# Protection of public forms

The forms of the form constructor (`skeeks/cms-module-form2`) are the shared
public entry of requests on every site. Protect them in that package, never per
site: a site only configures `modules.form2.protection` and listens to the
conversion and acceptance events. Settings, columns and the update order of a
site: the package README, «Protection of public forms».

## Contracts

- **One check of a public form.** `challenge`, `validate` and `submit` use
  `Module::findPublicForm()`: an allowlisted model (`Module::$formModelClasses`)
  and a form of the current site. Sites of one installation share
  `cookieValidationKey`, so the signed stamp names the site, the form model
  and the form; a stamp of one site never confirms another.
- **Never silently drop a request.** A submission that fails a check is saved
  with `check_status = suspicious` and reason codes; it gets no notification,
  no CRM deal (`ensureCmsLead()` refuses it; the admin «create lead» action
  answers 403) and no conversion. The visitor gets the same successful answer.
- **One acceptance.** Automatic or manual, acceptance happens once:
  manual is an atomic `UPDATE … WHERE check_status = 'suspicious'`, and only
  the request that changed the row creates the deal, queues notifications and
  fires `Form2FormSend::EVENT_ACCEPTED` after the commit. Project side effects
  listen to that event; `_afterSave()` runs for every saved submission and is
  not an acceptance hook.
- **Notifications are an outbox.** One row per recipient in
  `form2_form_send_notification`, created in the accepting transaction,
  delivered after the commit, claimed atomically, `send() === false` and
  exceptions recorded per recipient, retried alone; a delivered recipient is
  never notified again. Compare recipients exactly as the unique index does
  (its collation is case-insensitive) — a PHP mismatch makes a duplicate key
  that rolls the whole submission back. A row stuck in «sending» is an
  unknown result (the letter may have gone), never a reason to resend: it
  becomes `unknown` for a person to check. Never report «sent» without
  reading the rows.
- **Conversions come from the server.** The answer carries
  `data.form2.conversion`; the browser fires `form2Conversion` only for it.
  Goals listen to that event, not to `ajaxFormSuccessSubmited`. A lost first
  HTTP answer leaves a submission without a conversion (missed, never doubled).
- **A signed one-time stamp per widget instance.** The browser takes it on the
  first interaction (cached HTML stays valid); the script is attached by the
  widget's DOM id, because identical `registerJs()` strings are merged by Yii
  and two instances of one form would share one state. The stamp key is a
  unique column (`submit_key`): a repeated request returns the first result.
- **Pluggable bot checks.** `BotCheckInterface::issue($token)` /
  `verify($token, $answer)` → passed / failed / unavailable, browser solver
  `sx.Form2Protection.solvers[type]`. The built-in `PowBotCheck` is a SHA-256
  task without third-party services. Add captcha services as new
  implementations; do not put provider code into the controller.
- **Signals are not verdicts.** Fast filling, a shared IP, a direct source or
  no UTM are never a reason by themselves. Limits are site settings in the
  administration (`Form2ProtectionSettings`, a component settings record of
  the site; a project config key wins per key), approximate (counted
  before the save; simultaneous submissions may pass together) and count only
  accepted submissions of the site.
- **A phone only by an explicit field format.** A text property of the format
  «Телефон» (`PropertyTypeText::FIELD_PHONE` in skeeks/cms) renders `type=tel`
  with the soft mask `PhoneInputAsset` and validates by `PhoneValidator`. Only
  such fields give the limit key (E.164 by libphonenumber) and deal contacts;
  a name never does, on every site, with no switch back.
- **Untrusted values in cards and lists.** Escape visitor values everywhere
  they are shown (`Form2FormSend::propertyValueHtml()` for the card and the
  list columns); keep markup only of handlers that override `getAsHtml()` to
  build it safely (file links). Never
  send stored request dumps (cookies, session, raw request, full `$_SERVER`) to
  the browser, collapsed or not.
- **Code before migration.** `Form2FormSend::isProtectionReady()` requires every
  column, the unique index of `submit_key` and the notification table with its
  unique, status indexes and foreign key; until then the old behavior stays and nothing
  touches the new columns. A partial schema is not ready. Even then a visitor
  is told «sent» only for a saved submission: a page without the stamp is
  saved as ordinary, other bot signs get a temporary error. Migrations are
  resumable (MySQL DDL is not transactional): a repeated run adds missing
  columns, indexes and keys and keeps rows.

## Verification

`FORM2_PROTECTION_TEST=local php tests/form2-protection-contract.php`,
`php tests/form2-lead-contract.php`. On a stand with two sites of one
installation: forms of the other site in all three actions; two instances of
one form and a Pjax replacement; replay and parallel requests with one stamp;
autofill and an expired stamp; phones in ordinary and «Телефон» fields;
concurrent «Принять заявку» from two sessions, a manager, GET and bad CSRF;
a failing transport for one recipient and the retry; old, partial and full
schema; old cached HTML with `requireToken` on and off.
