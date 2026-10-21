# Rich-text editor field (`RichTextInputWidget`)

Verified 2026-10-21 on the skeeks.com Windows stand: task description, site
section description, CRM comments (admin and the UPA support cabinet), the
letter form of `skeeks/cms-communication`, content elements and products,
storage file descriptions, FAQ answers and the `cms-shop` site, store and
supplier forms. The widget replaced `skeeks/yii2-ckeditor` (removed, see
`backend-ui-assets.md`).

## Parts and owners (all in `skeeks/cms`)

| Part | Class / file | Role |
| --- | --- | --- |
| Field widget | `widgets\formInputs\richText\RichTextInputWidget` | toolbar, `contenteditable`, hidden textarea with the value |
| Assets | `RichTextAsset` (`rich-text.js`, `rich-text.css`) | registered only by pages with the field; a page that inserts editors later (AJAX) depends on it; depends on the sx core (`skeeks\sx\assets\Core`): `rich-text.js` adds `sx.RichText`, and loaded before the core it leaves a bare `window.sx` that stops the whole backend (blank page on forms rendered before the layout registers the core) |
| Server cleaning | `helpers\RichTextHtml` | tag/style allowlist by preset, `toText()`, `fromText()`, `uploadTokens()`, `replaceUploads()`, `isEmpty()` |
| Temporary files | `services\TemporaryUploads` | `put()`, `putBytes()`, `path()`, `store()`, `forget()`, session-bound tokens |
| Upload endpoint | `controllers\AdminRichTextController` (`cms/admin-rich-text/upload`, `import`, `file?t=`) | raster images only, 15 MB, admin access |
| Remote pictures | `services\RemoteImageDownloader` | SSRF-safe download of a pasted picture by URL |
| Model side | `behaviors\RichTextBehavior` | cleans changed attributes, stores pasted images on save |

`skeeks/cms-backend` was not changed: icons come from `BackendIcon` when it
has the name, otherwise from the widget's own glyphs (`RichTextInputWidget::icon()`).

## Presets: one value for the toolbar and the cleaning

A field's toolbar and its server cleaning must not diverge. Both take the same
preset (`RichTextHtml::PRESETS`, `RichTextInputWidget::PRESET_TOOLS`):

- `basic` — bold/italic/underline/strike, colors, clear, smileys, links,
  quote, alignment, lists. No headings, sizes, fonts, tables or images.
  Comments.
- `standard` — basic plus headings (Текст / Заголовок 1–4), text size and
  images pasted into the editor.
- `full` — standard plus tables kept from pasted or old texts. Task
  descriptions, future content elements and products.

Options override a preset: `['preset' => 'full', 'images' => true]` also keeps
old pictures by http(s), site-relative or `data:image` URL (old CKEditor
texts); `tables`, `sizes`, `colors`, `images` (`false` / `"uploads"` / `true`).
The font family is opt-in (`'fonts' => true`, toolbar tool `font`): owner
decision 2026-10-21 — only letters choose a font. An explicit `tools` list
changes the toolbar only; keep the cleaning options in step with it.

## Field contract

- The value lives in a hidden textarea and is written only when the text is
  edited (`input` events, toolbar commands, paste, image insert). An untouched
  field posts its old value byte for byte, so `RichTextBehavior` leaves it
  alone (`isAttributeChanged()`); old CKEditor HTML is cleaned only when an
  employee actually edits it. Browsers post every textarea with CRLF line
  breaks, so a value that differs from the stored one only by `\r\n`/`\n` is
  treated as untouched and the stored bytes are put back. Empty text (no
  letters, no pictures) posts `''`.
- Every typed line is a paragraph (`paragraph` tag): browsers wrap only the
  lines after the first Enter, so the line being typed is wrapped with
  `formatBlock` when it is bare text or inline markup at the top level.
- Old HTML is loaded through `DOMParser` with scripts, embeds, event handlers
  and `javascript:` removed, so opening a text never runs anything from it.
- Paste from Word/browsers keeps simple formatting and links; a pasted
  screenshot or dropped picture is uploaded and inserted as
  `<img src="…file?t=token" data-sx-upload="token">`. When the clipboard has
  both text (HTML) and a file, the text wins: Word adds a picture of the whole
  selection that must not replace it.
- Pictures inside pasted HTML (text copied from another site, owner request
  2026-10-21) are copied to us as if chosen from a disk: http(s) ones through
  `cms/admin-rich-text/import` (`importUrl`; server download into
  `TemporaryUploads::putBytes()`), `data:image` ones through the upload
  endpoint; at most 30 per paste. A picture that cannot be copied is removed
  with one summary message, so a saved text never links to someone else's
  server. Fields without images (`basic`) drop pasted pictures.
- The server download (`services\RemoteImageDownloader`) treats the URL as
  untrusted (SSRF): http(s) on ports 80/443 only, no credentials, every
  resolved address must be public (`isPublic()`: no loopback, private,
  link-local, reserved, CGNAT, unique-local or IPv4-mapped IPv6), the checked
  address is pinned with `CURLOPT_RESOLVE`, redirects (max 3) are followed by
  hand and rechecked, 15 MB and 15 s limits, raster type by content only.
  Never replace it with `storage->upload($url)`, which fetches any URL.
- While files are dragged over the page the editor shows drop zones:
  "Вставить в текст" (images) and, with `attachLabel`, "Прикрепить файлом".
  The second zone (and non-image files dropped or pasted into the text)
  triggers the jQuery event `sx-rte:attach` with the files on the root
  `[data-sx-rte]`; the host form attaches them its own way (mail does).
- Forms are not submitted while pictures are uploading (capture-phase
  `submit` guard). `sx.RichText.isBusy(root)` is available for AJAX forms.
- The toolbar is `position: sticky` under the shell header
  (`--sx-shell-top-offset`; `--sx-rte-sticky-top` overrides). `rich-text.css`
  sets `.sx-main-wrapper:has(.sx-rte) { overflow: clip }` because the shared
  wrapper clips overflow; only pages with the editor get it. Windows and
  panels (empty layout, `body.sx-empty`) have no header but keep
  `--sx-shell-top-offset` set, so there the toolbar sticks at 0.
- The `fullscreen` tool (last in every preset) makes the root
  `position: fixed; inset: 0` with the text scrolling inside and a readable
  column (max 960 px); Esc or the button returns it. Esc is stopped with
  `stopImmediatePropagation()` so it does not also close a window or panel.
- Menus near the right edge shift left to stay on screen (narrow screens).
- Colors only from `--sx-color-*` (light and dark verified); `plain` drops
  the frame when the editor is part of a bigger form (the letter);
  `paragraph` = `p` (default) or `div` (letters, comments).
- JS API: `sx.RichText.init/initAll/load/focus/isEmpty/isBusy/insertImages/isComplex`.
  Editors inserted later start by themselves (MutationObserver).

## Server side

- Give the model `RichTextBehavior` with the field's preset, or clean the
  value with `RichTextHtml::clean()` where the record is saved (comment
  actions attach the behavior to the new `CmsLog`). Text without tags (plain
  text from an API) is never cleaned.
- On save the behavior moves each `data-sx-upload` image from the temporary
  area into the storage (`TemporaryUploads::store()`, current site) and
  replaces it with `<img src="/uploads/…">`. A token of another session or an
  expired one is a validation error ("Картинка … устарела"). Nothing reaches
  the storage before the save; abandoned files are removed by the existing
  `ajaxfileupload/cleanup` agent. A picture deleted from a saved text stays
  in the storage until the storage usage registry exists (`skeeks/cms`
  `AGENTS.md`, planned).
- Trusted site HTML (`'attribute' => false`): no allowlist, only pasted images
  are stored. `CmsTree` and `CmsContentElement` (products included)
  `description_short`/`description_full` are trusted: they hold page markup
  (classes, `details`, `iframe`, `svg`, scripts).
- `ComboTextInputWidget` (modes Текст / Визуальный редактор / HTML) has no
  CKEditor since 2026-10-21: its visual mode is always this widget with
  `preset => 'full'` and `trusted` (options in `richTextOptions`); a text
  with scripts, styles, embeds or event handlers stays locked there with a
  hint to edit it in the HTML mode. Pasting pictures is offered only when
  `RichTextBehavior::covers($model, $attribute)` — the model has the behavior
  for that field (trusted `false` is enough); otherwise nothing would move
  them out of the temporary area. Brands, deliveries, legal texts, content
  property values (`UserPropertyTypeComboText`) and other ComboText fields
  without the behavior therefore edit text only; give the model the trusted
  behavior to enable pictures there.

## Preset per field (decided 2026-10-21 from stand data)

| Field | Preset |
| --- | --- |
| Task description, FAQ answer (`CmsFaq.response`), storage file `description_full` | `full` + `images => true` |
| Storage file `description_short`, `ShopSite.description_internal` | `standard` + `images => true` |
| `ShopStore.description` (store and supplier forms; CKEditor offered only bold/italic/underline/strike) | `basic` |
| CRM comments | `basic` |
| `CmsTree`, `CmsContentElement` descriptions | trusted (`false`) |

Company, contractor and project descriptions are plain text in plain
textareas (no tags in 477 local companies); they never used CKEditor (it was
commented out) and stay textareas.

## Consumers with their own upload endpoint

A section with its own permissions or limits passes `uploadUrl` (JSON
`success`, `token`, `url`) and uses `TemporaryUploads` with its own prefix;
mail uses `cmsCommunication/admin-cms-comm-message/upload` (send right, 25 MB,
prefix `mail`) and stores files itself when the letter is queued.

## Verification

`skeeks/cms` `tests/rich-text-editor.php` (presets, behavior, untouched and
plain values, expired uploads; SQLite memory). In the browser: type, paste a
screenshot, save, check `cms_storage_file` grew only on save and the
temporary directory is gone; open an old text without editing and save — the
value must stay byte for byte; light, dark and a 375 px viewport.
