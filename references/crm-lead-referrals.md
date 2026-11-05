# «Откуда узнали»: the advertising source of a deal

Verified in `skeeks/cms` 6.4.9.84 (stand, 05.11.2026). Words: «сделка» =
`CmsLead` (`crm-entity-names.md`).

Two different questions, two fields — never mix them:

- the **channel** `CmsLead.source_type` — how the request reached the CRM
  (`form`, `phone`, `email`, `max`, `manual`, `visit` …), set by the source
  adapter or the person, read by routing, feeds and the report «По источникам»;
- **«Откуда узнали»** `CmsLead.cms_lead_referral_id` — what advertising
  brought the client (Яндекс Директ, 2ГИС, Рекомендация …), an item of the
  site list `CmsLeadReferral` the client keeps himself. Empty is allowed.

Do not add per-client fields for this; extend the list or its rules.

## Model

- `cms_lead_referral` (`CmsLeadReferral`): `cms_site_id`, `name` (unique per
  site), optional `code` (unique per site, for API and packages), `sort`,
  `is_active`, `match_rules`. A used item is not deleted — switched off; an
  inactive item is never offered or matched, old deals and the report keep it
  (`LeadReferrals::items($siteId, $keepId)` shows the current inactive value).
- `cms_lead.cms_lead_referral_id` (FK SET NULL) and `cms_lead.referral_note`
  — why it was filled automatically; empty when a person chose it. Saving a
  changed item without a new note clears the note (`CmsLead::beforeSave()`).
- `CmsLead::referralReady()` guards every read and rule (column may come after
  the code, `package-migrations.md`). Migration `m261105_100000` seeds the
  starting list (`LeadReferrals::defaults()`) for every site by missing code
  or name; a new site gets it by «Стандартный список» in the list page
  (`ensureDefaults()`), never as a side effect.
- Channel `CmsLead::SOURCE_VISIT` («Визит в салон / офис»): chosen by hand, or
  a Form2 form with `Form2Form.lead_source_type = visit` (QR code in the
  showroom). A visit deal made from a form is a form submission:
  `CmsLead::isFormSubmission()` (channel `form`, or `visit` with
  `source_data.form_send_id`) is the one predicate for "the ingestion service
  writes the creation entry" — never compare `source_type === SOURCE_FORM`.
  Form2 looks a submission's deal up by every channel of a form
  (`Form2FormSend::leadSourceTypes()`): the form setting may change later.

## Filling: only an empty field, never over a person

`crm\LeadReferrals::fill($lead)` runs in `CmsLead::beforeSave()` for a new
deal (forms, letters, campaigns, API, manual). `fillByCall($lead, $called)`
runs from `CmsTelephonyCallLinkService::attachLead()` for an incoming call
with `provider_phone_to`: only the called number counts there, it saves with
`updateAttributes()` and writes one feed entry. Both swallow errors except a
lock conflict (`CmsLeadService::isLockConflict()`).

Order of `detect()`:

1. `LeadReferrals::EVENT_DETECT` (`crm\LeadReferralEvent`): a package names
   the item by `referralId` or `code` with a `note`; `handled` without an item
   stops the rules. The core depends on no package.
2. Rules without `referrer`: marks (the five utm columns, then any remembered
   parameter — `yclid`, `from` …), `channel=`, `phone=`.
3. Only when the visit has **no utm marks**: rules with `referrer=` against
   the first external page of the visit.

Inside a step: more conditions win, then the longer referrer pattern
(`yandex.ru/maps` over `yandex.ru`), then list order.

Context (`context()`): `params` from the source page query,
`source_data.utm` (cms-seo session values, `source`→`utm_source` aliases) and
the utm columns; `referrer` from `source_data.utm.sx_first_referrer`, else
`source_data.referrer`, only when its host differs from the source page and
the current host. In `beforeSave()` `source_data` is already JSON-encoded by
`HasJsonFieldsBehavior`: decode a string before reading it.

Rules text: a line is a rule, all its conditions must hold; a condition is
`name=v1,v2` (`*` any value, `yandex*` prefix); a new condition starts only at
a space before `name=`, so `phone=+7 900 000-00-00` keeps its spaces; `#`
comments. Keys: utm/parameters, `referrer` (domain with subdomains, optional
path prefix), `channel` (a `source_type`, validated), `phone` (last 10
digits). `validateRules()` names the bad line.

Letters of the CRM carry `utm_medium=email` (cms-communication link
tracking): the starting item «Рассылка» matches them and `channel=mailing`
without any cms-communication code.

## The first page of a visit (cms-seo)

`CmsSeoComponent` keeps visit marks in the session (`utms`, 24 h). Since
3.1.0.5 an ordinary GET page request with an external `Referer` and no marks
in the session yet stores `['sx_first_referrer' => …]` there too; marks of a
later visit replace it, a later page never does. Form2 copies `seo->utms`
into the submission and `source_data.utm`, so nothing else is needed for
forms. Search engines often send the origin only (`https://yandex.ru/`):
maps or catalogs without their own utm are then seen as search — advise the
client to put utm marks on links in their map and catalog profiles.

## Interface and API

- Deal create form: the chips `widgets\admin\LeadReferralInputWidget` right
  under the name, with the channel next to them (one click, a click on the
  chosen chip clears it); edit form; the card block «Источник и атрибуция»
  with «Изменить» → the small action `admin-cms-lead/referral` (only this
  field changes); list column and filter; the report table «Откуда узнали»
  with channels inside (`CmsLeadReportService::build(…, $referral)`,
  `children`) and its filter (`referral` = id or `none`).
- Setting `crmSettings->referralRequiredManual` (off by default): required
  only for a new deal by hand (`manual`/`visit`, no `source_ref`) on a site
  with active items (`CmsLead::referralRequired()`).
- List page `cms/admin-cms-lead-referral` (menu «Справочники»), admin only.
- cms-mcp: `cms_lead_referral_list`; `referral_id` / `referral_code` in
  `cms_lead_create` and `cms_lead_update` (inactive refused, null clears);
  `referral_id` (id or `none`) in `cms_lead_list`; `referral` in deal data
  (`auto`, `note`).

Tests: `cms/tests/lead-referrals-stand.php`,
`cms/tests/lead-referral-migration-db.php`,
`cms-mcp/tests/crm-lead-referrals.php`,
`cms-module-form2/tests/form2-visit-channel-stand.php`.
