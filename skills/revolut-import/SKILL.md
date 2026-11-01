---
name: revolut-import
description: Import a Revolut Business CSV transaction statement into Nordlet with the built-in Revolut template, which posts fees, card payments and top-ups, creates missing partners and links payments to invoices by reference. Use when the user mentions Revolut, a Revolut CSV, or card payments from a Revolut account.
---

# Revolut Business CSV import

Calls go through `call_operation` with the path in camelCase as `operationId` (`bank/statements/import` → `bankStatementsImport`). Conventions are in the `nordlet` skill; matching and bank accounts are in the `bank-reconciliation` skill.

## Steps

1. The user downloads the transaction statement of the Revolut Business account as CSV. The file must contain the columns `Date completed (UTC)`, `ID`, `Type`, `State`, `Description`, `Payment currency`, `Total amount`, `Reference`, `Sender name` and `Beneficiary name`.
2. Find the bank account with `bank/accounts/list`, or create one with `bank/accounts/create` (`name`, `iban`, `currency`). The `International account number` column of the file must match the account's IBAN, and only rows in the account's currency are imported; a Revolut account with several currencies needs one Nordlet bank account per currency.
3. Import:

```json
{ "operationId": "bankStatementsImport",
  "body": { "bankAccountId": "<id>", "templateId": "builtin-revolut", "content": "<the CSV text>" } }
```

4. Report the answer: `imported`, `skipped` (rows already on the account, recognised by `ID`), `posted`, `customersCreated`, `invoicesLinked`, `paymentsMatched` and every entry of `warnings`.
5. Rows that stayed `new` (transfers, payments without an invoice number) are matched as in the `bank-reconciliation` skill: `bank_transactions_list` with `status` eq `new`, `bank/transactions/suggest-matches`, `bank_transactions_match`.

## What the built-in template does

- Imports only rows with `State` = `COMPLETED`; `PENDING` rows are counted in `warnings` and come in with a later file in which they are completed.
- Each row becomes a bank transaction: date from `Date completed (UTC)`, amount from `Total amount`, currency from `Payment currency`, `ID` as `externalId`, `Description` as the description. Counterparty of money out: `Beneficiary name` and `Beneficiary IBAN`, or the merchant in `Description` for a card payment; of money in: `Sender name` and `Sender account`.
- Checks that the previous row's balance plus `Total amount` equals `Balance`, and lists every row where it does not in `warnings`.
- Creates a customer for every unknown sender of money in and a supplier for every unknown beneficiary or card merchant of money out (recognised by IBAN, then exact name; a company when the name carries a legal form, else a person).
- Matches every row whose `Reference` or `Description` contains the number of an open issued sale invoice (money in) or registered purchase invoice (money out) and posts the payment. A payment from an unknown payer is linked only when it equals the remaining balance and the number carries its series prefix.
- Posts the rows no invoice matched by `Type`:

| `Type` | Posting rule | Default account | Posting |
|---|---|---|---|
| `FEE` | `bank.fees` | 6800 | debit the rule's account, credit the bank account's ledger account; partner: a supplier named after the bank account |
| `CARD_PAYMENT` | `bank.cardPayments` | 6209 | debit the rule's account, credit the bank account's ledger account, without VAT |
| `TOPUP` | `bank.topUps` | 3010 | debit the bank account's ledger account, credit the rule's account |

  `TRANSFER` and other types stay `new` for matching.

## Changing the accounts or the behaviour

- Change the accounts with `ledger/posting-rules/update` (`rules: [{ key: "bank.fees" | "bank.cardPayments" | "bank.topUps", accountCode }]`); `ledger/posting-rules/list` shows the account in effect. The built-in template uses the new accounts from the next import.
- A posting rule cannot be turned off. To import card payments or top-ups without posting them, create a company template with `bank/import-templates/create` (`type: "revolut"`, same columns, rule rows `Type=FEE`, `Type=CARD_PAYMENT`, `Type=TRANSFER`, `Type=TOPUP` with or without `accountCode`) and import with its id. A card payment posted by type is not matched to the supplier's invoice when that invoice is registered later: post one of the two, not both.
- `bank/statements/import` with `format: "revolut-csv"` and no `templateId` imports the rows as plain transactions, with none of the steps above.
- `bank/import-templates/list-builtin` lists the built-in template, `bank/import-templates/get` with `id: "builtin-revolut"` returns its settings; `update` and `delete` refuse it with 400.
