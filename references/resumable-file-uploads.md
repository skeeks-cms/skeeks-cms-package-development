# Resumable file uploads

`skeeks/yii2-ajax-file-upload` owns the reusable browser-to-temporary-storage
upload protocol. Project forms may configure chunk size and timeout, but should
not reimplement chunk assembly.

## Protocol invariants

- Send a large file as bounded binary requests. Web-server and PHP body-size
  limits apply to one chunk, not to the complete file.
- Derive the temporary upload namespace from stable, user-scoped metadata:
  session, widget identity, sanitized basename, declared size and browser
  `lastModified`. Do not use a random per-selection ID as the only directory
  key; selecting the same file after a reload must find the existing partial.
- Treat the server-reported temporary-file size as the authoritative next
  offset. The client must not assume that every request appended its requested
  byte count.
- Append only when the requested offset equals the current server size. Reject
  mismatches and data beyond the declared total instead of producing a corrupt
  file.
- Sanitize the client filename to a basename before constructing a path. Keep
  partial files marked with `-loading`; rename to the final temporary name only
  after the assembled size exactly matches the declaration.
- Scope a completed temporary file to the authenticated session and stable
  widget/file identity. Cleanup of abandoned partials remains a separate,
  bounded retention task.

## Form integration

Use the big/chunked tool explicitly for fields expected to receive large
documents. Disable the form's save action between uploader `startUpload` and
`endUpload`; otherwise validation sees an empty hidden value while the browser
is still transferring chunks. Show a visible progress/waiting state and enable
save only after the server returns the completed temporary-file value.

## Verification

Check JavaScript syntax and PHP syntax, then exercise: a normal small upload, a
multi-chunk upload, interruption plus reselection of the same file, an offset
mismatch, and final-size equality. Confirm that retrying continues one partial
file rather than creating multiple `*-loading` files.

## Private files in the CMS storage

Verified 2026-10-21: `Yii::$app->storage` (`skeeks/cms` `components\storage\Storage`)
has only public clusters (`ClusterLocal` under `@webroot/uploads/all`), so every
`CmsStorageFile` is reachable by its path. `upload()` takes an `UploadedFile`,
a local path (the file is moved) or a URL, never raw bytes: write bytes to a
temporary file first. In queue and console code pass `cms_site_id` in `$data`;
the type is detected by the storage, keep it rather than a declared one.

For files that are not public (letters, client documents), never render
`src`/`absoluteSrc`. Serve them from an action that checks access to the owning
record and sends `rootSrc` with `Response::sendFile()`: show inline only
raster images by the stored type, send everything else as a download with
`X-Content-Type-Options: nosniff`. Reference: `skeeks/cms-communication`
`AdminCmsCommMessageController::file()` and `services\MailFiles`. A private
cluster outside the webroot would be a `skeeks/cms` configuration decision.

## Uploads before the owner is saved: the temporary area first

Owner decision of 2026-10-21, for every form that takes files before its
record exists or is saved (a letter, a task, news, an editor with pasted
images): an upload never goes straight to the storage. It waits in the CMS
temporary upload area, the `ajaxfileupload` module `private_tmp_dir`
(`@frontend/runtime/ajaxfileupload` for web and console alike), one random
directory per upload. The file moves into the storage only when the owner is
saved, and becomes the owner's file there (`storage->upload($path)` moves it).
A closed form, a removed attachment or an image deleted from the text leaves
nothing in the storage: the daily agent `ajaxfileupload/cleanup`
(`cms.cleanup-temporary-files`, `CleanupTemporaryFilesJobHandler`) removes
temporary files older than one hour. Do not add a second cleanup mechanism.

- The standard `AjaxFileUploadWidget` + `HasStorageFile(Multi)` behaviors
  already follow this flow; `admin-storage-files/store-uploaded-files` is the
  generic bridge from temporary paths to storage ids.
- A custom upload endpoint (an editor, drag and drop, paste) uses the shared
  `skeeks\cms\services\TemporaryUploads`: `put($file, $prefix)` saves into
  that directory and returns an opaque token `<prefix>-<32 hex>/<name>`, never
  a path; `path()` accepts only tokens of this session whose real path stays
  inside the temporary root; `store($token, $data)` moves the file into the
  storage when the owner is saved; `forget()` drops a token used otherwise.
  Serve previews only to the uploading session.
- Images inside rich text reference the token (`data-sx-upload`), are replaced
  on save by the stored file's reference (`RichTextBehavior`, see
  `rich-text-editor.md`), and are cleaned by the same agent if the text is
  abandoned.
- Shared endpoint for editor images: `cms/admin-rich-text/upload` and
  `file?t=` (`AdminRichTextController`). A section with its own rights or
  limits keeps its own action on `TemporaryUploads` (mail:
  `MailFiles::storeTemp()` with prefix `mail`, `storeFromTemp()` with
  same-content reuse).
- Drafts that must keep files longer than the cleanup window are owners of
  their own: store the files with the draft, not in the temporary area.
