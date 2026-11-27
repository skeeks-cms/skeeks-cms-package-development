# Site legal documents, consent checkboxes and local Google fonts (152-ФЗ)

Verified in source on 2026-11-19 (`skeeks/cms`, `skeeks/cms-module-form2`,
`skeeks/cms-theme-unify-v2`, `skeeks/theme-unify-shop`).

## Documents: the `legal` component

`skeeks\cms\components\LegalComponent` (component `legal`, settings
«Правовая информация») owns every legal document of a public site:

| Document | Route / URL | Field (own text) | Generated from |
|---|---|---|---|
| Политика конфиденциальности | `cms/legal/privacy-policy`, `/~legal-privacy-policy` | `doc_privacy_policy` | `template_privacy_policy` |
| Политика обработки файлов cookie | `cms/legal/cookie` | `doc_cookie` | `template_cookie` |
| Политика в отношении обработки ПДн | `cms/legal/personal-data` | `doc_personal_data` | `template_personal_data` |
| Согласие на обработку ПДн | `cms/legal/consent`, `/~legal-consent` | `doc_consent` | `template_consent` |

- A filled `doc_*` field wins; otherwise the template is filled by
  `fillTemplate()`: `{operator}`, `{site}` (falls back to the request host),
  `{email}`, `{details}`, `{contact}` («на адрес электронной почты …» or «по
  контактам, указанным на сайте») and `{url_cookie}`, `{url_personal_data}`,
  `{url_privacy_policy}`, `{url_consent}`.
- Templates describe a site of a shop/company (visitors, orders, delivery,
  Яндекс Метрика). Platform/SaaS wording does not belong in them.
- The short address `/~legal-<action>` is a project rule (present in
  `cms-basic-shop`, absent in `cms-basic`); links are always built with
  `Url::to(['/cms/legal/<action>'])`, which works without it. A new document is
  a new `LegalController` action and view; the side menu of documents is one
  partial, `views/legal/_menu.php`.
- Site switches in the same settings, fieldset «Защита персональных данных на
  сайте», both on by default: `is_local_google_fonts` (Google fonts from the
  site, read by `localFonts->isActive()`) and `is_auth_consent` (consent
  checkbox at login and registration). Component settings are cached without
  expiry and invalidated on save, so a change applies at once (except HTML
  already held in `PageCache`). Consumers read them with `isset()` /
  `method_exists()` so an older core keeps the old behaviour.
- The templates need the operator data («Правовая информация»: operator,
  details with address, e-mail for withdrawal). Without them the documents are
  incomplete; filling them is part of launching a site.
## Consent checkbox: one text, one target

Every checkbox of consent to personal data processing leads to the separate
consent document, not to a policy (since 01.09.2025 consent is a separate
document). The text comes from the core:

- `Yii::$app->legal->consentCheckboxText` — label HTML (setting
  `consent_checkbox_template`, `{url}` = consent URL); `consentUrl` — the URL.
- Consumers must keep working with an older core: check
  `method_exists(Yii::$app->legal, 'getConsentCheckboxText')` and fall back to
  the old text.

One checkbox for public auth forms: `legal->renderConsentCheckbox($id)`
(name `legal_consent`, `required`, classes `sx-legal-consent*`).

Consumers:

- Form constructor: `Form2Form::getLegalCheckboxText()` — empty
  `legal_checkbox_template` uses the core text; a custom template gets `{url}`
  = the consent URL (a link written into a custom text by hand stays as is).
  The cart (`cart-v2.php`) reuses the checkout form's text.
- Element ids (cms-module-form2 ≥ 3.0.2.17, theme-unify-shop ≥ 2.2.1.32): one
  page often has the cart and a modal form together, so only the cart checkbox
  keeps `id="sx-rules"` (project cart CSS targets it); a form-constructor
  checkbox gets `<widget id>-rules` with class `sx-rules`. Style and script form
  checkboxes by the class `.sx-rules`, never by `#sx-rules`. The cart script
  checks the `#sx-rules` that is not inside a `<form>`.
- «Купить в 1 клик» (`theme-unify-shop` `_product-price.php`).
- Site login widget (`cms-theme-unify-v2` `widgets\auth\AuthWidget`):
  `renderConsent($key)` adds the checkbox to the first step (phone, call-check
  phone, email), where an unknown contact also registers; the second step
  (password, code) has none. `is_show_consent` = null (default) follows the
  site switch `legal->is_auth_consent`; true/false overrides it in code.
- Registration pages: core `views/auth/register.php` and Unify
  `views/modules/cms/auth/register.php` (register by e-mail). Password login
  of an existing user (core `login.php`) has no checkbox. Both registration
  pages also follow `legal->is_auth_consent`.
- With an older core (no new methods) the widget hides the checkbox, the form
  constructor and «Купить в 1 клик» show the old text.

The checks are client-side `required` only: the server neither requires nor
stores the consent, so other clients of `/cms/auth/*` keep working. Recording
the fact of consent (time, document version, link to the request, order or
registration) is not implemented; see the platform plan.
## Analytics counters and the cookie window

Counter codes («Счётчики», `seo.countersContent`) are still printed on every
page regardless of consent, and the cookie window says «продолжая находиться
на сайте, вы соглашаетесь». Delaying counters until consent was postponed by
the owner on 2026-11-19 (promoted sites must not lose statistics). Do not add
consent gating of counters without a new owner decision.

## Google fonts from the site itself

`skeeks\cms\components\LocalFonts` (component `localFonts` in the common
config of `skeeks/cms`, so the console worker has the same settings;
bootstrapped only in web) keeps a copy of the Google fonts on the site. Goal:
simple, fast, stable; a page does only a file check.

- The copy lives in `<assetManager basePath>/sx-fonts/<md5(url)>/`
  (`fonts.css` with relative `url(<hash>.woff2)`). Pages link it by the direct
  static path; the web server serves it, PHP takes no part in loading fonts.
- No copy yet, or any internal failure: the page keeps the original Google
  link (owner decision 2026-11-19: the site must always look complete). The
  download is the cms-job type `cms.local-fonts` (core `jobRegistry`, queue
  `maintenance`, `transient`, `overlapPolicy` skip, 2 attempts; dedup and
  resource key = hash of URL + copies directory, so the same fonts on two
  sites are two jobs), handler `jobs\LocalFontsJobHandler`. cms-job and its
  workers exist in every project (owner); without the registered type nothing
  is queued and pages simply keep Google fonts.
- Queueing from a page: at most once per `retryAfter` (600 s) per URL — the
  `<key>.queued` mark is reserved atomically before `push()` (non-blocking
  `flock`, freshness re-checked under it, time written; only the request that
  wrote the mark pushes; a failed write means no push), so parallel cold pages
  and a broken queue do not repeat the DB insert; never during the failure
  pause (`<key>.failed`); at
  most `maxBuildsPerRequest` (3) per request. The next page after the copy
  appears links it. Clearing `assets` repeats this once; HTML already held in
  `PageCache` keeps its old path until the page cache refreshes.
- The handler: payload `url` and `dir`. A non-Google URL, a directory not
  named `<dirName>`, a parent outside `@root` after `realpath`, an `sx-fonts`
  that is a symlink or resolves outside `@root` is a `JobPermanentException`
  (no retry). Build settings come from the application's `localFonts`. Between
  steps it heartbeats and checks cancellation (`isCancelled()`, cached by the
  reporter for a few seconds); right before publishing it reads
  `cancel_requested_at` of its run directly (`build()`'s `$beforePublish`).
  A cancelled build publishes nothing and leaves no failure mark. A download failure is an ordinary error (retry)
  and leaves `.failed`; the job itself ignores the failure pause (`force`).
- `build()`: one build per key under `flock`, waiting at most `lockWait`
  (20 s; a busy lock is not a failure); `buildTimeLimit` (30 s) counted after
  the lock, each download limited by the time left; one curl handle per build (`curl_reset()` keeps the DNS cache and open connections: a Google set has dozens of files, and on a server with a slow resolver a lookup per file broke the time limit); no redirects; size limits
  while downloading (CSS, each file, the whole set); every write checked; the
  CSS is published only without external `url()`/`@import`; a temporary
  directory is renamed into place; cleanup errors are logged, not thrown.
  Old copies and service marks (`.queued`, `.lock`, `.failed`) are not removed
  automatically; clearing `assets` removes them. Worker and PHP-FPM must be able to
  write the same `assets` (same user or group); the lock file is `0666`.
- Rewriting changes only what the browser loads as styles: the `href` value
  of a real `<link>` tag (attributes are parsed in order with their quotes,
  so text inside another attribute is never taken for `href`; the first
  `href` decides; the whole value must be a Google CSS URL) and the address of
  a real `@import` directive in the CSS body of a `<style>` block (CSS
  comments and strings, e.g. `content`, and the attributes of the opening
  tag are skipped). Tag names are matched exactly
  (`<link-preview>` is not `<link>`); attributes are matched with quotes (`>`
  inside a quoted value does not end the tag). Left as is: HTML comments,
  text blocks `<title>`, `<textarea>`, `<script>` (unclosed ones up to the end
  of the document), opening tags of all other elements with their attributes
  (`data-template` etc.), other attributes of `<link>`, other `url()` in CSS,
  form values (e.g. `font_css`). On a PCRE error the original HTML is
  returned. The site setting is read only when the page has a Google fonts
  link. `Response::EVENT_BEFORE_SEND`, format HTML, not a stream:
  `response->data` and, when `data` is null, `response->content` (PJAX,
  `PageCache`). JSON and files are not touched.- Assets compression: `View::EVENT_END_PAGE`, attached first, acts only when
  the compressor downloads and merges external CSS itself (`enabled` +
  `cssFileCompile` + `cssFileRemouteCompile`): with a copy the Google entry of
  `$view->cssFiles` becomes the copy (merged with correct font paths by
  `prependRelativePath`); without a copy it stays. Without remote compilation
  (the default) fonts stay a separate `<link>` as before; JS is not touched.
- Switches: `localFonts.enabled` (config) and the site setting
  `legal->is_local_google_fonts` (on by default); `isActive()` needs both. An
  already queued job still runs after the setting is turned off (harmless:
  pages stop linking the copy). Code that needs Google fonts keeps writing
  ordinary Google URLs; do not hand-copy font files into packages.
- Not covered: Google URLs inside external CSS files, requests made by
  scripts, `dns-prefetch`/`preconnect` hints written by projects.