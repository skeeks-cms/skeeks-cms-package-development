# Shared lead ingestion

`skeeks/cms` owns the reusable lead model and source-ingestion service. A
source package such as Form2 owns extraction from its factual submission model
and passes normalized lead attributes into that service; it must not duplicate
lead persistence rules.

Every external submission uses a stable `source_type` plus `source_ref` and a
site scope when available. Enforce this identity with both a database unique
index and a service-level lookup. The service must also recover from a unique
constraint race by reading and returning the record that won the insert.

Persist the factual source submission, its related values, and the canonical
lead in one database transaction. The shared ingestion service must join an
already active adapter transaction and only commit or roll back a transaction
that it started itself; this prevents a failed lead projection from leaving a
partially saved source submission.

Capture source values only after the source model has loaded and validated its
input. For Form2, dynamic values live on `RelatedPropertiesModel`; snapshot its
actual attributes and matching `attributeLabels()` before saving
`Form2FormSend`. Do not infer values from raw request keys when the validated
model already owns their types and normalization.

Map a recognizable name into the lead and every recognizable phone and email
into the lead's ordered contact relations. Phones and emails may repeat across
different leads, but the same normalized contact is stored only once inside a
single lead. The first contact by `sort`, then `id`, is the default contact for
compact displays and prefilling downstream actions; it is not a separate copy.
Keep every non-empty source field as a labeled value in the description and as
structured `labels`, `values` and paired `fields` in `source_data`. Include the
source submission ID, source entity metadata, page URL, UTM values and a small
allowlist of request metadata. Do not duplicate raw cookies, sessions, full
server dumps or unrestricted request payloads into the lead.

Persist the complete operator-facing lead title in the canonical `name` field.
For Form2, build it during ingestion from the form name, submission number,
recognized contact name and main phone, for example
`Form on site «Callback» #243: Alexander, +7 900 000-00-00`. Lead lists and
card headers render that stored name directly, so a manager edit is reflected
verbatim and presentation code never wraps the editable value in a second
generated title. CRM creation forms that need a short client or company name
must derive or request it separately instead of treating the lead title as a
contact-only field.

Store canonical `utm_source`, `utm_medium`, `utm_campaign`, `utm_content` and
`utm_term` values in dedicated indexed lead columns for filtering and
analytics. The shared ingestion service normalizes explicit adapter values and
falls back to the source payload or `source_url`; adapters still retain the
original URL and payload so normalization never discards source evidence.

A lead card may link back to the source package's standard read-only submission
action when that package is installed. The factual source-submission card also
shows the reverse relation: an existing lead is a standard clickable backend
entity, while an older submission without a lead offers an explicit POST-only
manual projection through the same idempotent source adapter. Repeated manual
projection returns the lead identified by site, source type and source reference
instead of creating a duplicate.

Keep source viewing read-only: opening a historical submission must not assign a
manager, change its legacy status or create a lead. If the old source workflow
still needs editing during migration, expose it as a separate processing action
and keep the canonical ongoing work in the lead. Keep the source adapter
backward-aware: the source submission must remain usable when the lead class or
table is not yet available during a staged package rollout.

## Origin of a request

A package may know where a new request came from by evidence the core does
not have (cms-communication: the visit of a letter link kept in the browser
cookie `sx_visit`). The core asks through class-level events of
`CmsLeadService`, never by depending on that package; source adapters such as
Form2 stay unchanged.

- `EVENT_LEAD_ORIGIN` fires inside the creation transaction of a brand-new
  request only, before routing; a repeated `source_type` + `source_ref`
  returns its deal earlier and asks nobody. The `crm\LeadOriginEvent` handler
  names the client (`companyId`, `clientId`), the employee for a client
  without a main responsible (`executorId`, `executorRole` for the feed),
  `reason`, or an open deal the request goes on in (`continueLead`;
  `notifyContinue = false` when the handler already told its responsible).
  `origin` keeps the handler's data for the second trigger.
- `InboundQueue::route()` takes the event: the named client is linked, the
  deal goes in work to the main responsible of that client (a company — its
  own; a person — their own, else the one of their company, as
  `InboundQueue::mainOf()` decides for a contact match; a named company wins
  over a named person), otherwise to `executorId`; both must be active
  employees of the site with `cms/admin-lead`, else the usual route runs.
  The main responsible never changes; `afterRoute()` grants the level by
  `grantLeadExecutorLevel()` and writes "<reason>: …передана …". It does not
  depend on `inboundToMain`: named evidence is not a contact guess.
- `continueLead` must be open, of the same site and, for a partner request,
  of the same partner. The request still gets its own row and source
  reference, saved closed as added (`status=rejected`,
  `triage_result=merged`, `merged_into_id`, the target's executor and client,
  `suppressNotifications`), so `fromSource()` + `resolveMerged()` and a
  source card keep working; both feeds get the "Добавлено обращение" pair as
  in `LeadTriage::merge()`. The phones and emails of the request are saved
  to the target too, at once — the same as a repeated event of that request
  (`syncContacts()` of the lookup) would do, so the first request and its
  repeat give one result. The service returns the target deal.
- A handler swallows its own failures (the request is saved anyway) except a
  lock conflict (`CmsLeadService::isLockConflict()`): MySQL rolled the whole
  transaction back, so it is rethrown for the owner to repeat or fail.
- `EVENT_LEAD_ORIGIN_SAVED` fires with the same event after the request, its
  contacts and creation entry are saved, still in the transaction: `lead` is
  the saved request, `target` the deal where the work goes on. Handlers write
  their feed entries there and move their own records (an open call task) to
  `target`.
- A handler that creates its own request inside `EVENT_LEAD_ORIGIN` (the
  campaign request of a letter, created through `createFromSource()` again)
  guards against answering its own nested request (a static busy flag and
  skipping non-visitor sources).
- The feed keeps only absolute `http(s)` links (`helpers\CommentHtml`): build
  links of system entries with `Url::to([...], true)`.

Covered by `cms/tests/lead-origin-stand.php` (a test handler) and
`cms-communication/tests/link-requests-stand.php` (visits of letters).

## System activity entries

`CmsLead` owns the readable activity stream of a lead. Every system event is a
`CmsLog` with `LOG_TYPE_COMMENT` written through `CmsLead::addSystemActivity()`,
which fills `model_code`, `model_id` and `model_as_text` from the lead, escapes
nothing by itself and throws when the entry cannot be stored. Callers escape
every dynamic fragment with `Html::encode()` and own the surrounding
transaction, so a rejected entry rolls the domain change back instead of
leaving the stream out of sync. Do not create a parallel activity table and do
not write these entries from views or controllers of consumer packages.

`CmsLead::recordCreationActivity()` writes the single creation entry and is
idempotent per model instance. Its wording is derived from the source, not from
the calling surface: a partner lead reads `«<author> добавил лид «<name>»»`,
any other authored source reads `«<author> создал лид «<name>»»`, and the
author is resolved from `created_by`, then `submitted_by_id`, then
`partner_id`. When an explicit author is passed, set it as the
`BlameableBehavior` value on the log; assigning `created_by` alone is silently
replaced by the current session on insert.

Sources that persist a complete lead in one save record the entry from the lead
lifecycle. `SOURCE_FORM` is the deliberate exception: its contacts are stored
after the lead row, so the lifecycle skips it and `CmsLeadService` calls
`recordCreationActivity()` after the phone and email relations and before the
commit. That entry names the submitted form, its submission number from
`source_data.form_send_id` with a `source_ref` fallback, the lead name and the
main `CmsLeadPhone`. A lead returned from the `source_ref` lookup is not new and
must never receive a second entry; the same rule applies to contact
synchronization of an already ingested lead.

A financial outcome is announced only after its money exists. The shop listener
of the partner-success event writes its entry after the reward row and its
bonus transaction are saved, never before, and formats the amount as
`rtrim(rtrim(number_format((float)$value, 2, '.', ' '), '0'), '.')`.

A request added to another deal while sorting keeps its own row and
`source_ref`; the lookup follows `merged_into_id` (`CmsLead::resolveMerged()`)
and returns the deal where the work goes on (`crm-duplicates-and-triage.md`).

## Existing CRM identity matching

The candidate surface must use the same link-eligibility predicate as the
write action and domain service. For a lead that is not in work, keep the
evidence visible, replace link controls with an instruction to claim the lead,
and return expected write rejections to the card as a readable message.

Treat lead-to-client/company matching as read-only candidate discovery, not as
conversion. Load it independently from the main card so contact joins do not
delay the activity and workflow UI. Exact normalized phone and email evidence
comes before exact names; reuse the canonical entity queries and the factual
`CmsCompany2user` relation, and show the evidence that produced every candidate.

Viewing a lead must never create, link, claim or complete anything. Linking an
existing client, company, or their verified pair is an explicit authorized POST
action on an in-work lead. Recheck entity visibility and the client-company
relation on the server, never overwrite an existing different link, and record
the choice in the lead activity. Keep the original lead/source identity after
linking so attribution and submission audit remain intact.

Lead deletion is an administrator-only, single-record action using the
standard confirmed POST backend delete flow. Keep bulk deletion disabled.
Deleting a lead removes its phone and email rows through their database
cascades, preserves the factual source submission, and leaves canonical
telephony calls intact with their lead reference set to null. Do not widen
ordinary manager permissions or implement a parallel delete endpoint.

## Telephony activity

Persist the lead context on the canonical telephony call and point the lead
activity log at that same call ID. Do not copy duration, final status or audio
into a separate lead-specific record: webhook updates and recording downloads
must become visible through the existing phone-call renderer.

An outgoing call started from any lead phone carries the explicit lead ID
together with that phone number. The server verifies the number against all
lead phone records. For incoming calls and older call buttons, normalized phone
matching is only a fallback and may attach automatically only when one active
lead matches. Shared phone numbers are ambiguous and must remain unassigned.
Webhook retries and concurrent call registration must not create duplicate call
rows or duplicate lead activity entries.
