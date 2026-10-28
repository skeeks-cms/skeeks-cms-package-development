# Backend notifications

Use the standard SkeekS in-cabinet notification mechanism for events that a
signed-in user must see in the backend or customer cabinet.

## Kinds of events and one notify call

Verified 2026-10-27 (step У4a; `skeeks/cms` `notify\`, `skeeks/cms-communication`
`notify\`, `tests/notify-stand.php`):

- Create notifications through `Yii::$app->notifications->notify($userIds,
  $kind, $data)`, not `new CmsWebNotify()`. A package registers its kinds in
  its config (`components.notifications.kinds`: name, category, importance
  0/1/2, audience worker/client/any, `mutable`, `dealArea` for client
  notifications); categories are the core ones (tasks, clients, messages,
  finance, system, other) unless a package adds one. A direct row still
  works: it gets the category of its record (`modelCategories`) and stays in
  the CRM only.
- Format of a row (verified 2026-10-28, П2а): `name` is the event as plain
  text (grey line: «Фирстова Елизавета просит сделку по клиенту»), `model` is
  the record it concerns — shown as the bold link line by its `asText`, also
  the label in messengers and mail. `model` must be a record registered in
  `skeeks->modelsConfig` (lead, company, client, task…): a service row such
  as an access request has no config, `getModel()` returns null, and the
  text itself turns into the link. `url` only redirects the link (a list of
  requests). Keep the client's name out of `name` when it is the `model`;
  name a hidden client in the text only masked. Do not guess the gender of
  the actor («просит … ему»).
- `eventKey` stores one event once per recipient (a repeat returns nothing);
  with `collapse` an unread row of the same key takes the new text and
  `count` + 1 (bursts: new letters, chat messages). A read burst row loses
  its key and the next event starts a new row.
- Related events are one entry: wrap the cause and what it triggers in
  `notifications->group($key, fn)` (an opened proposal and the call task it
  creates; a visit by a letter link and its task). Rows created inside — in
  any package, the core task notification included — share `group_key`.
  The bell shows the first as the main line, later ones under it; actions
  apply to the whole group; messengers get one message.
- `notify()` skips muted kinds ("не уведомлять о таком") except urgent ones,
  and makes the row silent (read, not counted) when the person switched the
  category off in the CRM. Do not check settings in the caller.
- Settings are core (`cms_notify_setting`: category or kind × channel →
  all / important / off, kind × `*` → mute; `cms_notify_profile`: ready
  variant, quiet hours) and the page is the trait `UserNotifyActions`
  (profile "Уведомления"), reusable in client cabinets.
- Channels outside the CRM are `notify\NotifyChannel` objects registered in
  `components.notifications.channels` (a channel hides itself with
  `isAvailable()`); an entry implementing `NotifyChannelProviderInterface`
  gives several at once — messengers come from the installed bot drivers,
  coded by the driver channel, never a hand-written list. Ready variants of
  a package's channels are its `notifications.presets`. Settings are read
  anew on every `notify()`: queue workers live long. The delivering package listens to the component event
  `created` through the component config (`'on created' => [...]`), not a
  web bootstrap: notifications are created in console jobs too. The event
  runs inside the caller's transaction, so the handler only queues jobs.
- A link of a notification outside the CRM is `/~n/<id>` (built from
  `communication->siteUrl()` in jobs): the web request signs in, marks the
  group read and redirects to `CmsWebNotify::getOpenUrl()` with the right
  backend prefix; a stored `url` is followed only within the site.
- Bell actions that change rows (clear, read all, an entry action) accept
  POST only, so the CSRF check applies. Pages of the bell go by entries
  (groups), not rows: a row cursor repeats a group whose rows are far apart.
- A burst row that is silent (read from the start, CRM off) keeps
  collapsing; only a burst read by the person starts a new row. A conflict
  of `eventKey` from a parallel process is a duplicate, not an error of the
  caller.
- Delivery rules of `cms-communication` (`NotifyDeliveries`): one journal row
  per notification, channel and burst step; a short delay so a group goes as
  one message; quiet hours requeue the job; read in the CRM meanwhile — not
  sent (unless the row was silent from the start); a refusal starts the next
  connected channel, an unknown outcome (a broken mail connection included)
  never does. Jobs of a group may run in any order: a later event that runs
  first waits in the journal and the earlier one takes it; a fallback after
  a refused message carries every event of that message. The job checks the
  settings and mute again before sending (they may change while it waits),
  except for a fallback and the explicit channel check. A link check accepts
  no backslash or control character: browsers read them differently from
  `parse_url()`. Messenger
  messages go through `BotSend::queue()` and get their outcome from the send
  job (`NotifyDeliveries::onBotResult()`); one chat and one messenger person
  serve one employee, and a person linked to a client is never taken over by
  an employee's link.

## Standard contract

- Store notifications in `skeeks\cms\models\CmsWebNotify`.
- Set `cms_user_id`, a short actionable `name`, and, when the notification is
  tied to an entity, `model_code` plus `model_id`.
- The standard backend header notification center reads these records, marks
  unread entries, formats relative time and links model notifications through
  the configured backend entity controller.
- Reuse this mechanism instead of adding a project-only bell, notification
  table, dropdown or JavaScript transport.
- Keep recipient selection in server-side domain logic. Never rely on the
  currently open browser page to deliver a business notification.
- Do not notify the actor about their own action unless the workflow explicitly
  requires confirmation.
- Avoid duplicate rows for one event and recipient. For broadcasts, resolve a
  unique set of active user IDs before creating records.
- When the recipient must open an entity in a different application surface
  than the model's configured backend controller, store an explicit
  notification URL and prefer it while rendering. Keep `model_code` and
  `model_id` for context and fallback rendering.
- Activity/comment notifications should carry the canonical `sx-log-id` query
  parameter and `#sx-log-{id}` fragment so the recipient opens the exact log
  entry. Build partner-facing lead URLs from the configured UPA prefix, not
  from the backend active when the notification is created.

## Notifications created by background jobs

Verified 2026-10-22 with `skeeks/cms-communication` `MailNotifier` (new
letters, created by the receive job in the queue/console):

- Do not store a backend URL built in the console: the project's backend
  prefix (`backendAdmin.urlRule.urlPrefix`, `~sxx` on skeeks.com) is set in
  the web config only, so the console builds the package default (`~sx`).
  Refer to the record instead (`model_code` + `model_id`) and register the
  model in `Skeeks::$modelsConfig` with its `controller`: the header center
  renders `BackendEntityLink` at view time and opens the record's first
  allowed action in the panel. The model needs `asText` for the link label:
  a plain `yii\db\ActiveRecord` without the CMS `TActiveRecord` trait must
  also define `getAsText()` (a bare `asText()` method is not a property).
- `CmsWebNotify::getHtml()` catches a failing item, logs it and renders it
  without links, because the list endpoint renders all items in one response
  and a single exception left the header panel empty (verified 2026-10-25,
  `CmsCommConversation` without `getAsText()`). Keep new rendering code inside
  that guard instead of calling model properties in the controller.
- `modelsConfig` is a plain public array: a package config entry would
  replace the core list. Add the key at runtime instead
  (`$app->skeeks->modelsConfig[Model::class] = [...]` in the package
  bootstrap, as `CommunicationBootstrap` does).
- `getHtml()` renders backend widgets and works only in a web request.
- Recipient checks through core `forManager($user)` in a job need the
  `user` application component: relations such as `CmsCompany::getManagers()`
  and `User::getSubordinates()` read `Yii::$app->user->identityClass`, which a
  console application lacks. Set a session-less `yii\web\User` with that
  identity class only for the check and clear it afterwards
  (`MailNotifier::withUserComponent()`); never leave it registered.
- Collapse a burst into one row per recipient (several letters of one run —
  one notification pointing at the latest) and mark it read when the user
  opens the record (`CmsWebNotify::updateAll(['is_read' => 1], …)`).

`skeeks/cms/src/Skeeks.php` contains established task notification examples:
new assignments, executor changes, status changes and comments. Inspect those
rules before adding another task notification so the same event is not emitted
twice.

For a task authored by a customer (`created_by` equals `cms_user_id`), status
and employee-comment notifications to that author use the package UPA support
card instead of the Admin task controller. Comment notifications deep-link to
the originating `CmsLog`; keep the Admin link for employee recipients.

## Widget version compatibility

The admin header enables `CmsWebNotifyWidget::enableWorkReminders` only when
`property_exists()` confirms that the installed CMS widget supports it. CMS
6.4.9.5 lacks that property and must receive the legacy empty configuration.
A separately updated backend-admin package must not pass optional new widget
properties unconditionally while its supported dependency range includes older CMS.

## Unassigned client support tasks

A task created from the client cabinet uses the `client-support` scenario. If
project, company and client relations do not resolve an executor, keep
`executor_id = NULL`; this is a triage queue, not a random assignment.

The current temporary policy is:

- only active workers with `CmsManager::PERMISSION_ROLE_ADMIN_ACCESS` receive
  the notification and see the triage block; determine access through
  `Yii::$app->user->can(...)` for the current user and
  `authManager->checkAccess(...)` for another user, as in
  `CmsUserQuery::forManager()`;
- use the title `Новая неразобранная задача от клиента` and link it to the task;
- restrict the queue query with `forManager()` first, then narrow it with
  `unassignedFromClients()`;
- show the queue above the worker task calendar and link to the matching ready
  filter in the complete task list;
- after an executor is assigned, rely on the standard `Вам передали задачу`
  notification.

This administrator-only routing is an explicit interim fallback. Replace it with a
dedicated support-triage permission or configurable support group when that
role is introduced; do not widen the queue to every manager.
