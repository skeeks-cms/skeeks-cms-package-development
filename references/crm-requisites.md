# CRM requisites and lead-owned records

Verified in `skeeks/cms` (lead card addresses, links and requisites, 2026-10-19).

## Requisites (contractors) of an owner record

Requisites are shared `CmsContractor` records linked to an owner through a map
table: `cms_company2contractor`, `cms_lead2contractor`, `cms_contractor_map`
(users). The INN is unique across the whole base.

Add requisites to an owner only through a subclass of
`AdminCmsContractorAttachController` (`create?<owner_param>=<id>`), as
`AdminCmsCompanyContractorController`, `AdminCmsLeadContractorController` and
`AdminCmsUserContractorController` (clients, `cms_contractor_map`) do. It opens the full contractor form and:

- the "Заполнить по ИНН" button (`dadata-inn`) fills fields from our base first,
  then from DaData when the `dadataClient` component exists; without it the
  user gets a "fill manually" message, never a PHP error;
- on save, an existing contractor with the entered INN is only linked to the
  owner (no second record, no "INN already used" error); otherwise a new
  contractor is created and linked in one transaction;
- the "our company" flag is hidden: it belongs to the studio's own entities.

Do not add new INN-only modals that depend on DaData. A subclass supplies the
owner lookup with the manager scope, the access rule, the link save and the
return URL. Removing requisites from an owner deletes only the map row.

`CmsContractorQuery::forManager()` must expose contractors reachable through
every owner a manager can see; lead requisites are visible through
`CmsLeadQuery::forManager()`.

## Contact blocks of CRM cards

Phones, emails, addresses, requisites and links in the company and lead cards
are rendered only by `skeeks\cms\widgets\admin\CrmContactBlockWidget` (styles:
`CrmContactsAsset`, `crm-contacts.css`). It wraps `BackendSurfaceWidget` and
gives every block the same grammar: a quiet "?" hint in the header
(`hintIcon()`), rows with a type icon, muted label, value, optional note and
actions (`menu()` for the controller "⋮" menu, `iconButton()` for call/SMS/
write), a muted empty state and a dashed full-width "+ Добавить …" button
(`addButton()`). Cards pass data and permissions (`addRoute => null`, empty
actions) and may add density options through `surface`; they do not write
their own row markup. A new CRM card (client, deal) reuses the widget.

Phone and email row actions go through
`CrmContactBlockWidget::contactActions($type, $value, $owner, $defaults)`.
The card passes its defaults keyed by id (`sms`, `call`, `mailto`) and the
class-level `EVENT_CONTACT_ACTIONS` lets another package replace, add or
remove them (for example a "write a letter" dialog instead of `mailto`).
Handlers check their own permissions; a lead the employee cannot work with
passes no phone defaults. Do not edit card views to add channel buttons.

Icons are semantic `BackendIcon` names (`phone`, `mail`, `message`,
`map-pin`, `id-card`, `link`, `more-vertical`, `help`, `close`, `plus`), never
icon-font classes. `CrmContactBlockWidget::icon()` falls back to the matching
icon-font glyph when the installed `cms-backend` lacks a name, so `cms` can be
released before the icon set. The row "⋮" menu is shown on hover/focus only on
pointer devices (`@media (hover: hover)`); touch screens keep it visible.

## Records that belong to a lead

Phones, emails, addresses and links of a lead use
`controllers\traits\LeadContactActions`: create/update/delete only for who
works on the lead (`canBeWorkedBy`, `CrmAccess::isLeadWorker()`), the lead id
of a new record comes from `<FormName>[cms_lead_id]` in the request and of an
existing one from its saved value, and the flat cross-lead `index` list is for
administrators only. New lead-owned record types reuse the trait instead of
copying the checks.

Lead addresses and links mirror the company tables field for field, so
`AdminCmsLeadCompanyController::linkCompany()` copies them (and links the
lead's contractors) into the company created from the lead, skipping values the
company already has.
