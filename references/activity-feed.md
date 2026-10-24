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
