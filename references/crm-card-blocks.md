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

## Actions of the next step

The "Следующий шаг" block of the lead card has one menu "Действие": the core
"Поставить задачу" first, then actions of packages from the class-level
`CrmCardBlocksWidget::EVENT_ACTIONS` (`CrmCardBlocksEvent::$actions`:
id => `label`, `url` of a backend window, optional `icon` of `BackendIcon`;
`$priority` orders them; entries without a label or a URL are dropped;
`CrmCardBlocksWidget::actions($record)`). With no package actions the menu
stays the single "Поставить задачу" button. The menu is a native `<details>`
closed by an outside click or Escape, with the semantic menu tokens of the
lead kanban settings. Verified 2026-10-23 with `cms-proposal` ("Отправить
презентацию", "Отправить КП": pickers of ready documents with "+ Создать новое"). A
package checks `defined(CrmCardBlocksWidget::class.'::EVENT_ACTIONS')` so it
still works with an older core.

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
