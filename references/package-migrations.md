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

## Repeatable up, honest down

MySQL DDL is not transactional: a run may stop between statements and be run
again by hand. A `safeUp` adds only what is missing (`getColumn()` on a fresh
schema) and never rewrites data by an unconditional `UPDATE`: a repeated or
completed run must not reset settings users made meanwhile. When existing
rows need another value than new ones, let the DDL fill them — add the
column with the value for existing rows as its default, then change the
default for new rows — instead of a backfill after it (verified with
`skeeks/cms-communication` `m261109_110000`, campaign `lead_to`: existing
«Неразобранное», new «отправитель»). A `safeDown` that would drop columns or
tables holding data users set returns false with a message while such data
exists (the site restores its backup instead); it never erases it silently.
Check every case on a separate database: the old schema with rows, a second
run, an interrupted run, down with and without own data
(`cms-communication/tests/flow-migration-db.php`).

A repeated `cms/migrate` skips a version already written to the history, so it
proves nothing here: the dangerous case is a run stopped after some DDL and
before the history row (lost connection, killed deploy). Rules for every new
DDL migration (release audit N4/N5, 29.10.2026):

- every step of `safeUp` checks before it acts: a column, an index, a foreign
  key, a table is added only when missing; a dropped index is dropped only when
  present. In `skeeks/cms` 6.4.9.75+ use `skeeks\cms\db\RepeatableMigrationTrait`
  (`addColumnIfMissing`, `createIndexIfMissing`, `addForeignKeyIfMissing`,
  `createTableIfMissing` and the `…IfExists` drops); other packages may inline
  the same checks (`hasIndex` through `getSchema()->getTableIndexes($t, true)`,
  foreign keys through `getTableSchema($t, true)->foreignKeys[$name]`);
- a data step between DDL statements must be safe to repeat: copy only rows the
  target does not have yet (`NOT EXISTS`), fill only records still without a
  value (a main responsible only where none is set), and when a step must not
  rerun once users could change its result, run it before the last DDL of that
  table and skip the table when that last object exists (crm_protection: the
  index is created after the «always» carry-over);
- `safeDown` checks every kind of data it would erase — filled columns, rows of
  dropped tables, references to dropped tables — before its first DDL
  (`refuseRollbackWithData()` of the trait throws «Откат невозможен…»; a
  package without the trait returns false with a message), each check only for
  a column or table that still exists, and then drops only what exists, so a
  stopped rollback can be repeated too. A unique check on names is not enough:
  history rows with unique names are lost the same way;
- never return early because the table exists: a run stopped right after
  `CREATE TABLE` leaves it without its unique index and foreign keys, so each
  element is restored on its own. Before a unique index on a table that may
  already have been written, merge the duplicates the database groups by its
  collation (`GROUP BY … HAVING COUNT(*) > 1`), keeping the row that stays
  meaningful (the latest updated). Verified: `m261029_210000`
  `cms_crm_inbound_rotation` (release audit F4);
- what counts as data for an honest down: anything users entered (settings,
  lists in JSON columns) — refuse while a value differs from the default
  (NULL, '', `[]`, `{}`, `null` for JSON lists), as `m261029_220000` does for
  the queue settings of a pipeline (F5). Derived state the code rebuilds by
  itself (a round-robin cursor) may be dropped; say in the docblock what its
  loss changes.

Test it with the interruption harness `skeeks/cms`
`tests/support/migration-interrupt.php`: it stops `safeUp` before each
executed statement in turn and requires that up resumes to the schema of one
clean run and that down returns the schema before the migration (columns,
indexes, foreign keys, comments through information_schema). Examples:
`cms/tests/crm-migrations-repeat-stand.php`,
`cms-communication/tests/migrations-repeat-db.php`. They copy empty tables from
the stand database into a separate one, bring it back with the migrations' own
`safeDown` and drop it at the end. MariaDB keeps instantly dropped columns in
the row format until a rebuild, so the harness rebuilds a table after each
rollback; without that, dozens of cycles on a wide table end in «Row size too
large», which a single real run does not meet. Verify that a new test fails on
the old migration before trusting it.

Rollback guards are tested on rows the test creates itself in its own
database (structure copied with `CREATE TABLE … LIKE`, every required column
filled by type), never on rows that happen to exist in the stand database:
a fresh production snapshot may have none, and the check then fails or passes
for the wrong reason (`cms/tests/crm-migrations-rollback-stand.php`, audit
F4-02). Name each expected kind of data in the assertion on the refusal message.

## Data transition commands

A transition of old data is often several console commands with a plan mode and
`--apply` (in `skeeks/cms`: `cms/crm/assign-owners` → `assign-pool-main` →
`assign-owners` again → `protect-owners`). One command's empty repeated plan
does not prove the sequence is complete: a step that derives from another
step's result (a contact gets the main responsible of his company) sees nothing
while that result does not exist yet, and a later step that only processes
records already in its state (protection of records with an owner as main)
does not see the skipped ones either (release audit F4-01: 22 contacts of
companies given out from the pool stayed free, so «Работать» made a deal
without an approval). So:

- rerun a deriving step after every step that creates its input, and write the
  order into the release plan;
- let the later step refuse `--apply` while the input of its precondition is
  still pending, and name the step to run (`protect-owners` checks
  `OwnerAssignment::pendingOwnerContacts()`);
- test the composition on own fixtures that reproduce the problematic data (a
  record without an author and without history), with the real roles of the
  site, including the user-visible effect (a request instead of a deal), and
  check that every repeated plan of the sequence is empty
  (`cms/tests/crm-owner-assignment-stand.php`).

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
