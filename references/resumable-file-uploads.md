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
