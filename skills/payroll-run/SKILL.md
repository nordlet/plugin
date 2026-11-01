---
name: payroll-run
description: Run payroll in Nordlet under the rules of the company's country - employees and contracts, timesheets, sick-leave certificates and business trips, a trial calculation, the monthly run with typed payments, review, approval and posting, the SEPA payment file, the payroll filings and corrections by reversal. Use when the user asks to pay salaries, calculate a wage, add an employee or contract, fill a timesheet, approve or reverse a payroll run, or export salary payments.
---

# Payroll in Nordlet

No direct tool covers payroll; every call goes through `call_operation` with the path in camelCase as `operationId` (`payroll/runs/create` → `payrollRunsCreate`). Conventions are in the `nordlet` skill. Amounts are decimal strings with at most two decimals.

The rules are those of the company's country: income tax, contributions, allowances and the payment types all come from the country's payroll engine. `payroll/payment-types/list` returns the country's payment type codes with the taxes, contributions and reporting lines each one is subject to, and answers 422 when Nordlet has no engine for the company's country; call it first in a company you have not run payroll in. `hr/employees/fields` lists the country-specific details a filing needs about a person; their values are kept in the employee's `payrollOptions`.

## 1. Master data

- Reference lists: `payroll/departments/create` (`code`, `name`), `payroll/schedules/create` (`code`, `name`, `hoursPerWeek` default 40), `hr/positions/create` (`name`, optional `code`).
- Employee: `hr/employees/create` with `firstName`, `lastName`, and the fields the engine and the payments need: `personalCode`, `birthDate`, `iban` (required by the payment export), `socialInsuranceNo`, `hireDate`, `applyAllowance` (default true), `allowanceOverride`, `pensionAccumulation` (default false), `payrollOptions` (a map of strings, keys from `hr/employees/fields`), `address`, `email`. `hr/employees/update` changes them and sets `status` `terminated` with `terminationDate`. `hr/employees/list` filters `lastName`, `code`, `status`, `email`.
- Contract: `hr/contracts/create` with `employeeId`, `startDate`, `baseSalary`, `salaryType` (`monthly` default, `hourly`, `weekly`, `daily`, `yearly`), `type` (`permanent`, or `fixed_term` with `endDate`), optional `positionId`, `departmentId`, `scheduleId`, `workHours`, `contractNo`. One contract per employee per period: 409 when one overlaps; end the old one with `hr/contracts/end` (`id`, `endDate`, `endReason`).
- Leave: `hr/leave-balances/set` (`employeeId`, `year`, `entitledDays`, `usedDays`); `hr/leave-balances/list`.

## 2. The month's time

- `hr/timesheets/generate` with `year`, `month` and optional `employeeId` builds the timesheet from the contract (Monday to Friday, the contract's daily hours), replacing an existing one. Public holidays are filled in as `holiday` days only for a company in Lithuania; for every other country, set them with `hr/timesheets/upsert`. 422 when no contract covers the month.
- `hr/timesheets/upsert` with `employeeId`, `year`, `month` and `days` (`day`, `hours`, `type` `work`, `business_trip`, `vacation`, `sick`, `holiday`, `unpaid`) sets the exact days. The run reads the paid days, the days worked and the business-trip days from it; the payment types that need days (per diems, meal allowances) are refused without one.
- Sick leave: `hr/incapacity-certificates/create` (`employeeId`, `number`, `fromDate`, `toDate`); the run warns when a certificate covers working days of a month without a timesheet.
- Business trips: `hr/per-diem-rates/create` (`dailyAmount`, `validFrom`, per destination country), `hr/business-trips/create` (`employeeId`, `destinationCountryCode`, `purpose`, `startDate`, `endDate`; the per diem is the calendar days times the rate), `hr/business-trips/approve`. The automatic run adds the per diem of every approved trip ending in the month as an untaxed addition.

## 3. Trial calculation

`payroll/calc` with `taxableBase`, `date`, the employee settings (`applyAllowance`, `allowanceOverride`, `pensionAccumulation`, `fixedTerm`, `benefitInKind`, `options`) and optional `payments` (`paymentType` from `payroll/payment-types/list`, `amount`, `hours`) returns the allowance, income tax, contributions, components and net for one payment. Nothing is saved. Use it to answer "what would the net be" before touching a run.

## 4. Create the run

`payroll/runs/create` with `year` and `month`, optional `payDate` (default the last day of the month) and `notes`.

- Without `lines`: every active employee whose contract covers the month gets a line. The gross is the contract's monthly salary reduced to the paid working days of the contract and the timesheet, unless `grossOverrides` (`employeeId`, `gross`) sets it. Approved business trips ending in the month are added and linked to the run. Where the country's engine computes it, `includeNatura` (default true) adds the car benefit in kind, and a contract ending in the month gets the compensation for unused leave.
- With `lines`: only the named employees, each with `gross` (optional for a monthly contract), `additions` (`name`, `amount`, `taxable` default true, `paymentType` for a typed payment, `hours` where the type needs them) and `deductions` (`name`, `amount`). A payment that is not plain salary (bonus, overtime, allowance, severance, benefit) needs its `paymentType` so the engine applies the right taxes and reporting line; an untyped taxable addition falls under the default type that `payroll/payment-types/list` names.
- 422: no engine, an employee given twice, two contracts in the month, no contract and no gross, paid by the hour or day without a gross. 409: a run for the month exists, or the period is locked.
- Each line is computed with the engine, the employee's approved runs of the year and of the two previous years, and the month's timesheet.

Review with `payroll/runs/get`: per line the gross, the allowance, the income tax, the employee and employer contributions, the components, the net and the warnings. `payroll/lines/attendance` (`id`, `daysWorked`, `hoursWorked`, `registeredDays`, `averageHourlyEarnings`) records the time figures some countries report per employment, while the run is a draft. `payroll/runs/cancel` deletes a draft and releases its business trips.

Show the user every line with gross, tax, contributions and net, and get their agreement before approving: approval posts to the ledger.

## 5. Approve and post

`payroll/runs/approve` with `id` posts the journal entry dated the last day of the month (wage expense and employer contributions debited; net payable, income tax, social contributions and deductions credited) and sets the run to `approved`. The accounts default to the payroll accounts of the company's chart template; each can be given per call (`wageAccountCode`, `employerAccountCode`, `payableAccountCode`, `gpmAccountCode` for the income tax, `sodraAccountCode` and `employerSocialAccountCode` for the contributions, `deductionAccountCode`). 409 when not a draft or the period is locked, 422 when there is nothing to post. Emits `payroll_run.approved`. For a country whose payroll deadline has filing automation switched on (`declarations/automation/list`), approval also sends the month's submission and a failure lands in the run's `warnings` (the API documents this for the Irish `ie-payroll-withholding` deadline).

## 6. Pay

`payroll/payments/export` with `runId`, `bankAccountId`, optional `executionDate` (default today) and `locale` returns a SEPA credit transfer file (pain.001.001.03) in `xml`: one transfer to each employee's IBAN for the line's positive net, with the run period in the remittance text. Nothing is sent to the bank; the user uploads the file in their online bank. 422 when the bank account has no IBAN or is not in the base currency, or an employee has no IBAN.

When the salaries leave the bank account, match the statement line to the run: `bank_transactions_match` with `documentType: "payroll_run"` and the run id (the `bank-reconciliation` skill). The withholding and contribution payments to the administrations are recorded with `declarations/tax-payments/create` (`tax` `payroll_withholding` or `social_insurance`, `year`, `month`, `kind`, `amount`, `paidOn`, `description`).

## 7. File

The month's payroll filings (withholding returns, social insurance reports, employment register notices) are deadlines of the company's country in `calendar/list`; `calendar/submit` sends one and `calendar/download` builds the file, as described in the `vat-return` skill. The country-specific generators are also reachable as operations under `declarations/<country code>/...` (`search_operations`).

## 8. Correct

An approved run is never edited. `payroll/runs/reverse` with `id` and `reason` posts a mirror entry dated the last day of the month, sets the run to `reversed` and releases its business trips; 409 when the run is not approved, when part of it is matched to bank payments (unmatch first) or the period is locked. Then create the corrected run for the same month.

## Related

- `hr/employees/records/create` keeps education, qualification, certificate and training records; `hr/employees/anonymize` erases a person's data.
- `payroll/runs/list` filters `status`, `year`, `month`.
