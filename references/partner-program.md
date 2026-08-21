# Shared partner program

## Package boundary

The reusable lead workflow belongs to `skeeks/cms`, not to the partner or form
packages. `CmsLead` / `cms_lead` is the canonical inbox for manual, partner,
Form2, messenger, telephony, import and API leads. Its employee screen belongs
under CMS → Clients and deals → Leads.

`skeeks/cms-shop` owns only the financial extension of a partner lead: the
reward row, its bonus-ledger transaction, payout requests and payout locking.
Do not create a second lead status, executor, client/company link, description
or conversation in the shop package.

Use these shared contracts:

- `CmsLead` / `cms_lead` for every lead and its workflow;
- `ShopPartnerLead` / `shop_partner_lead` as the one-to-one financial result of
  a successful partner-linked `CmsLead`;
- `ShopPartnerPayout` / `shop_partner_payout` for payout requests;
- `PartnerProgramHelper` in `skeeks\cms\shop\helpers` for balance, reserved
  amount, available amount and payout locking;
- canonical Admin routes `/cms/admin-cms-lead` and
  `/shop/admin-partner-payout`;
- reusable partner-cabinet controllers, forms, presentation helpers and views
  in `skeeks/cms-shop`, exposed through `/shop/upa-partner-lead`,
  `/shop/upa-partner-bonus` and `/shop/upa-partner-payout`.
- Partner ActiveRecord relations and existence validators target the canonical
  `CmsUser` and `CmsSite` models directly. Do not resolve them through the web
  `user` component: shared domain models must also work in console and worker
  applications where that component is absent.

Keep project-specific landing pages, branding and menu composition in the
project. The project menu points at the package UPA routes; it must not copy the
controllers, forms or views. A project may keep a thin presentation-helper
alias for its public landing page, but shared cabinet copy and balance surfaces
belong to `skeeks/cms-shop`. All consumers use the shared models and helpers
instead of creating project tables or duplicating bonus queries.

Form2 remains the owner of the original form response. After inserting a
`Form2FormSend`, create the lead idempotently using the form-send ID as the
source reference. Put a readable label/value snapshot in the lead description,
retain structured values in source data, and copy page URL plus UTM fields.

## Financial invariants

- A successful partner-linked lead creates exactly one credit
  `ShopBonusTransaction` and one one-to-one reward extension row.
- Every newly inserted credit `ShopBonusTransaction` notifies its beneficiary
  once and links to the shared partner bonus ledger. Debit rows and later edits
  do not emit a credit notification.
- Public transaction titles use recipient-facing bonus wording such as
  `Начисление бонусов` and `Списание бонусов`; do not expose internal
  employee-oriented phrases such as `Начисление клиенту` in UPA notifications
  or the partner ledger.
- A paid payout creates exactly one debit `ShopBonusTransaction`.
- Employee payout and bonus-ledger sections use their normal section RBAC
  permission, then scope both collection queries and direct model loading
  through the beneficiary client's canonical `CmsUser::find()->forManager()`
  boundary. A hidden row must remain unavailable through a forged `pk`, and
  client selectors on these screens use the same manager scope.
- Payout notifications use the same boundary as payout visibility: enumerate
  active workers with the payout permission and retain only those for whom the
  beneficiary exists in `CmsUser::find()->forManager($worker)`. This includes
  administrators who can actually see the row, while excluding unrelated
  employees. Partner-facing status/comment notifications use the configured
  UPA prefix; employee comment notifications deep-link to the Admin activity
  entry with `sx-log-id`.
- Keep payout editing, result processing and the read-only card as separate
  actions. The initial workflow has exactly three public states: new, paid
  (shown as successful) and rejected (shown as cancelled). A paid result
  requires a partner-facing message; a rejected result requires a reason.
  Both Admin and UPA cards share the payout comment stream, but the UPA comment
  action must owner-scope the payout and set author/model fields server-side.
- The `CmsLead` success transition runs in an ActiveRecord transaction and
  emits the partner-success extension event. The shop listener creates its
  reward row; that row creates the ledger side effect in `beforeSave()` and
  stores the ledger transaction ID. An exception rolls back the lead result,
  reward and ledger transaction together.
- Use `lock_version` optimistic locking so a stale manager update rolls back
  its ledger side effect together with the domain save.
- `CmsLogBehavior` excludes `lock_version` from new logs, and `CmsLog` hides it
  when rendering historical data. It is a technical concurrency counter, not
  employee-facing activity.
- Keep workflow and source codes stable in storage. For readable activity
  entries, configure `CmsLogBehavior::$attribute_value_maps` from the model's
  canonical dictionaries, for example `status => Model::statuses()`. `CmsLog`
  applies these maps while rendering, so historical and new events display
  readable labels without rewriting stored JSON. Do not duplicate translations
  in views or persist localized labels as domain values.
- Lock the partner's `cms_user` row with `FOR UPDATE` while validating a payout
  reservation or creating its debit. This serializes concurrent payout
  requests for the same user.
- Successful/rejected leads and paid/rejected payouts are terminal. Do not
  rewind their status, delete financial records, or edit an already applied
  amount. Corrections are separate auditable ledger transactions.

## Transitional site scope

Keep nullable `cms_site_id` on `cms_lead` during the multisite transition and
keep required `cms_site_id` on payout rows while `shop_bonus_transaction`
remains site-scoped. A reward transaction inherits the lead site explicitly.

Package UPA create actions must also assign the active `cms_site_id` explicitly
before validation together with the authenticated partner ID. In shared models,
put defaults for server-owned required fields before their required/range
validators so a new record remains valid outside a particular controller.

This is a compatibility boundary, not a recommendation to expand multisite
architecture. When the CMS bonus ledger becomes global, remove site scope from
the ledger, partner models, aggregate helper and all consumers in one deliberate
migration. Do not remove it only from partner rows while balances are still
site-scoped.

## Lead card and partner conversation

- A partner-created lead starts as `new` without `executor_id`. The explicit
  employee action “Взять в работу” assigns the current employee as executor
  and moves the lead to `in_work` in the same optimistic-locking save. After
  assignment, employee-side edit, processing, conversion and comments are
  available to that executor. Other managers cannot change the lead; an
  administrator may explicitly reassign it. Show the executor in the card
  properties and keep the work action in the main-card action row, following
  the standard task-card pattern.
- Manual Admin creation defaults `executor_id` to the employee creating the
  lead, while allowing that value to be cleared or reassigned. An assigned lead
  is created as `in_work`; an unassigned lead remains `new`. Notify only the
  selected executor, except when that executor is the employee creating the
  lead. With no executor, use the normal eligible-manager notification set.
  Keep `manual` as the visible default source in the create form so an omitted
  choice never produces an ambiguous source.
- Link a converted lead to the canonical `CmsCompany` with
  `cms_company_id`. Reuse the standard company create controller/form through
  a thin package controller instead of copying CRM company fields into the
  shop package or making `skeeks/cms` depend on `skeeks/cms-shop`. The standard
  company form assigns the current employee as company manager; do not create
  a second manager-assignment mechanism for partner leads. After creation,
  link the company to the lead and add a company `CmsLog` comment that records
  the originating partner-lead ID.
- The employee lead screen opens a read-only `BackendSurfaceWidget` detail
  card first. Keep referred-client editing and status processing in separate
  update actions, and expose the standard `BackendModelLogAction` for the
  complete audit trail.
- Keep referred-client identity and contact fields out of the processing
  action. The decision form uses progressive disclosure: `in_work` has no
  outcome fields, `rejected` requires only a partner-visible rejection reason,
  and `success` requires both the reward amount and a partner-visible summary
  of the services actually delivered. Terminal outcomes and their decision
  fields are read-only.
- Render the partner with the standard clickable `CmsUserViewWidget` in the
  employee card instead of plain text, so managers can open the canonical
  client profile without introducing a package-specific link pattern.
- Both the employee card and the package UPA card render one conversation from
  `CmsLead::getLogs()->comments()` with `CmsCommentWidget` and
  `CmsLogListWidget`. Do not create a parallel comments table.
- A UPA consumer must not grant access to the global Admin
  `/cms/admin-cms-log/add-comment` action. Provide a package-owned hidden POST
  action that loads the lead through the same owner/site-scoped query as the
  card, then assigns `model_code`, `model_id`, author fields, `log_type` and pin
  state on the server after loading the request.
- Treat UPA comment HTML and attachment IDs as untrusted. Purify the comment,
  reject visually empty HTML and disable attachments until their ownership is
  verified server-side. `CmsCommentWidget::$isShowAttachments` and
  `$isShowPin` allow this restricted consumer while preserving the full Admin
  form by default.
- Scope the employee lead inbox through the same CRM access boundary as clients
  and companies. A non-admin employee may read a lead assigned to them or a
  subordinate, a lead whose partner/client/company is available through the
  canonical `forManager()` scopes, or an unassigned lead with no known CRM
  submitter/partner/client/company identity. Once an executor or CRM identity
  is known, do not leave the lead in the common queue. Apply this scope to the
  list, direct model loading, hidden AJAX actions and child create/update
  controllers. An inaccessible direct URL must resolve as not found. The
  partner-facing UPA card remains independently owner/site-scoped.
- Exclude the current worker's own user ID from the lead query's
  `CmsUser::forManager()` client subquery. A self-link in the manager relation
  must not expose a partner lead merely because that worker submitted it as a
  partner; visibility still requires another available CRM identity or an
  assigned executor.
- Automatic CRM identity matching uses normalized phones and exact emails only.
  Names and company names are display data, not identity evidence, and must not
  produce candidates by themselves.
- `CmsCompany` is global and has no `cms_site_id`. Match and link companies
  through `CmsCompany::find()->forManager()`; do not apply the generic
  `cmsSite()` scope to company queries. User candidates remain site-scoped.

## Lead notifications

- A new lead sends `CmsWebNotify` to the managers who can take it. If the
  authenticated submitter or partner belongs to a CRM company with responsible
  managers, or has directly assigned managers, notify only those employees who
  also have lead access. If a known CRM identity has no eligible responsible
  manager, notify active administrators who have both Admin and lead access as
  a narrow triage fallback; do not broadcast it to every worker. Use the
  all-active-workers fallback only for an anonymous lead with no submitter or
  partner identity.
- A status change or employee comment on a partner-linked lead notifies the
  partner.
- A partner comment notifies the responsible executor. Before assignment it
  notifies the same eligible-manager set used for a new lead.
- Link notifications and logs to the canonical `CmsLead` model code so every
  surface opens the same card and activity stream.
