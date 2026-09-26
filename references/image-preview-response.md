# Image preview response contract

The CMS thumbnail filter owns resizing and encoding; the preview controller
must serve its saved bytes unchanged using Yii response sendFile (inline).
Reopening a saved image with Imagine show() performs a second lossy encode
and may use a different quality from the saved preview.

Explicit Thumbnail q is part of filter configuration and signed preview URL.
When q is omitted (or zero), generation resolves seo.img_preview_quality;
it does not automatically enter the URL or invalidate existing static files.
Preserve this distinction when changing the response path.

Verify encoding support against the deployed Imagine version and actual image
driver. A shared local vendor can contain an older Imagine than the site.
The CMS package owns tests/webp-quality.php for real-library regression checks
without site bootstrap or database access.
