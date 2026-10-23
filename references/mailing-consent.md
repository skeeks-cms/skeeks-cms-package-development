# Advertising consent, opt-out and imports of recipients

Verified 2026-10-23 with `skeeks/cms-communication` stage 4A on the
skeeks.com Windows stand (test `tests/mailing.php`, a 15-row CSV, `curl`
one click, light, dark and 375 px). The domain rules live in the package
`ARCHITECTURE.md`; this file keeps the reusable contracts.

## Who owns consent

- Consent and opt-out belong to a **contact** — a normalized email (lower
  case) or phone (digits with the country code, `79991234567`), later a bot
  chat — per site (`cms_comm_contact`), never to a lead, company or client:
  one address is often all three, and consent to letters is not consent to
  SMS. A contact row is made lazily when something is recorded for it.
- State per topic (`cms_comm_contact_topic`): subscribed with a basis
  (`site`, `verbal`, `cold` — no consent) or unsubscribed, with who, when and
  where from (link, one click, manager, import, API, the letter). Keep the
  consent fields after an opt-out: they are its history.
- A stop list apart from topics (`cms_comm_suppression`: opted out of all
  advertising, complaint, undeliverable) stops every topic.
- One writer service. An import or an automatic source never undoes an
  opt-out and never weakens a real consent to `cold`; only an explicit
  manager choice or the client's own page may.

## The check before every advertising send

One method answers "may this topic go to this contact by this channel" with
a reason for the summary of excluded recipients
(`MailingPolicy::check()/checkMany()`), in two modes: a manager's own mailbox
(cold and unmarked contacts allowed, small personal letters) and a mailing
service (real consent only — 38-ФЗ, services ban cold lists). Service
messages and personal letters never ask it; a personal letter to a contact
on the stop list only shows a warning in the window (a debounced JSON check
of the recipients, the letter still goes).

## The opt-out link and one click

- A personal page by the contact's code (`/~unsub/<code>?t=<topic>&m=<letter>`)
  built like the pages of [client-link-pages.md](client-link-pages.md): own
  layout, `noindex`, no counters, `Referrer-Policy: no-referrer`. Accept the
  letter from the address only if it really went to this contact; the topic
  only if it is of the contact's site.
- RFC 8058: the same URL answers `POST List-Unsubscribe=One-Click` without
  CSRF (mail programs have no token; the code is the access) and opts out of
  the topic of the letter, of all advertising without a topic.
- "По всем каналам": every contact of the CRM records that have this contact
  (and the company and client of such a lead).
- The letter: a template placeholder link (`https://sx-unsubscribe.invalid/`,
  survives the rich-text cleaner like the send placeholder of
  [client-sends.md](client-sends.md)) becomes the personal link per
  recipient, or a line is appended; the letter stores the URL and the send
  job adds `List-Unsubscribe: <url>` and `List-Unsubscribe-Post`. One
  recipient per advertising letter. Build the URL from the current request
  host in the web (a stand links to itself) and from the site main domain in
  jobs.

## In CRM cards

Only through core places: a block in `CrmCardBlocksWidget::EVENT_BLOCKS`
(state of every email and phone by topics, an edit window per contact) and a
mark in `CrmContactBlockWidget::EVENT_CONTACT_ACTIONS` for contacts on the
stop list. Card pages do not load package styles: inline the color of such
a mark. A window URL from a card uses `enableEmptyLayout()` only — never
`enableNoActions()` (`_sxb[noa]`).

## Imports into leads

- Upload → map columns → preview with duplicates → import. Keep the parsed
  rows in the database (one row table, also the report and a future
  audience), not files between requests; cms-job workspaces are not for user
  uploads. Detect UTF-8/Windows-1251 (also in the file name), BOM and the
  separator; accept a table pasted from Excel (tabs).
- Duplicates are exact and batched: emails with `IN (LOWER(...))`, phones by
  their significant digits from one read of the CRM phone tables (their
  notation differs, no index helps), INN through the contractor maps, names
  without quotes and extra spaces (companies and leads; people's names repeat
  too often). SQLite `LIKE` is case-sensitive for Cyrillic, MariaDB
  `utf8mb4_unicode_ci` is not: when a coarse SQL filter precedes a PHP compare,
  query both the word and its capitalized form. A later row of the same file
  is a repeat of the earlier row, not a CRM duplicate.
- A lead per row with a stable `source_ref` (`<package>-import:<id>:<row>`),
  each row in its own short transaction, rows done skipped on a repeated or
  resumed run; a large file is a cancellable cms-job with the import as its
  resource.
- `CmsLead::afterSave()` notifies the responsible employee of every new lead
  when there is no web user (jobs): create the lead without the responsible
  and with `deferManagerNotifications`, then `updateAttributes()` the
  responsible and status, and send one notification about the whole import
  (`CmsWebNotify` with the import as `model_code`/`model_id`, registered in
  `Skeeks::$modelsConfig`). Set the Blameable value and `submitted_by_id` to
  the importing employee so the creation entry has an author in jobs.
- A new lead source is a core constant with a fallback in the package:
  `defined(CmsLead::class.'::SOURCE_MAILING') ? … : CmsLead::SOURCE_IMPORT`.
