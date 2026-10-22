# Creating CRM records from a source

"Создать задачу из письма", "лид из звонка", "сделка из лида": a new record
is created in the standard create window of its own controller, filled from
the source and linked back. Core `skeeks/cms` owns only the contract; the
package that owns the source implements it. Do not build a second create form
in the source package and do not teach core about the source.

## Contract

- Register the source in the `skeeks` component of the package config:
  `'recordSources' => ['<type>' => Class::class]` (`Skeeks::$recordSources`,
  registrations of packages merge). The class implements
  `skeeks\cms\source\RecordSourceInterface`.
- Open the standard create window of the target controller with
  `?source=<type>:<id>` (`RecordSources::PARAM`; type `[a-z][a-z0-9_-]*`).
  `RecordSources` handles every `BackendModelCreateAction` through
  class-level events attached in `Skeeks::bootstrap()` for web apps; the form
  posts to the same URL, so the parameter survives saving.
- `find($id)` returns the source only if the current employee may see it;
  `supports($modelClass)` limits target models. Unknown, hidden or
  unsupported sources are ignored silently: the window works as without the
  parameter.
- `prefill()` runs on `initFormModels`, before posted values are loaded:
  everything it fills stays editable. A line from `describe()` (plain text,
  encoded by core) stands above the form fields.
- `enforce()` runs on `beforeValidate` after the controller's own handlers
  (class-level handlers run after instance ones). Use it for attributes the
  form does not show or the controller resets (a lead's company, client,
  `source_type`, `source_data`).
- `created()` runs on `afterSave` inside a transaction opened on
  `beforeSave`; an exception rolls back the new record too. Link the record
  and write its feed entry here, never from the view.
- To show where a saved record came from, also implement
  `skeeks\cms\source\RecordOriginInterface::origins($record)` (each item:
  plain `label`, trusted `html`; link to the source only when the employee
  may open it). `RecordOriginWidget` renders them at the top of the task,
  lead and deal cards; add the widget to other cards the same way instead of
  writing a source-specific block into a core view.
- Put the work in a service of the source package that also has a
  `create($source, $type, $attributes)` method, so rules, API tools and bots
  create the same record without the window.

## Rules learned from the first consumer

- Verified with `skeeks/cms-communication` `MessageRecordSource` /
  `MessageRecordFactory` (2026-10-23): tasks, leads and deals from letters.
- Deal descriptions are visible to clients: never prefill them with internal
  text (letters, comments).
- A lead created by hand from a source keeps `source_ref` empty; the source
  package's own link (for mail, `cms_comm_link.link_type = created`) says
  which record came from which source. Do not let a manual window collide
  with the unique `(site, source_type, source_ref)` of automatic ingestion.
- Cards show a source's feed entries only through
  `CmsLogQuery::activity()`; the task card uses it too (core 2026-10-23).
- The window is opened with `sx.classes.backend.widgets.Action` and an
  `updateSuccessCallback` that refreshes the source screen. A saved create
  window marks the window that opened it as changed: if that window was
  itself opened by an `Action` without a callback, closing it reloads the
  whole page. Give such panels a callback that refreshes only what changed.
- Check that the employee may open the target create action before showing
  the button (permission `<controller>/create`, else `<controller>`, as
  `BackendEntityLink` does).

Test: `skeeks/cms/tests/crm-extension-points.php` (registry, parsing,
prefill/enforce order, rollback on a failing `created()`).
