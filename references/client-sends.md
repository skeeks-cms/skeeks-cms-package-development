# Sending a document to a client ("Отправить клиенту")

A commercial proposal, a bill, an act, later client notifications are sent to
the client through any channel: a letter, an SMS, later Telegram and MAX. The
send is one shared point of `skeeks/cms-communication`; the package that owns
the document only describes it. Do not build a send form, a channel or a
personal-link mailer inside a document package.

## Contract

- Register a source in the package config:
  `'components' => ['communication' => ['sendables' => ['<type>' => Source::class]]]`
  (`CommunicationComponent::$sendables`, registrations merge; type
  `[a-z][a-z0-9_-]*`; a document reference is `<type>:<id>`). The class
  implements `skeeks\cms\communication\send\SendableSourceInterface`:
  `title()` (menu name), `find($id, $user)` (the document only if the employee
  may send it — the package owns that right), `suggest($user, $q, $record,
  $limit)` (items `id`, `title`, `hint` for "Прикрепить из…", those of the card
  `$record` first).
- The document implements `send\SendableInterface`: `getSendTitle()`,
  `getSendText()` (short editable text), `getSendButton()` ("Открыть КП"),
  `getSendFiles()` (`SendFile::storage($cmsStorageFile)` or
  `SendFile::path($path, $name, $type)`; also called for the preview, keep it
  cheap or cached), `getSendRecords()` (lead, company, client, deal: their
  contacts are the recipients, the send lands in their feeds),
  `createSendLink(SendContext)` and `afterSend(SendContext, CmsCommMessage)`.
- One send = one `createSendLink()` (the package stores its pending send with
  its own code and returns an absolute URL; the context carries channel id,
  recipient — lower-case email or phone digits `79991234567` — name, CRM record
  and employee; `$context->ref` is free for the package) and one `afterSend()`
  with the same context and the created message, all inside the transaction
  that creates the message. An exception in the package rolls the whole send
  back; a request refused by checks never reaches the package. The message is
  queued, not delivered: read its status later, do not treat `afterSend()` as
  delivery.
- Channels are `communication.channels` (`send\SendChannelInterface`: title,
  icon, contact kind, `unavailableReason($user, $siteId)`, `renderForm()`,
  `send(SendRequest)`). The channel decides the look and creates the message;
  messenger drivers add channels without touching documents.
- UI: `MailUi::sendOnClick('<type>:<id>', $record)` opens the window
  (`admin-cms-comm-message/send`, empty layout only — never `_sxb[noa]`).
  Without UI (API, bots, AI): `SendService::request($ref, [...])` then
  `SendService::send()`; refusals are `SendException::$errors` by field.

## Rules learned from the first implementation

- Verified with `cms-communication` (2026-10-23), the test sample
  `tests/support/SampleSendable.php` and `tests/send.php`; two real letters
  from a stand mailbox.
- Mail reuses the ordinary letter form and `MailComposeService::queue()` for
  both the window and "Прикрепить из…" (`ComposeForm::$sendables`, kept by
  drafts): the personal link is a placeholder `https://sx-send.invalid/<type>/<id>`
  in the editor (the shared rich-text cleaner keeps only `https?:`, `mailto:`,
  `tel:` links and drops styles), turned into an inline-styled table button on
  send; a removed placeholder still gets the button at the end. Never put
  styled markup into editor HTML expecting it to survive cleaning.
- Several recipients in one letter are one send to the first "Кому" address.
- SMS is a `cms_comm_message` of channel `sms` in a conversation per phone
  (`thread_key = sms:<digits>`), so it is in the feeds like a letter; the job
  `comm.sms.send` calls the core `SmsHandler` directly with an explicit
  `user_ip`, because `CmsSmsMessage` defaults read the web request and fail in
  console workers. Keep the provider row id in `provider_ref`.
- A stand with a production copy limits phones too: `allowedPhones`; when
  `allowedRecipients` is set and `allowedPhones` is empty, no SMS leaves.
  Check again in the job, before the provider.
- Phone keys of `to_list` decode as PHP integers: cast with `(string)` before
  comparing or formatting.
- "Почта" lists only `channel = email`; SMS and messenger messages live in feeds
  and conversations.
- A new action from a deal (a letter, an SMS, "Прикрепить из…", a proposal
  and its sending) goes through `services\LeadActions::error($user, $lead)`
  (verified 2026-10-29, `tests/lead-actions-stand.php`): who works on the deal
  (core `CrmAccess::isLeadWorker()`), and a closed deal only for who acts on
  its client anyway (`canActOnClient()` without a deal: an administrator,
  "full", the main responsible). «Свои сделки» and "edit" return the deal to
  work first. `isLeadWorker()` says who belongs to the deal, not whether the
  action is allowed by its state: never use it alone for a new action. Check
  the card of the send and every `CmsLead` among `getSendRecords()`
  (`SendService::leadsError()`); models of a document check their own deal
  for the acting employee (`LeadActions::actingError()`), so the API and the
  UI refuse alike. Reading the history of a closed deal is not limited.

## Formatted text

`getSendText()` stays plain text: SMS and messengers send it as is. A
document whose text has formatting (a presentation written in the shared
rich-text editor) also implements `send\SendableHtmlInterface::getSendHtml()`;
the letter of "Отправить клиенту" starts with that HTML cleaned to its own
editor preset, and an empty HTML falls back to `getSendText()`. Return the
same text without tags from `getSendText()` (`RichTextHtml::toText()`).

## A document for everybody

A document may have no client of its own (one presentation sent to many
clients, verified with `cms-proposal` 2026-10-23). Store the client on the
package's send row from `SendContext::$record` (the CRM record of the
recipient's address, otherwise the card the window was opened from) and
refuse with `SendException(['sendable' => …])` when there is none: every
later event of that link (feed, notification, task) needs a client. Write
those events to the send's client, show such documents in a client card by
its sends, and show the state of the sends to that client, not of the whole
document.

## A real sendable: proposals (verified 2026-10-23)

`skeeks/cms-proposal` (`send\ProposalSendable`, `services\ProposalSends`),
real letter №1893 from the stand mailbox:

- Mail calls `createSendLink()` and only then `getSendFiles()`; SMS never
  asks for files. Refuse a document that must not go out (empty, expired)
  in `createSendLink()` with `SendException(['sendable' => …])`: the window
  and the letter form (`compose`, since this release) show it, nothing is
  created.
- Keep the PDF of the preview on disk (`@runtime`, keyed by version and
  day) and store the same file with the version in `afterSend()`. Date an
  unsent version "today" in the PDF so the attached and the stored files
  are equal. `Storage::upload()` moves the file and returns the
  `StorageFile` base model: upload a copy and keep only the id.
- Set the sender explicitly from `SendContext::$user` after saving the send
  row: blameable reads the web user and is empty in jobs and the API.
- The personal page of a sending is described in
  [client-link-pages.md](client-link-pages.md).
