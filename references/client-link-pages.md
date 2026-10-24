# Client pages by a personal link

A document sent to a client (a proposal; later bills, acts) opens on a
public page of the site by the secret code of one sending, without signing
in. Verified 2026-10-23 with `skeeks/cms-proposal` (`ProposalController`,
`/~kp/<code>`) on the skeeks.com Windows stand: a real letter, a browser,
`curl`, light page at 375 px.

## The page

- One row per sending with its own code (`CmsProposalSend::$code`, 24
  random bytes, url-safe): the view tells which channel and recipient it
  came from. The code is the only access, as in `cms-shop` `ShopBillController`;
  answer 404 for an unknown code and for a pending row the send never
  completed (no message id).
- A frontend controller of the package with a URL rule in the package web
  config (`'~kp/<code:[A-Za-z0-9_-]{16,64}>' => 'module/controller/view'`).
  Rules from package configs come after the CMS tree rule
  (`UrlRuleTree`), whose fallback resolves a multi-part path by the code of
  its last part: a fixed last part that may equal a section code (`tools`,
  `context`) is captured by that section. A random code is safe; fixed
  endpoints (API paths) must be prepended in a bootstrap with
  `urlManager->addRules($rules, false)`, as `skeeks/cms-mcp`
  `McpBootstrap` does. A `~` prefix alone does not help.
- Do not render it through the site's page events. Site packages hook
  `View::EVENT_BEGIN_PAGE/END_BODY` class-wide (theme scripts, cookie
  banner, generated keywords) and `cms-seo` appends `countersContent`
  (Metrika) and `header_content` to every HTML response whose `data` is a
  string. A fresh `View` instance does not help: the handlers are
  class-level. Use a layout without `beginPage()/head()/beginBody()/endBody()/endPage()`
  that prints its own `<link>`/`<script>` from
  `assetManager->getBundle(...)->baseUrl`, and return the page as
  `Response::$content` with `$data = null`.
- Headers: `X-Robots-Tag: noindex, nofollow, noarchive`, `Cache-Control:
  no-store`, `Referrer-Policy: no-referrer` (the code is in the address);
  the same robots meta in the page.
- Show the version that was sent by this link; a newer sent version is a
  notice with `?v=latest`, and every action of the page (`pdf`, signals,
  answers) carries the same parameter so it acts on what the client sees.
- PDF: the file stored with the version at the first sending; render only
  when it is missing.

## Honest "opened"

- Opening the link is not a view: mail link checkers, antiviruses and
  Telegram/MAX previews fetch it first. Count only a `POST` signal from the
  page script after the page has been visible for a moment
  (`document.visibilityState === 'visible'`, re-armed on `visibilitychange`).
- Robots that run scripts are matched by User-Agent and stored as robots
  without counting (history for support). Do not match broadly on
  `yandex` or `bot`: the Yandex app and Cubot phones are people.
- A repeat of the same browser (IP + User-Agent) within a window is not a
  new view; a signed-in employee with administration access is not a view.
  Say so on the page for that employee: on a stand the link opens in the
  browser signed in to the CRM, and a silent "not opened" looks like a bug.
- The first view of a sending is detected with a conditional update
  (`first_viewed_at IS NULL`), so two tabs cannot both be "first"; only
  then write the feed entry and the notification.

- Link checkers that run scripts in a real browser (Outlook Safe Links,
  Proofpoint, Yandex and Mail.ru antispam) still send the signal. Pass the
  signal through `skeeks/cms-communication` `services\LinkChecks::reason()`
  with the time the message actually left (`message_at` of a sent message,
  not the queueing time), the IP, the User-Agent and the page's browser
  signals (`navigator.webdriver`, the number of languages, the screen size:
  not the window, phone in-app browsers report a zero window). Keep such an
  opening in the history with its reason, write "возможно, проверка ссылок"
  to the feed once per sending, and give it no status, notification or
  task. Do not list public clouds as checker networks: clients use VPNs
  there. Verified with `cms-proposal` (stage E): an opening 18 seconds
  after a real letter left was marked `early`.

## A link that leads to another page

A document whose content is a page elsewhere (a presentation: a page of
the site, a case study) uses the same personal link and the same honest
view, then leads on. Verified with `cms-proposal` presentations
(2026-10-23, `views/proposal/presentation.php`, `services\PresentationLinks`).

- The target address is stored in the document by an employee (http(s)
  with a host, a site path completed with the site address) and never taken
  from the link: the page must not become an open redirect.
- The page script sends the view signal once the page is visible, then
  `location.replace()`s the target (Back does not return to the transition;
  a short timeout leads on if the signal hangs). Without scripts there is
  no view and no transition, only a plain "Перейти" link to the target.
- Add `utm_source=crm`, `utm_medium=<channel>`, `utm_campaign=<document
  kind>-<number>` to the target, keeping marks (any case) and the fragment
  already in the address, so visits are not direct ones in Metrika.
- Actions of the full document page (accept, ask, PDF) answer 404 for such
  a document; an expired one shows a notice with the manager's contacts and
  does not lead on.

## Client actions

- An irreversible client action ("Принять") asks for confirmation on the
  page, is allowed only for the newest sent version and a waiting, valid
  document, and is a conditional update (`status != accepted`): a second
  press returns "already done" and writes nothing.
- A question is an incoming message of the conversation through
  `skeeks/cms-communication` `services\SiteMessages` (channel `site`, no
  mailbox, linked to the document's CRM records so it is in their feeds),
  plus the package's own row for the card. Protect it with a hidden field
  answered like success and limits per link, per IP and per document.
- Notify the responsible employee through `CmsWebNotify` with
  `model_code`/`model_id` of the document (see
  [backend-notifications.md](backend-notifications.md)).

## Shared signal code

The page signal and the browser signals are one script of
`skeeks/cms-communication`: `assets\LinkSignalAsset` (`link-signal.js`,
`window.sxLinkSignal.whenSeen(delay, send)` — `send` returns a promise, a
failure re-arms on the next `visibilitychange`; `browserSignals()`). A page
with its own layout prints it before its own script
(`LinkSignalAsset::scriptUrl()`). The robots list is
`LinkChecks::ROBOT_PATTERNS` / `isRobot()`; adding marks to an address is
`LinkMarks::addParams()`. Do not copy them into a package. Verified
2026-10-24: the page of an offer counts a view through the shared script.

## Visits of our sites by links of letters

Every letter leaves through `MailComposeService::queue()`, so that is the
only place that marks links (`services\LinkMarks`, verified 2026-10-24 with
real letter №2148 from the stand mailbox, a browser, `curl` and a page of a
second installation):

- Only the text written now is marked, before the quote, the forwarded
  letter, the personal links of sendables and the opt-out link are added:
  earlier letters keep their codes, a link that already has `sx` is never
  touched, a resend reuses the stored body. Paths starting with `/~`
  (personal pages) and other hosts, `mailto:`, `tel:`, anchors stay. Missing
  `utm_*` are added, existing ones (any case) stay; there is no redirect.
- The code (`cms_comm_track`, 16 url-safe characters) is one per letter and
  its first "Кому" recipient with the recipient's CRM record; it is written
  in the transaction of the letter, and only when a link was marked.
- The site script (`assets\VisitAsset`) is registered by the package
  bootstrap on `View::EVENT_END_BODY` only for a GET page with `?sx=` or the
  visit cookie, never in a backend; the site's script compressor puts it in
  a separate compressed file, other pages are unchanged. It removes `sx` from
  the address bar at once, counts the visit only through `whenSeen()`, keeps
  the visit token in a first-party cookie (`sx_visit`, session) and reports
  next pages by the token (`POST /~comm-visit/seen|page`, fixed paths
  prepended in the bootstrap).
- `skeeks/cms-seo` canonical addresses redirect away unknown parameters on
  some sites (seen on skeeks-platform.ru): the bootstrap adds `sx` with
  `seo->canUrl->ADDminor_pname()` on frontend requests. A site on another
  installation must do the same before including `/~comm-visit/visit.js` of
  the CRM site.
- A site on another installation posts across sites: the endpoints answer
  with `Access-Control-Allow-Origin` only for origins on the configured
  domains (403 otherwise), read no cookies and set none; requests are simple
  (`FormData`, no preflight), so the browser sends the visit token, not a
  cookie of the CRM. A signed-in employee is recognized only on the CRM site
  itself.
- One feed entry per visit, its pages read when the feed is shown; a link
  check is written once per letter. The first counted visit of a code gives
  the call task unless the executor has an open `Позвонить…` task on that
  record (an offer's included): never two calls for one interest.
