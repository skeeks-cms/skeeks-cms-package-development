# CRM duplicates, sorting of «Неразобранное» and deal assignment

Verified in `skeeks/cms` 6.4.9.75 (П2а, 2026-10-27), `skeeks/cms-communication`
and `skeeks/cms-mcp`. Words: «сделка» = `CmsLead`, «услуга» = `CmsDeal`
(`crm-entity-names.md`).

## One duplicate check: `skeeks\cms\crm\CrmMatcher`

Component `crmMatcher` (`CrmMatcher::instance()` without it). Every place that
asks "is this client already in the CRM" uses it: the deal form, sorting,
imports (`cms-communication` `CrmDuplicates` is a thin wrapper keeping its
"first match" import contract), API (`cms_company_duplicate_check`,
`cms_lead_duplicate_check`). Do not write another lookup by phone or email.

- `rawFind($input)` searches the whole base of the site, including records the
  employee cannot see: companies, people (not employees), deals (open only by
  default, `leadStatuses = null` for every deal). Input: `phones`, `emails`,
  `inns`, `domains`, `name`, `siteId`, `excludeLeadIds`, `types`.
- Every candidate has all its evidence (`reasons`: by, value, strong). Email,
  phone (significant digits, 8/+7) and INN are strong; a normalized name
  (quotes and legal forms removed) and a domain of a site or of a company
  email are weak hints. Public mail domains (`publicMailDomains`) are never a
  company domain.
- It never links or merges anything. Names, domains and a single contact are
  hints for a person.
- `findMany($inputs, $siteId)` for batches (phones of the base read once);
  `rememberLeadPhones()` for deals made during the batch.
- `repeatsOf($candidates, $siteId)` — open deals of the found clients: the
  candidates for "the same request". A person decides: the same need — add to
  the open deal, a new need — a new deal.
- `present($candidates, $viewer)` gives what the employee may see:
  `CrmAccess::presentCandidate()`.

A repeated **event** (the same form, webhook, reply) is not a duplicate check:
it is `source_type + source_ref` in `CmsLeadService::createFromSource()`, which
follows `merged_into_id` (`CmsLead::resolveMerged()`), so a repeated event of a
request added to another deal lands in that deal.

## Hidden matches

A record the employee does not see (`forManager()`) is shown only as "такая
компания уже есть: «название», ответственный …"; a private person — without the
name, also in requests and their notifications
(`CrmAccessRequests::recordName($request, $viewerId)`). No id, contacts,
letters or amounts. The form gets an opaque reference
(`CrmAccess::candidateRef()`: authenticated encryption of Yii Security with a
random IV, valid one day) to act on it ("Это он", "Это другая компания");
`parseCandidateRef()` refuses forged and old ones. A signed but readable
"company:15:…" is not enough: never put a hidden id into HTML or API output.

## A strong match is never saved silently

A phone, an email or an INN that another company already has is saved only
with «Почему это другая компания» (`crm\CrmDuplicateGuard`, owner's decision
28.10.2026); the pair goes to `cms_crm_duplicate_mark`. No unique index: one
owner of two legal entities may share them. The check lives in the models
`CmsCompanyPhone` and `CmsCompanyEmail` (`duplicate_reason`,
`validateCrmDuplicate()`), so every entry point passes it — the company
create form (its phone and email fields; the reason field appears with a
strong match; the company and its contacts are one transaction), the phone
and email forms of a company card, the deal form (`duplicate_reason` of
«Это другая компания»), copying a contact of a request, imports and the API
(`cms_company_create_full` with `duplicate_reason`, child create tools). An
INN: `AdminCmsCompanyContractorController::attach()` with `CrmDuplicate[reason]`.
People: `cms_user_phone`/`cms_user_email` are already unique per site. A
public mail domain is not a match, the exact address is. A row is checked
when it is new, its value changed or its company changed (a contact moved to
a third company through a form or the API update tools), never against its
own row (`companiesWith(…, $exceptRowId)`). Existing duplicates
and merging are a later step.

### «Повторы клиентов»: strict or free

amoCRM and Bitrix24 let a second company with the same phone be saved, so
the site chooses (`crm\CrmSettings`, component `crmSettings` in
`config/common.php`; «Настройки → Компоненты» and «Повторы клиентов» in the
settings menu of the deals kanban, the component settings window of the
site; administrators only; kept per site, never per user —
`$_overridePath` is default + site):

- strict (`DUPLICATES_STRICT`, the default, also for a site that never saved
  the settings or has no `crmSettings`) — the rules above;
- free (`DUPLICATES_FREE`) — the same hints, «Использовать» and requests, the
  reason field stays but is optional («… (необязательно)»), the row is saved
  without a reason and the pair is still marked in `cms_crm_duplicate_mark`
  (for a later «Объединить»).

Only the guard decides: `CrmDuplicateGuard::refuses($reason)` (strict and
empty), `isStrict()`, `reasonLabel()`; callers never compare the reason
themselves — the phone and email models, the INN link, `LeadIdentityForm`
(«Это другая компания» requires a reason only when strict) and the API
(`cms-mcp` `companyCreateFull`, with a `method_exists` fallback for an older
core). Imports match existing records first in both modes (`AudienceImport`
uses `CrmMatcher` and creates a card only without a strong match). Hiding a
client of another responsible does not depend on the mode. Tests switch the
mode with `CrmDuplicateGuard::$mode` and reset it to `null`.

## Rules verified by the review (2026-10-27)

- Server fields of sorting (`triage_result`, `merged_into_id`, `triaged_*`)
  and link levels (`access_level`) are "!" in rules: validated, never loaded
  by a form; only the services write them.
- Rules of columns added by a migration are added only when the column exists
  (`CmsLead::triageReady()`, `CrmAccess::levelsReady()`): a validator reads
  the attribute before its `when`, so `when => hasAttribute()` does not protect.
- Every path that copies a contact of the request to a client card uses one
  check (`LeadTriage::mayCopyTo()`: the card is visible or created in the same
  operation); "Клиент сделки" with `contacts` is such a path.
- A request is decided or cancelled once: the pending row is taken by a
  conditional `UPDATE … WHERE status = pending` (inside the transaction of a
  decision), never by the loaded copy.
- The create window commits its transaction only when the whole save went
  through, including a record source's `created()` (a class-level `afterSave`
  handler attached last sets the flag); a PJAX error message rolls back.
  Contacts of a new person are validated by their models before the save.
- «Передать клиента» moves the tasks of the client and of its open deals; a
  kept deal keeps its tasks.

## Employee rights: `skeeks\cms\crm\CrmAccess`

Component `crmAccess`; controllers, views and packages ask it instead of
repeating checks:

- `canTriage($user, $lead)` — an unsorted deal (status `new`) is sorted by an
  employee who sees it (the evidence rule of `CmsLeadQuery::forManager()`, the
  queue of «Входящие заявки» below), its given employee when it has one,
  otherwise by an administrator; no separate permission.
- `canWorkLead`, `canAssignLead` (who works on a deal, `isLeadWorker()` in
  crm-access-levels.md; for an unsorted deal — whoever sorts it),
  `canSeeCompany`, `canSeeClient`, `canSeeLead`.
- `defaultExecutorId($companyId, $clientId, $actorId)` — the responsible of
  the client (the actor when he is one of them, else the first by
  `cms_company2manager.id`), else the actor.
- `companyBrief()` — the whole company or only "название + ответственный".
- Requests: `canDecideRequest()` — the main responsible of the client, his
  leaders, administrators; `requestDeciderIds()`.
- Leaders of an employee are `CmsDepartment::leaderIdsOf()`, the inverse of
  `subordinateIds()` on the same tree (`model-access-queries.md`).

SQL visibility of lists stays in the `forManager()` scopes for now; they are
meant to move into `CrmAccess` with grants (`cms_crm_grant`) later.

### Levels, the main responsible, a client of another employee

Since 28.10.2026: three levels `deal` / `edit` / `full`, exactly one main
responsible per company and client, «Работать с этим клиентом» instead of
«Запросить сделку себе» / «Запросить доступ», «Доступ только по
согласованию», «Доступ к клиенту» in the card, «Передать клиента» as a change
of the main responsible. The whole contract: `crm-access-levels.md`.

## Sorting and the client of a deal: `skeeks\cms\crm\LeadTriage`

One service for the admin windows, tests and future API:

- `accept()` — take into work with a found or created company and person
  (`LeadIdentityForm`), or without a client; the person is linked to the
  company (`CmsCompany2user`), gets no password or invitation; the contacts of
  the request stay with the deal («Из обращения») and go to the card only by
  choice (`contacts`, `copyContact()`).
- `merge()` — the request continues an open deal: contacts and open tasks go
  there, the request closes (`status=rejected`, `triage_result=merged`,
  `merged_into_id`). A partner request goes only into a deal of the same
  partner (the reward must not move).
- `support()` — a task on the client's service (or without one), the deal
  closes as `support`; `spam()` — closes as `spam`.
- `createLead()` = `prepareNewLead()` + save + `finishNewLead()`; a client of
  another responsible (`foreign`) gives the deal to that responsible, the actor
  stays the initiator (`created_by`) and by default requests the deal for
  himself, or access to the client, or only hands it over
  (`CrmAccessRequests`). "Это другая компания" creates the record and a
  `cms_crm_duplicate_mark` with the reason.
- `assign()` — give the deal to another employee; the company stays with its
  responsible, who is notified (`lead.delegated`).

Sorted deals are closed without being lost sales: `triage_result` is excluded
from the report and from the kanban "Отказ"; employees see the result
(`staffStatusName`, `CmsLeadStatusWidget` with `showStage`), partners still see
`statuses()`. Reopening clears `triage_result` and `merged_into_id`.

After commit the service fires `CmsLead::EVENT_TRIAGED` on the deal and the
application event `crm.leadTriaged` (`TriageEvent`: lead, result, target,
task, actorId). Packages bind the application event in their config
(`'on crm.leadTriaged' => [...]`), so it works without a web bootstrap;
`cms-communication` `TriageLinks` links letters and chats of the request to
the target deal or service.

## «Неразобранное»: two buttons and «Входящие заявки»

Owner's decision 29.10.2026 ([platform-plan `crm/links-contract.md`](https://github.com/skeeks-cms/platform-plan/blob/main/crm/links-contract.md) п. 13б; as
«Принять / Отклонить» of amoCRM plus the queue of Bitrix24).

- The header of an unsorted deal (`views/admin-cms-lead/_model_header.php`)
  has two buttons: «Взять в работу» (primary) and «Не продажа ▾» (a `details`
  menu: «Спам» posts `op=spam`, «Уже есть сделка» and «Поддержка» open the
  window in `mode=merge|support`). No «Разобрать», and no status badge for
  `new` — the stage bar shows «Неразобранное».
- «Взять в работу» is the one window of `LeadTriageActions::triage()`
  (`views/admin-cms-lead/triage.php`, `mode=take`): the client
  (`_identity` with the found candidates; a single strong visible company or
  contact is already chosen — `preselectCandidate()`; another company; a new
  client), the responsible is the actor (`LeadIdentityForm::$executor_id`),
  the stage is the first one (`CmsLead` gives a deal in work without a stage
  its first stage), one save = `LeadTriage::accept()`. Opened from the card
  (`from=card`) the window closes and the card reloads; from elsewhere the
  window goes on to the card (`data-sx-lead-open-card`, `lead-identity.js`).
- «Найдены совпадения» of the card (`_matches.php`, `canTake` =
  `canTriage()`): one line per match on the shared rows of the CRM matches
  (`sx-lead-hints`/`sx-lead-hint` of `lead-identity.css`, registered by the
  card header for every status since the block comes by AJAX): an icon of the
  kind, the name, `№id`, the main responsible, what matched, «возможный дубль»
  when an open `cms_crm_duplicate_mark` joins two found records, and «Это он»
  (an unsorted deal: a plain form to `triage` → `triagePlain()`, the same
  `accept()`, the responsible is the actor) or «Привязать» (a deal in work:
  `link-identity`); three lines, the rest under «ещё N». Two records with one
  name are duplicates of the base, not a doubled output; nothing merges them
  here.
- «Входящие заявки» — `crm\InboundQueue`. The queue belongs to the pipeline
  the deal is in (`CmsLeadPipeline`, its form: «Заявки из источников»
  `inbound_source_types`, «Кто разбирает» `inbound_worker_ids` /
  `inbound_department_ids`, «Как распределять» `inbound_mode`; JSON columns
  behind `inboundSourceTypes`, `inboundWorkerIds`, `inboundDepartmentIds`,
  `inboundReady()`, migration `m261029_220000`); a pipeline without its own
  employees takes the CRM settings (`CrmSettings::$inboundWorkerIds`,
  `$inboundDepartmentIds`, `$inboundMode`; «Входящие заявки» also in the
  kanban settings menu), its own mode still applies. `queueOf($pipeline)` /
  `queueOfLead($lead)` give `key` (the turn), `members`, `mode`; the
  visibility, the notifications and the turn read the pipeline the deal is in
  now, so a moved deal belongs to the queue of the new pipeline. A new
  request without a chosen pipeline lands in the first pipeline whose sources
  include its `source_type` (`CmsLeadPipeline::forSource()`, used by
  `CmsLead::preparePipeline()` and `InboundQueue::route()`), else in the
  default one. `CrmSettings::$inboundToMain` stays a CRM-wide switch.
  - no employee and no department anywhere — nothing changes (the base site);
  - members are active employees with `cms/admin-lead` (`memberIds()`,
    departments through `CmsDepartment2worker`, the turn by id);
  - `MODE_ALL` «видят все из списка»: `CmsLeadQuery::forManager()` adds the
    unsorted deals nobody was given of the pipelines whose queue the employee
    is in (`unsortedCondition()`, `CmsLeadPipeline::leadCondition()`), so
    `canTriage()` follows; an employee outside the queue keeps only the
    evidence rule;
  - `MODE_ROUND` «по кругу»: `CmsLeadService::createFromSource()` calls
    `route()` inside its transaction before the save; `next()` takes the turn
    from `cms_crm_inbound_rotation` (one row per site and queue,
    `INSERT IGNORE` + `SELECT … FOR UPDATE`; the queue key is `default` or
    `pipeline:<id>`; the row is created beforehand by its own short
    connection — `ensureRow()`, a clone of `db` in autocommit — so the
    deal's transaction takes only the exclusive lock of an existing row: an
    `INSERT IGNORE` inside it upgraded a shared lock and deadlocked 12
    parallel requests, release audit F3), so two requests at one moment go
    to two employees; the deal stays `new` with its executor (only he and his
    leaders see and sort it);
  - `$inboundToMain` «если клиент найден — сразу его основному»: `mainOf()`
    over the strong `CrmMatcher` candidates (a person of a company counts by
    the company's main one); one main responsible with the deals right → the
    deal is `in_work` with him, the single found company and person are
    linked, `afterRoute()` grants the level by `grantLeadExecutorLevel()`
    (journal) and writes the feed; several main responsibles → stays
    unsorted;
  - «Новая сделка в Неразобранном» (kind `lead.new`): with a queue —
    `InboundQueue::notifyIds()`: the queue (`MODE_ALL`), the given employee,
    the administrators with the deals right (their notification settings
    decide); without — the old `CmsLead::availableManagerIds()`. A deal given
    by turns tells its employee «Новая сделка в Неразобранном», not «Вам
    назначена».
  - A console process may keep an old copy of the CRM settings in its own
    file cache (a key without the site); after changing them in a test,
    invalidate the tag `CrmSettings::class` (`TagDependency::invalidate()`).
- `CmsLeadService::createFromSource()` repeats the whole creation (the
  repeat check, the route, the deal, the notifications) up to
  `LOCK_ATTEMPTS` times when MySQL rolls it back for 1213/1205/40001
  (`isLockConflict()`); inside a caller's transaction it never repeats. The
  notifications are queued inside the transaction (cms-communication
  `NotifyDeliveries`), so a repeat sends nothing twice.
- Tests: `cms/tests/inbound-queue-stand.php` (rolled back),
  `cms/tests/inbound-queue-race.php` (two processes, a separate test queue),
  `cms/tests/inbound-queue-load.php` (12+ processes through the whole
  `createFromSource()`, a new and an existing queue, a rolled back
  participant, one event twice, the delivery detached).

## Search fields and a pasted line (as in amoCRM and Bitrix24)

The client of a deal is chosen by two fields, «Компания» above «Контакт», not
by mode switches, as in Bitrix24 (`views/admin-cms-lead/_identity.php`):

- typing opens a dropdown (`_identity-dropdown.php`, the `match` action):
  own records are taken by a click; a client of another main responsible
  offers «Работать с этим клиентом» (`crm-access-levels.md`) and, for a
  strong match, «Это другая компания»; the last row is always «+ Создать новую компанию «…»»,
  kept under the scrolled list;
- a choice turns the field into its title with a badge («новая», «у другого
  сотрудника») and ⇄ to change it; an empty field is "no company";
- a new record keeps the field as its name and opens «Телефон» and «E-mail»;
  its name, phone and email are checked while typed (`match` with `check=1`,
  `_identity-check.php`): the popover «Есть совпадения: N» to the right of the
  fields (under them in a narrow window), each record once, in the group of
  its strongest evidence; an own record shows its contacts with the matched
  one in bold and «Использовать», a hidden one only the typed matched value,
  «Работать с этим клиентом» and «Это другая компания». The save is still refused by `CrmDuplicateGuard` without
  a reason;
- «Ответственный» is the last section of the deal form
  (`_identity-executor.php`), by default the employee who creates the deal.

One line pasted from a table or a site («Ромашка, +7 …, info@…,
sales@…») is split by `crm\CrmContactParser::parse()` (separators: comma,
semicolon, tab, line break; email, phone, INN, site, the first text is the
name, the next one the person) and searched by all its parts at once
(`CrmMatcher::searchLine()`, which adds the viewer's own records by a part of
the name). «Создать» puts the parts into the fields of the new company
(`company_phones`, `company_emails`) and, by choice, the contact. With a
strong match the new record is saved only through «Это другая компания» with
a reason (the server refuses otherwise). The create forms of a company and a client
get no separate search field (the owner found it too complex): their usual
fields — name, phone, email (a company gets «Телефон» and «Email» right after
the name) — are watched, and «уже есть, ответственный …» appears under them by
itself (`controllers\traits\CrmDuplicateCheckActions`, `views/crm/_duplicate-check.php`);
a line pasted into the name is split into the empty fields. A match with a
client of another main responsible offers there «Работать с этим клиентом» by
the opaque reference (`duplicate-request` action of the trait,
`LeadTriage::workWithClient()`); the reason field stays for «это другая
компания».
Words in the interface are those of amoCRM and Bitrix24: «Сделка»,
«Компания», «Контакт», «Неразобранное», «Сумма», «Ответственный»,
«Сменить ответственного»; never internal ones.

The deal name is optional (`CmsLead::$autoName`, `LeadTriage::suggestName()`:
«Компания — источник», else «Сделка №<id>»). «+ Сделка» is on the kanban toolbar.

Two amounts, one on the screen at a time (`CmsLead::getShownAmount()`):
`amount` — «Сумма», the expected amount (m261028_120000), shown for an open
or rejected deal in its forms, card, kanban and list; `deal_amount` —
«Сумма сделки», the result: required in the success form (prefilled from
«Сумма»), shown for a successful deal, with «ожидали …»
(`getExpectedAmountNote()`) when they differ. A rejected deal keeps «Сумма»
and never gets «Сумма сделки». Reports and API: amount is expected,
deal_amount is the result; never add them up.

## The deal form keeps the standard create window

«Новая сделка» is the standard `BackendModelCreateAction` (the drawer) with the
block "Кто это" as an `HtmlBlock` field and `LeadIdentityForm` as a second
form model. Order matters because record sources (`record-sources.md`) fill
the deal in `beforeValidate` after the controller and link it in `afterSave`:

1. `beforeValidate` — reset protected attributes, `LeadTriage::checkIdentity()`;
   a refusal is appended as a validator error of the deal name (validation
   clears errors, so add a validator, not an error) with `skipOnEmpty => false`:
   the name is optional, and a skipped refusal reaches the save as an error 500.
2. `beforeSave` — open the transaction, `prepareNewLead()`; commit on
   `Application::EVENT_AFTER_ACTION`, roll back on `EVENT_AFTER_REQUEST`.
3. `afterSave` — `finishNewLead()` and `afterSaveUrl` (the kanban when the
   deal went to another responsible and the actor no longer sees it).

An `InlineValidator` closure must not be `static`: it is bound to the model.

After the save the card of the new deal opens at once, as in Bitrix24 (no
filled form left to be saved twice): the create action replaces
`activeFormConfig.clientCallback` (`LeadTriageActions::createFormCallback()`,
the pattern of `crm-card-blocks.md`, "A create window that goes on to the next
step"). The card is recognised by the path of `afterSaveUrl` (in a backend
window both URLs carry `_sxb[...]`); a window triggers `model-create` with
`submitBtn: 'apply'` and goes on to the card keeping its `_sxb` layout (the
kanban refreshes when it is closed), a page follows `afterSaveUrl`. A deal
that went to the responsible of a foreign client (`afterSaveUrl` is the
kanban) closes the window as usual. The shared `ActiveFormAjaxBackend` is not
changed: other sections keep their behaviour.

## A simple site (the "base site" rule)

The core runs on every site; CRM packages only where needed. Every CRM change
in the core is checked on a site without `cms-communication` and without CRM
data (verified on the skeeks-platform.ru stand: form2, no communications, no
companies or managers):

- a form request comes as before: status `new`, the administrators with the
  deals right are notified (no «Входящие заявки» set), «Взять в работу» opens
  the one window (the client may stay empty), dropping it on a kanban stage
  takes it without a window unless the CRM knows its client (a strong match or
  a linked client);
- no new menu items; tabs that make sense only with CRM data hide themselves
  («Запросы доступа» without requests, «Передать клиента» without a responsible);
- the core never references package classes: packages join through events
  bound in their own config (`crm.leadTriaged`);
- migrations only add columns and tables; existing rows keep their behavior
  (`triage_result` null, link levels `full`).

## Incoming sources

- Form2 and partners go through `createFromSource()` into «Неразобранное».
- A reply to a campaign (`cms-communication` `CampaignTasks::onReply()`): a
  task inside the recipient's open deal (or the only open deal of the
  company/person), otherwise a deal in «Неразобранное» with source `mailing`,
  `source_ref = campaign-reply:<recipient>`, the reply and the campaign letter
  linked. Opening a letter, a link or an offer never creates a deal.
- Automatic mail linking (`AddressBook::findRecords()` without a viewer) does
  not spread a letter over several open deals of one address: it links the
  company and the person; the conversation's deal is inherited.
- Telephony keeps "one open deal with the number, otherwise none".
- `LeadImport` assigns through `save()` with `CmsLead::$suppressNotifications`
  (one notification per import), not `updateAttributes()`.

## Deals through REST/MCP (`cms_lead_*`)

Verified 2026-10-28 (`skeeks/cms-mcp` `services\CmsLeadService`,
`tests/crm-leads.php`). The API is one more entry of the same screen: it
calls the services above and never decides rights of its own.

- Tools: `cms_lead_list`, `cms_lead_get`, `cms_lead_stage_list`,
  `cms_lead_reject_reason_list` (scope `cms.company.read`); `cms_lead_create`,
  `cms_lead_update`, `cms_lead_move`, `cms_lead_close`, `cms_lead_triage`
  (`cms.company.write`); no delete. Deals are CRM clients' data, so they share
  the scopes of companies: existing tokens need no new grant.
- Visibility is `CmsLead::find()->forManager()`; a deal outside it is "not
  found". Rights are `CrmAccess` (`canWorkLead`, `canTriage`,
  `canAssignLead`) and the model (`canBeReopenedBy`, status transitions,
  required fields). A company the caller does not see comes as
  `companyBrief()`: name and responsible, `id: null`.
- Create = `LeadTriage::checkIdentity()` + `createLead()` with a
  `LeadIdentityForm` built from the arguments: an own record by id; a hidden
  one by the opaque `ref` of the duplicate check («Работать с этим
  клиентом»; a protected client goes through `workWithClient()`, which only
  sends the request); a new one from fields or a pasted line
  (`CrmContactParser`). Before a new client the server runs the duplicate
  check itself: a strong match returns `requires_confirmation` with the
  masked matches (choose the own record, the `ref`, or «Это другая
  компания» = `duplicate_of_ref` + `duplicate_reason`); open deals of the
  client need `confirm=true`. The duplicate checks return `ref` for every
  company and person match and accept `line`.
- A repeated event is `source_type + source_ref` (`CmsLeadQuery::fromSource()`
  + `resolveMerged()`), an `idempotency_key` without a source becomes
  `source_ref = api:<key>`; the unique index settles a parallel repeat.
- Stage and pipeline go through `CmsLeadBoardService::move()` and
  `LeadPipelines::move()`; the model writes `CmsLeadStageHistory`. An unsorted
  deal is sorted first (`cms_lead_triage`), as the kanban opens «Взять в
  работу» for a known client. Closing sets the same fields as the «Обработка» form;
  success requires an explicit `deal_amount` (never filled from `amount`
  silently).

Tests: `cms/tests/crm-lead-triage-stand.php` (stand database, rolled back),
`cms-communication/tests/triage-stand.php`, `tests/mailing.php`,
`cms-mcp/tests/crm-leads.php`.
