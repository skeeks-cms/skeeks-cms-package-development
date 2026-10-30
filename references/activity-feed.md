# Activity feed entries

Every entry of a CRM activity feed (lead, company, client, deal, task,
project, worker, the "Активность" log and the UPA support card) is drawn by
one frame: `skeeks/cms` `src/widgets/admin/views/_log-list-item.php`, rendered
by `CmsLogListWidget` and styled by `CmsActivityAsset`
(`cms-activity.css`). Packages give only the content.

## Content contract

`CmsLog::entry()` returns a `skeeks\cms\log\CmsLogEntry`:

- `icon` — a `BackendIcon` name of the event type;
- `tone` — the icon tile color: `neutral`, `accent`, `success`, `danger`,
  `warning`, `info` (`TONE_*`);
- `title` — plain text: "Исходящее письмо", "Входящий звонок";
- `badges` — trusted HTML from `CmsLogEntry::badge($label, $tooltip, $tone)`;
- `summary` — trusted HTML main line: the subject link, the proposal name;
- `details` — trusted HTML muted lines: "Кому: …", "Длительность: …";
- `snippet` — plain text, clamped to three muted lines;
- `body` — trusted HTML: comment text, a recording, chat lines.

The frame owns everything else: the time ("24 окт., 17:07", the year only
for other years, the full date in the tooltip), the author (18px avatar and
short name, a link to the person), the records the entry belongs to
(`sx-model` is hidden by `CmsLogListWidget::$is_show_model = false`), the
pinned mark, attached files and the actions. Do not print the time, author,
type label, pin or menu inside package content.

## The card feed of a company or a client

The feed of a company or a client person (`CmsCompany::getCardLogs()`,
`User::getCardLogs()`) is the record's own entries plus the entries of its
deals (`CmsLead`), as in Bitrix24:
- `CmsLogQuery::clientDealsCondition($record)` takes the deals of the client
  as an explicit id list, so the `(model_code, model_id)` index
  (`cms_log__model_code_id`) is used, not a scan of all deal entries;
- an event written to the client and to its deal is shown once: a deal entry
  is skipped when the client has a twin of the same type — the same `data`,
  or the same `data.id` for letters and calls (the company copy of a call
  keeps more fields). Only `phone_call`, `message`, `proposal` and
  `comm_visit` are compared: comments and changes are never twins, their
  `data` is often `[]`;
- the level of the employee decides through `crm\CrmAccess::feedCondition()`
  (see `crm-access-levels.md`); the letters and chats a feed entry opens
  follow the same rule in cms-communication (`CrmRecords::communicationQuery()`):
  for «Свои сделки» and "edit" the correspondence of the client itself is
  hidden both in the feed and in the mail and chat lists;
- pinned: `notPinnedOf($record)` leaves out only the card's own pinned
  entries (they are shown above the feed); a deal entry pinned in its deal
  stays in the feed as a normal entry;
- `CmsLogListWidget::$ownModel` = the card record: with
  `is_show_model = false` only its own link is hidden; a deal entry keeps a
  visible link signed «Сделка №927 Ромашка» (`name_one` of `modelsConfig`, the
  number, `asText`), which opens the deal in the backend window.

The «Активность» tab (`getLogs()`) has the deals' entries too, with the same
condition. Its other kinds (services, bills, payments, tasks, projects or
bonuses) also go in as id lists (`CmsLogQuery::feedIds()`), not subqueries:
with a chain of `OR … IN (subquery)` MariaDB merged the single-column indexes
over all ~70 000 entries and sorted them (150–740 ms per page). With lists it
reads `cms_log__model_code_id` (a few ms). A kind with more than
`FEED_IDS_LIMIT` records stays a subquery. The feed set did not change: old
and new ids were compared for 31 companies and 21 clients on a production
copy (30.10.2026).

Feed renderers are singletons of one request (`Skeeks::getLogRenderer()`).
A renderer that checks access per entry keeps its answer for the request,
keyed by the employee and the record: a group chat gives a digest a day, so
a card asked the same chat check 40 times. Examples:
`ChatDigestLogRenderer` (chat and `canRead`), `MessageLogRenderer` (the
letter under `forManager()`). Apply this only under
`CrmAccess::keepsAnswers()`, so the console and workers read every time.
Never keep an answer across requests or employees.

A package log type registers its renderer in `Skeeks::$logRenderers` (see
`model-activity-log.md`) and extends `skeeks\cms\log\CmsLogEntryRenderer`,
implementing only `entry(CmsLog $log)`. Its `render()` returns the same
content as a frameless HTML fragment for older callers. A renderer that
implements only the old `CmsLogRendererInterface::render()` still works: the
frame shows it as the body of a generic entry with the `info` icon and the
log type label, so migrate it rather than adding frame markup to it.

Core types are built in `CmsLog::entry()`: comment, phone call (in/out,
answered green, failed red with "Пропущен"/"Не дозвонились", duration,
recording), insert, update and delete.

## Icons and tones

Use one outline icon per event type from `BackendIcon` (24px grid, 2px
stroke, `currentColor`); add a missing one to the registry, never inline SVG
in a renderer. Feed icons: `comment`, `phone-in`, `phone-out`,
`phone-missed`, `mail-in`, `mail-out`, `sms`, `globe` (site message),
`chat`, `bot`, `file-text` (proposal created/version), `send` (sent), `eye`
(opened), `check-circle` (accepted), `shield` (link check), `pointer-click`
(site visit), `megaphone` (campaign), `plus-circle`, `edit`, `trash`,
`sparkles`, `pin`, `history`.

Tone rule: actions of the client (incoming letter or message, a visit, an
opened or accepted proposal, an answered call) are `success`; our actions
(an outgoing letter, a sent proposal, a bot reply) are `accent`; failures
are `danger`; technical events (created, changed, a comment, a link check)
are `neutral`. In the default palette `info` equals the accent, so do not
use it to tell incoming from outgoing.

Long service explanations go into a badge tooltip, not into a line: "Подготовлено
ИИ" with `AiMark::text()` as the tooltip.

## Actions and pinning

Pin, copy-link and the record menu (`AjaxControllerActionsWidget` on a
`<button>`) are borderless `sx-icon-action` icons in the entry header. They
appear on hover and `:focus-within`; under `@media (hover: none)` they stay
visible at reduced opacity, so phones keep every action. A pinned entry has
an accent inset left edge, a filled accent pin and "Закреплено" ("В
результате задачи" for task logs) in the meta line. The comment form pin is
the quiet `sx-comment-pin-toggle` next to "Прикрепить", never a framed
button. `cms/tests/cms-activity-surfaces.php` guards this markup.
