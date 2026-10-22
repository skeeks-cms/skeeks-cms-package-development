# Package migrations

## Registration and execution

Keep migration files in the owning package. Register them in its console config
under the application-level `controllerMap.migrate`:

```php
'controllerMap' => [
    'migrate' => [
        'migrationPath' => ['@skeeks/example/migrations'],
        // For NEW namespaced migrations instead:
        // 'migrationNamespaces' => ['skeeks\\example\\migrations'],
    ],
],
```

Use `migrationPath` for classes without a namespace. Use `migrationNamespaces`
for namespaced classes with a resolvable Yii alias and autoload mapping. Never
register the same directory in both. The standard Yii root `migrate` command
uses this registration directly. `cms/migrate` reads these two properties from
the same application config, plus its own module controller configuration.
Other settings (database, migration table, templates) remain controller-local;
configure them on the command that is actually used.

`skeeks\cms\console\controllers\MigrateController` delegates ordering, applying,
rollback, creation and history to Yii. It no longer copies or deletes migration
files in `@runtime/db-migrate`. Project `@console/migrations` comes first, also
as the default `cms/migrate/create` destination.

## Compatibility

`autoDiscoverMigrations` is false by default. Every required package must register
its migration paths explicitly. The legacy opt-in value true discovers existing
`<extension alias>/migrations` directories. Namespaced directories already registered
in `migrationNamespaces` are not also discovered as non-namespaced paths.
Unknown namespaced sources fail with a configuration diagnostic; never silently
skip all namespaced migrations.

There is no exclusion-list setting. Optional dependency migrations are simply
not registered. For example, cms-job registers its own migrationPath but not
the native yii2-queue migration namespace. Do not turn legacy discovery back on
in that configuration: it also finds optional dependency directories.

Before releasing the default-off loader to another project, compare its entire
migration inventory (not just pending migrations). Add missing registrations in
the owning packages before updating CMS. Existing packages cms-admin,
cms-comments and cms-hosting have explicit registration for their old migrations.
Release their changes with the loader wherever those packages are installed.

A new console configuration file must be declared in the package's Composer
extra.config-plugin.console. Updating only PHP files is insufficient when the
installed Composer metadata and Yii Config merge plan still describe an older
package. Normal package updates must install the new metadata and rebuild the
merge plan. In shared-vendor development, test the updated manifests explicitly;
do not mistake a hand-edited generated merge plan for a deployable solution.

Canonical paths are deduplicated. Different files declaring the same migration
identity cause a failure before migration SQL; the old collector silently
overwrote files. Resolve conflicts by selecting the intended source, not by
renaming an already applied class. Never add namespaces to existing deployed
migrations or rewrite their history keys. New migrations may use namespaces;
old global classes continue alongside them in the same Yii history table.

Setting `migrationPath = null` disables non-namespaced sources and discovery.
For a fully explicit module controller, set `autoDiscoverMigrations = false`
and `useApplicationMigrationConfig = false`, then supply its paths/namespaces.
Passing `--migrationPath` or `--migrationNamespaces` disables automatic
application-config merging and legacy discovery for that invocation; configure
both options when isolating both kinds of sources. `create` never discovers
vendor directories and otherwise follows standard Yii creation semantics.

## New columns read by other packages

A package release and its migration do not reach every site at the same
moment: Composer installs the code first, and dependent packages may be
updated without the owning one. When a new column of a shared model is read
by its own forms, cards or rules, or by another package, guard every access
with `$model->hasAttribute('<column>')` (rules, form fields, card rows,
consumers) and fall back to the previous behavior. An unguarded validation
rule on a missing column breaks saving the whole model. Do not add a hard
Composer constraint between packages only for such an optional column.
Verified consumers: `CmsContractor.registration_date` and
`CmsContractor.delivery_address` (skeeks/cms 6.4.9.48), read by
`skeeks/cms-shop` `ShopDocument::createFromBill()` (3.2.7.36).

## Verification

When a migration checks columns added by an earlier migration, request a fresh
schema with `getTableSchema($table, true)`. Persistent schema cache can still
describe the pre-update table and falsely report missing prerequisites.

A new index whose first column carries a foreign key makes MySQL/MariaDB drop
the index it created implicitly for that key (verified on MariaDB 10.5:
`cms_comm_message_mailbox__connection_id` was replaced by
`(cms_comm_connection_id, folder, uid)`). `down()` then cannot drop the new
index ("needed in a foreign key constraint"): recreate the single-column index
first when `SHOW INDEX` does not find it. MySQL DDL is not transactional, so a
failed `down()` leaves a half-reverted schema; run up → down → up on the real
database engine, not only on SQLite.

`skeeks/cms/tests/migration-loader.php` runs on a private SQLite memory database:
mixed migration styles, stable history identity, ordering, up/down/reapply,
creation, CLI selection, discovery, namespace diagnostics and duplicate-source
rejection. Its fixture files are created under a unique temporary directory and
removed afterwards. No site schema is involved.

`tests/migration-loader-site-audit.php` boots the consuming site's console config
and reads existing history without creating its table. It compares legacy and
native inventories and pending sets. A deliberate new explicit registration
can produce an expected inventory difference; review it instead of blindly
treating it as a regression. Never run `up`, `down`, `fresh` or history rewrites
on production as a compatibility test.

## Ownership of commercial seed data

Keep site-specific plan names, prices and commercial offers in the consuming
project's migrations. Shared packages own the schema and reusable resource
semantics. Update an existing plan by its stable business key while preserving
its ID and references; adding plans must be safe to repeat without duplicates.
Do not instantiate provisioned service models to seed catalog records: their
insert events may create external infrastructure, even with a local database.
