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
  changed and how much of the client is seen: only `full` sees the whole
  client (`CrmAccess::seesWholeClient()`, owner's decision 29.10.2026) —
  `edit` is «Свои сделки» plus changing the client data, nothing more.

## What each level may do (П2а+)

The level comes from `CrmAccess::clientDataLevel($user, $code, $id)`:
- null — the client is not visible;
- `full` — an administrator;
- otherwise the best of the employee's own link and his subordinates' links,
  never above it (owner's decision 29.10.2026): a leader whose subordinate is
  the main one or `full` gets `full` (and manages the access only when the
  subordinate is the main one, `canManageAccess()`), `edit` — `edit`, `deal`
  — `deal`, and his «своё» is then the work of the department (deals,
  bills… of him and his subordinates). A subordinate who came to a foreign
  client with his own deal never opens that client in full to the leader.
- A client person is equal to a company (owner's decision 29.10.2026), with
  two levels. His data (name, phones, emails, editing them) —
  `clientDataLevel('client', …)`: the best of the links of the employee and
  his subordinates to the person and to all the person's companies (people
  of a company are its data). His personal orders — services, deals, bills,
  payments, documents, proposals without a company (`cms_company_id` null) —
  `personalLevel()`: only the links to the person himself; a company link,
  even `full`, never opens them. `wholeClientLevel()` picks the right one for
  `seesWholeClient()` (feed, protection reason, MCP `cms_user_get`). In SQL:
  `clientRecordCondition()` opens a record without a company by a `full`
  person link only; `CmsLeadQuery::forManager()` opens a personal deal by the
  executor or a `full` person link, and takes the evidence of the person's
  companies (his contacts, submitter, partner) only for an incoming deal
  without an executor (the sorting). The protection of a person counts only
  his personal services and payments (`CrmProtection::compute()`): a payment
  of his company never protects him. A person without a company has his own
  main responsible, «Доступ» and the journal like a company (record code
  `client`); a person of an owned company is its data — seen through the
  company (`personCompanyLinks()`).
- The history of a client (owner's decision 29.10.2026):
  `CrmAccess::historyCondition()` / `hasHistory()` — a payment, a bill, a
  service or a deal ever, not an unsorted incoming deal (`STATUS_NEW`); of a
  person, his personal ones. A client with a history and no main one gets it by
  `CrmProtection::mainCandidate()` (the bill or the registration of the last
  payment, else the last service, else `HISTORY_FALLBACK_USER_ID`),
  `historyMainPlan()` / `cms/crm/assign-history-main`; «Работать с этим
  клиентом» assigns it first when missing (`assignHistoryMain()`, the deal
  being created is not counted as history), then the actor gets `deal`.

| | `deal` «Свои сделки» | `edit` | `full`, main, leader, admin |
| --- | --- | --- | --- |
| The card: data, people, phones, emails; call, write | yes | yes | yes |
| The client feed: the records of his deals (their letters, calls, chats, notes) and their tasks | yes | yes | yes |
| The records of the client itself: its letters, calls, chats not linked to his deal, comments and files of employees | no | no | yes |
| Act on the client itself (a note, a task, a proposal, a presentation, a letter, a call, a message without a deal) | no | no | yes |
| The feed records of other deals, services, projects, bills, payments, tasks not tied to his deals | no | no | yes |
| Pin a comment of the company or the person | no | yes | yes |
| Own deals: proposals, tasks, the deal's own requisites and links (`cms_lead_*`) | yes | yes | yes |
| Services, bills, payments, documents, tasks, proposals of the client: «своё» only (below) | own | own | all |
| Other deals of the company and their proposals | no | no | yes |
| Add a new phone or email: from his own deal only for `deal` (the deal contact block, contacts copied from a deal, API create with `cms_lead_id`) | own deal | yes | yes |
| Change the company/person data: name, form, requisites, existing phones and emails, addresses, links, people | no | yes | yes |
| Responsible employees and levels (the `managers` form field, «Доступ к клиенту») | no | no | main, leader, admin (`canManageAccess()`) |
| Deleting the company or the person | no | no | yes |

### Who works on a deal

`CrmAccess::isLeadWorker($user, $lead)` (owner's decision 29.10.2026) is the
only rule of working actions on a deal (`CmsLead`): an administrator, the
responsible (`executor_id`), his leaders (`CmsDepartment::leaderIdsOf()`), an
employee who sees the client of the deal in full (`seesWholeClient()` of
`wholeClientLevel()` for its company or person: his or a subordinate's `full`
link, the main responsible). `deal` and `edit` work only on their own deals; a
leader of a `deal` subordinate works on that subordinate's deals, not on other
deals of the company. Derived, never re-implemented:
- `canWorkLead()` (open deal) — `CmsLead::canBeWorkedBy()` delegates to it;
  the card actions (`canWorkWithModel()`), the stage bar and the board
  (`CmsLeadBoardService::denyReason()`, `LeadPipelines`), «Не дозвонились»,
  the client of the deal, lead contacts and requisites, SMS, pinning a deal
  comment, MCP `cms_lead_*`;
- `canReopenLead()` (closed deal) — `CmsLead::canBeReopenedBy()`;
- `canAssignLead()` — the same people (an unsorted deal — `canTriage()`); the
  «Ответственный» field of the edit form follows it, not the administrator role;
- a letter or a send in the context of a deal (`cms-communication`
  `ComposeForm::validateContext()`, `SendService::send()`) — `isLeadWorker()`.
`CmsLead::isManagedBy()` only says "he is the executor". The refusal text is
`CrmAccess::LEAD_WORK_DENIED`. Stand test: `cms/tests/crm-lead-worker-stand.php`.

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
- `seesWholeClient($level)` — the whole client (other deals, services,
  finances, tasks, proposals, the whole feed): `full` only. Every visibility
  decision uses it; `canEditClientData()` (≥ `edit`) decides changing data only;
- `dealOnlyCompanyIds($user)` — companies where the best link of the user and
  his subordinates is `deal` or `edit` (no `full` link among them): `CmsLeadQuery::forManager()` takes no company
  evidence from them (only own deals by the executor), so `canSeeLead()`, the
  «Сделки» tab and every list follow; `CmsProposalQuery::forManager()`
  (`cms-proposal`) opens a proposal of a company or a person only when its
  `cms_lead_id` is empty or visible;
- `clientRecordCondition($user, $table, $own)` — one visibility of the client
  records that are not deals (below, «Своё»); `ownerIds($user)` — he and his
  subordinates, the "himself" of «своё»;
- `feedCondition($user, $record)` — one condition for the feeds of a company
  or a person (`getCardLogs()` of the card, `getCompanyLogs()`/`getUserLogs()`,
  `getLogs()` of «Активность» add it for the current employee): for `deal` the
  records of the client itself (`model_code` of the company or the person —
  incoming calls, letters, chat messages are written there; not the comments
  of employees, owner's decision 29.10.2026), of his
  own deals and of the tasks with `cms_lead_id` of his deals — the same for
  `edit`; null — no limit (`full`, the main responsible, a leader through a
  `full` subordinate see the entries of all the
  client's deals in the card feed — `activity-feed.md`). The client's own
  records exclude entries another record wrote there (a proposal of the
  company — `ProposalFeed`) unless the employee sees that record: a log type
  whose renderer implements `log\CmsLogFeedVisibilityInterface`
  (`feedVisibleCondition($user, $logTable)`, `ProposalLogRenderer` — the
  proposal id in `data` among `CmsProposalQuery::forManager()`) adds the
  condition for `deal`/`edit` and for a client without the main one seen by a
  head;
- `canPinLog($user, $log)` (`AdminCmsLogController::togglePin()`): a deal
  comment — `CmsLead::canBeWorkedBy()`, a company or person comment —
  `canEditClientRecord()`, other records keep their rules;
- controllers of the rows (company and person phones, emails, addresses,
  links, people of a company) use the trait
  `controllers\traits\ClientDataGuard`: `guardClientData($actions)` adds the
  check to `create`/`update`/`delete`/`delete-multi`. For `create` it checks
  both the owner column in the query and in the posted form; `update` of
  an existing row refuses (403) a POST whose `cms_company_id`/`cms_user_id`
  differs from the stored owner (`clientDataOwnerChanged()`, release audit
  F03: a hidden field moved a phone to a foreign company). A row never moves
  between clients through a form; `tests/crm-client-data-owner-stand.php`
  checks it with a real GET + POST and CSRF;
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

### «Своё»: services, bills, payments, documents, tasks, proposals

Owner's decisions 28.10 and 29.10.2026 («Полноценная работа в рамках своих
сделок»). A service (`cms_deal`), a bill (`shop_bill`) and a document
(`shop_document`) are tied to a deal by `cms_lead_id` (migrations
`m261029_190000` of `skeeks/cms` and `cms-shop`, `CrmAccess::leadLinkReady()`
before them); a payment — through its bill (`shop_bill2payment`). Besides they
still link to services (`cms_deal2bill`, `cms_deal2payment`,
`shop_document2deal`). `forManager()` of
`CmsDealQuery`, `CmsTaskQuery` (`skeeks/cms`), `ShopBillQuery`,
`ShopPaymentQuery`, `ShopDocumentQuery` (`cms-shop`, through
`ShopBillQuery::clientCondition()` with a fallback for an older cms) and
`CmsProposalQuery` (`cms-proposal`) call
`CrmAccess::clientRecordCondition($user, $table, $own)`:
- the record of a company or a person is seen when the client is seen in
  full (`full`): the company is in `CmsCompany::forManager()` and not in
  `dealOnlyCompanyIds()`; the person has a link `full` of him or a
  subordinate, or he has no own link and the person belongs to such a
  company (`clientDataLevel()` in SQL);
- or it is «своё» (`$own`, OR):
  - a service — `created_by` of him or a subordinate (`ownerIds()`), or of his
    deal (`CrmAccess::ownLeadCondition()`: the executor of the deal among
    `ownerIds()` — a leader gets the deals of his subordinates);
  - a bill — `created_by`, or of his deal (`ShopBillQuery::ownLeadConditions()`);
  - a payment — `created_by`, or a payment of his bill (`shop_bill2payment`:
    the bill is his by `created_by` or by its deal);
  - a document — `created_by`, of his deal, or a document of his bill
    (`shop_document2bill`);
  - a task — the executor or the author, `cms_lead_id` of a visible deal, a
    visible project or service;
  - a proposal — `responsible_id` or `created_by`, a visible deal or service;
    a client proposal without a deal only with the client seen in full.
- Administrators: the scopes return before; levels off (`dealOnlyCompanyIds()`
  null) — the former rule "every visible client".

Everything follows the scopes: the sections and their CSV, the card tabs
(«Услуги», «Счета», «Документы», «Платежи», «Задачи» — the related grids apply
`forManager()`; the client tab «Счета» of `/shop/admin-bill` chains it in
`AdminUserController`), the mail linked to a service
(`CmsCommMessageQuery::linkedToVisibleCondition`), the «Услуга» select of the
task and proposal forms, the task RBAC rule (`CmsTaskRule`).

Writing such a record (audit F1/F2/F6, 29.10.2026). The models check who the
record belongs to by `CrmAccess::validateLeadLink($model)` (`CmsDeal`,
`ShopBill`, `ShopDocument` — `validateLead()`, before the "client required"
rules; `ShopPayment` — `validateClient()`) when the record is new or ANY field
of its belonging changes (the deal, the company, the person):
- the client must be available to the acting employee
  (`clientDataLevel()` not null; an administrator — any). No access is never a
  permission: `CLIENT_NOT_AVAILABLE`;
- the deal must be in his `CmsLeadQuery::forManager()` (`LEAD_NOT_AVAILABLE`)
  and of the same client (`LEAD_OTHER_CLIENT`; moving only the company while
  keeping the deal is refused); a record without a client takes the client of
  the deal;
- a closed deal gets no new records from anyone, administrators and `full`
  included (`LEAD_CLOSED`); reading and editing its existing records stay,
  a reopened deal takes records again;
- a client seen in part (`deal`, `edit`, a leader through such a subordinate)
  only within his own open deal (`canActOnClient()`, `OWN_LEAD_REQUIRED`); a
  payment (no `cms_lead_id`, tied to a deal through its bill) — only the
  available client.
The refusals never name the hidden client or deal, and
`forgetHiddenBelonging()` returns the model to its previous belonging, so a form
drawn again with the error and any serialization never show the hidden name.
Only an acting employee is checked (`actingWorker()`): the console, customers
of the site and system work wrapped in `CrmAccess::systemWrite(fn)` (the bill of
an order by its payment system, `PaysystemHandler::createShopBill()`) are not.
`CmsDeal::createBill()` gives the bill the deal of its service. Regressions:
`cms/tests/crm-lead-links-write-stand.php` (company/person and moves between
them × none/deal/edit/full/leader/admin × no/own/other/closed deal, through
`cmsMcp->executeTool` and the form with a CSRF token; after a refusal — no row,
no journal event, no hidden name) and `crm-leak-audit.php`. The forms
take the field from `crm\LeadLinkField::config($model)` (open deals of the
chosen client, the linked one stays) and `applyLeadClient($model)` fills the
client when the form is opened from a deal (`<Form>[cms_lead_id]=<id>`). The
deal card shows what is tied to it in the tabs «Услуги», «Счета», «Платежи»,
«Документы» (`AdminCmsLeadController`: related grids of the sections with their
own scopes; «Добавить» passes `cms_lead_id`; payments through their bills).
REST/MCP: `cms_lead_id` is writable for `cms_deal_*`, `shop_bill_*`,
`shop_document_*` (the same validation) and a filter of their lists.

The new responsible of a deal — a change of the responsible
(`LeadTriage::assign()`, the window, the edit form, MCP `cms_lead_update`),
«Разобрать» (`accept()`), taking a deal (the card, the board) — gets `deal` to
the client of the deal when he has no link: `CrmAccess::grantLeadExecutorLevel()`
(an existing level, the main one and the links of others stay; administrators
get no link; journal event `grant` with `cms_lead_id`). It is called by these
flows, not in `CmsLead::afterSave()`: «Работать с этим клиентом» writes its own
journal event `work` after the deal is saved. The event of the journal «Доступы»
and of the client feed: the actor, the employee, `нет доступа → Свои сделки`,
the comment «Смена ответственного сделки №N» and `cms_lead_id` (the journal
links the deal, the feed entry gets it in `data` and `AccessLogRenderer` links
it). A responsible who already has a level gets no journal event (the deal's
own event is enough); the method returns false then.

Writing levels never fakes a change: `setLevel()` returns whether the level
changed — no link is an error, the same level again is a success without a
journal event (`ClientTransferActions` «Уровень доступа не изменился»).
`CrmAccessLog::levelChange($log)` is the one «было → стало» of the journal
table, the section «Доступы» and the feed text (a new link: «нет доступа → …»).
Stand test: `cms/tests/crm-lead-links-stand.php`.

Answers of rights within one web request (audit F10: a feed asked the same
questions hundreds of times). `CmsDepartment::tree()` reads the departments and
their employees once (`subordinateIds()`, `leaderIdsOf()`), dropped by
`resetTree()` on a save or delete of a department or its employee.
`CrmAccess::memo()` keeps `canSeeCompany()`, `canSeeClient()`, `linkLevel()`
and the employee by id; it is dropped when the signed-in identity changes, on
a save or delete of `CmsCompany2manager`, `CmsUser2manager`, `CmsCompany2user`,
a department or its employee (class events), and by `resetMemo()` in the
writers that bypass the models (`setMain()`, `setLevel()`, the managers field,
`setApprovalRequired()`). A new writer of links with `updateAll()` must call
`resetMemo()`. Nothing is kept across requests, and the console and workers
keep nothing (`CrmAccess::keepsAnswers()`; `$keepAnswersInConsole` is only for
`cms/tests/crm-access-memo-stand.php`). RBAC rules ask these methods instead of
their own queries (`CmsCompanyRule` → `canSeeCompany()`). A feed page loads the
records of its entries in one query per class (`CmsLog::preloadModels()` in
`log-list.php`; `getModel()`/`getSubModel()` read once per entry).

The cards: `controllers\traits\ManagerScopedModel` — `getModel()` loads `pk`
through `forManager()` of the model query and throws 404 otherwise; used by
`AdminCmsBillController`, `AdminCmsDocumentController`,
`AdminCmsDealController` (`skeeks/cms`) and `AdminPaymentController`
(`cms-shop`), so every model action (view, update, tabs, delete) of a hidden
record is 404, as for a hidden deal or proposal. A controller as a related
grid of another card keeps the default lookup (`pk` is the parent's).
Verified by `skeeks/cms/tests/crm-leak-audit.php`: two runs, the employee with
`deal` and with `edit` — the same places hidden, `edit` may open the company
edit form, `deal` may not. The reason of the protection (it names payments and
services) is shown in «Доступ» only to `full` and who manages the access.

Not covered yet: calls of other employees' deals in the company tabs;
`cms_user_update` and `cms_contractor_*` of the API; the REST/MCP history and
overview tools (`cms-mcp`).

## Keeping one main

Keeping one main (`ensureMain()`): a new or deleted link model
(`afterSave`/`afterDelete`) gives a record without a main one by the rule of
«Participants only with the main responsible» (below). The managers field of the company and
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
  - A client with participants always has a main one (`ensureMain()`, section
    «Participants only with the main responsible» below; it never raises the
    level of a `deal`/`edit` participant); approving a request about a client without a main makes the approver the
    main one (`CrmAccessRequests::apply()`), the requester gets exactly the
    level chosen. Protected clients without a main: `php yii
    cms/crm/assign-main` (the plan; `--apply` writes) — the creator of the
    bill of the last payment, who registered it, the creator of the active
    service, else the first administrator (`CrmProtection::mainPlan()`).
  - `approvalRequired()` = not `free`; `setApprovalRequired()` is the
    «Авто / Всегда по согласованию» switch (`setAlways()`; the old column
    follows for a rollback). `describe()` — «Под защитой до …: последний
    платёж …», «…: действующая услуга №…».
  - «Доступ» shows the state and the switch; the company and client lists
    filter «Защита: …» (`responsibleFilterItems()`).
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
- The journal of the access to clients (owner's decision 29.10.2026,
  `crm\CrmAccessLog`, table `cms_crm_access_log`, migration `m261029_130000`
  copied the old requests in): every event is written where it happens —
  `request`, `request_approved/declined/cancelled/obsolete`
  (`CrmAccessRequests`), `work` (`LeadTriage::applyIdentity()`; `take` and
  `pool` remain in the history of the former general pool), `grant`/`level`/
  `remove` («Доступ», the add window), `transfer` (`CrmClientTransfer`),
  `main_auto` (the creator, the first payment, `assign-main`, the approver of a
  client without a main, `ensureMain()`): who, to whom, the client, the level before → after, the
  ground (deal, payment, request), a comment. `CrmAccessLog::write()` never
  breaks the action and also writes a client feed record (log type
  `crm_access`, `log\AccessLogRenderer`). Who sees:
  `CrmAccessLog::visibleCondition()` — an administrator all; else the events
  where he or his subordinate is the actor or the employee, or the main
  responsible of the client. The list is the section «Доступы» of «Клиенты и
  сделки» (`AdminCmsCrmAccessLogController`, a `BackendModelStandartController`
  on `CmsCrmAccessLog`: standard columns, filters, sorting, export) — not a
  hand-made page or a tab of «Компании» (the owner rejected that). Its saved
  public tabs «Ждут моего решения» (`is_default`, filter `scope=waiting` →
  `waitingCondition()`) and «Все» are `backend_showing` rows made by migration
  `m261029_150000` for every site; filters event, employee, client, dates. The
  menu item carries the counter of requests waiting for my decision and opens
  the default tab; `pageHeader.description` says how many wait. A pending
  request is decided in its row («Одобрить» / «Отказать» post
  `admin-cms-crm-access-request/decide`, `lead-identity.js`), never one's own
  request; an event opens the request card (`view` in the window), else the
  client. The history of a client is at the bottom of its «Доступ». Old
  addresses (`admin-cms-company/crm-access-log`,
  `admin-cms-crm-access-request/index`) redirect to the section.
- Who decides a request (`deciderIds()`, `requestsCondition(REQUESTS_TO_ME)`,
  the counter and the notification): the current main responsible of the
  client — without one, its `full` employees — their leaders and an
  administrator. Never the requester's leader: the request's `owner_id` (the
  main one when it was sent, possibly the requester himself) does not count.
  The requester's leader reads the subordinate's events in «Все» only, with
  no buttons, and a client he does not see is «клиент из базы»
  (`AdminCmsCrmAccessLogController::clientLink()`).
- The company and client lists share the CRM part (owner's decision
  29.10.2026): `crm\CrmClientGrid` with the record code (`company` for
  `AdminCmsCompanyController`, `client` for `AdminUserController`) — merge
  `filterFields($code, $table)` into the filter fields and `filterNames()` into
  `visibleFilters`, `managersColumn($code)` as the column `managers`, and
  `bulkActions($code)` into `actions()`; never copy them into a controller.
  - Filters: `crm_responsible` (`my_main`, `no_main`, `no_responsible` of
    `CrmAccess::responsibleFilterItems()`), `crm_protection`, `crm_main_user`
    and `crm_employee` (`CrmAccess::applyEmployeeFilter()`, main only or any
    link), `crm_created` (`DaterangeInputWidget::parseRange()`).
  - The column «Ответственный и сотрудники» (`managersCell()`): the main one
    first, marked «Ответственный», then the employees by level with the level
    name; empty for an employee's own record and for a person without links
    (a self-registered user of the site, п. 14).
  - Bulk actions «Сменить ответственного», «Добавить сотрудника», «Сменить
    защиту» (`crm-bulk-main/grant/protection`): `crm\CrmBulkAccessAction`
    (a `BackendModelMultiWindowAction`, `recordCode`) with
    `crm\CrmBulkAccessForm::apply()` for either record; the window shows
    «Будет изменено компаний/клиентов: N» (`selectedCount()`);
    `accessCallback` — an administrator or a leader; `eachAccessCallback` —
    `canManageAccess()` per record, an employee in the client list refused (a
    refused one refuses the whole selection); the selection query is narrowed
    by `applyClientVisibility()`. Every change is journaled
    (`main_auto`/`transfer`, `grant`/`level`, `protection`; the card's
    protection switch writes `protection` too). The grid lists only multi
    actions whose `isAllow` is true (cms-backend
    `BackendGridModelAction::_initMultiActions()`), so a forbidden bulk button
    is not shown at all.
  - A saved view (`backend_showing`, the default one too) keeps its own
    `visibleColumns` and `visibleFilters`: a new column or filter of a list
    needs a migration adding it to the views already saved
    (`m261029_140000` companies, `m261029_180000` clients — the column only to a
    view that shows the name, not to a one-column export view).
- (Before the journal) «Запросы доступа» — a section of its own in «Клиенты и сделки», after
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
- Leaks closed after the audit (`tests/crm-leak-audit.php`): «Доступ» of a
  card opens only to who sees the client (`canSeeCompany()` /
  `canSeeClient()` in `clientAccessAction()`), not merely because the client
  has links; the protection reason (it names payments and services,
  `CrmProtection::describe(..., $withReason)`) is shown only to who manages
  the access or has "edit"/"full". `AjaxSelectModel` names a selected value
  only within its `searchQuery` (the default `valueCallback` reuses it), so
  a field whose search is limited by `forManager()` never names a hidden
  record put into the form address — give every CRM select a limited
  `searchQuery`, not a bare `find()`.

## The shield of protection and the protection of new clients

Owner's decision 29.10.2026:

- A protected client stands out everywhere, a free one shows nothing: one
  helper `CrmProtection::shield($record, $withReason)` /
  `shieldFor($record)` (the reason — payments, services — only to who sees
  the whole client or manages the access) / `shieldIcon($state, $text)` for
  a match (`presentCandidate()` carries `protection`, `protectionText`). An
  icon `fa-shield-alt` with the hint «Под защитой до …: …» or «Всегда по
  согласованию: …», asset `CrmShieldAsset` (registered by the helper;
  `LeadIdentityAsset` depends on it for the AJAX matches). Used at the name in
  the company and client lists, in the card headers (`titleSuffix` of the
  standard `BackendModelHeader`: the company controller sets `modelHeader`,
  the client — `views/admin-user/_model_header.php`), in the duplicate search
  and the company choice of the deal form. Never a text badge «Свободная».
- The employee setting «Новые компании и клиенты, которые я создаю, —
  „Всегда по согласованию“» — `cms_user.crm_new_always` (migration
  `m261029_160000`), switched in «Клиенты в CRM» of the profile by him, his
  head or an administrator (`canEditCrmSettings()`).
  `CrmProtection::onCreated()` on `EVENT_AFTER_INSERT` of a company or a
  person (attached in `CrmAccess::keepLinks()`): the creator — `created_by`,
  else the web user — decides, not the main responsible he assigns; so the
  forms, the deal form, an import, REST and MCP all follow. Journal
  `protection`.

## «Свои сделки» and "edit" work only through their deals

Owner's decision 29.10.2026 (it replaced «the client's own records are
seen»). "edit" differs from `deal` only by changing the client data (phones,
emails, people, addresses, site, requisites).

- One rule of «his deals»: `CrmAccess::ownLeads($user, $record, $openOnly)` —
  the deals `forManager()` shows him, of this company or person, whose
  executor is he or his subordinate (`ownerIds()`); `ownOpenLeads()` — new or
  in work. The feed, the server checks and the card actions block
  (`CrmCardActionsWidget::ownDeals()`) use it.
- The feed (`feedCondition()`): `deal`/`edit` see only the records of their
  deals and of the tasks of those deals — nothing of the client itself. A
  person of a company the employee has in `full` also shows the person's own
  records (the person is the company's data); his personal orders stay by
  `personalLevel()`. A client without the main one (old data, seen by a head
  of a department) shows the client's own communications without comments of
  employees (`clientOwnFeedCondition()`). The card feed drops the
  deal copy of a letter or a call that also lies on the client ("twin",
  `CmsLogQuery::clientDealsCondition()`) only for a whole-client viewer: in a
  limited feed the client copy is hidden, so the deal copy must stay.
- Server checks — `CrmAccess::canActOnClient($user, $record, $leadId)`
  (whole client, or `$leadId` among `ownOpenLeads()`), the message
  `CrmAccess::OWN_LEAD_REQUIRED`; `clientActionError($companyId, $clientId,
  $leadId)` checks the acting employee (`actingWorker()`: a web or API
  session; the console, receiving and agents are not limited):
  - `CmsLog` rule: a comment on a company or a person;
  - `CmsTask` rule: a task of a client without his open deal (a task of a
    service or a project follows those);
  - cms-proposal: `CmsProposal::validateClient()` and `CmsProposalSend` — a
    proposal or a presentation of the company or the person itself;
  - cms-communication: `ComposeForm::validateContext()` (a letter from the
    client card) and `SendService::send()` (SMS, chat, e-mail sends of a
    record);
  - `TelephonyController::actionCall()` →
    `CmsLeadTelephonyService::outgoingCallError()`: a phone of a client he
    sees only by his deals is called only with `lead_id` of his open deal; the
    call attaches to that deal also by a phone of its client.
  The API goes through the same models and services.
- Incoming (owner's decision 29.10.2026) — `crm\CrmInbound` (component
  `crmInbound`), one rule for letters, site messages, private chat messages
  and calls. The usual linking runs first: the deal's own e-mail or phone
  (one such open deal), the records of the page, and a reply in a
  conversation keeps the links of the earlier letters
  (`MessageLinker::inheritFromConversation()`, it wins). A record still
  without a deal goes by the clients it was matched to by their card
  contacts (`route()`): one open deal of these clients — the record lies in
  it too (a copy on the client and one on the deal); several — no guess, the
  executors are notified once per event (kind `crm.inbound_unlinked`,
  category «Клиенты», `eventKey` per record, per conversation for a chat,
  `collapse`) with the page `admin-cms-lead/attach-inbound?type=&id=`
  («Привязать к сделке №…», only his and his subordinates' open deals of the
  client, `leadsFor()`/`attach()`); none — the client only (the main one and
  `full` see it). Kinds of records are attachers
  (`CrmInboundAttacherInterface`: find, clients, isLinked, attach, describe)
  in `crmInbound.attachers`: `call` in the core (`CrmInboundCallAttacher`,
  from `CmsLeadTelephonyService::attachToLead()`, incoming calls), `message`
  in cms-communication (`services\InboundLeads`, from `MailIngestService`,
  `SiteMessages::receive()`, `ChatFeed::onMessage()` of a private dialog; a
  chat is attached whole by `ChatLinks::attach()`). A record older than a day
  (a history import) only goes to the one deal, nobody is notified.
- Reading and writing the correspondence (cms-communication, release audit
  N1/N2, 29.10.2026). One policy for letters and chats —
  `services\CrmRecords::communicationQuery($type, $user)`: a letter or a chat
  linked to a company or a person is read only when the client is seen in
  full (the company outside `dealOnlyCompanyIds()`; a person — a `full` link
  of him or a subordinate, or a person of such a company), an administrator —
  all; deals, services and tasks keep their scopes (they already give «своё»).
  `visibleQuery()` stays for choosing and linking records. Users:
  `CmsCommMessageQuery::linkedToVisibleCondition()` (the mail list, a letter,
  the search, files, REST/MCP — all go through `forManager()`) and
  `ChatAccess::linkedCondition()` (the chat list, a chat, REST/MCP). A full
  access to a mailbox or a bot (`read_level = all`) stays a right of its own.
  Writing to a chat with «own» sending — `ChatAccess::sendRefusal()`: a chat
  the employee took, or linked to an open deal he works on
  (`CrmRecords::workingLeadsQuery()` — `isLeadWorker()` in SQL: his and his
  subordinates' deals, every deal of a client seen in full) or to a client
  seen in full; otherwise the refusal asks to link the chat to his open deal.
  It is checked in the chat page, `cms_comm_chat_reply_check`/`_send` and
  once more right before the queue (`BotSend::queue()` for the acting worker,
  so a repeat by an old token after the deal was closed is refused); «any»
  sending on the bot writes to every chat it reads. Regressions:
  cms-communication `tests/crm-communication-access-stand.php` (company-only
  and person-only letters, deal/edit/full, his, a foreign and a closed deal, a
  head of department, an administrator, a fake messenger), and the
  company-only letter among the forbidden markers of `crm-leak-audit.php`.

## Participants only with the main responsible

Owner's decision 29.10.2026. `CrmAccess::ensureMain($code, $id, $added,
$formerMainId)` runs on every saved or deleted link (`CmsCompany2manager`,
`CmsUser2manager`): a client with participants always has a main one — a
participant with `full` (the responsible of an import, the API, a campaign),
else the administrator or head of a department who adds the first
participant, else the head of the department of the former main one (or of a
participant), else `CrmProtection::HISTORY_FALLBACK_USER_ID` / the first
administrator; written as `main_auto`. `setMain()` suppresses it while it moves
the mark. A head of a department manages the access of a client without the
main one (`canManageAccess()`), so the «Доступ» window and the bulk actions give
its first participant («Сделать
основным ответственным» makes the employee the main one). Old clients with
participants and no main: `php yii cms/crm/assign-orphan-main` (the plan;
`--apply` writes, `CrmProtection::orphanMainPlan()`).

## Pitfall: the pk of a selection over a scope with joins

`forManager()` / `forManagerOrOrphans()` join `managers`: a bare `['id' => …]`
over them is ambiguous for a non-administrator (an administrator has no join
and never meets it). Always `Model::tableName().'.id'`;
`BackendModelMultiWindowAction::applySelection()` (cms-backend) qualifies the
pk itself. Test such actions under a leader.

## There is no general pool: a CRM client always has a main responsible

Owner's decision 29.10.2026 (cms-communication `ARCHITECTURE.md` п. 13а; it
replaced the general pool, the right «Общий пул», «Взять в работу» and the
filter «Общий пул»).

- A new company or person gets its main one at insert
  (`CrmAccess::assignCreatorMain()`, from `keepLinks()`): links given with it
  (a form) make one of them the main one (`ensureMain()`), else the creator —
  `created_by`, else the acting employee (`actingWorker()`); a company created
  without an employee (a console job) gets the default administrator; a
  person who registered on the site himself gets nobody — a user of the site,
  not a CRM client (п. 14). The creating code may give its responsible right
  after (an import «я / сотрудник», the API `manager_ids`, a campaign, the deal
  form `LeadTriage::newClientMain()`): a repeated link of the creator merges
  into the automatic one (`CmsCompany2manager::insert()`, `isAutoCreatorLink()`,
  `confirmAutoMain()`), another employee's `full` link takes the main place and
  the creator's automatic link goes (`onLinkInserted()`, journal `main_auto`
  «ответственный, заданный при создании»). Only within the process that created
  the record; links `setMain()` inserts itself never count.
- A client without the main one is old data (`isOrphan()`: no employee at all,
  not an employee, not a person of an owned company): seen by an administrator
  and the heads of departments (`canSeeOrphan()`, `applyClientVisibility()` →
  `CmsCompanyQuery/CmsUserQuery::forManagerOrOrphans()`; a person only when an
  employee created him), the card shows `views/crm/_no-main-banner.php`, a head
  gives it a main one. «Работать с этим клиентом» on it assigns the main one by
  the history first (`CrmProtection::assignHistoryMain()`, else
  `HISTORY_FALLBACK_USER_ID`), the employee gets `deal`. The old ones:
  `php yii cms/crm/assign-pool-main` (`CrmProtection::poolMainPlan()`: by the
  history, else `--user`/Семенов; persons only when created by an employee or
  with a history; the plan by default, `--apply` writes). The RBAC item
  `cms/crm-pool` is removed by migration `m261029_170000`.
- The dismissal: an employee who is the main one of clients is not
  deactivated, made a non-employee or deleted (`User::init()` events,
  `CrmAccess::deactivationError()` / `mainClientsCount()`; every way — the form,
  one field, the bulk deactivation, the delete — saves or deletes the model).
  «Передать всех клиентов» (`AdminWorkerController` action
  `crm-transfer-clients`, `views/admin-worker/crm-transfer-clients.php`, a button
  in «Клиенты в CRM» of the profile; an administrator or his head —
  `canTransferClientsOf()`): `CrmClientTransfer::workOf()` (his clients, his
  open deals and tasks of them) and `transferAllOf()` — the receiver (the
  nearest head by default, `nearestHeadId()`) becomes the main one, the deals
  and tasks move except the kept ones; «Сотрудник увольняется» removes the
  former main one from the clients and deactivates him. Journal `transfer`
  (and `remove`).
