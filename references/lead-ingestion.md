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

Store canonical `utm_source`, `utm_medium`, `utm_campaign`, `utm_content` and
`utm_term` values in dedicated indexed lead columns for filtering and
analytics. The shared ingestion service normalizes explicit adapter values and
falls back to the source payload or `source_url`; adapters still retain the
original URL and payload so normalization never discards source evidence.

A lead card may link back to the source package's standard read-only submission
action when that package is installed. Keep the source adapter backward-aware:
the source submission must remain usable when the lead class or table is not
yet available during a staged package rollout.

## Existing CRM identity matching

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
