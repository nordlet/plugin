---
name: nordlet
description: Work with a company's books in Nordlet through the nordlet MCP connector. Use when the user asks to create, find, issue or send invoices, register supplier bills, import or match bank transactions, read journal entries, run reports (trial balance, financial statements, VAT summary, debt aging), or calls anything in the Nordlet API.
---

# Nordlet

Nordlet is an accounting API. Every operation is `POST /v1/{module}/{resource}/{action}` with a JSON body. The `nordlet` MCP server exposes those operations as tools. Each tool call is an ordinary API request: it is authenticated with the plugin's API key, rate limited and metered like one, and a mutating call changes the company's books.

The API key is scoped to one company. Every call acts on that company. To work on another company, the user needs a key of that company.

## Tools

Three tools cover every operation:

| Tool | What it does |
|---|---|
| `search_operations` | Finds operations by words in their name, path, summary or description, optionally within one module (`sales`, `bank`, `reports`, ...). Without a query it lists the modules. |
| `describe_operation` | Returns the method, path, description and JSON request schema of one operation by its `operationId`; `includeResponse: true` adds the schema of the 200 response. |
| `call_operation` | Runs one operation with a JSON body. Takes an optional `idempotencyKey`, sent as the `Idempotency-Key` header. |

The most used operations are also direct tools with their own request schemas: `partners_list`, `partners_get`, `partners_create`, `partners_find_or_create`, `catalog_items_list`, `sales_invoices_list`, `sales_invoices_get`, `sales_invoices_create`, `sales_invoices_issue`, `sales_invoices_send`, `purchases_invoices_list`, `purchases_invoices_create`, `purchases_invoices_register`, `bank_transactions_list`, `bank_transactions_import`, `bank_transactions_match`, `ledger_journal_transactions_list`, `reports_trial_balance`, `reports_financial_statements`, `reports_vat_summary`, `reports_debt_aging`. The direct tools do not send an `Idempotency-Key`; use `call_operation` when one is needed.

For anything else: `search_operations` → `describe_operation` → `call_operation`. Read the request schema before calling; do not guess field names.

## Conventions

- Monetary amounts are decimal strings with up to 4 decimal places, such as `"121.0000"`, never numbers. VAT is computed per line with half-up rounding.
- Dates are `YYYY-MM-DD`; timestamps are ISO 8601 UTC.
- Every `list` operation takes `page`, `pageSize`, `sort: [{ field, dir }]` and `filter: [{ field, op, value }]`. Filter operators: `eq`, `ne`, `contains` (text fields only), `gte`, `lte`, `in` (up to 500 values). Sortable and filterable fields are whitelisted per operation; an unknown field answers 400. `totals: ["field", ...]` sums numeric fields over every matching row; `totalsByCurrency` splits the sums by currency. Responses are `{ rows, total, page, pageSize, totals?, totalsByCurrency? }`.
- Scopes: a key carries `{module}:read` (get and list), `{module}:write` (mutations), `{module}:*` or `*`. A missing scope answers 403 `forbidden`; that is the key's scope, not a wrong request.
- Rate limit: 300 requests a minute per key. On 429, wait for the `retry-after` header.

## Idempotency

Send an `idempotencyKey` (any string up to 255 characters) with every mutating `call_operation`, one key per intended action, for example `sales-invoice-create-<order id>`:

- Same key and same payload again → the stored response is replayed (`x-idempotent-replay: true`).
- Same key, different payload → 422 `idempotency_key_reuse`.
- The same call still in flight → 409 `idempotency_in_progress`.
- Keys expire after 24 hours. 4xx answers are replayed too; 5xx answers are not stored, so a retry re-executes.

## Errors

One envelope everywhere:

```json
{ "error": { "code": "validation", "message": "Request validation failed", "requestId": "k1x…", "key": "request_invalid", "fieldErrors": { "lines.0.unitPriceExclVat": ["Invalid value"] } } }
```

| Code | Status | Meaning |
|---|---|---|
| `validation` | 400, 422, 413 | Invalid request; `fieldErrors` names the fields. 413 when a file or body is too large. |
| `unauthorized` | 401 | Missing or invalid key. |
| `payment_required` | 402 | The trial or prepaid credit has ended; the message says how to continue. |
| `forbidden` | 403 | The key lacks the scope. |
| `not_found` | 404 | No such record in this company. |
| `conflict` | 409 | For example a posting dated inside a locked accounting period. |
| `idempotency_key_reuse` | 422 | Same key, different payload. |
| `idempotency_in_progress` | 409 | A duplicate while the first call runs. |
| `rate_limited` | 429 | Wait for `retry-after`. |
| `internal` | 5xx | Report with the `requestId`. |

A country-specific declaration answers 422 unless the company is registered in that country. When the message is a fixed one, `key` names it and `params` holds its values (for example `{ "period": "2026-09" }` for a locked period). Quote the `requestId` when reporting a problem.

## Accounting rules the API enforces

- Issuing a sales invoice allocates a gapless number, checks that the accounting period is open and posts the balanced journal entry in the same transaction.
- Journal postings are balanced; document numbers are gapless per series and year.
- A locked accounting period rejects any posting dated inside it (409).
- Soft-deleted documents keep their audit trail. Issued or registered documents cannot be deleted.
- Every mutation is recorded and queryable with `audit/list`.

Because a mutating call changes the books and an issued document cannot be deleted, show the user the request body of a mutating call and get their agreement before sending it, unless they asked for the change in that message.

## The invoicing cycle

| Step | Operations |
|---|---|
| Customer or supplier | `partners/find-or-create`, `partners/create`, `partners/list` |
| Sales invoice | `sales/invoices/create` → `sales/invoices/issue` (allocates the number, posts the entry) → `sales/invoices/send` |
| Supplier bill | `purchases/invoices/create` → `purchases/invoices/register` |
| Bank statement | `bank/statements/import` (camt.053 XML) or `bank/transactions/import` (JSON) |
| Match suggestions | `bank/transactions/suggest-matches` |
| Reconcile a payment | `bank/transactions/match` (posts the payment, updates `paymentStatus`) |
| Pay suppliers | `bank/payments/export` (pain.001 SEPA file) |
| Reports | `reports/trial-balance`, `reports/financial-statements`, `reports/vat-summary`, `reports/debt-aging` |
| Migration from another system | `migration/books/validate` → `migration/books/import` |

Use `search_operations` to get the `operationId` of an operation that is not a direct tool.

## Sandbox companies

A company created with `isSandbox: true` behaves like a real one (same modules, same API), is badged as test data, and its filings and e-invoices go to the administrations' test endpoints. Usage in a sandbox company is metered and billed like usage in a real one. The key decides which company a call acts on; there is no sandbox switch on a call.

## Reference

- MCP server: https://nordlet.com/en/docs/guides/mcp/
- API conventions: https://nordlet.com/en/docs/guides/api-conventions/
- Getting started: https://nordlet.com/en/docs/guides/getting-started/
- API reference: https://nordlet.com/en/docs/api/
