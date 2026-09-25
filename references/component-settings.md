# Component settings overrides

`skeeks\cms\base\Component` stores configurable settings in
`cms_component_settings` at three levels: default (`cms_site_id` and `user_id`
empty), site and user. At runtime the effective value is merged along the
component's `overridePath`, by default `default → site → user`; a later level
wins for every key it contains, even when it holds only one attribute.

## Which record administration edits

`AdminComponentSettingsController::actionIndex()` decides the level from the
current site:

- the default site (`cms_site.is_default = 1`) reads and saves only the
  default record (`overridePath = [default]`);
- any other site reads `default → site` and saves its site record.

`actionSite()` (`/admin-component-settings/site?site_id=N`) explicitly edits a
site record, including the default site's one. The ordinary settings page of a
default site therefore never shows a site-level record, while that record still
overrides the page's values on the storefront.

## Contract for programmatic writers

Every writer outside administration — MCP/REST tools, migrations, console
commands, installers — must save to the same level administration uses for that
site: `OVERRIDE_DEFAULT` on the default site, `OVERRIDE_SITE` elsewhere. Do not
create a site-level record for the default site as a side effect of a partial
update: the user will edit the visible default page and the change will
silently have no effect for the keys in that record.

Save a partial update with `save(true, $attributeNames)`; `Component::save()`
merges those keys into the existing record value instead of replacing it.
Restore the shared component's `overridePath` after writing, because
`Yii::$app->get()` returns the application singleton.

When a site-level record already exists on the default site and contains a key
being written to the default record, report it: the stored default is shadowed
until that site-level key is removed. There are no delete tools in
`skeeks/cms-mcp`; removal is done through administration (settings removal,
`do=sites`) or by an explicitly authorized database operation, followed by
cache invalidation of the component.

`skeeks/cms-mcp` `cms_component_settings_update` implements this contract and
returns `override` and optional `warnings` in its response.
