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
