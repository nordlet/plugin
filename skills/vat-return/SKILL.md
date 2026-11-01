---
name: vat-return
description: Prepare and file VAT in Nordlet for a company in any of its 30 countries - the national VAT return as the boxes of the country's form, the country's VAT registers and listings, the quarterly OSS and monthly IOSS returns, the filing calendar with submission and download, and recording the VAT payment. Use when the user asks about the VAT return, VAT due, a VAT register or listing, OSS, IOSS, filing deadlines, or submitting a declaration.
---

# VAT returns in Nordlet

Direct tool: `reports_vat_summary`. The rest goes through `call_operation` with the path in camelCase as `operationId` (`declarations/eu/vat-return/compute` → `declarationsEuVatReturnCompute`). Conventions are in the `nordlet` skill.

A country-specific declaration answers 422 unless the company is registered in that country. Filings of a sandbox company go to the administrations' test endpoints. Close the month first (`month-end-close` skill): the return is computed from issued sale invoices and registered purchase invoices, so a draft is not in it.

## 1. Which return

`declarations/eu/vat-return/packs/list` lists the national forms Nordlet computes, with country, form key, filing frequency and source. `calendar/list` (below) shows which deadlines apply to this company.

## 2. Compute

- National return: `declarations/eu/vat-return/compute` with `countryCode` (the company's country), `year`, `month` and `months` (1 for a monthly return, 3 for a quarterly one). The answer is the numbered boxes of the country's form as `{ code, label, amount }`, with `formKey`, `formName`, `warnings`, `notes` and the official `source`. 422 when no pack is registered for the country; `declarations/eu/vat-return/packs/list` names the registered ones.
- The country's own VAT files (sales and purchase registers, control statements, EC sales lists, the full return file where the administration takes a file) are country operations under `declarations/<country code>/...`; find them with `search_operations` using the form's name or the country code, and read the body with `describe_operation`. A filing that has a deadline in the calendar (step 4) is also built from that deadline, which fixes the period.
- Union OSS: `declarations/eu/oss/compute` with `year` and `quarter`, from issued documents with `vatScheme` `oss_union`: one row per Member State of consumption and rate, in EUR at the rate of the last day of the period, plus `corrections` for credit notes that credit invoices of earlier periods. IOSS: `declarations/eu/ioss/compute` with `year` and `month` for `vatScheme` `ioss`. Both answer 422 when the company is not established in the EU. Warnings name non-statutory rates, documents without `vatCountryCode` and credit notes without `creditedInvoiceId`.

## 3. Reconcile before filing

`reports_vat_summary` with `fromDate`, `toDate` and `side` `sales`, then `purchases`: net, VAT and gross per rate and the number of documents, credit notes negative. `reports/vat-detail` lists every document behind a rate. Compare the output VAT with the return's sales boxes, the input VAT with the purchase boxes, and both with the VAT accounts of `reports_trial_balance` for the same period. Show the user the boxes and every difference with its amount before anything is filed.

## 4. File

`calendar/list` with `from`, `to` and `includeDone: false` lists the company's deadlines in due-date order (the VAT return of its country, the registers and listings, OSS and IOSS when they apply), each with its `key` (`ruleKey:period`, for example `de-vat-return:2026-03`), its submissions, and whether it can be filed with `calendar/submit`, amended, or built with `calendar/download`. `calendar/get` with the `key` reads one.

- `calendar/submit` with `key` generates the filing and sends it to the administration; `amend: true` files a correction of a return already submitted for the period (only formats with a correction mark accept it). It needs the country's filing system configured (`declarations/configs/list` shows the systems and their stored values; `declarations/automation/list` shows which deadlines Nordlet can file by itself, whether the system and certificate are in place, and whether it would go to the test or the production service).
- `calendar/download` with `key` builds the file for the company to upload in the administration's own portal or program; nothing is sent and no filing is recorded.
- The country support table at https://nordlet.com/en/docs/filing-support/filing-support/ says, per country and filing, whether Nordlet sends it, builds the file, or gives the figures to key in.

Filing is outward-facing and cannot be undone by Nordlet: get the user's explicit agreement on the computed figures before calling `calendar/submit`.

## 5. Record the payment

When the VAT is paid, `declarations/tax-payments/create` with `tax: "vat"`, `year`, `month`, `kind` (`final` for the return's amount, `advance`, `refund` for money received), `amount`, `paidOn`, `description` and optional `reference`. The bank line itself is matched or booked as in the `bank-reconciliation` skill.

## Related

- VAT treatment of a single sale (scheme, country, rate): `reference/vat/resolve`, described in the `sales-invoicing` skill.
- Distance-sales threshold: `declarations/eu/distance-sales-threshold/get`; SME exemption position: `declarations/eu/sme-threshold/get`; Union turnover for the cross-border SME scheme: `declarations/eu/union-turnover/get`.
