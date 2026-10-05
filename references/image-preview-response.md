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

## JFIF sources and thumbnail consumers

Imaging accepts `.jfif` as a JPEG source. When WebP is disabled, normalize the
output extension to `.jpg`; retain `ext=jfif` in the signed parameters so
ImagePreviewController reconstructs the original source path. With WebP enabled,
use `.webp` and the same original-extension parameter. Do not ask Imagine to
encode `.jfif`. The CMS test `tests/jfif-preview.php` verifies actual 60x60
JPEG/WebP generation, first-response byte identity and existing-format URLs
without bootstrapping a site or using its database.

Templates consuming thumbnailUrlOnRequest must constrain their image layout:
unsupported source formats deliberately return the original URL. The Unify
left-news item reserves 60x60 pixels with object-fit: cover even for that fallback.
After a template change, invalidate the relevant widget TagDependency using the
real web cache configuration; console and web FileCache paths can differ.
