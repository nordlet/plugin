---
name: month-end-close
description: Close a month in Nordlet - check that every document is in, release prepaid expenses, post depreciation and deferred revenue, run the trial balance, VAT, ageing, partner balances, financial statements and cash flow, book adjustments, and lock the period. Use when the user asks to close the month or year, prepare monthly reports, check the books, post depreciation, or lock a period.
---

# Month-end close in Nordlet

Direct tools: `reports_trial_balance`, `reports_financial_statements`, `reports_vat_summary`, `reports_debt_aging`, `ledger_journal_transactions_list`, `sales_invoices_list`, `purchases_invoices_list`, `bank_transactions_list`. The rest goes through `call_operation` with the path in camelCase as `operationId` (`ledger/periods/lock` → `ledgerPeriodsLock`). Conventions are in the `nordlet` skill.

Dates are `YYYY-MM-DD`; a month runs from its first to its last day. Report what each step found before the next one, and get the user's agreement before anything that posts (steps 3 to 5, 7 and 8).

## 1. Is the period open?

`ledger/periods/list` with filters `year` eq and `month` eq. A locked month refuses every posting dated in it with 409; `ledger/periods/unlock` opens it again.

## 2. Is everything in?

- Sales drafts not issued: `sales_invoices_list`, filter `status` eq `draft`.
- Purchase drafts not registered: `purchases_invoices_list`, filter `status` eq `draft`.
- Scanned documents not confirmed: `capture/documents/list`, filter `status` eq `extracted`.
- Bank lines not matched: `bank_transactions_list`, filters `status` eq `new`, `date` lte the month end.
- Bank feeds not synced: `bank/feeds/connections/list` shows `lastSyncedAt` and errors.

List what is open with the amounts. Finish those with the `sales-invoicing`, `supplier-bills` and `bank-reconciliation` skills before the reports.

## 3. Prepaid expenses

`purchases/deferrals/list` with filters `status` eq `pending`, `scheduleDate` lte the month end shows the releases due. `purchases/deferrals/post` with `asOfDate` = the month end posts one journal entry per schedule date, debiting the expense and crediting the prepaid account. The scheduler also runs this for every company, so it may already be done.

## 4. Depreciation

`assets/depreciation/preview` with `year` and `month` shows, per active asset, the amount and the months it covers (earlier unposted months included) and whether the month is already posted. `assets/depreciation/post` with the same body posts one entry dated the last day of the month. `journalTransactionId` null means nothing was due.

## 5. Deferred revenue

`sales/recognition/compute` with `asOfDate` previews the ratable tranches due. `sales/recognition/run` (`asOfDate`, optional `postingDate`, or exact `scheduleIds`) posts them; 422 when nothing is due. Locking the month (step 8) posts the pending ratable tranches by itself, so this step is optional. Milestone and percent-complete tranches are listed with `sales/recognition-schedules/list`.

## 6. Reports

| Report | Call | Body |
|---|---|---|
| Trial balance | `reports_trial_balance` | `fromDate`, `toDate`; check that the debit and credit totals agree |
| Profit and loss, balance sheet | `reports_financial_statements` | `fromDate` (first day of the fiscal year), `toDate`; `category` `micro`, `small`, `medium`, `large` (default from `reports/size-category`) |
| VAT by rate | `reports_vat_summary` | `fromDate`, `toDate`, `side` `sales` or `purchases`; `reports/vat-detail` gives one row per document |
| Receivables and payables ageing | `reports_debt_aging` | `side` `receivables` or `payables`, `asOf` |
| Open balance per partner | `reports/partner-balances` | none |
| Cash flow | `reports/cash-flow` | `fromDate`, `toDate`; `balanced` says whether opening cash plus the net change equals closing cash |
| One account's card | `reports/gl-detail` | `accountCode`, `fromDate`, `toDate` |
| Every posting | `reports/general-journal` | `fromDate`, `toDate`, `page`, `pageSize` |
| Twelve-month trend | `reports/monthly-summary` | `months` (default 12) |

Sale VAT is counted by issue date and purchase VAT by registration date. For an XLSX or PDF, `reports/jobs/create` with `reportType` (for example `trial-balance`, `vat-summary`, `debt-aging`, `cash-flow`), `params` (the body of that report) and `formats`; follow it with `reports/jobs/get` and download the output with `files/get`.

Compare the receivables and payables totals of the ageing with the balance sheet, and the VAT summary with the VAT accounts in the trial balance. Name every difference with its amount.

## 7. Adjustments

`ledger/journal/transactions/create` with `date`, `description` and `entries` (`accountCode`, `debit` or `credit` as decimal strings with at most two decimals, optional `costCenterId`, `projectId`). Base currency only; debits must equal credits; 409 in a locked period. Postable accounts: `ledger/accounts/list`. Posted entries are never edited; a mistake is corrected by a reversing entry.

## 8. Lock

`ledger/periods/lock` with `year` and `month`. Before locking it posts the pending ratable revenue tranches up to the month end and, when the month is the fiscal year end and no close is posted yet, the year-end close that moves the income and expense balances to retained earnings (posting rule `closing.retainedEarnings`). `ledger/periods/unlock` reverses that close when the month is unlocked again.

After locking, run the trial balance once more to confirm nothing changed.
