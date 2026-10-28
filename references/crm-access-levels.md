# CRM: the main responsible, three levels and «Работать с этим клиентом»

Owner's decision 28.10.2026 (cms-communication `ARCHITECTURE.md`, П1/П2
пп. 10–11). Code: `skeeks\cms\crm\CrmAccess`, `CrmAccessRequests`,
`CrmClientTransfer`, `LeadTriage::workWithClient()`; migration
`m261028_140000__alter_table__crm_main_responsible`.

## The link of an employee to a company or a client

`cms_company2manager` / `cms_user2manager` (`CrmAccess::linkTable()`):

- `access_level` — `deal` «Свои сделки» (the card and everything of his deals
  in it), `edit` «Свои сделки и данные компании» (plus phones, emails, contact
  persons, addresses, requisites), `full` «Полный». The former `sales` became
  `edit`. Not safe for forms (`levelRules()`, "!"); `grantLink()` only raises
  (`levelRank()`), the access page sets (`setLevel()`).
- `is_main` — exactly one link is the **main responsible**, always `full`:
  approves requests, gets the notifications about the client, is listed first
  (`companyResponsibleIds()` / `clientResponsibleIds()` order by it,
  `mainResponsibleId()`).
- The levels are enforced (П2а+, `CrmAccess::$enforceLevels = true`;
  `false` only gives a site the old "every level is full" back).
  `forManager()` still decides visibility; the level decides what may be
  changed.

## What each level may do (П2а+)

The level comes from `CrmAccess::clientDataLevel($user, $code, $id)`:
- null — the client is not visible;
- `full` — an administrator, or a leader who sees the client through a
  subordinate's link;
- otherwise the employee's own link. A person without an own link takes the
  best level of the employee's links to the person's companies (people of a
  company are its data).

| | `deal` «Свои сделки» | `edit` | `full`, main, leader, admin |
| --- | --- | --- | --- |
| The card: data, people, phones, emails; call, write | yes | yes | yes |
| The client feed: the client's own records (calls, letters, chats, comments), his deals' records and their tasks | yes | yes | yes |
| The feed records of other deals, services, projects, bills, payments, tasks not tied to his deals | no | yes | yes |
| Pin a comment of the company or the person | no | yes | yes |
| Own deals: bills, payments, documents, proposals, the deal's own requisites and links (`cms_lead_*`) | yes | yes | yes |
| Other deals of the company and their proposals | no | yes | yes |
| Add a new phone or email: from his own deal only for `deal` (the deal contact block, contacts copied from a deal, API create with `cms_lead_id`) | own deal | yes | yes |
| Change the company/person data: name, form, requisites, existing phones and emails, addresses, links, people | no | yes | yes |
| Responsible employees and levels (the `managers` form field, «Доступ к клиенту») | no | no | main, leader, admin (`canManageAccess()`) |
| Deleting the company or the person | no | no | yes |

One place decides, everything uses it:
- `canEditClientData($user, $code, $id)` (level ≥ `edit`);
- `canEditClientRecord($user, $model)` takes a company, a person or a row with
  `cms_company_id` / `cms_user_id` (`clientRecordOf()`). It is true for
  employees and the user himself: they are not client data;
- `canAddClientContact($user, $model, $leadId)` — a new phone or email:
  `edit` and above anywhere; `deal` only from a deal of this client he works on
  (`$leadId`: `?lead_id=` of the deal contact block, `LeadTriage::mayCopyTo()`,
  `cms_lead_id` of the API). The company card never offers it to `deal`, a
  post from the card without the deal is 403. The phone and email controllers
  say so by `clientDataAddable()` of `ClientDataGuard` (create only);
- `dealOnlyCompanyIds($user)` — companies where the user's link is `deal` and
  no subordinate works on them: `CmsLeadQuery::forManager()` takes no company
  evidence from them (only own deals by the executor), so `canSeeLead()`, the
  «Сделки» tab and every list follow; `CmsProposalQuery::forManager()`
  (`cms-proposal`) opens a proposal of a company or a person only when its
  `cms_lead_id` is empty or visible. Bills, payments and documents carry no
  `cms_lead_id` and are not hidden;
- `feedCondition($user, $record)` — one condition for the feeds of a company
  or a person (`getCardLogs()` of the card, `getCompanyLogs()`/`getUserLogs()`,
  `getLogs()` of «Активность» add it for the current employee): for `deal` the
  records of the client itself (`model_code` of the company or the person —
  incoming calls, letters, chat messages, comments are written there), of his
  own deals and of the tasks with `cms_lead_id` of his deals; null — no limit
  (`edit`, `full`, the main responsible, a leader see the entries of all the
  client's deals in the card feed — `activity-feed.md`);
- `canPinLog($user, $log)` (`AdminCmsLogController::togglePin()`): a deal
  comment — `CmsLead::canBeWorkedBy()`, a company or person comment —
  `canEditClientRecord()`, other records keep their rules;
- controllers of the rows (company and person phones, emails, addresses,
  links, people of a company) use the trait
  `controllers\traits\ClientDataGuard`: `guardClientData($actions)` adds the
  check to `create`/`update`/`delete`/`delete-multi`. For `create` it checks
  both the owner column in the query and in the posted form;
- `AdminCmsCompanyController` and `AdminUserController`:
  - `update`, `update-eav`, `add-user`, `add-contractor`, `client-person`,
    activate/deactivate by `canEditClientRecord()`;
  - `delete` by the level `full`;
  - the `managers` field is unset and its POST refused without
    `canManageAccess()`;
- requisites attach (`Admin*ContractorController::canAttach()`) and their
  `update`/`delete` in `AdminCmsContractorController::canEditContractor()`:
  refused when the user works on one of the owners with `deal` only (owners he
  does not see do not block);
- REST/MCP (`cms-mcp` `CmsCompanyService`): `assertCanEditClient()` for
  `cms_company_update`, `cms_company_category_replace`, phone/email update,
  address and link create/update; `assertCanAddContact()` for phone/email
  create; `manager_ids` needs `canManageAccess()`. The refusal names the level
  and what it allows;
- `LeadTriage::mayCopyTo()` (contacts of a deal to the card) and linking a
  person to an existing company;
- cards: `$canEditData` hides the «⋮» menus and the «Добавить …» buttons; the
  deal contact block (`_client.php`) hides them by `canEditClientData()`.

Notifications of client events reach the main responsible as well (no
duplicates, the employee's settings of the kind decide the channels):
- a letter linked to the client (`cms_comm_link`) — `MailNotifier::mainResponsibleOf()`
  adds him as a "linked" candidate (only when he can open the letter, no
  mailbox access needed), kind `mail.new`;
- a chat message — `ChatNotifier`: an assigned chat notifies the assignee and
  the main responsible of the linked client (if he can read it); an unassigned
  one already notified all responsible employees, kind `chat.message`;
- a finished incoming call (answered by another employee or missed) —
  `crm\CrmClientCallNotifier` from `CmsTelephonyCall::afterSave()`, kind
  `crm.client_call` («Клиенты и предложения»), the company (else the person) as
  the link; never breaks the call save.

Not covered yet: tasks and calls of other employees' deals in the company
tabs; `cms_user_update` and `cms_contractor_*` of the API.

## Keeping one main

Keeping one main (`ensureMain()`): a new or deleted link model
(`afterSave`/`afterDelete`) gives a record without a main the author's link
(`created_by`), else the earliest one. The managers field of the company and
client forms rewrites the links (`RelationalBehavior` deletes and inserts):
`CrmAccess::keepLinks()` (attached in `CmsCompany::init()` and `User::init()`,
skipped for employees) snapshots the levels and the main before the update and
restores them after, then calls `ensureMain()`. `setMain()` grants the link
first and only then moves the mark — a grant of a link triggers `ensureMain()`,
which would otherwise pick the former main again.

## A client of another main responsible

A client is "foreign" when `forManager()` hides it (no link, not a client of a
subordinate). Matches show «Работать с этим клиентом» and, for a strong match,
«Это другая компания» (`views/admin-cms-lead/_identity-foreign-actions.php`,
shared by the deal form dropdown, its check and the create forms of a company
and a client):

- default — the deal is the employee's at once (`LeadTriage::applyIdentity()`:
  executor = actor), linked to the client, the employee gets `deal`
  (`grantLink()` in `applyIdentity()`). No request (owner's decision
  28.10.2026): the main responsible and his leaders get the notification
  `lead.delegated` «Имя: работа с клиентом «…» по своей сделке» with the deal as
  the link; the level is raised in «Доступ». A `work` request remains only for
  a client «по согласованию» (below) and for an incoming request of such a
  client.
- The protection of the client (П2д, `crm\CrmProtection`, migration
  `m261028_170000`) replaces the hand flag: `cms_company`/`cms_user`
  `crm_protection` (`free` | `protected` | `always`), `crm_protected_until`,
  `crm_protection_reason`.
  - `protected` — an incoming payment (`shop_payment.is_debit = 1`,
    `COALESCE(operation_at, created_at)`) within `crmSettings::$protectionMonths`
    (12) or an active `CmsDeal` (`is_active`, `end_at` empty or ahead);
    `always` — by hand, never expires; else `free`. An open deal of another
    employee never protects.
  - Recalculated on `ShopPayment` and `CmsDeal` saves (events in
    `Skeeks::bootstrap()`, no cms-shop change) and daily by the agent
    `cms/crm/protection` (`recalcMany()`; `--all` for every client with
    payments or services). The migration made the old flag `always` and
    calculated the rest.
  - The first incoming payment of a client without a main responsible makes
    the creator of its bill (else who registered the payment) the main one
    (`onPayment()`): a paying client always has one.
  - Nobody but an explicit choice makes a main responsible: `ensureMain()`
    promotes only a link that already is `full` (never raises `deal`/`edit`);
    approving a request about a client without a main makes the approver the
    main one (`CrmAccessRequests::apply()`), the requester gets exactly the
    level chosen. Protected clients without a main: `php yii
    cms/crm/assign-main` (the plan; `--apply` writes) — the creator of the
    bill of the last payment, who registered it, the creator of the active
    service, else the first administrator (`CrmProtection::mainPlan()`).
  - `approvalRequired()` = not `free`; `setApprovalRequired()` is the
    «Авто / Всегда по согласованию» switch (`setAlways()`; the old column
    follows for a rollback). `describe()` — «Под защитой до …: последний
    платёж …», «…: действующая услуга №…».
  - The general pool: a `free` client without a main responsible (no links).
    `canTakeIntoWork()`, a match shows `canTake` → «Взять в работу» (the deal
    form, the duplicate search, sorting): `applyIdentity()` makes the actor
    the main responsible (`setMain()`, `full`), no request, no notification;
    the next one gets «Работать с этим клиентом». «Вернуть в общий пул»
    (`returnToPool()`, `canManageAccess()`, refused for a protected client)
    removes every link.
  - «Доступ» shows the state, the switch and «Вернуть в общий пул»; the
    company and client lists filter «Защита: …» and «Общий пул»
    (`responsibleFilterItems()`).
  - The pool is seen by every CRM employee (owner's decision 28.10.2026):
    `CrmAccess::inPool()` (no links), `canSeePool()` (the right to the
    companies or clients section). One visibility of the lists, the card search
    and the API — `CrmAccess::applyClientVisibility()` →
    `CmsCompanyQuery::forManagerOrPool()` / `CmsUserQuery::forManagerOrPool()`
    (a client person of the pool is a non-employee without links). Never use it
    for deals, services, bills, payments, documents or projects: they keep
    `forManager()`, so a pool company shows none of them. The card `view`
    action allows `canSeePool()`; the other tabs keep the manage right and are
    hidden; the data is read only (`canEditClientRecord()` is false), the feed is
    the client's own records (`feedCondition()`), `views/crm/_pool-banner.php`
    at the top offers «Взять в работу» (action `crm-take`,
    `ClientTransferActions::clientTake()` → `setMain()`), then the card opens
    in full. A protected client without employees shows the state; a leader
    or an administrator assigns it. The match search keeps the old visibility
    (a pool company is `visible: false`, `canTake: true`).
- A protected client (formerly «Доступ только по согласованию»;
  `approvalRequired()`, shown in a match
  as `byApproval`) — a deal by hand is refused (`checkIdentity()`,
  `applyIdentity()`); the button sends only the request
  (`workWithClient()`, actions `admin-cms-lead/work-request` and
  `duplicate-request` of the create forms); the approval creates the deal for
  the employee (`CrmAccessRequests::createLead()`). An incoming request of the
  client does not wait: sorting (`accept()`, `$incoming`) gives the deal to the
  main responsible and sends the request; the approval hands the deal over.
- One kind of request: `work` («Работать с этим клиентом»).
  `CrmAccessRequests::request()` refuses any other kind. The old kinds
  `delegate` («Сделка себе») and `access` («Доступ к клиенту») are history
  only (`CmsCrmAccessRequest::getIsLegacy()`):
  - they are never decided (`canDecideRequest()`, `decide()`);
  - they are not counted or listed in «Ждут моего решения»
    (`requestsCondition()` takes `kind = work`);
  - their cards are read only.
- Migration `m261028_150000` (`CrmAccessRequests::closeLegacy()`, also on
  production) closed the pending ones as `obsolete` («Устарел»):
  - each sender was notified how access is given now. The client is the
    link only when the sender sees it; otherwise it is named in the text.
    Visibility is checked by the links of the client, not `forManager()`,
    which needs the web user in the console;
  - the deciders' bell notifications about them were marked read.
  `safeDown` does nothing.
- Request and notification texts carry no grammatical gender: «Имя: запрос на
  работу с клиентом «…»», «Имя: работа с клиентом «…» по своей сделке, запрос
  уровня доступа» — no «ему/ей», «просил/просила».

Deciders (`deciderIds()`): the main responsible and his leaders, else the
`full` employees, else administrators.

## Pages

- Access management lives in one tab «Доступ» of the company and client
  cards (`ClientTransferActions::clientAccess()`, `views/crm/access.php`,
  action `crm-access`):
  - the main responsible with the leaders who see the client, and the
    «Передать клиента» block — `setMain($new, LEVEL_EDIT)`: the former main
    stays with `edit` and may be removed; other links stay; open deals and
    tasks move except the kept ones. The `transfer` action is hidden
    (`isVisible = false`): a GET of its old address redirects to
    `crm-access`, the POST of the block goes to it;
  - who works with the client: the level in words and where the access comes
    from («по своей сделке» — `deal` of an executor of a client's deal,
    «выдан вручную»); the level select and «Убрать»;
  - «+ Добавить сотрудника» — the `crm-access-add` action in the standard
    backend window (small, `enableNoModelActions()`): the employee, the level
    as three cards with `CrmAccess::levelHints()` («Свои сделки» by default),
    «Добавить» / «Отмена»; on success the window closes and the card reloads.
    An existing link is changed in the list;
  - pending `work` requests about the client, links to their cards;
  - «Доступ только по согласованию».
  Changed by the main responsible, his leaders and administrators
  (`canManageAccess()`); others see it read only with their own level
  («Ваш доступ к клиенту: …»). The tab shows where the client has employees or
  to who may transfer it.
- One row and one add button for the tab and the card: `views/crm/_access-row.php`
  (mode `tab` — the level select and «Убрать»; mode `card` — a cross on hover)
  and `views/crm/_access-add-button.php`; the rows come from
  `CrmAccess::accessRows()` (`accessList()` + where the access comes from).
- The profile block «Работают с компанией» / «Работают с клиентом»
  (`views/crm/_access-block.php`): the main responsible first («Основной»),
  each employee with the level in words; who manages the access adds and
  removes (the main responsible is never removed, only transferred); others
  read only; «Настроить доступ →» opens «Доступ».
- The company and client forms have no employees, people or requisites fields
  (people and contacts — «Профиль», employees — «Доступ», requisites — their
  tab): a form that does not post a relation never rewrites it
  (`RelationalBehavior` saves only populated relations; `keepLinks()` guards
  the rest). A new company or client gets the author as the responsible in
  `updateFields()`; imports and the API write the links directly.
- «Запросы доступа» — a section of its own in «Клиенты и сделки», after
  «Клиенты» (`AdminCmsCrmAccessRequestController`, the deals right
  `cms/admin-lead`): the menu item for administrators, main responsible
  employees, leaders and senders (`canUseRequests()`) with the counter of
  pending requests the user decides (`pendingRequestsCount()`, the menu
  `badge`); the list «Ждут моего решения» (default) / «Мои запросы» / «Все»
  (leaders, administrators) by `requestsCondition()` — the same rule as
  `isRequestDecider()` in SQL (the main link of the record belongs to the user
  or his subordinate, or `owner_id`); the card (`view`) in the backend window:
  who asks and the level, «Одобрить» with `deal`/`edit`/`full» or «Отказать»,
  the client with its employees and open deals, the deal, the decision. Seen by
  the deciders and the sender only (`canSeeRequest()`, otherwise 403); no
  create/delete; `update` only redirects to the card (a record link opened in a
  new tab falls back to it). The old `admin-cms-lead/requests` redirects here.
- A request notification: the client in the text, the request itself as the
  bold link (`model` = `CmsCrmAccessRequest`, registered in `modelsConfig`), so
  the bell opens its card in the window and the short link of mail and MAX
  (`CmsWebNotify::getOpenUrl()`) leads to `view`. Companies, contacts,
  employees and deals in the list and the card are `BackendEntityLink`s (an
  employee — `admin-worker` when the user may open it, else `admin-user`).
- Lists: «Мои клиенты (основной)» and «Без ответственного»
  (`responsibleFilterItems()`, `applyResponsibleFilter()`), the main
  responsible first in the «Сотрудники» column of companies.

## Pitfall: `data-confirm` on a button inside a form

`yii.js` handles any clickable element with `data-confirm`: after the
confirmation a button inside a form submits that form. A button that sends its
own AJAX inside a create form (here — it created empty deals) must use another
attribute (`data-sx-confirm`) and confirm itself.
