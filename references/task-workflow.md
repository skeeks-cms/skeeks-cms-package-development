# Task lifecycle and time accounting

`skeeks/cms/services/TaskWorkflow` owns task status transitions used by both
`CmsTaskBtnsWidget` and `cms-mcp/CmsTaskService`. Do not reproduce a transition
by saving only `CmsTask.status` or by separately inserting a time interval.

The service locks the executor row and task row in one transaction, reloads
the task, checks participant permissions, applies the model's transition
validation, and saves status plus time records atomically. Starting work opens
the employee schedule when needed and one task schedule. Pausing or completing
work closes the task schedule only; the employee schedule remains open.
An existing open interval on another task blocks starting a new task.
Same-state retries do not create another interval. Inconsistent or missing
intervals fail instead of inventing timestamps.

The core model rejects unknown status values. Preserve the historical paused
code `on_pouse` for compatibility. UI hints must tolerate invalid legacy values
so one damaged record cannot break a company listing.

MCP owns action schemas, authorized record lookup and safe response fields.
Its create/update endpoints reject direct status writes. Administration and
API use the current authenticated user, never an actor ID supplied by a client.
An administrative recovery is separate from ordinary transitions: only unknown
statuses with an unbroken audit chain and no open interval can be restored to
a verified accepted/paused state, with normal audit and recalculation events.

## Tasks made automatically by a package event

A package may give an employee a core task when something happens outside
the CRM (a client opened a proposal: `cms-proposal` `services\ProposalTasks`,
verified 2026-10-23). Rules:

- Create a plain `CmsTask` through the model, with `executor_id` and
  `created_by` set to the employee: a task to oneself is accepted at once,
  and the status rule never reads `Yii::$app->user->identity` (a guest on
  a client page has none). The blameable behavior overwrites `created_by`
  with the (absent) signed-in user on insert, so set it again with
  `updateAttributes()` after saving.
- `plan_start_at` may not be sooner than 30 minutes from now (the model
  rule). "Today" means the first working period of the employee's
  `work_shedule` (`CmsScheduleHelper::getSchedulesByWorktimeForDate()`)
  that ends after now + 31 minutes, otherwise the start of the nearest
  working period; use a default schedule for employees without one.
- Core tasks know only the project, company, lead and client. Keep the link
  to the package record in a package table (`cms_proposal_task`), use it to
  skip a new task while the employee has an open one (`CmsTask::activeStatuses()`)
  and show it at the top of the task card through `RecordOriginInterface`
  of the package's record source.
- The event must not fail because of the task: catch, log and go on. Let
  the site switch the rule off with a component setting.

## Task comments and results

Task-card descriptions use `cms/helpers/TaskContentHelper::render()` at output,
without rewriting the stored value. It escapes raw-text/document tags before
HTML parsing, then applies Yii HtmlPurifier with `Core.EscapeInvalidTags`.
Literal technical examples such as an unclosed `<title>` stay visible and
cannot swallow the page; normal rich-text formatting survives and unsafe
attributes/URLs are removed. Keep this output boundary when extending the card.

A task comment is a `CmsLog` with `log_type=comment`, `model_code=CmsTask`
and `model_id` = task id; administration leaves `cms_company_id` and
`cms_user_id` empty. The core `CmsLog` after-insert handler notifies the task
author and executor, so API writers must save the model normally and must not
send their own notifications.

A task result is a pinned task comment (`is_pinned=1`). There is no separate
result table or flag: the task report
(`AdminCmsTaskController::getTaskReportResults()`) selects
`CmsLog::find()->comments()->pinned()` by task model, and the admin pin button
reads "Результат задачи" for task logs. Pinning uses the narrow toggle-pin
write (`is_pinned`, `updated_at`, `updated_by`) under administration access;
editing text or files uses `cms/admin-cms-log/update-delete`, which
`CmsLogRule` limits for workers to their own unpinned comments within 24 hours.
Only comment records may be pinned or edited; generated history is immutable.
