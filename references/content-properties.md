# Content properties (характеристики)

Applies to every programmatic writer of `cms_content_property` and
`cms_content_property_enum`: migrations, server scripts, imports and API
services. Administration (`/cms/admin-cms-content-property`) is the reference
behavior.

## Prefer the API on installed sites

`skeeks/cms-mcp` exposes `cms_content_property_create/update` and
`cms_content_property_enum_create/_create_batch/update` (administrator role,
`cms.content.write`). Use them for site data instead of one-off server scripts.
Migrations remain the tool for properties that project code depends on.
The package's `AGENTS.md` owns the tool contract.

## Model contract

- `component` is a `Yii::$app->cms->relatedHandlers` id
  (`PropertyType*` class). Do not write `property_type` or `is_multiple` by
  hand: `RelatedPropertyModel::_processBeforeSave()` copies them from the
  handler on every save.
- Store `component_settings` as the handler's `toArray()` after
  `load($settings, '')` and `validate()` on a cloned handler, as the
  administration create/update actions do. `PropertyType::toArray()` drops
  `property`; do not pass `id`, `code`, `name` or `property` as settings.
- `getHandler()` caches the handler on the model. Set `component` and
  `component_settings` before anything reads `$model->handler`, or reload the
  model; otherwise save derives the type from the stale handler.
- Bind contents and sections through the `cmsContents` / `cmsTrees` relations
  (`RelationalBehavior` rewrites `cms_content_property2content` /
  `cms_content_property2tree` after save). `cmsContents` is required; trees
  require `cms_site_id` and should belong to that site.
- `code` is unique per `cms_site_id`. Shop flags live in the same table:
  `is_offer_property`, `is_img_offer_property`, and `is_sticker` (one per site,
  unique index, `0` is normalized to `null`). `is_vendor`, `is_vendor_code` and
  `is_country` are also unique per site and are not edited in administration.
- Administration does not allow changing a list (`L`) property with enum
  values to a non-list component.

## Enum values

- `cms_content_property_enum` rows belong to a list property only.
- The model's default `code` is `md5(rand(1, 10) . time())`: it repeats when
  several values are created within one second. Generate a random code
  explicitly in bulk writers. `code` is at most 32 characters.
- `value` comparisons follow the table collation (`utf8_general_ci`, case
  insensitive); idempotent writers should look up an existing value first.
