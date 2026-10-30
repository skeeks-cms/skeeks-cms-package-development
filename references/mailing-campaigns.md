# Mailing campaigns

Verified 2026-10-23 with `skeeks/cms-communication` stage 4B and
`skeeks/cms-proposal` on the skeeks.com Windows stand (tests
`cms-communication/tests/campaign.php`, `cms-proposal/tests/client.php`; two
campaigns of five test leads sent from a stand mailbox with a lowered
limit, a presentation for everybody and a КП from a template, an opening, a
one-click opt-out, light, dark and 375 px). Domain rules live in the package
`ARCHITECTURE.md`; consent and opt-out are
[mailing-consent.md](mailing-consent.md); sendable documents are
[client-sends.md](client-sends.md).

## The audience and the recipients

- Read the audience in the launching employee's web request through the
  core `forManager()` scopes; fix the recipients at launch as rows (one per
  CRM record, unique campaign + record type + id, so a campaign never writes
  twice to one record). Keep a record type and id on the row, not a lead
  foreign key, so companies and clients can become audiences later.
- Store excluded recipients too, with the reason: the summary before launch
  and the report after it read the same rows. Reasons: no address, the
  consent policy ([mailing-consent.md](mailing-consent.md)), a repeated
  address, the stand allowlist, no sender.
- Ask the policy again right before each letter: an opt-out after the
  launch must exclude the recipient.
- Launch is one conditional `UPDATE … WHERE status = draft` inside the
  transaction that inserts the recipients: two clicks never fix them twice.

## The letter of each recipient

- A campaign letter is an ordinary message of the mail service
  (`MailComposeService::queue()`), not a second mailer: feeds, "Sent", the
  opt-out link and headers, replies and links come for free. The form gets
  code-only values that are never loaded from a request: the topic, the
  campaign mark, the CRM record of the recipient (checked at launch — the
  job does not ask visibility again) and documents prepared in code.
- A document per recipient from a template (a КП from a template) is made
  by the document package (`CampaignSourceInterface::createFromTemplate()`
  returns its id) inside the transaction that sends the letter and marks the
  recipient: a failed letter leaves no document. Claim the recipient
  (`pending → sending`) inside the same transaction: a crash rolls back to
  waiting and nothing was handed to the mail job.
- Placeholders fill plain values HTML-escaped; an unknown name disappears
  with its comma, at the start of a paragraph with the comma after it and a
  capital letter. The link placeholder becomes the sendable placeholder link
  of [client-sends.md](client-sends.md) (the mail service makes it a
  button); the opt-out placeholder is the one of
  [mailing-consent.md](mailing-consent.md).
- The rich-text editor of a campaign has no pictures: uploads are temporary,
  a campaign body is kept for days.
- Preview the letter in a sandboxed `iframe srcdoc`, on white in both themes:
  mail programs show it so, and its own styles never leak into the cabinet.

## Recipients added one by one and personal letters

- A recipient added before launch (by an employee or by AI through MCP)
  is a recipient row in a `draft` status: the CRM record (a lead, a company
  — its email or one of its people's — or a client), the address chosen
  from that record's addresses, the name and an optional personal letter.
  One service method serves the card and the tools; it checks the record's
  visibility for the caller and keeps who prepared the letter (employee or
  AI, through which client, which CRM user, when) for the "prepared by AI"
  mark of the letter and the feed.
- At launch the added rows join the audience (their address, name and
  letter win over the computed ones) and pass the same exclusions; a
  campaign may have only added recipients.
- A personal letter must contain the opt-out placeholder and, with an
  offer, the link placeholder; an empty subject and text return to the
  template. Change a waiting letter with a conditional update on the
  waiting status: the sending job may take it at the same moment.
- The sender of a company or a client is its first manager; do not read
  relations that use the web user component (`Yii::$app->user`) in code that
  jobs or API calls may reach.

## Channels and the pace of a mailbox

- How letters leave is a registered campaign channel
  (`communication.campaignChannels`): the policy mode it needs (a manager's
  mailbox may write to cold contacts, a mailing service only with consent),
  the sender of each recipient at launch, how it starts and how it sends one
  prepared letter. The audience, the letter and the report stay with the
  campaign, so a mailing service driver adds only a channel.
- A manager's mailbox sends one letter per job step, then the same run
  waits in the queue (`JobContext::requestRequeue($seconds)`, called as the
  last statement outside any transaction) a random pause or until the next
  working period. One job per mailbox (`resourceKey` and `dedupKey` of the
  mailbox, overlap `skip`): pushing again while a run waits is a no-op, and a
  scheduled tick (every few minutes) is only a safety net that restarts
  running campaigns and finishes empty ones.
- The daily limit counts every letter that left the mailbox that day, also
  those the manager wrote in the mail service itself (outgoing rows of the
  mailbox and of its "Sent" copies), not failed ones.
- Days and hours are those of the CRM — the formatter's time zone, as core
  reports and task schedules count them — never `date()`: a console worker
  may run in UTC, and "9:00" would move by hours.
- New pace settings apply from the next step: a mailbox already waiting
  until tomorrow starts with them in the morning. Do not edit queue rows to
  wake a waiting run; it can lose its delayed message.

## The report

- Read the outcome of each recipient from its own sources when the report
  is opened, and store it on the recipient row for counts and filters: the
  message status (failed, unknown), the document package
  (`CampaignSourceInterface::states()` — opened, accepted, by message id),
  the first incoming letter of the conversation after ours (replied), an
  opt-out of the topic or the stop list after the letter. Percents are of
  the letters sent.
- Mark replies as they are received too (after the receive job's
  notifications, never undoing received letters on error): the "replied"
  task of the campaign needs the moment.
- The "opened" task stays the document package's rule; it asks the
  campaign (`Campaigns::taskOnOpen($messageId)`: true, false, or null for a
  letter of no campaign) instead of its own section setting.

## Personal links made in jobs

A job has no web request: build absolute links from one place
(`CommunicationComponent::siteUrl()`: the request host, otherwise
`communication.baseUrl`, otherwise the main domain of the site) and build
paths of public pages by hand (`/~kp/<code>`, `/~unsub/<code>`): a console
application has no web URL rules. A local stand with a copy of the
production database sets `baseUrl` to itself, otherwise its letters lead to
production.

## Model defaults

`init()` of an ActiveRecord runs after the constructor's configuration and
also for every record `find()` instantiates: set a default only when the
attribute exists and is still `null`, and read a single setting of another
record with a scalar query instead of instantiating it.

## Lists of recipients and the rules of every mailing

Verified 2026-10-24 with `skeeks/cms-communication` stage 4D step 1 on the
skeeks.com Windows stand (test `tests/lists.php`; lists filled by hand, by
the bulk action of "Компании" and "Лиды", by an import; a campaign by two
lists with an excluding list, light, dark and 375 px).

- A member of a list is a CRM record (lead, company, client), never an
  address: the channel picks the contact at sending, so one list serves
  mail, SMS and bots, and consent stays with the contact — an opted-out
  member stays in the list and the summary excludes it with the reason.
- A member keeps who added it, when and from where (by hand, a CRM section,
  an import, AI with its client); removing marks it, adding again brings the
  same row back. One service operation serves the list card, CRM cards, the
  bulk action, imports and MCP; records read through the section's own
  scoped query are not checked for visibility a second time.
- Campaign audiences read lists in a few queries: active members once each,
  addresses and companies of many records batched per type, visibility by
  one `forManager()` query per type (invisible members stay in the summary
  as excluded).
- An excluding list works for every audience; a company in it excludes its
  people and its leads too.
- Rules that protect every mailing are exclusions of the summary with their
  own reasons, after the policy: the frequency per contact over all
  campaigns of the site (a site setting; a letter sent in N days, being sent,
  or waiting in a running campaign), "не мешать менеджеру" (a letter or a
  message linked to the record or its company, a chat linked to them, a call
  — never a mailing letter — in the campaign's N days) and one letter per
  company (the recipient added one by one, then the company itself, then the
  first of its people). Ask the frequency (letters that left or are leaving)
  and the quiet days (since the launch) again before every letter.
- A list may fill itself by rules: saved filters of CRM sections counted with
  the section's own code (see "Counting a section's filter outside its page"
  in [crm-card-blocks.md](crm-card-blocks.md)) hourly by a job and before
  every campaign by the list; a rule adds only records that never were in the
  list, so a member removed by hand stays removed.

## Mailing services and delivery events

Verified with `skeeks/cms-communication` 1.19.6 and the driver package
`skeeks/cms-communication-unisender` 1.0.0 (test `tests/unisender.php`; real
sends and event exports on the stand and «Проверить подключение» with the
webhook on skeeks.com, 30.10.2026; two external reviews).

- A mailing service is a send-only mail driver
  (`mail\MailingServiceInterface`) behind an ordinary mailbox, not a parallel
  sender: campaign letters, "Написать" and "Отправить клиенту" from that
  mailbox all go through `MailComposeService::queue()` and the mail send job,
  so feeds, rights, the opt-out and link marks need nothing new. Receiving,
  "Sent" copies and a manager's working mailbox skip such mailboxes.
- The driver enforces the service's rules at the last moment, whatever sent
  the letter: an advertising letter (with a topic) only to contacts with
  consent (`MailingPolicy::MODE_SERVICE`), never a letter of a campaign of
  managers' mailboxes. Its campaign channel asks the same mode, so cold
  contacts are excluded in the summary with the reason.
- One API call per letter with our ids in the service's metadata (letter,
  site, installation, campaign, recipient); the service's job id is kept on
  the letter (`provider_ref`).
- Several installations share one account of a service (a stand with a copy
  of the site's database, two sites), so letter ids collide. An event belongs
  to a letter when its job id equals the letter's `provider_ref`; a letter
  with an unknown outcome (no job id) takes only an event of its own
  installation and site. The installation id is random, made once and kept in
  a file of the installation (`runtime/…`), never in the database (a copied
  database would carry it) and never derived from the DSN (two Docker
  installations share `host=db`). A refused match never falls back to the job
  id, and an event of another installation touches no stop list.
- Everything a person wrote stays literal for the service's template engine:
  break `{{`/`}}` (HTML entities, a zero-width space in text and subject)
  before adding the service's own tags (`{{UnsubscribeUrl}}`). Switching the
  engine off may break the service's own headers (Unisender Go `Sender`).
- A service campaign never leaves through an ordinary mailbox, checked at the
  channel and again in the mail send job (the mailbox may be switched to
  another connection after the launch), as the reverse is checked by the
  service driver.
- "Sent" means only that the service took the letter. A refusal "for now"
  (429) re-queues the letter for a limited time; a server error or a broken
  connection is an unknown outcome and is never resent.
- Webhooks: limit the body before the signature (Content-Length, bounded
  read of the input, gzip unpacked piece by piece under the same limit),
  check the signature, store the body once and answer at once (services
  retry for a day and then stop the webhook), apply by a coalescing job that
  works in portions until nothing waits, retries a failed call later and has
  recovery attempts; an agent wakes it for waiting calls (a run lost to a
  crash after the service got 200), a repeated call wakes it too. Clear the
  raw body once applied (readers' IP and device), keep its hash against
  repeats. The rollback of the table refuses while calls are new or failed.
- «Проверить подключение» compares the webhook with the exact address of this
  connection: the core passes the connection to the driver
  (`ConnectionDriver::getConnectionId()`, set by
  `CmsCommConnection::createDriver()`, not a setting). One writer of events (`services\DeliveryEvents`): an event is unique
  per provider and event key (retries carry new bodies), the letter's state
  follows its first "To" only and never goes down in rank, a bounce reaches
  the stop list only when the address itself is dead (not our sender's
  reputation), a complaint always, an opt-out through the service's link
  opts out of the letter's topic. Consequences apply only when the event's
  address is a recipient of our letter.
- A service may replace the Message-ID: match a reply with an unknown parent
  to our latest letter from a service mailbox to its author with the same
  subject without "Re:".
- A stand gets no webhooks: read the same events through the service's
  event export and record them with the same writer.
- Features a service must allow (Unisender Go: no tracking `track_*=0`, no
  own opt-out block `skip_unsubscribe`) are settings that default to the safe
  value and say in their hint that letters fail until the support allows it;
  until then the service's opt-out tag replaces our link, so the letter has
  one opt-out line, not two.

## The audience of the new CRM model: cards, not deals

Verified 2026-10-28 with `skeeks/cms-communication` П5 on the skeeks.com
Windows stand (test `tests/audience-stand.php`; a base of 14 rows and pasted
addresses imported into a list, a campaign by the list with a presentation,
a real view of the presentation link making the company warm with a call
task). Words: «сделка» = `CmsLead` ([crm-entity-names.md](crm-entity-names.md)).

- A cold base is an audience of companies and contacts in a list, never deals:
  a deal comes from interest (a reply, a request; `CampaignTasks`)
  or from the manager. An import has an explicit mode chosen before the
  columns: a base into a list, or real requests into deals (a found client
  with an open deal the importing employee sees gets a feed entry of the
  repeated request; an open deal hidden from him is never written to — the
  row becomes a new unsorted deal without the import executor; otherwise a
  new deal, unsorted unless a responsible is chosen).
- Every row of a base goes through the one core duplicate check
  (`CrmMatcher::findMany()`, [crm-duplicates-and-triage.md](crm-duplicates-and-triage.md)).
  Only one strong match (email, phone, INN) links; several matches, or a name
  only, leave the row out for a person — never a new duplicate. A found card is
  added to the list as it is: the import never changes its responsible,
  contacts or details.
- Rows left out keep their reason, counted in the preview by reason: a
  client of another manager (shown as name and responsible only, no id), a
  current client (an active service or a recent payment), a recent refusal
  (a pause after a rejected deal, not a sorted-out request), every email
  opted out or on the stop list. The preview and the run use the same
  outcomes; the preview writes nothing.
- A new card: a company when the row names one, a contact linked to it
  (contacts go to the person, else to the company), the responsible chosen
  in the import — «Я» by default or another employee, never none (a CRM
  client always has a main responsible) — no password or
  invitation, the mailing source and the cold
  state in the package's own table (the core card has no source field), the
  topic with the cold basis on its contacts. A contact without a name gets its
  main email/phone through the core setter (`User::$email`), so its card is
  named by them. Pasted addresses become the same import of lines split by
  `CrmContactParser`: a list never holds a bare address.
- A signal of interest (the first counted visit by a link of a letter, the
  first counted view of a presentation or an offer) marks the recipient's
  company or contact warm once, with the campaign and the letter, and the
  campaign report counts it. An opening of a letter is only a report mark.
- The flow is always a campaign (a bot through MCP/API, a mailing, a
  mailing service; a bot names the campaign on every send): a deal is made
  by a person by hand, and by the flow only by the campaign's rules. The
  campaign holds the responsible of NEW companies (the sender — by default,
  for a bot the user of its authorization — or an employee; never none), what makes a deal (a reply and a request from
  the letter always; a view of the presentation or offer and a visit by a
  link by checkboxes), who gets the deal («Неразобранное», the sender, an
  employee), its pipeline, and the call task on an event that makes no
  deal. One rule (`CampaignTasks`) for every event: the client's only open
  deal continues (a task in it, a view or a visit — a feed entry) and moves
  forward to the stage of the event; otherwise a new deal, once per
  recipient and kind by its source reference, at the stage «Ответил» /
  «Смотрел» of the pipeline (the core `OutboundSales` names them). The flow
  takes an existing company as it is, skips a protected client of another
  main responsible at launch and gives such a client's deal to its main
  responsible; a deal of the flow never changes the main responsible of the
  client. The event runs in one transaction under `SELECT … FOR UPDATE` of
  the client row: two events at once make one deal
  (`tests/flow-race-stand.php` — parallel processes).
- Entry points of the rule live in the packages that record the event and
  call it by the letter: the offer package (`Campaigns::onOfferView()`,
  `onOfferRequest()`), link visits (`onLinkVisit()`), the mail receive
  (replies), the API send of a draft with `cms_comm_campaign_id` (the letter
  and its recipient row join the campaign). A company a bot creates without
  explicit managers gets the campaign's responsible of new companies
  (`Campaigns::newCardOwnerId()`), a company its creator may not open is
  reported created with `visible: false`, not as an error.
- «My defaults» of the flow are the core CRM settings, set «Исходящие
  продажи» (`CrmSettings::$outgoing*`, beside the pipeline and stages of
  personal offers): a new campaign copies them and may change them, a bot
  without a campaign works by them. A package reads such a core setting
  with `property_exists()` and a fallback for an older core; a conditional
  `required` of such a form needs `whenClient` too, or the browser demands
  the field always.
- Old deal members of lists move to the deal's company, its contact, one
  card found by its contacts, or a new contact — repeatable, the deal member
  marked removed with its origin copied, the deal untouched.
- The core never references the audience: the import of a base is a package
  feature added to core sections through `actionsMap` (a tab of "Компании"),
  and signals are wired inside the packages that record them.
- Code that reads a new column or table of the audience asks one schema
  check first (`CmsCommImport::audienceReady()`): the code may reach a site
  before its migrations, then imports stay requests and the audience actions
  (the tab, pasted addresses, warm signals) are off instead of failing.
