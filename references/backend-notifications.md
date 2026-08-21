# Backend notifications

Use the standard SkeekS in-cabinet notification mechanism for events that a
signed-in user must see in the backend or customer cabinet.

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

`skeeks/cms/src/Skeeks.php` contains established task notification examples:
new assignments, executor changes, status changes and comments. Inspect those
rules before adding another task notification so the same event is not emitted
twice.

For a task authored by a customer (`created_by` equals `cms_user_id`), status
and employee-comment notifications to that author use the package UPA support
card instead of the Admin task controller. Comment notifications deep-link to
the originating `CmsLog`; keep the Admin link for employee recipients.

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
