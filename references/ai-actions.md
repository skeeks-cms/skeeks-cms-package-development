# Action tools for AI clients (MCP/REST): prepare, people send

Verified on skeeks.com, 24.10.2026 (`skeeks/cms-communication`
`services\AiActions`, `services\AiMark`, `mcp\CommunicationActionToolProvider`;
`skeeks/cms-mcp` `McpComponent`; `skeeks/cms-proposal`
`ProposalSendableSource`). `tests/ai-actions.php` of cms-communication is the
executable check. Read tools are in [ai-read-api.md](ai-read-api.md).

## AI prepares, a person acts

- An AI action creates something a person finishes: a draft in the
  responsible employee's "Черновики", a draft campaign with personal letters.
  No tool launches a campaign; say so in the tool description, so a client
  does not look for a way around it. Sending one letter through the API is a
  separate, separately granted step (below).
- Save only with `confirm=true`. Without it the tool checks every row and
  returns what would be done (`requires_confirmation`), saving nothing.
- Every action runs as the OAuth user: `forManager()` for records, the
  user's own send rights for a chosen mailbox, the section right for
  campaigns. Never accept another user, a mark or a status from arguments;
  unknown argument fields are ignored, not applied. The one exception is a
  draft of an employee the user manages (below): then the draft's owner is
  taken from the stored draft, never from an argument.

## The same path as the screen

Reuse the service the UI uses and run the UI's validation: a draft is
`MailDrafts::save()` for its owner, after the owner's `ComposeForm` was
validated with the service attached (mailbox, recipients, "own clients
only", topic policy, attached documents). A recipient is
`Campaigns::addRecipient()`. A rule written only in the AI service drifts
from the screen; a rule in the shared form holds for both.

## Checks and refusals per row

A batch call returns one row per client: `status` (`ready`, `created`,
`refused`), `reason` (a stable code) and `message` (text for people); one
refused row does not stop the others. Whole-call errors (no topic, over the
per-call limit, no section right) are exceptions. Typical checks: the client
is visible, has a contact, the contact is allowed by the mailing policy for
the topic, no recent letter or send of the same offer (`recentDays`), nothing
already waiting for the client (an AI draft, a campaign), the offer belongs
to this client, one row per client in a call.

A document package answers "when was this document last sent to this
record" through an optional interface of the sending package
(`send\SendHistoryInterface::lastSentAt()`), so the mail package stays free
of the document package.

## "Подготовлено ИИ"

- The mark is data of the record (`prepared_by=ai`, `prepared_client`
  `mcp:<OAuth client name>`, `prepared_user_id`, `prepared_at`), set only by
  the server. The client name comes from the token of the running tool:
  `McpComponent::getCurrentClientName()` (valid only inside `execute()`).
- It moves only along trusted links: a letter takes it from the sender's
  own draft (looked up by the owner, so a foreign draft id gives nothing), a
  campaign letter from its recipient row, a document send from the letter's
  send context into its feed entry data.
- A person's change does not remove it: the record says "changed by an
  employee" (`prepared_edited` by a fingerprint of the AI subject and text;
  `text_edited_by` for a campaign letter). Who sent stays the employee
  (`sent_by_user_id`).
- Show it where people look: the drafts list, the letter form ("check and
  send yourself"), the letter page, the card feed ("Подготовлено ИИ (client)
  по запросу … · отправил …"), the campaign card, the document's sends. Use
  the shared `sx-status` label, no new global CSS.

## Limits and journal

Limits (`perCall`, `perDay` per CRM user in the CRM time zone, `recentDays`,
`sendPerDay`) and a site-wide switch of API sending belong to whoever
manages that area — a package permission of its own, given to the
administrator role by default (RBAC in the package config), not a hard
check of the administrator role: a settings page of the package on the site component
settings ([component-settings.md](component-settings.md)) wins, the
application configuration is the fallback for an empty field, then the
defaults; one reader returns the value and its source, and the page shows
both. Production configuration is not deployed, so a limit only in config
cannot be changed there. A switched-off send refuses the check and the send
with a reason that says where to turn it on; preparing still works. Count the day from the marks the action leaves
(drafts, letters sent from them, campaign letters) instead of a new counter
table. The journal is `ApiLogService`: `McpComponent::execute()` logs every
call with `oauth_client_id` and the user; the action adds an entry with the
created ids and refusal reasons in the category of its transport, so the
site's log routing keeps it with the call.

## Sending through the API: a checked draft, an idempotency key

Verified 24.10.2026 (`services\AiSend`, `cms_comm_api_send`,
`tests/ai-send.php`).

- Separate OAuth grants: preparing (drafts, campaigns, topics, checks) and
  sending are different scopes; a grant given for preparing never sends.
  Do not reuse an existing "send" scope that old tokens already hold for
  other purposes — they would gain sending silently.
- Send only a saved draft, never arguments: what was checked and shown to
  the person is what leaves. The draft is the OAuth user's own or of an
  employee he manages (next section); anyone else's is refused with a
  stable reason (`not_own`) whose message says who may work with it.
- The check reads the draft through the screen's path (fill the letter
  form, validate it) plus the API rules, with no side effects (no letter,
  no personal link, no opening; a dead button in the preview), and returns a
  fingerprint of everything the client will see. The send recomputes it and
  refuses on any change ("check again").
- Idempotency: a row per (site, user, key) with the fingerprint, written in
  the transaction that queues the letter — either both exist or neither. The
  same key and data return the first result; other data with the key is
  refused; a refused send does not take the key; a parallel repeat waits on
  the unique index and returns the first result. A client that lost the
  answer reads the status by the key instead of resending.
- Report only what is known: queued, sending, sent (accepted by the outgoing
  server), failed, unknown (never resent). Mail gives no delivery proof:
  "not confirmed" unless a bounce came back. Openings come from the
  document package (honest views, link checks apart); reading a status
  opens nothing.
- Use the package's existing operation for the letter (`MailComposeService::
  queue()`: personal link, opt-out, feed, links, job), never direct SMTP or
  an own tracker. A document that cannot be sent now answers before the
  letter (`send\SendRefusalInterface`), and still refuses when the link is
  made.
- Reference data the send needs (mailing topics) is managed by API with the
  section's right; a send never creates it, and an empty list says what to
  set up.

## Drafts of managed employees: acting as the owner

Verified 03.11.2026 (`services\DraftOwners`, `AiSend`, `AdminCmsCommMessageController`,
`tests/draft-owners-stand.php` of cms-communication).

- Who is available is the core rule, not a copy: `CmsUserQuery::managedBy()`
  over `CmsDepartment::subordinateIds()` — subordinates by departments,
  every employee for an administrator, inactive ones included (their drafts
  must stay reachable); only an active employee (or the actor) receives a
  draft. No consent of the employee: the leader reads, edits, checks, sends
  and hands over the drafts himself. One service answers it for the screen
  and the API alike (find, list, transfer, refusal reasons).
- Working with such a draft means acting as its owner: the letter form, the
  mailbox send rights, "own clients only", visibility of the client, the
  signature and the day limit of sends are the owner's (`MailDrafts`,
  `MailComposeService`, the check of the owner). Never give the owner's
  rights to the actor or the actor's mailboxes to the owner's draft. Every
  rule that asks "who acts" must get the owner explicitly — a rule that
  reads the session user (the right to act on a client, a deal) checks the
  actor instead. A stored draft keeps its owner on a change: never fall
  back to the mailbox or responsible of the client; no mailbox — a refusal.
- What links a draft to letters and cards (the answered and forwarded
  letter, its attachments, the client) comes from the stored draft when
  the actor is not the owner, never from the request: the owner's rights
  would otherwise open the owner's letters to the actor. Suggestions in
  such a window (recipients, documents) are asked as the owner, for a draft
  the actor may reach.
- One draft makes one letter: the send locks the draft row
  (`SELECT … FOR UPDATE`) inside its transaction and compares what it
  checked (owner, mailbox, last change, files) before queueing; a parallel
  send or hand-over then finds it gone or changed. The check token includes
  the owner. A window opened for another owner (the draft was handed over)
  neither saves nor sends; a window whose draft is gone sends nothing —
  never the actor's own letter instead.
- The letter stays the owner's (`sent_by_user_id`, his mailbox) and records
  who actually sent it (`acted_by_user_id`), set only in code from the
  session or OAuth user; the feed, the letter page and the API show
  "отправил … по поручению сотрудника …". An AI mark of a change made by
  the actor names the actor ("по запросу"), and his preparing limits count.
- The send row keeps both: the caller (idempotency keys and status reads
  stay his) and the owner of the draft (the day limit of sends counts
  against the owner's drafts, whoever sent them). Every channel that shares
  the limit (letters, chat messages) reads the same counter. Preparing
  limits of the actor are asked before a change that counts it today; test
  mode asks the actor's role.
- Handing a draft over changes only its owner, and only between employees
  the actor manages (himself included). The mailbox stays when the new
  owner may send from it, otherwise it becomes his default one; the former
  owner's signature block gives way to the new owner's; an untouched AI
  text stays untouched (its fingerprint follows the new signature).
- Mass actions on drafts (delete, hand over) take draft ids through the
  same service and never reach letter actions (read marks, moves, the row
  menu): the list says which kind its checks choose. A hand-over from an
  open window happens only after the last version is saved; a failed save
  leaves the window open. The API has the same hand-over tool, and every
  draft tool refuses with one structure (`success=false`, `reason`
  `not_found` or `not_own`, `message`).

## Purpose, files and test sends of an API letter

Verified 24.10.2026 (`services\LetterPurpose`, `services\AiDraftFiles`,
`tests/ai-letters.php`).

- The purpose is explicit in the request and checked against data, never
  inferred from the text, an individual text, a manual send or an attached
  document: advertising (topic, opt-out, mailing checks), personal (the
  client wrote within N days), service (active deal or paid bill). Personal
  and service take no topic and no opt-out link; complaints and bounces stop
  any purpose. The letter form validates the same purpose, so the screen and
  the API refuse alike. The purpose is part of the check fingerprint and of
  the send journal.
- Files are referenced by storage id, never by path or URL of the server:
  only the OAuth user's own files of the site, the type by content
  (`finfo`), count/size limits, a copy in the package's own storage so
  removing a file from the letter never touches the user's upload, and the
  same content is not added twice. Pictures in the text go the screen's way
  (an upload token in the HTML → `cid:` part at send). The fingerprint takes
  the files' content hashes, not one-time upload tokens.
- An upload tool must never pass a caller's string to a storage that treats
  relative strings as local paths: allow only public http(s) URLs, resolve
  and pin every hop to public IPs, cap the size, refuse script extensions.
- A test mode is a server list of exact record–address pairs in site
  configuration (no tool edits it), for administrators only. It skips only
  the repetition checks and keeps rights, address, purpose, files,
  confirmation, fingerprint, idempotency, day limits and suppression; it is
  shown in the check, part of the fingerprint and journaled. No `force`.
- Whether a letter carries the opt-out line and List-Unsubscribe is decided
  in one place of the letter form (`ComposeForm::withOptOut()`), not by "has
  a topic": an individual offer keeps its topic for the opt-out checks
  (stop list, topic opt-out, complaint, bounce, repetition) and shows no
  opt-out; a campaign letter always carries it, so an individual offer is
  refused there. Never relax a check because the visible link is absent.
- What the sender adds by hand in the screen (the mailbox signature of the
  employee) is added by the server to an AI draft the same way, once, inside
  the text — the check preview then equals the letter and the fingerprint
  covers it. AI never invents such parts.

## Testing

Run the tools through `McpComponent::executeTool()` with an unsaved token
carrying a real `client_id`, in a transaction that is rolled back; create
the test clients inside it. Check the dry run saves nothing, the mark comes
from the token and ignores argument fields, each refusal reason, the limits,
sending from the draft as the screen does (mark, topic opt-out, offer send),
and that no tool launches. For sending also check: a token without the send
scope is refused, the check creates nothing, a change after the check
refuses, the same key returns the same letter, another payload with the key
is refused, the states and a bounce, and that reading the status opens
nothing.
