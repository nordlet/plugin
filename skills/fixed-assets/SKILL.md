---
name: fixed-assets
description: Keep the fixed asset register in Nordlet - asset groups with their accounts, registering an asset, straight-line depreciation preview and posting (or automatic monthly posting), modernisations, disposal with gain or loss, and the input VAT facts of capital goods. Use when the user asks to add a fixed asset, depreciate, sell or scrap an asset, see net book values, or set up asset groups.
---

# Fixed assets in Nordlet

Calls go through `call_operation` with the path in camelCase as `operationId` (`assets/depreciation/post` → `assetsDepreciationPost`). Conventions are in the `nordlet` skill.

Depreciation is straight-line and monthly: each month's amount is the remaining depreciable value (cost plus modernisations less salvage value less accumulated depreciation) divided by the months left.

## 1. Asset groups

`assets/groups/list`; `assets/groups/create` with `code`, `name`, `assetAccountCode` (the asset account), `depreciationAccountCode` (accumulated depreciation), optional `expenseAccountCode` (default the chart's depreciation expense account) and `defaultUsefulLifeMonths`. Each account must exist and be postable (422); a duplicate code is 409. Postable accounts: `ledger/accounts/list`.

## 2. Register an asset

`assets/assets/create` with `groupId`, `code`, `name`, `acquisitionDate`, `acquisitionCost`, optional `salvageValue` (default 0, must be below the cost), `usefulLifeMonths` (default the group's; 422 when neither is set), `depreciationStartDate` (default the acquisition date), `notes`, `documents` (`name`, `ref`).

The purchase itself is a supplier bill whose line carries the group's asset account as `accountCode` (`supplier-bills` skill); the register entry does not post anything. Assets brought over from another system come in with their accumulated depreciation through the migration package (`onboarding-migration` skill).

`assets/assets/update` changes an asset that has no posted depreciation yet (409 afterwards or once disposed). `assets/assets/get` returns the totals: cost including modernisations, accumulated depreciation, net book value, total life, the input VAT facts and the disposal. `assets/assets/list` filters `code`, `name`, `groupId`, `status`, `acquisitionDate`, `acquisitionCost`, `accumulatedDepreciation`.

## 3. Depreciate

- `assets/depreciation/preview` with `year` and `month`: per active asset the amount and the months it covers (earlier unposted months included) and whether the month is already posted. Nothing is written.
- `assets/depreciation/post` with the same body: one journal entry dated the last day of the month, debiting each group's expense account and crediting its accumulated depreciation account, for every active asset whose depreciation has started, covering the month and any earlier month not yet posted. An asset whose life or value is used up becomes `fully_depreciated`. 409 when the period is locked; `journalTransactionId` null when nothing is due.
- Automatic: `assets/settings/update` with `autoDepreciation: true` makes the scheduler post the previous month's depreciation once a day; `assets/settings/get` shows the setting.

Show the preview and get the user's agreement before posting.

## 4. Modernise

`assets/assets/modernize` with `id`, `date`, `amount` and optional `addedLifeMonths`: the amount is added to the cost and the months to the useful life from the month of `date`, and the remaining depreciable value is spread over the months left. 422 when the asset is disposed or no depreciation month would be left (give `addedLifeMonths` of at least 1); a fully depreciated asset becomes active again.

## 5. Dispose

`assets/assets/dispose` with `id`, `date`, `reason` (`sold`, `scrapped`, `written_off`) and `proceeds` (the sale price excluding VAT; 0 when scrapped or written off). It removes the cost and the accumulated depreciation, books the net book value as a disposal loss and the proceeds as a disposal gain (posting rules `assets.disposalLoss`, `assets.disposalGain`, `assets.disposalProceeds`; `ledger/posting-rules/list` shows the accounts) and stops the depreciation. Depreciation must be posted for every month before the disposal month: run `assets/depreciation/post` up to that month first.

A sale to a customer is also invoiced (`sales-invoicing` skill); the disposal records the register side.

## 6. Input VAT of capital goods

`assets/assets/input-vat` records what the annual VAT return needs for the adjustment of the deduction over the adjustment period (Article 187 of the VAT Directive): `inputVatAmount`, `inputVatFirstUseDate`, `inputVatDeductiblePercent` at first use, `inputVatRealEstate` (ten-year period instead of five) and `inputVatUseChanges` (`year`, `percent`, `reason` `use_change`, `sale`, `withdrawal`). Allowed after depreciation has been posted.

## Related

- The depreciation step of the `month-end-close` skill is step 3 of this one.
