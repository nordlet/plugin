---
name: inventory
description: Keep stock in Nordlet - warehouses and tracked items, receipts (manual, from goods receipts and registered bills), issues through invoices, write-offs, transfers and stock takes, landed costs, lots and expiry, reorder rules, the negative stock policy, and the stock reports (levels, balance at a date, ageing, movement, shortages, write-off acts). Use when the user asks what is in stock, to receive, move, count or write off goods, to allocate freight to stock, or which items need reordering.
---

# Inventory in Nordlet

Direct tools: `catalog_items_list`, `sales_invoices_issue`, `purchases_invoices_register`. The rest goes through `call_operation` with the path in camelCase as `operationId` (`inventory/stock/write-off` → `inventoryStockWriteOff`). Conventions are in the `nordlet` skill.

Stock is valued FIFO: every receipt opens a cost layer, every issue consumes the oldest layers first, and the ledger side goes through the posting rules `inventory.stock` and `inventory.cogs` (`ledger/posting-rules/list`) or the stock account of the layer. Every change emits `stock.changed`.

## 1. Setup

- Warehouses: `inventory/warehouses/create` (`code`, `name`, `isDefault`, `countryCode`); `inventory/warehouses/list`.
- Items: `catalog/items/create` with `type: "product"`, `name`, `code`, `unit`, `tracking` (`none`, `lot`, `serial`), `barcode`, prices, `purchaseAccountCode`, `saleAccountCode`; `catalog_items_list` filters `name`, `code`, `barcode`, `type`.
- Negative stock: `inventory/settings/get`; `inventory/settings/update` with `negativeStockPolicy` `reject` (an issue above the stock on hand is refused with 422) or `allow` (the missing quantity is written as a shortage priced at the last receipt's unit cost and settled by the next receipt, with any cost difference posted between `inventory.cogs` and `inventory.stock`). An issue from a named lot is refused under both policies when the lot is short.

## 2. Goods in

- From a purchase order: `purchases/receipts/create` (`orderId`, `receiptDate`, `lines [{ orderLineId, quantity, lotNumber, expiryDate }]`, `warehouseId` when the order has none) against an approved order; product lines go into stock at their share of the order line's net (the `supplier-bills` skill).
- From a supplier bill without an order: `purchases_invoices_register` with `warehouseId` receives the product lines.
- By hand: `inventory/stock/receive` with `warehouseId`, `itemId`, `date`, `quantity`, `unitCost`, optional `lotNumber` (required for a lot- or serial-tracked item; an unknown lot is created), `expiryDate`, `notes`. A serial-tracked item is received one unit at a time; a serial number already in stock is 409.
- Freight, duty and other costs of a receipt: `inventory/landed-costs/create` with `date`, `amount` (whole cents), `method` (`by_value` default, `by_quantity`), and `goodsReceiptId` or `movementIds`, optional `sourceInvoiceId`. The share of the quantity still on hand raises the layers' cost; the share of quantity already sold is posted to `inventory.cogs`.

## 3. Goods out and between warehouses

- Sold: `sales_invoices_issue` with `warehouseId` takes the sold goods out of that warehouse (`sales-invoicing` skill).
- Written off: `inventory/stock/write-off` with `warehouseId`, `itemId`, `date`, `quantity`, optional `lotNumber`, `expenseAccountCode` (default the chart's inventory expense account), `inventoryAccountCode`, `notes`: consumes the oldest layers and posts expense against the stock account. 409 in a locked period.
- Moved: `inventory/stock/transfer` with `fromWarehouseId`, `toWarehouseId`, `itemId`, `date`, `quantity`, optional `lotNumber`: the source layers are consumed oldest first and received in the target with the same costs, lots and stock accounts. 422 when both warehouses are the same.
- Counted: `inventory/stock/take` with `warehouseId`, `date` and `lines [{ itemId or barcode, countedQty, unitCost?, lotNumber?, expiryDate? }]`: the difference to the quantity on hand is written off (oldest layers first) or received as a surplus at `unitCost` (default the item's purchase price), and the net shortage less surplus is posted in one journal entry between the expense account and the stock accounts; `journalTransactionId` is null when the net is zero. A surplus of a tracked item needs `lotNumber`.

Get the user's agreement before a write-off, a transfer or a stock take: each one changes stock and, except the transfer, posts to the ledger.

## 4. Lots and reordering

- `inventory/lots/list` (filters `itemId`, `lotNumber`, `expiryDate`) shows each lot's quantity on hand; `inventory/lots/update` changes `expiryDate` and `notes`.
- `inventory/reorder-rules/create` with `itemId`, `minQty`, optional `warehouseId` (else over all warehouses), `reorderQty`; one rule per item and warehouse. While active, an issue or reservation that takes the available quantity (on hand less reserved) from above `minQty` to at or below it emits `stock.reorder_needed` with the `reorderQty`.
- `inventory/reorder-rules/check` returns the active rules at or below their minimum with the quantities, without emitting anything. The order is placed with `purchases/orders/create` (`supplier-bills` skill).

## 5. Reports

| Question | Call | Body |
|---|---|---|
| What is on hand now | `inventory/stock/levels` | optional `warehouseId`, `itemId`; quantity and value per item and warehouse |
| What was on hand at a date | `reports/stock-balance` | `asOf`, optional `warehouseId` |
| How old the stock is | `reports/stock-aging` | `asOf`, optional `warehouseId`; remaining layers by 0 to 30, 31 to 60, 61 to 90, over 90 days |
| Opening, in, out, closing per item | `reports/stock-movement` | `fromDate`, `toDate`, optional `warehouseId`, `itemId` |
| Reserved more than on hand | `reports/stock-shortage` | optional `warehouseId` |
| Write-offs and count differences | `reports/write-off-acts` | `fromDate`, `toDate`, optional `warehouseId` |
| Every movement | `inventory/stock/movements/list` | filters `itemId`, `warehouseId`, `lotId`, `direction`, `documentType`, `date` |

`reports/jobs/create` with `reportType` `stock-balance` or `write-off-acts`, `params` and `formats` builds the XLSX or PDF; follow it with `reports/jobs/get` and `files/get`.

The stock value of `inventory/stock/levels` should agree with the inventory account in `reports_trial_balance` at the same date; name a difference with its amount.

## Related

- Assembling products from components: the bill of materials guide, https://nordlet.com/en/docs/guides/bill-of-materials/.
- Opening stock from another system: the `stock` section of the migration package (`onboarding-migration` skill).
