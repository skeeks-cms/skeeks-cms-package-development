# Model activity logs

Attach `skeeks\cms\behaviors\CmsLogBehavior` in the owning model's
`behaviors()` and use `skeeks\cms\models\behaviors\traits\HasLogTrait`.
The trait supplies `skeeksModelCode` and `getLogs()`; attaching the behavior
alone to the base CMS ActiveRecord is insufficient. `CmsContentElement`,
`ShopBrand` and `ShopCollection` are examples. Keep activity logging out of
controllers and import adapters: model `save()` (including `save(false)`),
`update()` and `delete()` emit the events for every caller.

Register model names and the existing admin controller in
`skeeks->modelsConfig` through the owning package's bootstrap. `CmsLog` uses
this registry to resolve its model and render a human-readable entity title;
without it, an inserted/deleted entity's title falls back to its PHP class.
Preserve project overrides when adding default registry entries.

`updateAll()`, `deleteAll()`, `updateAttributes()`, counter updates and direct
SQL bypass these model events. Business changes requiring an activity event
must use the model lifecycle. Do not add a duplicate transport-level log.

Use `no_log_fields` for technical model fields such as `show_counter`.
The behavior merges these with its standard identity, timestamp, author,
site and optimistic-lock exclusions. An unchanged or excluded-only update
must produce no event. Deactivation is an update of the active attribute.

Provide explicit `relation_map` entries when the relation cannot be inferred
from `<name>_id`, including non-ID keys such as `country_alpha2 => country`.
The behavior resolves old and new values independently from the relation's
link, retaining readable snapshots even after the related record changes.
It does not rely on the owner's cached relation for a simple foreign key.

Handler order matters: `RelationalBehavior` must save its changes before
`CmsLogBehavior` reads them. The log must run before `HasStorageFile` removes
the previous image in `afterSave`; otherwise `old_as_text` may fall back to an
ID because the old file row no longer exists. Preserve the other behavior
contracts when ordering these handlers.

`CmsLog` is saved synchronously on the application's database connection.
With the owner on that same connection and transactional tables, an outer
transaction or nested savepoint rolls back the model change and its log
together. The behavior does not create a transaction for the caller. Callers
requiring atomic business writes must establish one and propagate failures.
Database rollback does not restore physical files deleted by storage.

Verify real insert/update/deactivate/delete operations, readable old/new
relations, excluded-only saves, and rollback of both owner and log. The
`cms-shop/tests/reference-activity-log.php` integration test exercises real
models and the GPD writer in a disposable MariaDB, with storage clusters
disabled so the test cannot touch actual files. Enabling the behavior only
affects future model events; do not backfill history during deployment.
