# Link preview card (og:image)

Messengers (MAX, Telegram) and social networks show `og:image` of a shared
link; without it they fall back to the bare site logo. `skeeks/cms` gives
every ordinary page a picture: the record's own picture when it has one,
otherwise a generated 1200x630 card.

## Ownership

- `skeeks/cms` owns the mechanism: component `ogImage`
  (`skeeks\cms\components\OgImage`, registered in `src/config/web.php`), the
  GD renderer `components/og/OgImageRenderer.php` and the bundled fonts
  (static Nunito ExtraBold/SemiBold instances, OFL).
- A site owns only the style: colors, glows, logo, plate, site name in the
  `ogImage` component config of the project. Do not fork the renderer in a
  project or a theme. SkeekS sites take the logo from `skeeks/skeeks-brand`
  (`@vendor/skeeks/skeeks-brand/brand/logos/skeeks/logo-on-dark-128h.png`,
  `logoPlate => false`).

## Flow and invariants

1. `TreeController`, `ContentElementController` and `SavedFilterController`
   register the record picture as `og:image` (keyed `'og:image'`) as before,
   and always call `ogImage->setPage(seoName, label)`. Label: parent section
   for a tree page (none for level 1), the element's section or content name,
   the filter's section.
2. `skeeks\cms\web\View::endPage()` calls `ogImage->apply($view)` before the
   head is rendered, so every controller and template has already had its
   chance to give its own picture.
3. `apply()` works only for a web GET, non-AJAX/PJAX, HTML, status 200 page
   whose controller is not a `skeeks\cms\backend\BackendController` (admin
   and cabinets get nothing).
4. Own picture wins: `og:image` registered under the key or without a key (old
   templates) suppresses the card. `twitter:card=summary_large_image` is added
   in both cases. The card adds `og:image`, `og:image:width`,
   `og:image:height`, `og:image:type`.
5. Without `setPage()` the title comes from `$view->title` with the
   `" | Site"` suffix cut, so project controllers get a card too.
6. The file is `@webroot/uploads/og/<2 chars>/<md5>.jpg`, the hash covering
   `OgImage::TEMPLATE_VERSION`, the whole style, the logo file and its mtime,
   the title and the label. It is drawn once (40-80 ms with GD, atomic
   rename) and then served as static. Requests with a query string only reuse
   an existing card and never draw one: search and filter titles must not
   fill the disk. The folder may be deleted at any time.
7. Any change of drawing in `OgImageRenderer` must bump `TEMPLATE_VERSION`.
   A style change in config needs nothing: the hash changes by itself.
8. Requirements: GD with FreeType (`imagettftext`). No Imagick and no
   headless browser. Without FreeType the card is skipped silently; drawing
   errors are logged and never break the page.

## Choosing your own picture in a template

Register it with the key, `registerMetaTag([...], 'og:image')`; an unkeyed
tag is still detected but a keyed one also replaces the controller's record
picture instead of duplicating it.
