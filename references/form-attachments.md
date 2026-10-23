# Form attachments (`AttachmentsInputWidget`)

Verified 2026-10-23 on the skeeks.com Windows stand: the CRM comment form
(`CmsCommentWidget`: lead, company, client, deal, task, project, worker and
the UPA cards that render it) and the letter form of
`skeeks/cms-communication` (window "Написать", quick reply, "Отправить
клиенту"). Use it for files attached to a record; it replaces the bulky
`AjaxFileUploadWidget` block ("Выбрать файл" with a dropdown) there. Image
pickers (logos, avatars, galleries) are a different need and keep their widget.

## Parts and owners (all in `skeeks/cms`)

| Part | Class / file | Role |
| --- | --- | --- |
| Field | `widgets\formInputs\attachments\AttachmentsInputWidget` | chip list, hidden file input, chip template; `button()` renders the light "Прикрепить" button |
| Assets | `AttachmentsAsset` (`attachments.js`, `attachments.css`) | registered by the field; a page that inserts the field later (AJAX) depends on it (`MailClientAsset` does) |
| Endpoint | `controllers\TemporaryUploadController` (`cms/temporary-upload/attach`) | any file, 50 MB, signed-in users (admin and UPA), POST; `TemporaryUploads::put($file, 'att')` |
| Temporary files | `services\TemporaryUploads` | session-bound tokens, `store()` on save (see `rich-text-editor.md`) |
| Model side | `models\behaviors\HasStorageFileMulti` | stores tokens of the field on save; a stale token is a validation error of the field |

## Contract

- Files are uploaded at once into the temporary area; the chip keeps the
  token in `<name>[]`. Nothing reaches the storage before the record is
  saved. The field always posts `<name>=''` too, so removing every file is
  posted.
- `HasStorageFileMulti` (both `fields` and `relations`) recognizes a token
  (`TemporaryUploads::isToken()`) and moves it with `TemporaryUploads::store()`;
  numeric values stay storage ids. Any model with the behavior (`CmsLog`,
  `CmsTask` `fileIds`) therefore needs no controller code. Other consumers
  call `TemporaryUploads::store()` per posted token themselves.
- The host form places `AttachmentsInputWidget::button()` next to its send
  button (one row, like the letter). The button opens the field with the id
  in `for`, otherwise the first field of its form.
- Drop and paste area: the closest `form` (or `[data-sx-attach-zone]`) of the
  field, `drop => false` turns it off. Paste outside text fields only; a
  `RichTextInputWidget` of the area with `attachLabel` hands over its
  "Прикрепить файлом" zone, non-image files and, in a text without images
  (`basic`), pasted screenshots (`sx-rte:attach`). Give the editor
  `attachLabel` whenever the form has this field.
- A form is not submitted while files upload (capture-phase `submit` guard);
  AJAX senders check `sx.Attachments.isBusy(form)`. The event
  `sx-attach:change` on the field marks the form changed (mail drafts).
- Options for a host with its own rules: `uploadUrl` (same JSON: `success`,
  `token`, `name`, `size`, `sizeText`, `message`), `maxSize`,
  `maxSizeMessage` (`{name}`, `{size}` in MB), `name` + `storedName` when new
  uploads and stored files post under different names, `files` / `uploads`
  for chips of an opened draft, `listOptions` (the host's data hooks),
  `listContent` (the host's own chips; a chip with `sx-attach__file` looks
  like a file and keeps the list visible).
- The letter keeps its own endpoint (`upload`, mailbox size limit and the
  "почтовые серверы" message, prefix `mail`), its `uploads[]` / `file_ids[]`
  names and its "Прикрепить из…" chips (`MailUi::sendableChip()` uses the
  shared `sx-attach__*` classes). Do not fork the field per section.
- Markup is semantic `sx-attach*` only; colors from `--sx-color-*` (light,
  dark and 375 px verified).
