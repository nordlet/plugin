---
name: sales-invoicing
description: Issue sales documents in Nordlet - invoices, credit notes, proformas, quotes and advance invoices - from finding the customer and the VAT treatment to issuing, e-mailing, the payment link and the PDF. Use when the user asks to invoice a customer, bill something, credit an invoice, send a quote, apply an advance, or list unpaid invoices.
---

# Sales invoicing in Nordlet

Operations are called through the `nordlet` MCP connector: the direct tools where they exist, otherwise `call_operation` with the `operationId`, which is the path in camelCase (`sales/invoices/payment-link` → `salesInvoicesPaymentLink`). Read the request schema with `describe_operation` before a call whose fields are not listed here. The general conventions (decimal strings, list queries, errors, idempotency) are in the `nordlet` skill.

## 1. Find or create the customer

- Search first: `partners_list` with a filter on `name` (`contains`), `code` (`eq`), `vatCode` (`eq`) or `email` (`eq`).
- `partners_find_or_create` returns the live partner matched by `code`, then `vatCode`, then `email` (at least one of the three is required), or creates one from the body; `created` in the answer says which happened. Fields: `type` (`company` default, `person`), `name` (required), `code`, `vatCode`, `email`, `paymentTermDays`, `invoiceCurrency`, `invoiceLocale`, `address { street, city, postalCode, countryCode }`, `isCustomer` (default true), `isSupplier`.
- A `vatCode` is checked against VIES at creation; an invalid number opens a VAT review. `partners/validate-vat` checks a number on demand.

## 2. Decide the VAT treatment

For a customer in another country, or when unsure, call `reference/vat/resolve` with `partnerId` and `supplyType` (`goods`, `services`, `digital`). It returns `scheme`, `vatCountryCode`, the rates of the country whose VAT applies and the legal basis. It answers 422 when the partner has neither an address country nor a VAT code.

| Scheme | Lines carry VAT? | Note |
|---|---|---|
| `domestic` | yes, home rates | default when `vatScheme` is not sent |
| `intra_eu_b2b` | no (0 %) | goods to a VIES-valid business in another EU country |
| `reverse_charge` | no | services to a business in another EU country |
| `oss_union`, `ioss` | yes, rates of the consumption country | `vatCountryCode` required, must be EU |
| `export`, `out_of_scope`, `marketplace_deemed`, `sme_exempt` | no | |

A line with VAT under a scheme that charges none is refused with 422, and so are lines with VAT when the company is not a VAT payer.

Rates on a line: `vatRatePercent` directly, or `vatClassifierCode` from `reference/vat-classifiers/list` (the line takes the classifier's rate when neither the line nor the item has one; Lithuanian codes are `PVM1`, `PVM2`, ...).

## 3. Create the draft

`sales_invoices_create`:

- `partnerId` and `lines` are required. `type`: `invoice` (default), `credit_note`, `proforma`, `advance`.
- A line: `itemId` (from `catalog_items_list`) and/or `description`, `quantity` (default 1), `unit`, `unitPriceExclVat` or `unitPriceInclVat`, `vatRatePercent` or `vatClassifierCode`, optional `costCenterId`, `projectId`, `recognition` (ratable, milestone or percent-complete revenue recognition).
- Optional: `currency`, `issueDate`, `dueDate`, `discountPercent`, `notes`, `orderNumber`, `locale` (language of the PDF and e-mail), `documentSeriesId`, `payLinkEnabled` (default true), `debtRemindersEnabled` (default true), `vatScheme`, `vatCountryCode`.
- Totals are computed from the lines; no number is assigned yet. Read it back with `sales_invoices_get`; change it with `sales/invoices/update` (`lines` replaces every line; 409 once issued or locked).

Credit note: `type: "credit_note"` with `creditedInvoiceId` (an issued document of the same partner and currency), or `creditedInvoiceReference` plus `creditedInvoiceDate` for an invoice issued outside Nordlet. Credit notes may not exceed the credited total (422).

Advance invoice: `type: "advance"`. Once it is fully paid, `sales/invoices/apply-advance` (`advanceId`, `invoiceId`, optional `date`, `amount`) settles a final invoice of the same partner with it.

## 4. Issue

`sales_invoices_issue` with `id`, optional `issueDate` (else the draft's, else today), `series`, `warehouseId` (takes sold goods out of stock), `returnToStock` (credit note of an invoice only).

Issuing assigns the next gapless number, fills a missing due date from the partner's payment term, and for an invoice, credit note or advance posts the journal entry in the same transaction. A proforma is numbered but not posted. Answers 409 when already issued, locked, or the accounting period is locked. An issued document cannot be deleted; correct it with a credit note.

For a create or issue that may be retried, use `call_operation` with an `idempotencyKey` instead of the direct tool.

## 5. Deliver

- E-mail: `sales_invoices_send` with `id`, optional `to` (else the partner's e-mail), `locale`. 422 when e-mail sending is not configured, the document is not issued, or no recipient exists.
- Quote: `sales_invoices_send` with `quote: true` on a draft e-mails it as a quote and leaves it a draft.
- Payment link: `sales/invoices/payment-link` returns the Stripe card page when the company's Stripe account is connected, else the link from the company's payment link template; `url` is null when the company has neither. 409 when already paid.
- PDF: `sales/invoices/pdf` returns the document base64-encoded (`locale` optional; the document's own locale wins).
- Peppol: `sales/invoices/peppol-send` needs the company's own access point (422 without); `sales/invoices/peppol-xml` builds the file to download. `sales/invoices/einvoice-send` is the national channel for Italy, Poland and Romania.

## 6. Follow up

- Open invoices: `sales_invoices_list` with filters on `paymentStatus`, `partnerId`, `status`, `issueDate`, `dueDate`, `fullNumber`; add `totals: ["grossTotal"]` for the sum over every matching row.
- Money received in a period: `sales/payments/list` (`from`, `to`), one row per bank payment, card settlement or cash receipt.
- Payments are recorded by matching bank lines (see the `bank-reconciliation` skill), not by editing the invoice.
- `sales/invoices/lock` protects a document from update, delete, issue and apply-advance until `unlock`.
