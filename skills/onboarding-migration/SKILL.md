---
name: onboarding-migration
description: Set up a company in Nordlet and bring its books over from another system - the chart of accounts and posting rules, numbering series, operation types, bank accounts, warehouses, cost centers and owners, then the historical-books package (accounts, partners, items, opening balances or full journal history, open invoices, fixed assets, stock) validated and imported in one transaction. Use when the user starts a new company in Nordlet, migrates from another accounting program, asks about opening balances or the cut-over date, or wants invoice numbering to continue.
---

# Onboarding and migration in Nordlet

Calls go through `call_operation` with the path in camelCase as `operationId` (`migration/books/validate` → `migrationBooksValidate`). Conventions are in the `nordlet` skill. Guide: https://nordlet.com/en/docs/guides/migration/.

What the API key cannot do: the company itself, its profile (name, registry code, VAT number, VAT payer flag, VAT period, fiscal year end, address, bank details, invoice prefix, default invoice language) and the API keys are managed by a signed-in user in the Nordlet app under Account. The `account/...` operations that manage the company and its keys answer 401 "A user session is required for this action" to an API key. Ask the user to do those steps in the app.

## 1. Chart of accounts

- `ledger/accounts/list` shows the chart the company was seeded with (filters `code`, `name`, `type`, `isPostable`).
- `ledger/accounts/apply-template` adds the accounts of the company's chart template that are missing and sets the template's parent links; existing codes are kept.
- `ledger/accounts/switch-chart` replaces the seeded chart with the chart template of the company's country and switches the posting defaults with it. Only for a company that has posted nothing: 409 when it already uses that chart, has journal entries, has hand-made accounts, or has settings naming an account the new chart lacks.
- `ledger/accounts/create` adds an account (`code`, `name`, `type` `asset`, `liability`, `equity`, `income`, `expense`, optional `parentId`, `isPostable` default true).
- `ledger/posting-rules/list` shows every default account the documents post to (receivables, payables, VAT, bank fees, inventory, retained earnings and the rest) with the template default and the account in effect; `ledger/posting-rules/update` with `rules [{ key, accountCode }]` changes them.

## 2. Numbering

- `document-series/create`: a series for `sale_invoice`, `sale_credit_note`, `sale_proforma` or `sale_advance` with `prefix`, `nextNumber`, `numberLength` (zero padding), `printSeries`, `isDefault`, optionally bound to an operation type or a warehouse (`sales/invoices/issue` takes the goods from it). 409 when the prefix exists for the document type. `document-series/list` shows them.
- Continuing the old system's numbers without a series: `reference/series/create` with `documentType: "sale_invoice"`, `prefix`, `year` and `startAt` (the next number to hand out); `reference/series/list` shows the next number of each.

## 3. Operation types and master data

- `operation-types/create` when some invoices must post to other accounts than the defaults, or stay out of the VAT registers or the SAF-T file (`includeInVatRegister`, `includeInSaft`). Guide: https://nordlet.com/en/docs/guides/operation-types/.
- Bank accounts: `bank/accounts/create` (`name`, `iban`, `currency`, `accountCode`), one per currency (the `bank-reconciliation` skill).
- Warehouses: `inventory/warehouses/create` (`code`, `name`, `isDefault`), needed before stock is imported.
- Cost centers: `ledger/cost-centers/create` (`code`, `name`, `groupId`). Owners: `ledger/owners/create` (`name`, shares, `equityAccountCode` default the share capital account).
- VAT classifiers of the country: `reference/vat-classifiers/list`. Exchange rates: `reference/exchange-rates/sync` for a date.

## 4. The historical-books package

Pick the cut-over date: the first day the company is run in Nordlet. Every amount in the package is as of the day before it. The API key needs `ledger:write` plus the write scope of every section included (`partners`, `catalog`, `sales`, `purchases`, `assets`, `inventory`); bodies up to 32 MB.

`migration/books/validate` runs every check and rolls back; `migration/books/import` writes everything in one database transaction, or nothing. Run validate until it reports no errors and the user accepts its warnings, then import the same body. Every section is optional; references between sections are by code (account, partner, item, asset group, warehouse), to a record that exists or one created earlier in the same package.

| Section | Rows | Rules |
|---|---|---|
| `accounts` | `code`, `name`, `type`, `parentCode`, `isPostable` | existing codes are left untouched and counted as `existing` |
| `partners` | `code`, `name`, `type`, `vatCode`, `email`, `isCustomer`, `isSupplier`, `paymentTermDays`, `address` | existing codes are skipped, not updated |
| `items` | `code`, `name`, `type` `product` or `service`, `unit`, `barcode`, `vatRatePercent`, prices | existing codes are skipped |
| `openingBalances` | `date` (default the cut-over date), `balancingAccountCode`, `entries [{ accountCode, debit, credit }]` | the old system's closing trial balance, posted as one journal transaction; must balance, or the difference goes to `balancingAccountCode` (reported as `balancingAmount`); only one per company (409 on a second) |
| `journal` | `date`, `description`, `reference`, `entries` | full history instead of or before the balances; every transaction balances on its own; dated before the cut-over and, with `openingBalances`, on or after their date; never balances and history for the same period |
| `openReceivables` | `partnerCode`, `number`, `issueDate`, `dueDate`, `grossTotal`, `vatTotal`, `outstanding`, `currency`, `fxRate` | created as issued sale invoices with `paidAmount` = gross less outstanding, so matching, ageing and reminders work; not posted again (their balance is in the opening balances); `number` unique in the company; `fxRate` required for a currency other than the base currency |
| `openPayables` | `partnerCode`, `documentNumber`, `documentDate`, the same amounts | created as registered purchase invoices; requires `openingBalances` in the same package; unique per partner and number |
| `assetGroups`, `fixedAssets` | group accounts; `groupCode`, `code`, `name`, `acquisitionDate`, `acquisitionCost`, `salvageValue`, `usefulLifeMonths`, `accumulatedDepreciation`, `depreciatedMonths` | depreciation continues from the remaining value over the remaining months; nothing posted |
| `stock` | `itemCode`, `quantity`, `unitCost`, `warehouseCode` (default the default warehouse), `lotNumber`, `expiryDate` | one incoming FIFO layer per row dated on the cut-over; nothing posted |

Invoice lines are not imported; an open invoice carries totals only. A customer balance without open items becomes one invoice per partner. When an imported sale invoice number has the form `PREFIX-N` and its issue date is in the cut-over year, the series for that prefix and year is set to `N + 1` (never lowered); the answer lists the resulting `numberSeries`.

Every month touched by the opening balances or the history must be an open period (409 otherwise).

Checks: 422 for missing or non-postable accounts, unbalanced transactions, unknown codes, duplicates inside the package, outstanding above gross, accumulated depreciation above the depreciable value, missing FX rates; 409 for numbers or codes that already exist. `warnings` compare the sub-ledgers with the ledger in the same package: open receivables against the receivables account, open payables against the payables account, stock cost against the stock account, each asset group's cost and depreciation against its accounts. A warning means the old system's sub-ledger and general ledger disagree; resolve it with the user before importing.

The answer gives per section the counts and totals, the opening-balance transaction id and `balancingAmount`; `dryRun: true` marks a validate run.

## 5. After the import

- Lock the periods before the cut-over with `ledger/periods/lock` so nothing is posted into migrated history by accident.
- Run `reports_trial_balance` and `reports_financial_statements` as of the cut-over date and compare them with the old system's closing reports; name every difference.
- Continue with the normal flow: `sales-invoicing`, `supplier-bills`, `bank-reconciliation`. The migrated open invoices settle like any other.

Not covered by the importer: invoice lines and VAT registers of past periods, payroll history, and documents other than invoices (orders, waybills, contracts).
