---
name: supplier-bills
description: Register supplier bills and purchase credit notes in Nordlet, typed in by the user or read from a PDF or photo through document capture, with expense accounts, prepaid-expense deferrals, purchase orders and goods receipts. Use when the user asks to book an expense, enter a supplier invoice, scan a receipt, record a purchase credit note, or list unpaid bills.
---

# Supplier bills in Nordlet

Direct tools: `purchases_invoices_list`, `purchases_invoices_create`, `purchases_invoices_register`, `partners_find_or_create`, `partners_list`, `catalog_items_list`. Everything else goes through `call_operation` with the path in camelCase as `operationId` (`capture/documents/confirm` → `captureDocumentsConfirm`). Conventions are in the `nordlet` skill.

A purchase document is a draft until it is registered. Registration posts the journal entry; a registered document cannot be deleted.

## Path A: from a document file

1. `capture/documents/upload` with `fileName`, `mimeType` and `content` (base64; PDF, image, Word or PowerPoint, up to 10 MB). The answer holds the extracted supplier, dates, totals and lines, the partner matched by VAT code, then partner code, then name, and `status` `extracted` or `failed` with `error`. It needs a configured reading provider, is refused with 402 for a company on trial before its first top-up, and every page read counts as usage.
2. Show the user the extraction against the document. Fix what is wrong. `capture/documents/extract` re-reads a stored capture and replaces the draft (409 once linked to an invoice).
3. `capture/documents/confirm` with `id`, `documentNumber`, `documentDate`, `lines` (required), and `partnerId` or `newSupplier { name, code, vatCode, countryCode }`; optional `type` (`invoice` or `credit_note`), `dueDate`, `currency`, `notes`. Lines with the opposite sign go in `oppositeLines`, saved as a second document of the opposite type for the same supplier, numbered `oppositeDocumentNumber` (default the document number plus `-CR` or `-INV`). It saves the reviewed draft as a purchase invoice and attaches the original file.
4. Read the result with `purchases/invoices/get` and register it as in step 3 of path B when it is still a draft.

Documents can also arrive by e-mail: `capture/settings/get` returns the company's intake address and whether e-mailed documents are read on arrival. Unprocessed captures: `capture/documents/list` with a filter on `status`.

## Path B: typed in

1. Supplier: `partners_list` (filter `name` contains, `code` or `vatCode` eq) or `partners_find_or_create` with `isSupplier: true`, `isCustomer: false`.
2. `purchases_invoices_create`: `partnerId`, `documentNumber` (the supplier's number) and `documentDate` are required, plus `lines`. Optional `type` (`invoice` default, `credit_note` with `creditedInvoiceId` of the same partner), `dueDate`, `currency` (default the company's invoice currency), `purchaseOrderId`, `operationTypeId`, `notes`, `documentRef` (a file uploaded with `files/upload`, which returns the `storageKey` to pass). The same `documentNumber` for the same partner is refused with 409.
   A line: `description` and/or `itemId`, `quantity` (default 1), `unitPriceExclVat` or `unitPriceInclVat` (a line without a price takes the item's supplier or purchase price), `vatRatePercent` or `vatClassifierCode`, `accountCode` (the expense account), optional `costCenterId`, `projectId`, and `deferralStartDate` with `deferralEndDate` for a cost that belongs to a period (insurance, subscriptions, rent paid ahead).
3. `purchases_invoices_register` with `id`, optional `registrationDate` (default the document date), `warehouseId` (receives product lines into stock; 422 on an order-linked invoice), `returnFromStock` (credit note of an invoice only).

What registration posts: each line's expense account (the line's `accountCode`, else the item's purchase or expense account, else the operation type's account, else the posting rule `purchases.goodsForResale` or `purchases.defaultExpense`), VAT receivable (a non-VAT-payer books the VAT into the expense), payables, and output VAT for self-assessed VAT (reverse charge). Lines with deferral dates are debited to the prepaid-expenses account and get a monthly release schedule, posted by `purchases/deferrals/post` (see the `month-end-close` skill). A credit note posts the reverse and is applied to the credited invoice as a payment up to its open amount. Answers 409 when the accounting period is locked.

A draft can be changed with `purchases/invoices/update` (`lines` replaces all lines) or removed with `purchases/invoices/delete`; both answer 409 once registered.

## Purchase orders and goods receipts

When the company orders before it is billed: `purchases/orders/create` (`partnerId`, `orderDate`, `lines`; the number comes from the PO series) → `purchases/orders/submit` → `purchases/orders/approve` (needs the `purchases:approve` scope) → `purchases/receipts/create` (`orderId`, `receiptDate`, `lines [{ orderLineId, quantity }]`, `warehouseId` when the order has none) as goods arrive. The bill then carries `purchaseOrderId`, and `purchases/invoices/match` (`invoiceId`, `priceTolerancePercent`) compares it with the order and the goods received without changing anything: rows are `matched`, `not_received`, `over_invoiced`, `price_mismatch`, `not_on_order` or `not_invoiced`.

## Follow up

- Unpaid bills: `purchases_invoices_list` with a filter on `paymentStatus`, `dueDate`, `partnerId`, `status`; `totals: ["grossTotal"]` sums them.
- Payment is recorded when the outgoing bank line is matched to the bill (see the `bank-reconciliation` skill); `bank/payments/export` builds the pain.001 file to pay suppliers.
- Chart of accounts for `accountCode`: `ledger/accounts/list`.
