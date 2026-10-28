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
| Own deals: bills, payments, documents, proposals, the deal's own requisites and links (`cms_lead_*`) | yes | yes | yes |
| The company/person data: name, form, requisites, phones, emails, addresses, links, people, contacts copied from a deal | no | yes | yes |
| Responsible employees and levels (the `managers` form field, «Доступ к клиенту») | no | no | main, leader, admin (`canManageAccess()`) |
| Deleting the company or the person | no | no | yes |

One place decides, everything uses it:
- `canEditClientData($user, $code, $id)` (level ≥ `edit`);
- `canEditClientRecord($user, $model)` takes a company, a person or a row with
  `cms_company_id` / `cms_user_id` (`clientRecordOf()`). It is true for
  employees and the user himself: they are not client data;
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
- requisites attach (`Admin*ContractorController::canAttach()`);
- `LeadTriage::mayCopyTo()` (contacts of a deal to the card) and linking a
  person to an existing company;
- cards: `$canEditData` hides the «⋮» menus and the «Добавить …» buttons; the
  deal contact block (`_client.php`) hides them by `canEditClientData()`.

Not covered yet:
- the REST API (`cms-mcp` `CmsCompanyService`: update, phones, emails,
  addresses, links) checks only visibility;
- the client feed for `deal` and the notifications of client events to the
  main responsible: feed entries (`cms_log`) carry no deal, and mail and chat
  recipients are decided in cms-communication.

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
  (`grantLink()` in `afterIdentity()`), a `work` request with the deal goes to
  the main responsible and his leaders. They raise the level or decline; a
  decline keeps the deal and the level (`CrmAccessRequests::decline()`).
- «Доступ только по согласованию» (`cms_company.crm_access_by_approval`,
  `cms_user.crm_access_by_approval`, `approvalRequired()`, shown in a match
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

- «Доступ к клиенту» in the company and client cards
  (`ClientTransferActions::clientAccess()`, `views/crm/access.php`): the main
  responsible first, levels, «Убрать», the flag. Changed by the main
  responsible, his leaders and administrators (`canManageAccess()`).
- «Передать клиента» — `setMain($new, LEVEL_EDIT)`: the former main stays with
  `edit` and may be removed; other links stay; open deals and tasks move
  except the kept ones.
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
