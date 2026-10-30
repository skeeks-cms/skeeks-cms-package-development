# Messenger bots (Telegram, MAX) in cms-communication

Verified in `skeeks/cms-communication` stage 5.1 (`tests/bots.php` on
SQLite with a fake messenger, `tests/bots-stand.php` on the stand database
with rollback). Messenger facts (limits, headers, update types) and the
rollout of a site's bot belong to the package `ARCHITECTURE.md`, section
"Мессенджеры и боты".

## Driver boundary

A messenger driver package (`cms-communication-telegram`, `-max`) extends
`bot\MessengerDriver` (a `ConnectionDriver`) and only speaks the protocol:

- `updateKey(array)` — a repeated delivery gives the same key, different
  updates never share one. Telegram has `update_id`; MAX has no update id,
  so the key is composed (`update_type` + message `mid`, + `timestamp` for
  edits, `callback_id`, `chat_id` + `timestamp` for a start);
- `parseUpdate(array)` → `bot\BotEvent[]` in the common form (ids are strings);
- `send(BotOutgoing)` → the message id; one message (a text part within
  `textLimit()`, or one file);
- `downloadFile()`, `setWebhook()/getWebhook()/deleteWebhook()`, `poll()`,
  `chatInterval()`, `webhookSecretHeader()`, `getMe()`;
- optional capabilities have a refusing default in `MessengerDriver`, so a
  driver declares only what its API gives: `reactionChoices()` (empty — the
  CRM shows no reaction picker) and `react()`; `webhookOutdated(getWebhook())`
  — true when the subscription lacks update types the driver now asks for,
  then `BotSubscriptions::ensure` sets the webhook again (a driver adding an
  update type needs no manual resubscription on sites).

A reaction event (`BotEvent::REACTION`) carries the person's whole current
set of reactions on the message (`reactions`, empty = removed), not a
delta: Telegram sends it so, and replacing is idempotent for repeats.
Reactions are kept apart from messages (`chat\ChatReactions`) and are not
client-history entries; they touch the message `updated_at` so an open chat
redraws it.

An API refusal (nothing was sent) is `bot\BotException` (`blocked`,
`retryAfter`); any other exception means the outcome is unknown, and a send
is never repeated then. Token, webhook secret and proxy are encrypted
connection secrets (`secretAttributes()`), see `external-service-secrets.md`.
Conversations, identities, files, links, notifications and rights are never
implemented in a driver.

When updates do not carry a group title (MAX), the driver answers
`chatTitle()`; `BotIngest` asks it once for a new chat and goes on without a
title if the call fails. The API address and a proxy are bot settings: the
documented MAX address uses a Russian Trusted Root CA certificate that PHP
does not trust by default (verified 24.10.2026), so the driver defaults to
the address that works and a site may switch. Tokens go only to the API host:
file hosts (uploads, downloads) get no Authorization header. Telegram puts the
token into every address (`/bot<token>/…`, files `/file/bot<token>/…`): a
Telegram driver never logs or throws an address and clears the token from API
and transport errors.

## A messenger the server cannot reach

Verified with `cms-communication` 1.17.0 and `cms-communication-telegram`
1.0.0 on skeeks.com, 25.10.2026 (Telegram from a server in Russia, both ways):

- Outgoing: the bot setting "Прокси для запросов к API" (`proxy`, an encrypted
  secret, may carry `user:pass@`) is the only channel of every driver request:
  sends, polls, webhook calls, getMe and file downloads. Use a CONNECT proxy
  restricted to the API host, so TLS stays end to end.
- Incoming: the bot setting "Адрес приёма вебхука" (`webhookBaseUrl`, HTTPS)
  makes `Bots::webhookUrl()` give `<relay>/~comm-bot/<id>`; the hourly
  subscription check keeps it. The relay forwards the path, the body and the
  secret header unchanged; the site checks the secret as usual, so a relay
  holds no secret. The messenger must trust the relay certificate with its
  full chain (Telegram reports `certificate verify failed` otherwise).
- Without a relay the bot works in the poll mode through the same proxy.

A new bot connection has no `cms_comm_account_access` rows: nobody, not even
an administrator or an OAuth client of the CRM tools, sees its chats until
rights are granted per employee (verified on skeeks.com, 25.10.2026).

## A bot is an account of its channel

A bot connection gets a `cms_comm_account` with the driver's channel
(`bot\Bots::account()`), so employee rights (`cms_comm_account_access`),
visibility through `forManager()` and "default" work as for a mailbox.
Consequence for every consumer of the shared account table: code that means
mailboxes must ask `CmsCommAccount::findMailboxes()` (channel `email`), not
`find()`; otherwise a bot appears among mail folders and "От кого". The
"default" flag is unique per site AND channel.

## Durable receipt, then applying

- The webhook (`POST /~comm-bot/<connection id>`) checks the secret header
  with `hash_equals`, stores the raw update with one insert (unique
  connection + update key; a repeat is not written) and answers 200 at once.
  A failure to store answers 500 so the messenger delivers again. Nothing
  is applied in the web request: MAX allows 30 seconds and unsubscribes a
  bot after 8 hours of failures.
- A poll stores its batch and the protocol cursor in one transaction
  (`external-change-streams.md`); webhook and poll are mutually exclusive.
- The job `comm.bot.apply` (resource per bot, `coalesce`) applies rows in id
  order, each in its own transaction with its journal row; a failure is
  retried up to 5 times, then stays `failed` with the reason.
- Message identity is `<connection id>:<chat id>:<message id>` per site
  channel. The bot must be in the key: Telegram numbers messages per bot and
  a private chat id is the person's id, so two bots of one site get equal
  chat and message ids from one person (verified by `tests/bots.php`). An edit of
  an unknown message creates it (edits may arrive first); an imported
  history never doubles a message and never reaches handlers.
- Event handlers of other packages (`communication.botHandlers`,
  `bot\BotEventHandlerInterface`) run inside the event transaction: they
  must be repeatable and may only queue external work. A side effect that
  is not the message itself (a notification) catches and logs its own
  failure: an exception there rolls back the received message, and the
  update fails five times and is lost from the chat (skeeks.com 28–30.10.2026:
  emoji in a sender name hit the utf8 `cms_web_notify`; `cms` migration
  m261030_120000 converts it to utf8mb4). Every table that stores people's
  names or texts must be utf8mb4.

## The bot never hears itself

Messengers send no updates about the bot's own messages, so whatever the bot
says goes through `bot\BotSend::queue()` (message + job in one transaction),
or it is missing from the CRM. A long text goes in parts; "too many
requests" keeps the parts already sent in the message and requeues the job
(`JobContext::requestRequeue()`), so only the rest is sent.

## A stand with production bots

There are no test bots: the stand uses production ones, so
`communication.botReceive = false` there — no webhook, no poll, no cursor
moves (either would take updates away from production) — and
`communication.allowedChats` limits sending (empty with `allowedRecipients`
set — nothing leaves). Production tokens in the copied database cannot be
decrypted on a stand with its own secret key.

## Dialogs, visibility, notifications ("Чаты")

Verified with `cms-communication` `tests/chats-stand.php` (stand database,
rollback) and in the browser, 2026-10-24:

- Chat visibility repeats mail: read "all" on the bot account, or the chat
  is taken by / written to by the employee, or it is linked (whole-chat
  `cms_comm_conversation_link`) to a record visible through `forManager()`
  and the bot exposes linked access. Writing: "any" — every visible chat,
  "own" — taken or linked ones. Administrators follow their own rows.
- "Взять диалог" is the conversation's responsible person; a dialog taken
  by a colleague is passed explicitly (the receiver must see the chat and
  is notified), never taken over silently.
- The dialog state lives on the conversation: `waiting_since` starts with
  the first unanswered client message and ends with any message of ours
  (from the CRM or an employee's own account in a group); a client message
  reopens a closed dialog for the same employee.
- Personal read marks are one row per chat and employee (last read message
  id), not per message: chats are long.
- Notifications are a bot event handler (`communication.botHandlers`), one
  unread `CmsWebNotify` per employee and chat whose text is updated by later
  messages; groups stay silent unless somebody leads the dialog. The
  notification refers to the conversation (`modelsConfig` → the chat
  controller, whose `view` opens the chat in the panel).

## Whose a chat is: links, feed, people

Verified with `tests/chat-links-stand.php` and in the browser, 2026-10-24:

- A chat belongs to records as a whole (`cms_comm_conversation_link`), not
  message by message. The feed follows the kind of chat: a private dialog
  puts every message into the record's feed (the ordinary message link and
  log entry), a group writes one summary per chat and CRM day that every
  new message of the day recomputes. Every writer of chat messages (receipt,
  sending, history import) calls the feed hook, so links made earlier keep
  working for later messages.
- A person is linked once and their private chats follow: a client is kept
  on the identity, a lead or company on the chats; a new private chat of a
  linked person (another bot of the site) inherits the records.
- Identification by events: a one-time start key (no personal data, used
  once) and the person's own phone verified by the messenger. The chat of
  the event must be passed explicitly — a chat opened by "Start" has no
  messages of the person yet, so "chats where the person wrote" misses it.
- Notifications of a chat nobody leads go to the people of linked records
  who can open it (company and client managers, the lead's executor).

## Testing a webhook in a stand test

Build a `yii\web\Request` with the path, raw body and header, and set it as
`Yii::$app->request` (and a fresh response) before `runAction()`: the
controller reads the application's request, so a request object that only
resolved the route makes every secret check fail.

## Work from chats (tasks, "Отправить клиенту")

Verified with `cms-communication` `tests/chat-tasks-stand.php` (stand
database, rollback, fake messenger) and in the browser, 2026-10-24:

- A chat message is an ordinary record source `message`: the source checks
  chat visibility with `chat\ChatAccess` (not `forManager()`, which knows only
  mailboxes) and fills a task from the chat's whole-chat links and the
  person's client link. A group message that became a record is also linked
  to the chat's records on its own: the day summary does not say which
  message became a task.
- A command in a chat (`/task …`, `/task` in answer to a message, a mention
  of the bot) is a bot event handler. It creates the task through the same
  source factory, keeps a package row (chat, command message, task) so a
  repeated event does nothing, and answers through `BotSend::queue()`; a
  refused send (a stand) never fails the task. In a job there is no signed-in
  user: set `created_by` explicitly and again after saving (blameable).
- Telling a chat about task statuses is a class-level `CmsTask`
  `EVENT_AFTER_UPDATE` listener that only queues a bot message, remembers
  the last status told and never throws into the task's save. Only tasks
  asked for in the chat report back, and the chat can switch it off.
- "Это наш сотрудник" is the identity's `cms_user_id` pointing to an active
  employee: their messages are ours, so dialog state and sides follow.
- "Отправить клиенту" through bots is one channel for all messengers: chats
  are a contact kind of `SendContacts` (chats linked to the client's records
  and private chats of people linked to the client, only those the employee
  may write to); the personal link is a URL button, files go as separate
  messages after the text.

## Chats for AI clients (MCP/REST)

Verified with `cms-communication` `tests/chat-mcp-stand.php` through
`McpComponent::executeTool()` (stand database, rollback), 2026-10-24:

- Chat tools take the rights of the OAuth employee from `chat\ChatAccess`,
  never `forManager()`: a mailbox query does not see bot chats. The same goes
  for client-history describers of feed entries that point to chat messages,
  otherwise a chat message reads as "a letter of a mailbox you cannot open".
- A reply is checked, then sent: the check returns exactly what the client
  gets (the bot's signature line included) and a token of user, chat,
  answered message and text; the send recomputes it, needs a new
  idempotency key (the shared `cms_comm_api_send` table, so a status tool
  must tell a chat send from a letter) and `confirm=true`. The AI mark is
  written by the server from the OAuth client.
- Grant model: preparing (`write`) never implies sending; sending to a
  messenger reuses the existing client-messaging scope instead of a new one,
  because a new scope means a `cms-mcp` release and re-authorizing clients.

## Order of messages: time, not id

An imported history gets ids in the order it was stored, and a messenger
gives history pages from the newest back, so ids of imported messages are not
in time order (verified 2026-10-24, `cms-communication` 1.16.2: a group read
in pieces and scrolling up never reached its start). Show and page a chat by
`message_at` with the id as a tie-break, take the "last message" of a chat by
time, and poll for new messages from the largest id of the chat, not from the
last one shown. An import collects all pages first and stores them oldest
first, so new imports are numbered in time order too.
