---
name: debt-collection
description: Chase overdue receivables in Nordlet - the debt ageing and partner balances, the automatic payment reminders (schedule, preview of what goes out, history), the credit check of a customer before a new sale, late-interest and credit-limit settings on the partner. Use when the user asks who owes money, how overdue an invoice is, which reminders will be sent, whether a customer is over its credit limit, or wants to switch reminders on or off.
---

# Debt collection in Nordlet

Direct tools: `reports_debt_aging`, `sales_invoices_list`, `sales_invoices_get`, `sales_invoices_send`, `partners_list`, `partners_get`. The rest goes through `call_operation` with the path in camelCase as `operationId` (`partners/debt-reminders/preview` → `partnersDebtRemindersPreview`). Conventions are in the `nordlet` skill.

## 1. Who owes what

- `reports_debt_aging` with `side: "receivables"` and `asOf` (default today): per partner, the open amount not yet due and 1 to 30, 31 to 60, 61 to 90 and over 90 days past due. Documents dated on or before `asOf` and payments up to that date count.
- `reports/partner-balances`: the open balance per partner, receivables and payables, in the base currency at each document's rate.
- The invoices behind a balance: `sales_invoices_list` with filters `partnerId` eq, `paymentStatus` ne `paid`, `dueDate` lte today, sorted by `dueDate`; `totals: ["grossTotal"]` sums them and `totalsByCurrency` splits the sum by currency.
- Payments are recorded by matching bank lines (the `bank-reconciliation` skill), so a balance is current as of the last matched statement.

## 2. Automatic reminders

Reminders are e-mailed by a daily job; there is no operation that sends one on demand.

Who gets them: a partner with `autoDebtReminder` true that is active and has an e-mail address, for issued invoices and advance invoices with `debtRemindersEnabled` (set when the invoice is created, default true) that have an unpaid remainder.

- Schedule: `partners/debt-reminders/settings/get` returns `daysBefore` (the "payment due soon" reminder that many days before the due date; null means none) and `repeatDays` (days between overdue reminders to one partner). `partners/debt-reminders/settings/update` changes either (1 to 90; `daysBefore` null switches the early reminder off).
- Switch a partner on or off: `partners/update` with `autoDebtReminder`. `lateInterestPercent` on the partner is the yearly rate the reminder's interest is computed with (remainder × rate × days late / 365).
- What would go out today: `partners/debt-reminders/preview` runs the overdue selection without sending or recording anything and returns the reminders with the invoices, the interest per invoice and the totals per currency. A partner that got an overdue reminder within the last `repeatDays` days is not in it.
- What comes next: `partners/debt-reminders/upcoming` lists the next reminder per unpaid invoice with its date and kind (`before_due` once per invoice, `overdue` from the day after the due date, repeated every `repeatDays` days).
- What was sent: `partners/debt-reminders/list`, one row per partner and currency with the invoice ids and the kind; filter `partnerId`, sort `sentAt`.

To put an invoice in front of the customer right now, `sales_invoices_send` e-mails the invoice itself again (`id`, optional `to`).

## 3. Credit check before a new sale

`partners/credit-check` with `partnerId` and optional `additionalAmount` (the sale about to be made) compares the open receivables (unpaid remainder of issued invoices, advances and credit notes, converted to the base currency at each invoice's rate) plus that amount with the partner's `creditLimit`. The answer gives the open amount, `available` and `exceeded`; without a credit limit `available` is null and `exceeded` is false. The limit is set with `partners/update` (`creditLimit`).

## 4. Record the follow-up

`partners/update` carries `firstCallDate`, `lastCallDate`, `nextCallDate` and `rating` for the collection history of a partner, and `isActive: false` stops its reminders. `partners_get` shows them with the partner's payment term (`paymentTermDays`).

## Related

- The money arrives: match the bank line to the invoice in the `bank-reconciliation` skill; the invoice's `paymentStatus` and the ageing follow.
- A disputed amount is settled with a credit note (`sales-invoicing` skill), not by editing the invoice.
