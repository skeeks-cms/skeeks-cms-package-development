# Action tools for AI clients (MCP/REST): prepare, people send

Verified on skeeks.com, 24.10.2026 (`skeeks/cms-communication`
`services\AiActions`, `services\AiMark`, `mcp\CommunicationActionToolProvider`;
`skeeks/cms-mcp` `McpComponent`; `skeeks/cms-proposal`
`ProposalSendableSource`). `tests/ai-actions.php` of cms-communication is the
executable check. Read tools are in [ai-read-api.md](ai-read-api.md).

## AI prepares, a person acts

- An AI action creates something a person finishes: a draft in the
  responsible employee's "Черновики", a draft campaign with personal letters.
  No tool sends a letter to a client or launches a campaign; say so in the
  tool description, so a client does not look for a way around it.
- Save only with `confirm=true`. Without it the tool checks every row and
  returns what would be done (`requires_confirmation`), saving nothing.
- Every action runs as the OAuth user: `forManager()` for records, the
  user's own send rights for a chosen mailbox, the section right for
  campaigns. Never accept another user, a mark or a status from arguments;
  unknown argument fields are ignored, not applied.

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

Limits are component configuration (`perCall`, `perDay` per CRM user in the
CRM time zone, `recentDays`). Count the day from the marks the action leaves
(drafts, letters sent from them, campaign letters) instead of a new counter
table. The journal is `ApiLogService`: `McpComponent::execute()` logs every
call with `oauth_client_id` and the user; the action adds an entry with the
created ids and refusal reasons in the category of its transport, so the
site's log routing keeps it with the call.

## Testing

Run the tools through `McpComponent::executeTool()` with an unsaved token
carrying a real `client_id`, in a transaction that is rolled back; create
the test clients inside it. Check the dry run saves nothing, the mark comes
from the token and ignores argument fields, each refusal reason, the limits,
sending from the draft as the screen does (mark, topic opt-out, offer send),
and that no tool name launches or sends.
