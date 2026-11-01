---
name: bank-reconciliation
description: Bring bank movements into Nordlet and match them to invoices - camt.053 and MT940 statement files, JSON lines, PSD2 bank connections, import templates that create partners, link invoices by reference and post bank charges, match suggestions, matching and unmatching. Use when the user asks to import a bank statement, reconcile payments, connect a bank, mark an invoice as paid from the bank, or fix a wrong match. For Revolut CSV files use the revolut-import skill, for Stripe the stripe-import skill.
---

# Bank reconciliation in Nordlet

Direct tools: `bank_transactions_list`, `bank_transactions_import`, `bank_transactions_match`. The rest goes through `call_operation` with the path in camelCase as `operationId` (`bank/statements/import` → `bankStatementsImport`). Conventions are in the `nordlet` skill.

How it works: every import lands as bank transactions in status `new` on one bank account. A transaction is posted to the ledger only when it is matched to a document, or when an import template posts it to an account. Re-importing a file is safe: lines already on the account (same `externalId`) are skipped and counted in `skipped`.

## Bank accounts

`bank/accounts/list`; `bank/accounts/create` with `name` (required), `type` (`bank` default, `stripe`, `broker`), `iban`, `currency` (default the company base currency), `accountCode` (default the chart's bank account). One account holds one currency; a statement or line in another currency is refused with 422. With an IBAN set, a file for another IBAN is rejected.

## Import a statement file

`bank/statements/import` with `bankAccountId`, `content` and either `format` (`camt053` default, `mt940`) or `templateId`.

- camt.053 is the ISO 20022 XML that EU online banks export ("ISO XML", "camt.053"). Only booked entries (`Ntry/Sts` = `BOOK`) are imported; a file may hold several statements, those of another IBAN or currency are left out. Entry references become `externalId`.
- MT940 files are parsed when the content starts with `{1:` or `:20:`, or with `format: "mt940"`.
- Limits: 10 MB and 50 000 lines; every 10 imported entries count as one API request.
- The answer reports `imported`, `skipped`, per-statement metadata (period, opening and closing balances) and, with a template, `posted`, `customersCreated`, `invoicesLinked`, `invoicesCreated`, `paymentsMatched` and `warnings`.

Anything else is pushed as JSON with `bank_transactions_import`: `bankAccountId` and `transactions` (up to 1000 per call), each `{ date, amount (signed, "-25.00" for money out), currency?, counterpartyName?, counterpartyIban?, description?, externalId }`. Give a stable `externalId` so a repeat is skipped.

## Import templates (ISO 20022)

A template makes the file import create partners, link invoices by reference and post charges. Create one with `bank/import-templates/create`:

- `name`, `type: "iso20022"`, `fields` (omit for the standard list; a file missing a listed field is rejected with 422 and the missing names).
- Posting rows, each `{ name, accountCode }`: `Ntry/BkTxCd=CHRG` (bank charges, usually 6800), `Ntry/BkTxCd=INTR`, `Ntry/BkTxCd=TAXE`, `Ntry/CdtDbtInd=DBIT` (other money out), `Ntry/CdtDbtInd=CRDT` (other money in), `Ntry/Amt` (everything else). An entry is posted to the first row it matches: code family, then direction, then the plain amount row. Leave the direction and amount rows without an account so payments stay `new` for invoice matching. MT940 has no transaction code, so only the direction rows apply to it.
- `createPartner: true` on `TxDtls/RltdPties/Dbtr/Nm` creates a customer for every unknown payer of incoming money, on `TxDtls/RltdPties/Cdtr/Nm` a supplier for every unknown payee, on the `=CHRG` row a supplier named after the bank. Partners are recognised by IBAN, then by exact name.
- `invoiceMetaField` set to `TxDtls/RmtInf/Ustrd` (or `TxDtls/RmtInf/Strd/CdtrRefInf/Ref`, `TxDtls/Refs/EndToEndId`) links every new line whose reference, remittance or description contains the number of an open issued sale invoice (money in) or registered purchase invoice (money out) and posts the payment. A payment from an unknown payer is linked only when it equals the remaining balance and the number carries its series prefix; a payment above the remaining balance is never linked. With `advanceInvoices: true` an unmatched incoming payment from a known partner gets an issued advance invoice at `invoiceVatRatePercent` or the rate of `invoiceItemId`.
- Then import with `templateId` instead of `format`. Template names are unique per company (409).

## Match what is left

1. `bank_transactions_list` with filters `bankAccountId` eq, `status` eq `new`, `date` gte/lte. Sort by `date`.
2. For each line, `bank/transactions/suggest-matches` (`transactionId`, `limit` default 5): issued sale invoices and advances for incoming money; registered purchase invoices, issued sale credit notes and approved payroll runs for outgoing money, scored by equal remaining balance, document number in the description and counterparty IBAN or name. 409 when the line is already matched.
3. `bank_transactions_match` with `transactionId`, `documentType` (`sale_invoice`, `purchase_invoice`, `payroll_run`) and `documentId`; `invoiceAmount` only when the invoice is in another currency than the line. It posts the payment between the bank account and the receivable, payable or net payable, with any exchange difference, raises the paid amount and sets the line to `matched`. 409 when the amount exceeds the document's remaining balance or the period of the line date is locked.
4. A wrong match: `bank/transactions/unmatch` (`transactionId`, optional `date`, default today, never before the original posting) writes a reversing journal transaction, restores the invoice's paid amount and returns the line to `new`. The reversal date must be in an open period.

A line with no document (a loan, a tax payment, an owner's transfer) is booked with a manual journal transaction (`ledger/journal/transactions/create`), or by giving the template's rule row an account before the next import. A payment whose amount does not match any invoice stays `new`; tell the user instead of forcing a match.

## Bank connections (PSD2 feed)

1. `bank/feeds/banks/list` with `country` lists the banks (503 when the Nordlet instance has no feed provider).
2. `bank/feeds/connections/start` with `aspspName`, `aspspCountry`, `psuType` (`business` default), optional `redirectUrl`, `validForDays` (default 90). The answer's `url` is where the user authorises access at the bank; give it to them to open in a browser. `reference` identifies the connection.
3. After the bank redirects back, `bank/feeds/connections/complete` with `reference` and the `code` from the redirect. The connection becomes active and its accounts are recorded.
4. `bank/feeds/accounts/link` for each feed account: `id` plus `bankAccountId` or `createBankAccount { name, accountCode? }`, optional `syncFrom` (default: the last 90 days).
5. `bank/feeds/accounts/configure` with `id`, `importTemplateId` (a template of type `bank_connection`, same options as above with the feed field names `transaction_amount.amount`, `debtor.name`, `creditor.name`, `remittance_information`, `reference_number`, `bank_transaction_code=CHRG`, `credit_debit_indicator=DBIT`) and `syncSchedule` (`manual`, `daily`, `weekly`, `monthly`; a scheduled account is synced by Nordlet by itself).
6. `bank/feeds/sync` with `connectionId`, optional `feedAccountId`, `dateFrom`, `dateTo`. Pending transactions are skipped until booked. 409 when the connection is not active or the bank returns more than 20 pages (narrow the range).

Consents expire (commonly 90 or 180 days); a failed sync marks the connection with the bank's error, and the user connects the same bank again. The linked accounts and templates are kept.

## Related

- Revolut Business CSV: the `revolut-import` skill. Stripe exports and payouts: the `stripe-import` skill.
- `bank/match-rules/create` is only for linking incoming payout lines to Stripe settlement batches; it is not a general matching rule.
- Outgoing payments to suppliers: `bank/payments/export` (pain.001 SEPA file).
