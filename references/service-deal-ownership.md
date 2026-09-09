# Service ownership and deal-derived state

Applies to billable services in `skeeks/cms-hosting` (VPS, sites, DNS zones)
and to any package entity that a customer pays for.

## Contents

- [Ownership contract](#ownership-contract)
- [Deal-derived service state](#deal-derived-service-state)
- [Collection grids](#collection-grids)
- [Access scoping](#access-scoping)
- [Shared form composition](#shared-form-composition)

## Ownership contract

A service belongs to a company **or** to a private person, and is paid for by a
deal of that same owner. The entity therefore carries three nullable columns:

- `cms_company_id`;
- `cms_user_id`;
- `cms_deal_id`.

Do not add a second, service-specific owner column. Do not duplicate the
company/person choice as an enum: which column is filled is the answer.

## Deal-derived service state

`CmsDeal` is the single source of truth for whether a service is paid and until
when. Read `cms_deal.is_active` and `cms_deal.end_at`; never mirror them into
per-service `active_to`/`is_active` columns that then have to be kept in sync.

Two details the executable code must handle:

- `CmsDeal::getIsEnded()` compares `time() > end_at` and therefore reports an
  empty `end_at` as ended. A service with an open-ended deal must not rely on
  it — treat empty `end_at` as "no expiry" explicitly.
- A service with no deal at all is an internal/system record. Decide its meaning
  per entity and state it in the model; `DnsZone` treats it as a permanently
  active service zone.

Expose the decision as model properties rather than repeating the condition in
controllers and views, for example `getIsServiceActive()`, `getServiceActiveTo()`
and a human-readable `getServiceStateText()`.

Gate write operations on that state. `DnsZone::getIsEditable()` combines the
entity's own lifecycle status with `isServiceActive`, so an unpaid service
becomes read-only without a separate flag.

## Collection grids

Sorting by expiry cannot use a relation accessor. Join the deal and project the
two values as expressions, then declare them in `sortAttributes`:

```php
$query->joinWith('deal as deal');

$query->select([
    static::tableName().'.*',
    'calc_active_to' => new Expression("IF (deal.id IS NOT NULL, deal.end_at, 0)"),
    'is_deal_active' => new Expression("IF (deal.id IS NULL, 1, IF (deal.is_active, 1, 0))"),
]);
```

Read them back through `$model->raw_row`, not through the model attributes.
`AdminVpsSiteController` and `AdminDnsZoneController` both use this shape; keep
new service grids consistent with it so that "expiring soon" tone rules and
default ordering stay comparable.

## Access scoping

Visibility follows the deal, not the service. A query class extending
`CmsActiveQuery` implements `forManager()` and `forClient()` by joining the deal
and constraining it to `CmsDeal::find()->forManager()` / `->forClient()`.

Records without a deal are not visible to non-administrators: an unowned service
record has no customer to scope it to.

## Shared form composition

`skeeks\hosting\controllers\traits\HostingClientDealTrait` owns the whole
owner-and-deal form block for the hosting package:

- `registerClientSwitcher($model, $modelId)` — CSS/JS for the Company/Person
  toggle; `$modelId` is the lowercased model name used in form markup ids
  (`hostingvps`, `hostingvpssite`, `dnszone`);
- `getClientFieldSet($modelId)` — the toggle plus both `AjaxSelectModel` fields,
  each marked `data-form-reload` so choosing an owner reloads the form;
- `getClientDealItems($model)` — deals of the chosen owner, empty until an owner
  is chosen;
- `getDealFieldSet($model)` — the deal select built from those items.

The deal list is deliberately empty before an owner is selected: offering every
deal in the system in that field is never correct.

The fieldset is labelled **"Клиент"** across all sections. The same label is used
by the equivalent blocks in `skeeks/cms` (`AdminCmsDealController`,
`AdminCmsProjectController`), `skeeks/cms-shop` (`AdminPaymentController`) and
project-owned screens; keep new screens on that wording rather than reintroducing
per-screen variants.

Controllers must consume the trait instead of copying the switcher. Before this
trait existed the same 72-line method was duplicated verbatim in
`AdminVpsController` and `AdminVpsSiteController` and drifted out of any single
owner — that duplication is what the trait exists to prevent.
