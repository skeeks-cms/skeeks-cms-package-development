# Task lifecycle and time accounting

`skeeks/cms/services/TaskWorkflow` owns task status transitions used by both
`CmsTaskBtnsWidget` and `cms-mcp/CmsTaskService`. Do not reproduce a transition
by saving only `CmsTask.status` or by separately inserting a time interval.

The service takes the executor's layout lock (`TaskPlanner::lockExecutors()`,
before any row lock — see «Task start»), locks the executor row and task row in
one transaction, reloads the task, checks participant permissions, applies the model's transition
validation, and saves status plus time records atomically. Starting work opens
the employee schedule when needed and one task schedule. Pausing or completing
work closes the task schedule only; the employee schedule remains open.
Starting work fixes the task on when its work first started (the first task
schedule, else now; `is_fixed_start = 1`, `CmsTask::$startingWork` lets that
past time through validation) before the queue is laid out (owner's decision
3, verified 2026-10-31 `tests/task-review-regressions.php`).
An existing open interval on another task blocks starting a new task.
Cancelling a new or accepted task and resuming a cancelled one are allowed to
its author, its executor and an administrator (`PERMISSION_ROLE_ADMIN_ACCESS`):
one check `CmsTask::canCancel()` serves the model rule, `TaskWorkflow` and the
card buttons, so an administrator can clean up tasks of closed deals through
administration or `cms_task_action`. Other transitions stay with participants.
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
- The start of such a task comes from `TaskPlanner::nearestStart($executor,
  $duration, $notBefore)` — the nearest free window of the executor's
  schedule that fits the duration (see «Task start» below). Keep a fallback
  to the schedule alone (`class_exists(TaskPlanner::class)`) while the
  package supports an older `cms` (`CampaignTasks::startFor()`,
  `ProposalTasks::startFor()`, verified 2026-10-31).
- A calculated start is not a person's choice: set `$task->isAutoStart = true`
  (guarded by `$task->hasProperty('isAutoStart')` for an older `cms`). The new
  task stays auto and takes its place in the queue by that time — before the
  first auto task starting later (`CmsTask::afterSave()`); without the flag a
  start given to a new task fixes it (`CampaignTasks`, `LinkVisits`,
  `ProposalTasks`, verified 2026-10-31).
- Core tasks know only the project, company, lead and client. Keep the link
  to the package record in a package table (`cms_proposal_task`), use it to
  skip a new task while the employee has an open one (`CmsTask::activeStatuses()`)
  and show it at the top of the task card through `RecordOriginInterface`
  of the package's record source.
- The event must not fail because of the task: catch, log and go on. Let
  the site switch the rule off with a component setting.

## Task start: fixed or planned

Verified 2026-10-31 (`skeeks/cms` `services\TaskPlanner`, `tests/task-planner-stand.php`).
`plan_start_at` is NOT NULL (m261031_150000): `CmsTask::beforeSave()` gives a
task without a start the planner's provisional time (`TaskPlanner::estimate()`,
the end of the executor's queue; no executor — now) and keeps it auto. Every
way of creating a task (form, API, packages, import) must go through the
model; there is no direct insert into `cms_task` (checked 2026-10-31).
`is_fixed_start = 1` — a person chose
the time and nothing moves it (any future time, outside the schedule or over
other tasks allowed; `TaskPlanner::warnings()` only warns). `0` — the planner
gives the start:

- `TaskPlanner::plan($user, $sortIds)` lays out the executor's active tasks:
  fixed ones and the task in work take their time out of `work_shedule`
  (default weekdays 9–18); auto tasks follow `executor_sort` into the free
  time, a long one continues after a break, starts on a 5-minute grid. It
  writes `plan_start_at`, `plan_end_at`, `executor_end_at`, `executor_sort`
  (and the reminder, see «Task reminders») with `updateAll()` — no history, no
  `afterSave()`. A task in work is fixed on when its work first started.
- Locks (verified 2026-10-31, `tests/task-concurrency-disposable.php`): every
  change of a task takes the layout lock of its executor (and of the former
  one) before it reads the queue — `CmsTask::beforeSave()`,
  `TaskWorkflow::transition()`, `plan()`; locks go in the order of ids
  (`TaskPlanner::lockExecutors()`, `GET_LOCK('cms-task-plan-<id>')`, the site
  has no mutex component). Every task save is a transaction
  (`CmsTask::isTransactional()`), and inside a transaction the lock is held
  until it is committed or rolled back (`Connection` commit/rollback events);
  `plan()` reads the queue with `FOR UPDATE`, so a layout always sees the
  committed result of the previous one, even in a transaction that began
  earlier. A lock not given within `LOCK_TIMEOUT` is an error of the operation
  (`TaskPlanner::BUSY_MESSAGE`; beforeSave adds it to the model), never a
  silently skipped layout. A new path that writes tasks takes the lock first,
  before any row lock.
- Tasks do not jump (owner's clarification 31.10.2026): the planner runs only
  on events — `CmsTask::afterSave()` on insert and on changes of executor,
  duration, start, mode or status (closing included), a move through
  `reschedule`, a changed `work_shedule` (`User` after update; a vacation is
  a schedule change). Never replan when a page opens or by a timer;
  `cms/task/plan` is a manual console command. `recalculateTasksPriority()`
  is a wrapper (a manual reorder passes the auto task ids).
- The pin is a toggle: `CmsTask::setFixedStart(bool)` fixes on the current
  start or gives the start back to the planner (which replans at once), refuses
  closed tasks and the task in work, and writes a history record in words
  («Закреплено на …» / «Закрепление снято (было …)», `is_fixed_start` itself is
  in `no_log_fields`). Views call `admin-cms-task/pin` through
  `[data-sx-task-pin]` of `task-calendar.js` (unfixing asks first,
  the standard `sx.backend.confirm()`, see `backend-ui-assets.md`); the API takes `is_fixed_start` without a new start and
  calls the same method without a confirmation. Every view renders the toggle
  with `CmsTask::pinToggleHtml($label)` (owner, 01.11.2026, like the pin of a
  comment): an empty grey pin — auto, a filled accent pin (`.is-fixed`, svg
  `fill: currentColor`) — fixed; the card adds the state as text. A static mark
  of a fixed task (`CmsTask::icon('pin')`) is filled as well. A fixed task has
  no drag handle in the list: it is not dragged at all.
- A noticeable move of an auto task (another day or `NOTICEABLE_MOVE`, 30
  minutes) keeps `replanned_from` / `replanned_at`; views draw
  `CmsTask::getReplannedIconHtml()` next to the bell, the card shows
  «Перенесена». A person choosing the time clears it.
- The model decides the mode: a new task with a start is fixed (unless
  `isAutoStart`); on update a changed start fixes it unless `is_fixed_start`
  was set in the same save; an emptied start returns it to the planner.
  Compare with `isAttributeChanged($name, false)` — forms post strings. A form
  must show an auto task without its start (the planner's time is not a choice).
- A closed task and the task in work do not move, whoever saves them:
  `CmsTask::startChangeError()` (by the saved status) is a validation rule of
  `plan_start_at`, `is_fixed_start` and `plan_duration` (the duration of a task
  in work may change — the rest of the work), so a form, a view and the API
  are refused before anything is saved; `reschedule`, `pin` and
  `setFixedStart()` answer with the same text.
- `beforeSave()` recomputes `plan_end_at` only for a new task or a changed
  start or duration: an edit of anything else keeps the end the planner
  calculated after a break.
- Moving a task from a view goes through one action,
  `admin-cms-task/reschedule`: `plan_start_at` (a time the person saw while
  choosing it — calendar, Gantt), `day` + `keep_time=1` (a fixed task to
  another day at the same time of day, never shifted, only warned), `day`
  alone (an auto task to the first free time of that day), `mode=auto`,
  `plan_duration` alone; the manager query, `cms/admin-task/manage`, no move
  of closed tasks or the task in work, model validation. The answer carries
  `previous` for «Отменить»: `fixed` + `restore=1` + `remind_at`/`reminded_at`
  bring back the former start (even a past one), mode and reminder exactly
  (`CmsTask::$restoringStart`, the reminder's author stays).
- A fixed time changes only by a choice a person sees (owner, 31.10.2026):
  a list never moves a fixed task (no handle, 01.11.2026); calendar and
  Gantt drag in steps of 5 minutes showing the new time; the Gantt's month
  moves by whole days keeping the time of day (`day` + `keep_time`, a day is
  44 px there; «Сроки» uses it for a fixed task too). Every view moves
  through `sx.taskMove(url, id, data, {kind, done})` of `task-calendar.js`:
  one message «Было → стало» / «Перенесено на …» with the reminder's time,
  warnings and «Отменить». API: `cms_task_*` takes `plan_start_at` and
  `start_mode` (auto/fixed), answers `start_mode`.

## Task views

Verified 2026-10-31. The views «Задачи» (day list), «Сроки», «Календарь»,
«Гант» of `admin-cms-task` and of an employee's card (`admin-worker`) share:

- `widgets\admin\TaskViewContext` — whose tasks and the search: in the task
  section `fromRequest()` (GET `executor_id` among `CmsUser::managedBy()`,
  oneself by default; GET `q`), in a card `forUser($model)` (no choice).
  `applySearch()` narrows any task query (name, number, project, company,
  client).
- `widgets\admin\CmsTaskViewToolbarWidget` — one toolbar: the view's own
  controls (`start`, `tools`), then employee, search, «Добавить задачу»
  (executor preset), all 32 px high. A new view uses it instead of its own
  bar; a form `GET` keeps the other query parameters.
- Icons of the views and of task marks come from the backend icon library
  (`skeeks\cms\backend\helpers\BackendIcon`, `'svg:<name>'` in an action's
  `icon`): `task-list`, `kanban`, `calendar`, `gantt`, `table`, `chart-bar`,
  `plus`, `grip` (drag handle), `pin`, `bell`, `history`, `alert-circle`,
  `chevron-left/right`. Marks are drawn by `CmsTask::icon()` and the model's
  `get*IconHtml()`; a browser-drawn view receives the rendered icons in its
  data. Do not add icon-font classes to task views; a missing icon is added to
  the library in its line style (24 grid, stroke 2, round caps).
- A list of task rows loads `TaskViewContext::ROW_RELATIONS` with `with()`:
  the row widgets read project, company, their images, client and work
  intervals — without it a long list costs five queries per task.

## Task deadline

`deadline_at` is optional (m261031_140000). Empty — nothing is checked. When
the planned end (`plan_end_at`) of an active task is after it,
`CmsTask::isBehindDeadline()` is true and every task view draws
`getDeadlineMarkHtml()` (red «не успевает», the deadline and the planned end
in the title); the API answers `behind_deadline`. No notification is sent
(owner's decision 31.10.2026).

## Task reminders

Verified 2026-10-31 (`services\TaskReminders`, `tests/task-reminder-stand.php`,
`cms-mcp/tests/task-start-reminder.php`). `remind_at` — when;
`remind_offset` — seconds before the start: the reminder follows the start
(model and planner recompute it); `remind_author`; `reminded_at` — sent. A
reminder of its own time (`remind_offset` null) follows the task as well,
keeping its distance to the start, on every path — a model save of a new
start (form, API, views), a layout of the planner, unpinning
(`CmsTask::followedReminderAt()`: one that would fall into the past reminds
now, one already due stays). A reminder of a task in work does not move.
A reminder moved to the future is sent again. The form and API set the virtual
`reminder` ('' or `none`, seconds from `CmsTask::reminderOffsets()`,
`custom` with `remind_at`, never in the past).

The agent `cms/task/remind` (job `cms.task-remind`, every minute) sends each
due reminder in its own transaction: a locking read of the row (still active,
not reminded, due — a task closed or moved meanwhile is skipped), the mark
`reminded_at` and `notify()` (the stored notification and its deliveries)
commit together; a failure before that rolls the mark back and the next run
sends it. The notification system delivers it to the person's channels.
A recipient who set the reminder (`remind_by`, written only when a person
chooses the reminder in this save — never when it follows a move by someone
else or comes back with «Отменить») gets `task.reminder_self`
(no quiet hours); others get `task.reminder` (waits for quiet hours). Both
are important and not mutable. A missed reminder is never skipped: it goes
with «опоздало на N мин», still once. A task with a reminder shows
`CmsTask::getReminderIconHtml()` (the bell, its time in the title) in every
task view; do not draw a bell of your own.

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
