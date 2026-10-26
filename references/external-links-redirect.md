# External links transition page (`/~redirect`)

Owner: `skeeks/yii2-external-links` (component, controller, view).
`skeeks/cms-external-links` only wires it into the CMS: the `~redirect` URL
rule, the `externallinks` module, `CmsExternalLinksComponent` with settings
from administration ("Преобразование внешних ссылок"). Contract since 1.2.0
of `yii2-external-links`.

## Contract

- The component rewrites absolute links in the `<body>` of an HTML response to
  `/~redirect?url=<base64>&sign=<hmac>` and builds them only through
  `ExternalLinksComponent::createRedirectUrl($url)`. Any other code that needs
  a transition link uses the same method; never hand-build `?url=`.
- `sign` is `hash_hmac('sha256', 'externalLinks|' . $url, $key)` cut to 32
  chars; `$key` is `signKey` or the request `cookieValidationKey`. Pages in
  full-page cache stay valid while the key is unchanged; changing the key
  turns cached links into unsigned ones (still usable, see below).
- The page accepts only `isAllowedUrl()`: absolute URL with a host and a
  scheme from `allowedSchemes` (`http`, `https`), no whitespace, control
  chars or backslash, at most 4096 bytes, strict base64. Everything else is
  `400`. `javascript:`, `data:`, relative and `//host` values never render.
- A valid signature means the link came from this site: the page redirects
  automatically after 2 s (old behavior).
- No or wrong signature (old links from caches and search engines, or links
  made by somebody else) is never redirected automatically: the page shows
  the host, the full address and a "Перейти" button. This keeps old links
  working without an administration allowlist and stops the page from being
  an open redirect under the client's design.
- Output: `Html::encode` / `Html::a` in markup, `Json::htmlEncode` in JS.
  Hrefs found in the HTML are `Html::decode`d before signing, so `&amp;` in
  the source does not end up inside the target address.
- The page sends `X-Robots-Tag: noindex, nofollow` and the robots meta tag.
  Do not add `Disallow: /~redirect` to robots.txt: a blocked page cannot show
  its noindex to search engines.
- Not replaced: relative links, `tel:`/`mailto:` and other links without
  `://`, the current host, `noReplaceLinksOnDomains`, links failing
  `isAllowedUrl()`.
