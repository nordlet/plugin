# Nordlet Accounting plugin for Claude Code

Connects Claude Code to a company's books in [Nordlet](https://nordlet.com), the accounting API for the EU/EEA market, and teaches Claude the API's conventions.

The plugin contains:

- **An MCP connector** to `https://api.nordlet.com/mcp`, Nordlet's own MCP server. Its tools search the API's operations, read their request schemas and call them, and the most used operations (partners, catalog items, sales and purchase invoices, bank transactions, journal entries and reports) are direct tools. The server is documented at https://nordlet.com/en/docs/guides/mcp/.
- **Fifteen skills** that teach Claude the API's conventions and Nordlet's workflows:

  | Skill | What it covers |
  |---|---|
  | `nordlet` | The API conventions: decimal-string amounts, list queries, scopes, idempotency keys, the error envelope, the accounting rules the API enforces and the invoicing cycle. |
  | `sales-invoicing` | Invoices, credit notes, proformas, quotes and advance invoices: finding the customer, the VAT scheme, create, issue, e-mail, payment link, PDF, Peppol. |
  | `supplier-bills` | Supplier bills and purchase credit notes, typed in or read from a PDF or photo through document capture; expense accounts, prepaid-expense deferrals, purchase orders and goods receipts. |
  | `bank-reconciliation` | camt.053 and MT940 statement import, JSON bank lines, PSD2 bank connections, import templates, match suggestions, matching and unmatching. |
  | `revolut-import` | Revolut Business CSV import with the built-in Revolut template: fees, card payments and top-ups posted, partners created, payments linked to invoices by reference. |
  | `stripe-import` | Stripe Payments export through a Stripe import template, the plain Stripe balance CSV, and payout reconciliation through settlement batches. |
  | `month-end-close` | Checks that everything is in, prepaid expenses, depreciation, deferred revenue, the reports, adjustments and locking the period. |
  | `vat-return` | The national VAT return of the company's country as the boxes of its form, the country's VAT registers and listings, OSS and IOSS, the filing calendar with submit and download, recording the VAT payment. |
  | `debt-collection` | Debt ageing and partner balances, the automatic payment reminders (schedule, preview, history), the credit check of a customer, late interest and credit limits. |
  | `payroll-run` | Employees, contracts, timesheets, sick leave and business trips, a trial calculation, the monthly run with the country's payment types, approval and posting, the SEPA payment file, filings and reversals. |
  | `fixed-assets` | Asset groups, registering an asset, depreciation preview and posting or automatic posting, modernisations, disposals, the input VAT facts of capital goods. |
  | `onboarding-migration` | Chart of accounts and posting rules, numbering series, operation types, master data, and the historical-books package validated and imported in one transaction. |
  | `recurring-billing` | Agreements with a billing period and invoice-template schedules, the billing run, the generated invoices and the daily automation. |
  | `inventory` | Warehouses and tracked items, receipts, write-offs, transfers and stock takes, landed costs, lots, reorder rules, the negative stock policy and the stock reports. |
  | `developer-webhooks` | Webhook subscriptions and the event catalogue, the delivery envelope and headers, signature and timestamp verification, retries, the delivery log and redelivery. |

## Install

```sh
claude plugin marketplace add nordlet/plugin
claude plugin install nordlet@nordlet
```

Claude Code then asks for your **Nordlet API key** (`nl_...`). Create one in the Nordlet app under Account → API keys. The key is stored in your operating system's credential store, not in a settings file.

To try the plugin from a local checkout without installing it:

```sh
claude --plugin-dir /path/to/plugin
```

## What the plugin sends and where

- Every tool call is an HTTPS request to `api.nordlet.com` with your API key as the Bearer token. Nothing is sent anywhere else.
- Each call is authenticated, rate limited (300 requests a minute per key) and metered like a direct API request.
- The key is scoped to one company. A key with `*:read` scopes can only read; a mutating tool call (create, issue, register, import, match) changes that company's books.

## Surfaces

Built for Claude Code. In claude.ai chat the skill loads, but the connector has to be added there separately from the plugin's Connectors tab.
