# Client access to deals ("Сделка как услуга")

A deal is a service the client bought. What a person of the client (not an
employee) may do in it is checked by one core service, `Yii::$app->dealAccess`
(`skeeks\cms\deal\DealAccess`, skeeks/cms). Cabinets, bots, notifications, API
and AI tools use it instead of their own company/project rules. Employees keep
`forManager()` scopes; this service is only for client people.

## Model

- **Access areas** — what can be opened in a service: core `tasks` (levels 10
  view, 20 comment, 30 create tasks), `reports`, `documents`, `billing` (10),
  `manage` (10 view, 20 manage). Levels are integers: 0 none, 10, 20, 30.
  Never call them "sections" (site tree) or deal types.
- **Grant** `cms_client_access`: person + company + deal, or deal empty = all
  deals of the company (including future ones). Areas and levels are stored as
  rows (`cms_client_access_area`), copied from the chosen role; `is_full` means
  every area including areas added later; `is_custom` marks levels changed by
  hand. `scope_key` (`c:<company>` / `d:<deal>`) makes (person, scope) unique.
- **Roles** `cms_deal_role` (+ `_area`): editable ready-made sets (full,
  accountant, marketer, viewer seeded by migration, codes stable). Editing a role
  does not change existing grants.
- **Openness**: a deal type may open areas up to a level (`cms_deal_type_area`),
  a deal may narrow an area (`cms_deal_area`; no row — as the type). A type
  without rows opens only the core areas; an area of a package
  (`openByDefault` false unless it says otherwise) opens only where a type row
  allows it, so a new possibility of a package never reaches clients by itself.
- **Effective level** = max(owner of a deal without a company → full;
  deal grant; company-wide grant) limited by openness.
- The contact (`cms_deal.cms_user_id`) of a **company** deal gets nothing by
  itself; the owner of a deal **without** a company has full access without a
  row. When such a deal is given to a company, `CmsDeal::afterSave` gives the
  former owner a full grant on it.
- `CmsCompany2user::is_root` equals a company-wide full grant; both models keep
  it in sync (`skipAccessSync` / `skipRootSync` stop the echo) with validated
  saves inside the model transaction (`transactions()` OP_ALL), so the flag and
  the grant are written together or not at all. After any save or delete of a
  grant `CmsClientAccess::syncRoot()` recounts `is_root` for the previous and
  the current company (a company-wide full grant narrowed to a deal clears it). `scope_key` is computed again
  in `beforeSave()`: `save(false)` cannot write a grant without it. Removing a
  person from a company removes all their grants on its deals. Code that still
  reads `is_root` keeps working; new code reads the service.
- **A deal does not silently change its client.** `CmsDeal::validateClientChange`
  refuses changing the company (or company → person) while the deal has client
  grants or tasks of the previous company, with a message saying what to clean
  up. Writers that skip validation hit the safety net in `afterSave()`: grants
  of the previous company's people on that deal are removed and the deal feed
  says so. Personal → company keeps the owner a full grant.

## Service API

```php
$da = \Yii::$app->dealAccess;
$da->can($user, $deal, DealAccess::AREA_BILLING);              // one record
$da->level($user, $deal, 'tasks'); $da->levels($user, $deal);   // effective levels
$da->dealsQuery($user, 'tasks', DealAccess::LEVEL_FULL);        // lists, counts
$da->dealCondition($user, 'billing', 10, 'deal');               // inside other queries
$da->usersQuery($deal, 'billing');                              // recipients
CmsDeal::find()->forClient($user, 'billing', 10);               // same condition
CmsDeal::find()->forClient($user);                              // no area: FULL access only (legacy screens)
\Yii::$app->user->can(CmsDealAreaRule::PERMISSION, ['deal' => $deal, 'area' => 'billing']);
```

**`forClient()` without an area means full access** (owner, full grant —
`DealAccess::fullCondition()`), exactly the boundary older screens were
written for (is_root, deal contact). Screens of other packages that still call
it (GPD, analytics, promotion, the cabinet home) therefore never widen to a
limited role; a screen opens to limited roles only when it asks for its area.
`dealsQuery($user)` without an area (any grant) is for "my services" lists.

Every answer is one SQL condition (EXISTS subqueries, no joins), so a card, a
list, a count and recipients agree and lists never duplicate deals. RBAC only
gives the standard entry point: permission `cms/deal-area` with
`CmsDealAreaRule` (role `user`) calls `can()`; `rbac/init` creates it.

## Extension points

Packages register through configuration of the `dealAccess` component; the
core never names a package:

```php
'dealAccess' => [
    'areas' => ['chat' => ['name' => 'Чат по услуге', 'levels' => [10 => 'Читать', 20 => 'Писать'], 'priority' => 600]],
    'resourceProviders' => ['hosting_site' => ['class' => HostingDealResourceProvider::class, /* ... */]],
],
```

- A new area appears in role and grant forms (`DealAccessLevelsWidget`) by
  itself; full roles include it at once.
- A **resource provider** (`DealResourceProviderInterface`: `label`,
  `findByDeal`, `findOne`, `move`) answers from the package's own `cms_deal_id`;
  the core stores no copy of the link. A resource carries two links, one per
  cabinet: `url` — the employee card, an **absolute** admin route
  (`['/hosting/admin-vps/update', 'pk' => 1]`; a relative route resolves against
  the current module and breaks), opened in the standard backend window;
  `clientUrl` — the same service in the client cabinet, turned into a link by
  `DealAccess::clientUrl()` with the UPA prefix from any context (empty when the
  client has no page for it). `DealAccess::moveResource()` moves a
  resource in a transaction and writes the move into the feeds of both deals.
  Tasks and composition lines of the resource stay in the previous deal as its
  history with their resource link (validation checks the link only when it is
  set or changed); new links are made only within the resource's deal.
  Verified provider: `skeeks/cms-hosting` (site, VPS, SSL, DNS zone; only the
  service's own deal counts).

## Tasks of a deal

`cms_task.cms_deal_id` (+ optional `deal_resource_type/id`, `is_internal`).
`CmsTask::validateDeal` fills the deal's company (or the owner client of a
personal deal) and refuses a deal of another company; the resource must belong
to the deal. Projects stay for internal work.

`CmsTaskQuery::forClient($user, $level)`: an internal task is never shown,
its author included. A task **with** a deal is decided only by the rights in
that deal (`tasks` open and granted at `$level`) — authorship or project
participation does not bypass them, which matters for tasks moved from
projects. Only tasks **without** a deal keep the old rules (own tasks and,
temporarily, projects where the person takes part). A task linked only to a
company or a client, without a deal, is internal work and is never shown. The UPA support controller uses this scope; a
client comments with `tasks` ≥ 20 and creates tasks only in deals with
`tasks` = 30.

## Code before migrations

Shared code may be installed before its migration runs:
`CmsDealQuery::dealAccessReady()` and `CmsTaskQuery::hasDealColumn()` keep the
previous rules until the tables exist. The migration that creates the tables
also carries existing access over (is_root → company-wide full grant, contact
of a company deal → full grant on it), so no client loses access between the
code update and later data steps.

## Moving projects into deals

`php yii cms/deal-access/migrate-projects` prints a plan (nothing changes);
`--map=<project>:<deal>` decides ambiguous projects; `--apply` sets the deal on
the project's tasks (the project stays) and gives project participants a
grant with the marketer role. Each project is moved in its own transaction
together with its journal row in `cms_deal_project_migration` (run key, task
and grant ids), so a later failure never leaves moved data without an undo
record; the JSON file is only an export. `cms/deal-access/revert --run=<key>`
undoes a run: tasks lose the deal only if they still point at it; each grant
is journaled with a snapshot (`grantSnapshot()`: person, scope, role, full,
custom, levels, comment) and removed only while it still matches — a change
in the same second is caught too; changed ones are kept and reported; a
reverted run is not undone twice.

## Managing access in CRM cards

Managers set all of this in the cards (skeeks/cms, verified 2026-10-26 on the
skeeks.com stand: promotion and hosting deals, light, dark, 375 px). Every
screen reads the service; none counts rights by itself.

- **Openness.** The deal type form has «Настроить, что можно открыть клиенту»
  (`CmsDealType::$clientAreasCustom` / `$clientAreaLevels`, saved by
  `DealAccess::saveTypeOpenLevels()`: null removes the settings; an array stores
  every area, closed ones as 0, so a type can close everything). The deal card
  window `admin-cms-deal/client-open` only narrows the type
  (`saveDealOpenLevels()` stores rows below the type and writes the change into
  the deal feed). `openInfo($deal)` answers per area: type level, deal level,
  open level and what limits it (`type` / `deal`).
- **Effective rights with their origin.** `DealAccess::explain($deal)` gives
  every person of the deal (owner, deal and company-wide grants) the level per
  area, the granted level, the source (`owner`, `company`, `deal`), the grant
  and the limit that cut it — in PHP from `dealGrants()` and `openInfo()`, no
  query per area and level. It must equal `levels()`; the test checks it for
  every person and deal. A covering full access (owner, company-wide full
  grant) is named as the origin; covered deal grants are shown as «не действует».
- **Deal card «Доступ и уведомления»**: chips of what is open, a person × area
  table with the origin under each level (on a phone a card per person), the
  grants of each person (click edits the grant window), channels per person.
  The «Состав» block (lines of `cms_deal_item`) is postponed: its interface
  waits on branch `deal-composition` of `skeeks/cms` with
  `docs/deal-composition.md`; the table and the model stay released.
  Every window that takes its deal or company from the request
  (`?CmsClientAccess[cms_deal_id]=`, `?CmsClientAccess[cms_company_id]=`,
  `?deal=`) checks `forManager()` right
  after loading it, before reading people or building the form,
  and again before saving: the form itself would show a foreign client's data.
  The link of a person to a company (`admin-cms-company2user`, its `is_root` is
  full access) opens only for who may `cms/admin-company/manage` that company,
  and its company and person are not changed by the form.
  **«Ресурсы»**: «Перенести в
  другую сделку» is `admin-cms-deal/move-resource?type=&id=` — deals of the same
  client only, through `moveResource()`.
- **Company card «Люди клиента и доступы»**: the role in the company per person
  (full / company-wide role / chosen services); a click opens
  `admin-cms-company/client-person?user=` (role in the company, every active
  deal with what is given and «Выдать роль», channels). Adding a person asks
  for access first and defaults to «Только выбранные услуги» with chosen deals
  and a role: the database default `is_root = 1` must never decide it. The
  person, the link and the grants are one transaction; when a chosen deal no
  longer passes the filter (company, `forManager()`), nothing is saved and the
  employee is told — never a person with less access than asked. Lists of
  deals in these windows have no silent limit.
- Presentation is `ClientAccessAsset` (`client-access.css`), registered only by
  these views; semantic `--sx-color-*` tokens.

## Channels of a client person and the place of subscriptions

`ClientChannelsWidget` (core) shows «Каналы связи» in the client card (clients
only) and as compact icons in the deal table and the person window. Core lists
emails and phones (keys `email:<lower case>`, `phone:<digits>`, confirmed or
not); packages add channels or notes through the class-level
`ClientChannelsWidget::EVENT_CHANNELS` (`ClientChannelsEvent::$channels`: kind,
icon, label, value, state ok/warning/danger/off, notes, priority) and never
make the core know them. Verified providers: `cms-communication`
`helpers\ClientChannels` (bounces, complaints, opt-out of advertising, a row
per active bot channel of the site: `ok` only with a private chat of the
person with an active bot — found by the person's messages in it, as
`ChatIdentities::privateChats()`; a profile known only from a group is
`warning`, the bot cannot write to it personally) and `cms-mobile` `MobilePush::channelRow()` (installed, notifications
allowed). Guard the subscription with `defined(...::EVENT_CHANNELS)` for an
older core.

Client notification subscriptions live in `DealAccessNotifyEvent::EVENT_NOTIFY`
(class-level, `collect($deal, explain())`): a notification package fills
`cells` (user id => HTML of the «Уведомления» column) and `footer`; without a
handler the block shows a quiet note. The core keeps no subscription data.

Test: `skeeks/cms/tests/deal-client-access.php` (Composer autoload, SQLite memory);
the channels and subscription places — `tests/crm-extension-points.php`.
