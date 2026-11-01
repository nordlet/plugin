---
name: recurring-billing
description: Bill customers on a schedule in Nordlet - either from an agreement (a contract with a value or items and a monthly, quarterly or annual billing period, prorated, with ratable revenue recognition) or from an invoice template on a weekly to annual schedule that can issue and e-mail the invoices by itself; the billing run, the generated invoices and the daily automation. Use when the user asks about subscriptions, retainers, rent, service contracts, recurring or repeat invoices, or why an expected invoice was not generated.
---

# Recurring billing in Nordlet

Direct tools: `sales_invoices_list`, `sales_invoices_get`, `sales_invoices_issue`, `sales_invoices_send`, `partners_list`. The rest goes through `call_operation` with the path in camelCase as `operationId` (`agreements/agreements/billing/run` → `agreementsAgreementsBillingRun`). Conventions are in the `nordlet` skill.

Nordlet has two mechanisms. They differ in what is stored and what happens to the invoice:

| | Agreement | Invoice template schedule |
|---|---|---|
| Stores | the contract: number, dates, value or items, billing period, renewal, type | an invoice template and a schedule on it |
| Periods | monthly, quarterly, annual from the start date, prorated by days | weekly, monthly, quarterly, annual, every `interval` periods |
| Invoice | a draft; revenue recognised ratably over the period | a draft, or issued (`autoIssue`) and e-mailed (`autoSend`) |
| Run | `agreements/agreements/billing/run` or `autoBilling` | `sales/recurring-invoices/run` or the scheduled job |
| Event | `agreement.invoice_generated` | `recurring_invoice.generated` |

## A. Agreements

1. Optional type: `agreements/types/create` (`code`, `name`).
2. `agreements/agreements/create` with `partnerId`, `number` (unique in the company, 409), `startDate`, optional `endDate`, `autoRenew`, `billingPeriod` (`monthly`, `quarterly`, `annual`), `currency` (default EUR), `kind` (`customer`, `supplier`, `employment`, `bank`, `lease`, `insurance`, `other`; a partner agreement defaults to `other`), `typeId`, `name`, `notes`, `documentRef` (the `storageKey` of a file from `files/upload`), and either `items` (`itemId` or `description`, `quantity`, `unitPrice`, `vatRatePercent`) or `value` (one line per period). `status` default `active`.
3. Billing:
   - One agreement: `agreements/agreements/generate-invoice` with `id` and optional `asOfDate` (default today) creates a draft sale invoice for the next unbilled period. The period follows the grid from `startDate`, continues after the last period invoiced, and is prorated by days when it starts late or is cut by `endDate`; an `autoRenew` agreement has its `endDate` moved forward by whole terms instead. The lines come from the items, else one line of `value`, with the revenue recognised ratably over the period. 422 when the agreement is not active, has no partner, billing period, items or value, carries VAT while the company is not a VAT payer, is fully billed through its end date, or when the next period starts after `asOfDate`.
   - All agreements: `agreements/agreements/billing/run` with optional `asOfDate` generates every invoice due for every active agreement with a partner and a billing period, one invoice per period and at most 60 per agreement in one run; a fully billed agreement past its `endDate` becomes `expired`; a failing agreement is listed in `errors` and does not stop the run.
   - Daily: `agreements/settings/update` with `autoBilling: true` makes a scheduled job run the same billing once a day; `agreements/settings/get` shows the setting.
4. The generated invoices are drafts: issue and send them with the `sales-invoicing` skill, or review them first. `sales_invoices_list` with a filter on `agreementId` lists an agreement's invoices.
5. `agreements/agreements/update` changes the end date, renewal, value, billing period, status (`draft`, `active`, `expired`, `terminated`), type and notes; the party, number, start date, currency and items are not changed by it. `agreements/agreements/delete` marks it deleted; it is no longer billed. `agreements/agreements/list` filters `partnerId`, `kind`, `status`, `startDate`, `endDate`.

## B. Invoice templates and schedules

1. `sales/invoice-templates/create` with `name` (unique, 409), the header defaults (`partnerId`, `currency`, `paymentTermDays`, `vatScheme`, `vatCountryCode`, `supplyType`, `documentSeriesId`, `agreementId`, `discountPercent`, `notes`) and `lines` (`itemId` or `description`, `quantity`, `unitPriceExclVat`, `vatRatePercent`, `unit`, `projectId`). `documentType` `invoice` (a `credit_note` template cannot be scheduled).
2. `sales/recurring-invoices/create` with `templateId`, `startDate` (the first run), optional `endDate`, `frequency` (`weekly`, `monthly`, `quarterly`, `annual`; default monthly), `interval` (default 1), `partnerId` (else the template's; 422 when neither has one), `locale`, `autoIssue`, `autoSend` with `sendTo` (else the partner's e-mail; `autoSend` needs `autoIssue`).
3. Generation:
   - `sales/recurring-invoices/generate` with `id` creates the next invoice now, dated the schedule's `nextRunDate`: a draft from the template (due date from the template's payment term, VAT scheme from the template or else resolved for the partner and supply type), issued when `autoIssue` is set, e-mailed when `autoSend` is set, then the schedule moves to its next occurrence or to `finished`. A failure to issue or send comes back in `error` and is stored as `lastError`; the draft stays.
   - `sales/recurring-invoices/run` with optional `asOfDate` generates every invoice due from the active schedules, catching up to 60 missed occurrences per schedule, and lists what was generated and which schedules failed.
   - The scheduled job `recurring-invoices` does the same by itself.
4. `sales/recurring-invoices/update` pauses and resumes (`status` `active` or `paused`), moves `nextRunDate`, changes the frequency, interval, dates, template, partner and the issue and send flags. `sales/recurring-invoices/delete` removes the schedule; its invoices stay. `sales/recurring-invoices/list` filters `status`, `partnerId`, `templateId`, `frequency`, `nextRunDate`, `autoIssue`.

## When an expected invoice is missing

- Agreement: `agreements/agreements/get` shows the status, the end date and the items; the 422 reasons above are the usual causes (not active, fully billed, next period in the future, no value).
- Schedule: `sales/recurring-invoices/get` shows `status`, `nextRunDate` and `lastError`.
- The run answers (`errors`, failed schedules) name the agreement or schedule and the reason.
