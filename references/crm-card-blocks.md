# Package blocks in CRM cards and versioned client documents

Verified 2026-10-23 with `skeeks/cms-proposal` (the "КП" block) on the
skeeks.com Windows stand: lead, company and deal cards, light, dark and a
375 px viewport.

## Card blocks (`CrmCardBlocksWidget`)

A package that owns CRM data of its own (proposals, later bills, messenger
chats) shows it in the lead, company and deal cards through the core place
`skeeks\cms\widgets\admin\CrmCardBlocksWidget` (`skeeks/cms` 6.4.9.64). Core
renders only what handlers return and knows nothing of the packages; do not
add a package block to a core view.

- Subscribe in the package bootstrap (web applications only), guarded by
  `class_exists()` so the package still works with an older core:
  `Event::on(CrmCardBlocksWidget::class, CrmCardBlocksWidget::EVENT_BLOCKS, fn(CrmCardBlocksEvent $e) => ...)`.
- The event carries `owner` (the card record: `CmsLead`, `CmsCompany`,
  `CmsDeal`), `position` (`POSITION_ASIDE` in lead and company cards,
  `POSITION_MAIN` under the deal card, which has no aside), `blocks`
  (id => trusted HTML) and `priority` (id => order, lower first, 100 by
  default; equal priorities keep the order handlers added them). Empty HTML
  is dropped.
- The client card (`/cms/admin-user/view`, after "Реквизиты") is a place
  too, with a `CmsUser` owner. Employees open the same view: a handler for
  clients checks `!$owner->is_worker` and renders nothing for employees;
  record source values refuse employees as well (`proposal-client:user.ID`).
  Verified 2026-10-23 with `cms-proposal` (client card 1672).
- A block looks like the neighbouring contact blocks (owner decision
  2026-10-23): a `BackendSurfaceWidget` (`titleTag h2`, `raised`,
  `responsive`) with the `CrmContactBlockWidget` markup and classes —
  `sx-contact-list`, rows `sx-contact-row` (icon, `sx-contact-label`,
  `sx-contact-value`, `sx-contact-note`, status in `sx-contact-actions`; the
  hover plate comes with them), `sx-contact-empty` for "… пока нет" and the
  full-width `sx-contact-add` "+ Создать …" under the list, not a button in
  the header. Register `CrmContactsAsset`; no package CSS. The handler checks its
  own permissions and the visibility of every record it lists (its
  `forManager()` scope); it renders nothing for a guest or an employee
  without administration access.
- "Создать …" in a block opens the standard create window of the package
  controller with a record source (`?source=`, see
  [record-sources.md](record-sources.md)) through
  `sx.classes.backend.widgets.Action`; the default `updateSuccess` reloads
  the card after saving. Do not add `_sxb[noa]` to such URLs.
- Test: `skeeks/cms/tests/crm-extension-points.php` (order, priority,
  position, empty blocks, the three views render the place).

## Phone and email rows, and the client's contacts in a deal card

Verified 2026-10-28 (П2г-1) on the skeeks.com stand: deals 670 (company),
481 (a person), 64 (no client); an administrator and an employee with
«Свои сделки и данные компании».

- A phone or email row of a company, a client or a deal is built by
  `CrmContactBlockWidget::phoneItems($phones, $controllerId, $owner, $options)`
  / `emailItems()`: the "⋮" menu of the row controller (`editable`), SMS
  and «Позвонить» (`call` adds attributes such as `data-lead-id`), «Письмо»
  (`mailto`, replaced by the mail package) and package actions through
  `contactActions()`. Company, client and deal cards use the same rows; do
  not rebuild them in a view.
- `limit` + `moreUrl` of the block show the first N rows and «ещё N» to the
  card that has them all.
- Contacts live in the company or the person (cms-communication
  `ARCHITECTURE.md`, п. 12). The deal card (`views/admin-cms-lead/_client.php`)
  shows the people (first 3), the phones and the emails (first 5) of the
  company, else of the person, with the deal as the `$owner` (the context of
  calls and letters). «Добавить телефон/email» opens the create window of the
  company or client phone/email controller, so the model checks repeats
  (`CrmDuplicateGuard`); «Добавить человека» is the company card's partial
  `views/admin-cms-company/_add-person.php`. The buttons need
  `CrmAccess::canEditClientData()` ([crm-access-levels.md](crm-access-levels.md)).
- The deal's own phones, emails and addresses are «…из обращения»: shown
  only while the deal has neither a company nor a person.

## Record source type of a package

`Skeeks::$recordSources` keeps one provider per source type. A package whose
source only creates its own records must not register a generic key such as
`lead` or `deal`: core or another package will need it for its own target
("сделка из лида"). Prefix the type with the package and put the kind into
the id: `proposal-client:lead.15`, `proposal-client:company.3`,
`proposal-client:deal.7` (`ProposalClientSource`, `supports()` only its own
model). `enforce()` sets the client again before validation, so the form may
hide the client fields when it is opened from a card.

## A client document whose sent version never changes

A document the client receives by a personal link (a proposal; the same
shape suits other sent documents) keeps its content in versions:

- The document row holds the client, the responsible employee, the status,
  the validity and `current_version_id`; texts, lines, totals and a copy of
  our requisites (as `ShopBill::fillContractorSnapshot()`) live in the
  version, lines in their own table with `sort`.
- A version gets `sent_at` at its first sending. From then on the model
  refuses any change except the attributes the sending itself stores (the
  PDF file id) in `beforeSave()`, and refuses `delete()`; line models check
  the version's `sent_at` in `beforeSave()`/`beforeDelete()`. A rule, not a
  form check: services, API tools and bots meet the same refusal.
- One service writes content for every caller: saving into an unsent
  current version edits it in place; saving while the current version is
  sent creates version N+1 and makes it current; "Новая версия" copies the
  current version on purpose. Feed entries are written by the service for a
  new version, never by the editing window.
- The first version is made in the document's `afterSave($insert)` with
  `transactions()` returning `OP_INSERT`, so a document never exists without
  content whichever way it was created (window, record source, API).
- The content editor is a hidden-model `BackendModelUpdateAction` tab with
  a form model only (`on initFormModels`, `isSaveFormModels = false`, the
  service in `on beforeSave`), see "Single-record domain operation forms" in
  [backend-model-controller.md](backend-model-controller.md). A tab of a
  card uses `'buttons' => ['apply']`: `save` is "save and close" and leaves
  the page.

Test: `skeeks/cms-proposal/tests/proposal.php` (SQLite memory: numbering per
site, first version from a template, editing in place, the locked sent
version, a new version on editing, feed entries of the client and its
company).

## Sending a ready document from the block

When a team sends the same few ready documents to many clients (shared
presentations of `cms-proposal`, verified 2026-10-23), give the block a
second full-width action next to "+ Создать …" that opens a small picker
window with the client's record source value (`?source=proposal-client:lead.29`):
the ready documents, the most sent first, a client-side search once the
list is longer than about seven, and per row what this client already got
("Отправляли …", "Открывал …"; sending again stays allowed). A row is a
plain link to `MailUi::sendUrl('<type>:<id>', $record)` inside the same
window, so "Отправить клиенту" opens with this client's contacts: two
clicks from the card. Without documents the picker links to creating one.

## The block of actions over the feed (`CrmCardActionsWidget`)

Lead (deal), company and client cards have one block of actions over the
feed, as "Дело / Комментарий / Задача / Ещё" of Bitrix24:
`skeeks\cms\widgets\admin\CrmCardActionsWidget` (`skeeks/cms`, verified
2026-10-28 on the skeeks.com stand: deals 880, 795, 670, company 3074,
client 1801, an administrator and a manager, light, dark, 375 px). A row of
action tabs, under it "Примечание": a one-line field that opens into
`CmsCommentWidget` with "Отправить" and "Отмена" (`isShowCancel`,
`data-sx-comment-cancel`). Do not add separate button rows or a separate
comment surface to a card.

- A card connects it with one call inside the pjax container of its feed (a
  sent note reloads the container and the block folds back):
  `CrmCardActionsWidget::widget(['model' => $model, 'commentUrl' => [...add-comment route...], 'actions' => $cardActions])`.
  `commentUrl => null` drops the note tab (no rights to write).
- The card's own tabs come first, in the order given (`actions`: id =>
  `label`, `icon` of `BackendIcon`, and either `url` of a backend window
  opened by `sx.classes.backend.widgets.Action` with extra `window` options,
  or `options` of a button — `onclick`, `data-*`, a delegated class such as
  `sx-telephony-btn`; `more => true` puts a tab into "Ещё"). The deal card
  gives "Задача", "Позвонить", "Письмо", "Не дозвонились"; company and client
  give "Задача" with the record preselected (`CmsTask[cms_company_id]` /
  `CmsTask[cms_user_id]`, not for an employee).
- A package adds its tab through the class-level
  `CrmCardBlocksWidget::EVENT_ACTIONS` (`CrmCardBlocksEvent::$actions`:
  id => `label`, `url` of a backend window, optional `icon`, optional
  `more => true`; `$priority` orders them, entries without a label or URL
  are dropped; `CrmCardBlocksWidget::actions($record)`). `$event->owner` is
  the card record (`CmsLead`, `CmsCompany`, `CmsUser` — check `is_worker`);
  the handler checks its own permissions and returns nothing otherwise
  (`packageActions => false` hides package tabs, e.g. for a read-only deal).
  A package checks `defined(CrmCardBlocksWidget::class.'::EVENT_ACTIONS')` so
  it still works with an older core. References: `cms-proposal`
  ("Отправить презентацию", "Отправить КП"), `cms-communication`
  ("Добавить в список рассылки").
- The row stays one line at any width: every row tab has a hidden copy in
  "Ещё", and `crm-card-actions.js` (`CrmCardActionsAsset`) moves the tabs
  that do not fit into the menu on load, resize and `pjax:end`; `rowLimit`
  (6) caps the row before measuring. "Ещё" is a native `<details>` closed by
  an outside click, a chosen item or Escape, with the semantic menu tokens.
  Tabs only open their action: the server side of each action does not
  change when it moves between the row and the menu.

## A create window that goes on to the next step

The ajax form of `BackendModelCreateAction` (ActiveFormAjaxBackend) closes
its window a second after saving, whatever button was pressed; in the pjax
path `submit-btn=save` closes it too. A window that should go on (a new КП
to its items, then to "Отправить клиенту") sets the next URL in the create
action's `on afterSave` (`$action->afterSaveUrl`, returned in the ajax
response) and replaces the form's `activeFormConfig.clientCallback`: on
success it triggers `model-create`/`model-update` on the opener with
`submitBtn: 'apply'` (the card still refreshes when the window closes) and
sets `window.location` to `data.afterSaveUrl`. An extra "Сохранить и …"
button in an update form sets `submit-btn` to its own value and submits the
form; the update action keeps that value in `data.submitBtn`. Reference:
`cms-proposal` `ProposalUi::nextStepFormCallback()` (verified 2026-10-23).

## A tab of a package in a core section

A package adds a tab to a core section (the leads section: "Канбан",
"Таблица", "Отчёт", then "Импорт" of `cms-communication`, verified
2026-10-23) without a core change: `BackendController::$actionsMap` is
merged into `actions()`, and controllers are created through the DI
container, so the package bootstrap (web only) extends the controller's
container definition, keeping whatever is there:

```php
$class = \skeeks\cms\controllers\AdminCmsLeadController::class;
$definition = \Yii::$container->getDefinitions()[$class] ?? [];
$definition['actionsMap']['comm-import'] = [
    'class' => \skeeks\cms\backend\ViewBackendAction::class,
    'name' => 'Импорт', 'icon' => 'fas fa-file-import', 'priority' => 28,
    'callback' => [\vendor\package\controllers\AdminImportController::class, 'renderLeadsTab'],
];
\Yii::$container->set($class, $definition);
```

Show the package's records in such a tab with its own standard grid, not a
hand-made table: create the package controller, give its `index` action the
tab's `configKey`/`backendShowingKey` through `actionsMap` (grid settings
are kept per tab), set the action's `url` to the tab's `urlData` and return
`$index->run()` — the way core shows another controller's list in a tab
(`AdminCmsCompanyController::documents()`; with a parent record
`BackendGridModelRelatedAction` does the same). Reference:
`cms-communication` `AdminCmsCommImportController::renderLeadsTab()`.

The tab inherits the section's permission (`cms/admin-lead`); the package
grid filters anything narrower. Rows, "create" and every next step open in
the standard backend window (`sx.classes.backend.widgets.Action` with
`isOpenNewWindow`; links built with `enableEmptyLayout()` only, never
`_sxb[noa]`); forms inside the window post and redirect with
`BackendUrlHelper::...->setBackendParamsByCurrentRequest()` so the next step
stays in the window; the window page triggers
`sx.Window.openerWidgetTriggerEvent('model-update', {submitBtn: 'apply'})`
and the list reloads when the window closes. Hide the package controller's
own `index`/`create` tabs (`isVisible => false`) so the window has no tab
bar. Prefer this to a separate menu item when the feature belongs to the
section.

## A bulk action of a package in a core section

A package adds a bulk action to a core section ("В список рассылки" of
"Компании", "Лиды", "Клиенты", `cms-communication`, verified 2026-10-24) the
same way as a tab: an `actionsMap` entry in the container definition of the
core controller with `BackendModelMultiWindowAction` (see "Bulk actions and
iframe mass editing" in [backend-model-controller.md](backend-model-controller.md)).
The marked rows and "для всех" (every row of the section's current filters,
collected by the grid) come for free; give the action
`modelsQueryCallback` with the section's own scope (`forManager()`,
`cmsSite()`), a `permissionName` of the package and a small form model of
the package (`formModel` may be a config array with the section name).

- A section whose default columns have no `checkbox` gets it through the
  same definition: `actionsMap['index']['grid']['visibleColumns'] = ['checkbox']`
  is merged before the controller's own list, so it comes first. Saved user
  column settings still win.
- A static callback of the form for the action fields must not be called
  `fields()`: it collides with `yii\base\Model::fields()` and the class fails
  to load (a 500 in the window; `php -l` does not see it).
- The built-in browser of the desktop app answers "no" to the native
  `confirm()` that "для всех" and `data-confirm` use; test those with
  `window.confirm` replaced.

## Counting a section's filter outside its page

A package that saves what an employee filtered in a core section (rules of
mailing lists, `cms-communication` `ListRules`, verified 2026-10-24: the
companies filter "category + Нет оплат" gave 1720 on the page, in a web
request and in a console job) counts it with the section's own code instead
of a filter builder of its own:

- create the section controller (`Yii::createObject($class, [$id, $module])`,
  also in a console application) and take `actions()['index']`;
- run its `grid['on init']` with an event whose sender has the
  `dataProvider` (a component with that property) — this is the section
  scope (`forManager()`, `cmsSite()`);
- build `DynamicConfigModel` from `filters['filtersModel']`, set the saved
  values, keep only explicit fields with `on apply`, create the fields with
  `skeeks\yii2\form\Builder` and trigger `apply` with a `QueryFiltersEvent`
  on each — what `QueryFiltersWidget::run()` does without rendering;
- filters with `HAVING` (`addSelect` of a count) keep their select: add the
  id column under an alias instead of replacing the select.

Read the values from the section page address: `f<widget id>[...]`
(`findex`) over the saved view `sx-bs` (`BackendShowing::config_jsoned`,
`FiltersWidget` → `filterValues`). Automatic column filters
(`StringFilterField`, `NumberFilterField`) have no `on apply` in the action
config: a field the section configures over a column without a class (lead
status, stage, source with `defaultMode`) gets the class the page would give
the column and applies itself (`FilterField`, value `{mode, value: [[...], end]}`);
columns the section does not configure are refused as saved rules.

Store the raw page values, never their meaning, so the section's current
code counts them. Before every count check that each key is still a field of
the section and each chosen option is still among its `items`: a section
silently ignores an unknown key or option, and the saved filter would widen.
Stop such a rule with the reason instead.

`forManager()` without an argument reads `Yii::$app->user`. Count on behalf
of the employee who saved the filter by setting a fresh
`yii\web\User` component (`enableSession = false`, `setIdentity()`) for the
duration and restoring the previous one — a fresh component has no access
cache of another employee, and a console application gets a user only for
that call.
