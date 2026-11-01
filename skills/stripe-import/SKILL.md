---
name: stripe-import
description: Book Stripe activity in Nordlet - the Payments export through a Stripe import template (fees, customers, invoices, credit notes), the plain Stripe balance CSV, and payout reconciliation through settlement batches that clear the Stripe balance to the real bank account. Use when the user mentions Stripe, Stripe fees, Stripe payouts, card payments from an online shop, or a marketplace commission split.
---

# Stripe in Nordlet

Calls go through `call_operation` with the path in camelCase as `operationId` (`bank/settlements/post` → `bankSettlementsPost`). Conventions are in the `nordlet` skill; matching and bank accounts are in the `bank-reconciliation` skill.

Two files, two steps:

| What | File from Stripe | Lands on | Operation |
|---|---|---|---|
| Charges, fees, refunds | Payments → Export (all columns, with the metadata columns) | the Stripe bank account | `bank/statements/import` with a Stripe template |
| Payouts to the real bank | Reports → Payouts → Payout reconciliation, itemized | the real bank account | `bank/settlements/import` then `bank/settlements/post` |

## 1. The Stripe bank account

`bank/accounts/create` with `type: "stripe"`, a `name` such as "Stripe", the `currency` charged in and the `accountCode` (a Stripe account has no IBAN; one with an `iban` is refused with 422). The Stripe balance behaves like a bank account: charges come in, fees go out, payouts move the balance to the real bank.

## 2. The Stripe template

`bank/import-templates/create` with `type: "stripe"`. `fields` defaults to the columns of the Payments export without the 19 `Card …` columns. The usual setup:

| Field | Setting | Effect |
|---|---|---|
| `Fee` | `accountCode` of the PSP fee expense (6800 in the standard chart), `createPartner: true` | every fee is posted as an expense with a supplier named Stripe |
| `Amount` | no account | charges stay open for invoice matching |
| `Amount Refunded` | no account | refunds go through credit notes |
| `Customer Email` | `createPartner: true` | a customer per unknown e-mail, named from `Customer Description`, recognised by e-mail on later imports |

- `metaFields`: the metadata keys the charges carry (Stripe exports each as `<key> (metadata)`), for example `["orderNo"]`.
- `invoiceMetaField`: the key that identifies an order, with `invoiceVatRatePercent` and optionally `invoiceItemId` (the catalog item every created line references). With it, charges and refunds are grouped by the key's value: a value that matches an existing sale invoice (a sales order's `externalRef` or an invoice `fullNumber`) links the group as payments; a value with no invoice creates one for the customer (one line per charge, VAT-inclusive at the rate, dated on the first charge), issues and posts it and matches the charges, so it shows as paid; each refund becomes an issued credit note against it. Do not also map `Amount` to an account in such a template.
- `companyMetaField`: a key whose value `true`, `1`, `yes`, `company`, `business` or `b2b` makes the created customer a company instead of a person.
- The marketplace fields (`payoutOperationTypeId`, `commissionOperationTypeId`, `lenderMetaField`, `authorizationOperationTypeId`, the refund labels) are for a platform that collects money for others; an ordinary shop leaves them empty.

## 3. Import the Payments export

`bank/statements/import` with `bankAccountId` (the Stripe account), `templateId` and `content` (the CSV). A marketplace adds `transfersCsv` (the Stripe transfers export). Rows that moved no money are skipped (failed, cancelled, not captured). The answer reports `imported`, `skipped`, `posted`, `customersCreated`, `invoicesCreated`, `invoicesLinked`, `creditNotesCreated`, `paymentsMatched` and `warnings` (groups without a customer e-mail, mixed currencies, a linked invoice that is a draft or already paid; their rows stay `new`). A large file takes minutes because invoices are issued one by one.

Without a template, `format: "stripe-csv"` imports a Stripe balance CSV or Payments export as plain transactions: one for the gross amount, a second with the opposite sign for a non-zero fee (`externalId` `<id>:fee`, counterparty Stripe), a third for a non-zero `Amount Refunded` (`<id>:refund`); payout rows come in as single negative transactions. These are then matched by hand.

## 4. Payouts: settlement batches

1. `bank/settlements/import` with `bankAccountId` (the real bank account), `provider: "stripe"` and `content`. Accepted: the itemized payout reconciliation report (one row per balance transaction; Stripe downloads it one file per section, charge, refund and fee; import them all, rows of a payout not yet posted merge into its batch) or the unified Payments export with its `Transfer` column (test mode). Rows are grouped by payout into batches with gross, fee and net totals; every charge, refund and dispute row is matched to an issued sale invoice through the order's `externalRef`, an invoice number in the columns, or the charge of an earlier match.
2. Review with `bank/settlements/list` (filter `status`, `payoutId`, `bankAccountId`) and `bank/settlements/get`. Fix a line with `bank/settlements/match` (`lineId`, `invoiceId`; `invoiceId` null clears it); only while the batch is `imported` (409).
3. `bank/settlements/post` with `id`, optional `date` (default the payout date) and `commissionPercent`. One balanced journal transaction: the bank account receives the net payout, matched invoices are settled against receivables (capped at each invoice's remaining amount, excess to suspense with a warning), fees go to `settlements.fees` (6800). A line whose charge, fee or refund the Payments import already put on the Stripe bank account is not booked again: its net is credited to the Stripe account, so the Stripe balance clears payout by payout; `clearedNet`, `clearingDifference` and `clearingOpenCount` on the batch show whether it agrees. 409 when already posted or the period is locked.
4. The incoming payout line on the real bank statement is attached to the batch by `bank/settlements/link` (`id`, `bankTransactionId`), or automatically by a rule from `bank/match-rules/create` (`name`, `pattern` such as `STRIPE`, `provider: "stripe"`, optional `payoutIdPrefix`, `bankAccountId`, `dateWindowDays` default 5).

Unmatched charges on a marketplace: with `commissionPercent` on the post, or a per-line rate or amount from `bank/settlements/commission`, they are split into commission revenue (`settlements.commissionRevenue`, 5001) and a seller liability (`settlements.sellerPayable`, 4499); without it they are parked on `settlements.suspense` (4440). The four accounts are changed with `ledger/posting-rules/update`.

A payout in another currency converts at the rate of the posting date (the company's own rate from `reference/exchange-rates/set` when it is at least as recent, else the official one); receivables are relieved at each invoice's own rate and the difference goes to `bank.fxGain` or `bank.fxLoss`. Without a rate on or before the date the call fails with 422 and names `reference/exchange-rates/sync`. A payout mixing currencies is refused with 422.

## Order

Import the Payments export before posting the payout that contains its charges. A Payments export imported after its payout was posted skips those movements and names the payout in `warnings`.
