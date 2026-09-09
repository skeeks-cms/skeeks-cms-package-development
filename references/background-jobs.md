# Background jobs

`skeeks/cms-job` owns universal background jobs: the queue, run history,
progress, artifacts and the worker.

**Verification status.** Verified against `yiisoft/yii2-queue` 2.3.8 installed
through Composer, MariaDB 10.5 and PHP 8.2: the domain layer plus the transport
integration, lane routing, transactional enqueue, isolation, two parallel
workers on one lane and graceful shutdown.

`tests/job-recovery-regression.php` also exercises real Symfony `Process`
timeouts before and after claim, crash redelivery, parent-issued attempt tokens,
isolated worker limits and switching isolation off. Redis/AMQP drivers remain
unverified; delayed crash redelivery currently requires the DB driver.

## Package boundary

The composition packages `cms-basic` and `cms-basic-shop` require
`skeeks/cms-job: ^1.0 || dev-master`. Keep this installation dependency out of
the CMS core, since cms-job itself requires cms. Installing the composition
package does not provision workers; deployments must still arrange consumers
and migrations, and satisfy cms-job's PHP requirement.

Scheduling is only one trigger among manual, API, event and child runs, so the
job core must not live inside `skeeks/cms-agent`; the dependency runs the other
way. `cms-agent` stays a scheduler that pushes jobs.

Domain packages register job types and implement handlers. They must not build
their own queue table, claim loop, backoff or stale-lock sweeper. A
domain-local queue is the failure mode this package exists to remove: the
pattern gets reimplemented per subsystem, each copy without history, progress
or an interface.

The core depends only on `skeeks/cms` and `skeeks/cms-backend`. It must not
reference a consumer package; a run records its trigger as free-form
`trigger_ref` (`cms_agent:72`) rather than a foreign key, precisely so the core
does not depend on the scheduler.

## Lane ownership

Lane ownership: cms-job registers only `default` and `maintenance`. Consumer
packages register their lanes in `components.jobQueueFactory.queues` together
with real types in `jobRegistry.types`; project configuration can override
them. Do not register speculative domain lanes in the core or empty integrations
in consumers. Workers are deployed explicitly, never started by registration.
Per-type timeout and leaseSeconds determine message TTR; lane TTR is not a
universal execution limit. Regression fixture lanes must stay in test config.

cms-hosting's `hosting.vps-site.update-disk-usage` type owns the `hosting`
lane. It scans all hosting sites, so dedup/resource keys are installation-wide,
not site-specific. It performs remote SSH `du` reads and stores measurements;
a local database copy does not make an execution isolated from real servers.
See the scheduler bridge section below for its resumable per-site execution.

## Migrating an existing domain queue

cms-hosting requires cms-job and publishes individual DNS operations directly to
`hosting-dns`, separately from disk usage on `hosting`. The payload contains all
operation input; there is no DNS-local queue, claim/retry loop, mode switch or
terminal-state projection. DnsOperation is an in-memory DTO, not an ActiveRecord.
Active deduplication is handled by cms-job, not permanent request IDs.
Before removing an old queue, stop producers and drain old consumers. The DNS
retirement migration refuses unfinished legacy work and existing active DNS jobs
before DDL, removes the old schedule and archives completed history without live
foreign keys (an empty table is dropped). Runtime never reads the archive. Applied migration files remain
unchanged. The disposable-DB tests execute with no old queue table; real PowerDNS
I/O and production cutover need separate verification. DNS-JOBS.md owns rollout.

## Active record base class for high-write tables

Models written on every progress update must extend `yii\db\ActiveRecord`
directly, not `skeeks\cms\base\ActiveRecord`. The shared base attaches:

- `HasTableCache`, which invalidates a cache tag on every save — for a job
  reporting progress roughly once per second per running job that is constant
  cache churn;
- `BlameableBehavior` without `preserveNonEmptyValues`, which overwrites
  `created_by` from the current session; an initiator that must be assigned
  explicitly, including from console, cannot survive it.

Attach `TimestampBehavior` explicitly instead. The same reasoning applies to any
future high-frequency or explicitly-attributed table.

## `save(false)` and default values

A `default` validator never runs under `save(false)`. Infrastructure rows that
are written with validation skipped — job events, artifacts, log-like records —
must obtain `created_at` from `TimestampBehavior`, not from
`[['created_at'], 'default', 'value' => ...]`. This fails at insert time with
`Field 'created_at' doesn't have a default value`, not at review time.

## MySQL/MariaDB constraints that shape the schema

- **No partial unique indexes.** Uniqueness among unfinished rows is expressed
  with a second column that mirrors the key while the row is active and is set
  to `NULL` on completion; `NULL` does not collide in a unique index. The job
  core uses `dedup_key` plus `dedup_active` this way.
- **MariaDB 10.5 has no `SKIP LOCKED`** (added in 10.6). Claiming a run is an
  `UPDATE ... WHERE id = :id AND status = 'queued'` with an affected-rows check:
  the winner is whoever changed the row. Note this covers claiming the *run*;
  reserving a transport *message* is the driver's job, and `yii\queue\db\Queue`
  does it under a `mutex` component, which SkeekS CMS did not previously
  configure. See the fencing section for why a zero affected-rows result is
  ambiguous.
- Index string keys at 190 characters so they fit a `utf8mb4` index.

## Concurrency contracts

- **Mutual exclusion belongs to the data, not to the schedule row.** A boolean
  such as `is_running` on a scheduler row only protects that row from itself;
  a full and an incremental job over the same supplier are different rows over
  the same data. Use a `resource_key` derived from the entity.
- Resource locks live in their own table keyed by `resource_key`, acquired with
  `INSERT`. The primary key makes the acquisition atomic; a read-then-write
  check would race.
- **Overlap policy is mandatory per type**, not optional. Without `skip` as the
  scheduler default, a short-interval job accumulates dozens of pending copies
  while a long one holds the resource, then runs them all pointlessly. A
  skipped push must be recorded on the active run, otherwise the job silently
  appears not to run at all.
- **Never retry what is not declared idempotent.** An unknown `Throwable` is
  classified as permanent unless the type sets `idempotent = true`, and an
  expired lease times the run out instead of requeueing it: the worker may have
  died after the external call succeeded.
- **Distributed deletion is a state machine, not one `DELETE`.** Delete each
  external replica idempotently, retain the local row while any replica still
  serves the object, and let the domain retry only the missing cleanup. A
  successful primary deletion is not proof that secondaries stopped serving.

## Business result is not the process exit code

A run that processed every item but failed on some of them is
`succeeded_with_warnings`, decided from reporter counters after the handler
returns. Handlers do not set the final status. Swallowing per-item errors and
exiting zero is the defect this replaces.

Per-item errors go to a streamed CSV artifact through
`JobReporterInterface::itemError()`, never to the event table and never to a
dedicated errors table: at import scale that table would outgrow everything
else and be read once.

## Progress writes must be buffered

A handler may call `advance()` per item. The reporter buffers counters and
issues one `UPDATE` at most once per second, and that same statement extends
the lease — there is no separate heartbeat. Without buffering a 20k-item import
becomes 20k updates of a single row that the UI is simultaneously polling.

Cancellation is cooperative and re-read on the same throttle, not on every
check.

## Reducing load is not the same as slowing down

Throttling (`chunkDelayMs`, `dutyCycle`, `allowedWindow`) lowers the peak but
not the total work. Real reduction comes from the chunk being a first-class
unit: preload lookups once per chunk and open one transaction per chunk instead
of per item. A per-item transaction with per-item lookups is what makes large
imports expensive, and no amount of pacing fixes it.

`ChunkedJobHandler` owns the loop, cursor, throttling, window, cancellation and
self-requeue; a handler implements only `loadChunk`, `processChunk` and
`nextCursor`. Saving the cursor and restarting the process is the intended way
to survive memory growth on large runs, rather than fighting the ActiveRecord
identity map.

## Legacy console commands

`ConsoleCommandJobHandler` gives history, separated stdout/stderr, exit code
and working cancellation to existing console commands without editing them.
Fine-grained progress is impossible here — the command knows nothing about the
reporter — so it appears only in rewritten handlers.

Three rules that must not be relaxed:

- **Build the command as an array and run it without a shell.** The historical
  agent runner concatenated a free-text database field into `system()`, which
  makes shell metacharacters in an argument executable. Array-form `proc_open`
  removes the shell entirely; the route and every argument are additionally
  validated by pattern, and an allowlist restricts routes where the pusher is
  not necessarily an administrator.
- **Extend the lease while the child runs.** A command running for hours would
  otherwise look abandoned and be reaped.
- **A zero exit code with non-empty stderr is a warning, not a clean success.**
  Legacy commands print per-item failures to stderr and still exit zero; the
  run is marked `succeeded_with_warnings` so it does not look flawless.

Cancellation sends `SIGTERM` to the child and escalates to `SIGKILL` after a
grace period. Marking a row cancelled without signalling the process — the
previous behaviour — leaves the work running.

## Scheduler bridge

A `cms_agent` row with `job_type` pushes a job and returns immediately instead
of running anything; `is_running` is not used on that path. The push carries
`triggerType = schedule`, `triggerRef = cms_agent:{id}`, a dedup key of
`cms_agent:{id}` and `overlapPolicy = skip`, which is what actually prevents a
short-interval agent from stacking up behind a long-running one.

Keep the columns optional. Legacy command-only schedules work without cms-job,
but a schedule with a job type must fail closed if the runtime or type is
missing; never fall back to executing its name as a console command.
The scheduler has no hard Composer dependency on cms-job.

Native system schedules are registered in `CmsAgentComponent::$jobs`, keyed by
a stable code, with `jobType`, `jobPayload`, `interval` and `name`. They have no
command and synchronize as `job:<code>` in cms_agent. Synchronization is
transactional and scoped to the current CMS site. Existing commands and their
jobType bridge remain supported. No second schedule table is introduced.

The standard admin form exposes executionMode, registered job_type and a JSON
object payload. Lane selection belongs to the type definition. Server-side
validation checks registry membership, job permission and system-field
immutability; the config loader uses its explicit trusted scenario. Existing
broken schedules can be disabled with their routing/payload unchanged, but
cannot be reactivated while invalid. Manual push leaves schedule dates intact
and shares the scheduled push's deduplication key and skip policy.

The hosting disk-usage job calls the shared site measurement method directly.
It snapshots IDs in the cursor and processes one site per delivery, keeping the
same run on continuation. Never paginate by OFFSET over disk_usage_updated_at
while updating that field. Its lease exceeds the synchronous SSH timeout;
progress and cooperative cancellation occur between measurements, not inside
the blocking call. The migration preserves the old schedule ID and history.
Deployment and isolated checks are documented in cms-hosting/DISK-USAGE-JOB.md.

`CmsAgentComponent::$onHitsEnabled` runs the whole scheduler synchronously from
a web request. Leave the default alone for compatibility, but any project with
a cron or worker container should set it to `false`.

## Administration

The scheduler's read-only `view` uses `BackendModelViewAction`; its `jobs` tab
uses `BackendGridModelRelatedAction` to reuse the job controller's standard
grid and filters. Always constrain that history by both the parent site's
`cms_site_id` and exact `trigger_ref = cms_agent:{id}`. Job type and resource
keys may be shared by other schedules and must not substitute for this link.
Keep the tab optional when cms-job is absent. Legacy direct command executions
have no job-run history; the empty state must explain that distinction.

`AdminCmsJobRunController` extends `BackendModelStandartController`. Four
details are easy to get wrong and were verified in a browser:

- **A model query class must extend `CmsActiveQuery`, not `yii\db\ActiveQuery`,
  whenever the table has `cms_site_id`.** `TBackendModelController::getModel()`
  calls `->cmsSite()` unconditionally in that case, and a plain query fatals.
  When overriding a `CmsActiveQuery` method, keep the parent signature —
  `active($state = true)` and `createdBy($user = null)` — or pick a different
  name. A job query means "unfinished" by `active()`, which is a different
  concept from the `is_active` column, so it is named `unfinished()` instead.
- **Filter fields are `SelectField` with `items`.** There is no
  `elementOptions`; the default `StringFilterField` rejects it and the page
  500s.
- **An integer timestamp column needs an explicit grid column.** The
  auto-column prints the raw value.
- **Do not hand-roll per-record action buttons in a custom card view.**
  Register them as controller model actions with `method`, `request` and
  `confirm`; the standard header renders them with access checks, POST and the
  Ajax runtime. A second set of buttons in the card body only duplicates them.

`StorageFile` exposes `src`, not `url`.

Poll progress from a small dedicated JSON action rather than reloading the
collection, and only while the run is unfinished.

## Private diagnostic log storage

`jobLogs` (`JobLogStorage`) owns disposable diagnostic files, not CMS uploads.
`JobReporter::addArtifact(TYPE_LOG|TYPE_ERROR_REPORT)` stores a relative `log_path`;
exports and source files remain CMS-storage artifacts. Error CSVs are streamed
in full (the console byte limit must not truncate CSV rows). Each continuation
owns a separate randomly named CSV fragment, including its download name; never
overwrite or append to another delivery's file. Both diagnostic types share
private downloads, retention and orphan/history cleanup. Existing CMS-storage
artifacts are not automatically migrated or deleted. SkeekS common config uses
`@root/console/runtime/cms-jobs/logs`, because web and console have different `@runtime`
aliases. A project override must point every worker, web app and cleanup process
at the same private persistent filesystem with compatible Unix users/groups.

Partition by queue / UTC year/month/day / `intdiv(runId,1000)` / random-suffixed
run filename. Attempt files must not overwrite each other. Console output is
bounded (default 5 MiB) without stopping pipe draining; register metadata before
execution so a killed worker leaves a discoverable log. Backend downloads check
administrative and job-type permission plus site scope, and send private/no-store
responses. Never expose the root through the web server.

The run card previews private logs inside the standard `PjaxLazyLoad` widget.
Read at most the final 256 KiB only in that PJAX request, using the same access,
site and expiry guard as downloads. Strip terminal controls and HTML-escape the
text; keep the preview scrollable (maximum height 320 px), explain truncation,
and retain full download and explicit refresh. Do not embed the entire file in
the initial card HTML or add a publicly accessible file URL.

After lazy rendering, `job-log-stream.js` reads `log-chunk` by artifact ID and
byte offset every three seconds. Initial/reset reads are bounded to 256 KiB,
incremental reads to 64 KiB; base64 keeps split UTF-8/ANSI sequences intact in
transit. The browser retains at most 256 KiB and uses textContent, not HTML.
Every request repeats download authorization/site/expiry checks. Drain remaining
chunks before stopping on a terminal run; abort on detach/pagehide, pause while
hidden, and preserve scroll unless the reader was already at the bottom.

`cms-job/worker/cleanup-logs` expires finished-run logs (default 14 days from
artifact creation), retains metadata, and removes old orphan files. Active runs
are protected. `cleanup` history removes referenced private logs before deleting
rows; direct/cascade deletions may leave orphans for the next log sweep. Existing
CMS-storage logs are neither migrated nor physically deleted by these commands.
Backup exclusions and periodic cleanup are deployment responsibilities.
The release harness verifies private paths, expiry, active-run protection,
orphan cleanup, download authorization and headers on a disposable database.

## Notifications

Job completion publishes a domain event; the core knows nothing about delivery
channels. In-cabinet notification of the result uses `CmsWebNotify` under the
rules in [backend-notifications.md](backend-notifications.md) — do not add a
second notification table or bell for jobs. External channels
(email, push, Telegram) belong to a delivery table in the notification package.

Technical messages such as a single email do not deserve a visible run. They
still need a queue row, so the core marks them `visibility = 'transient'` and
deletes the row on success, keeping it only on final failure.

## Transport boundary

`skeeks/cms-job` owns the business layer; `yiisoft/yii2-queue` owns message
transport only. The split is enforced by keeping every `yii\queue\*` type
inside `src/transport/yii2queue/`. The domain sees two package-owned
interfaces, `JobPublisherInterface` and `JobConsumerInterface`, plus a
package-owned `WorkerOptions` and transport DTO.

Only the run id and an envelope version travel through the queue. Payload,
settings and handlers stay in `cms_job_run`. This is what makes a message
survive a code deploy: an old message always means the same thing — "execute
run N" — and the current code decides what that is from the current row.

`cli\Queue::run()` is internal API of the library and must stay inside the
consumer adapter. The public `cms-job/worker` command is a facade over
`JobConsumerInterface`; internal driver commands must never become part of the
SkeekS contract.

Swapping to `yiisoft/queue` later replaces the publisher, the consumer, the
envelope, the queue configuration and the transport migration, and the
transport integration test has to be rewritten against the new API. The
domain — models, registry, runner, reporter, handlers, locks, business retry,
admin UI, scheduler — must not change. Do not describe this as a three-class
swap.

## Atomic enqueue without an outbox

`yii\queue\db\Queue::pushMessage()` writes through `$this->db->createCommand()`.
Give the driver the same `yii\db\Connection` instance the application uses and
the message insert joins the caller's transaction, so the run row and its
message commit or roll back together. A transactional outbox is unnecessary
under those exact conditions and must not be added speculatively.

The property disappears with any broker outside the database. Introducing
Redis or AMQP means introducing an outbox at the same time.

## Fencing: affected rows are ambiguous

Every write to a running job must be conditioned on `execution_token`, not on
the run id: the id survives a takeover, the token does not. After a lease
expires another worker legitimately claims the run while the previous process
may still be alive — hung on a network call rather than dead — and will
otherwise overwrite the new owner's progress and finish its work.

MySQL and MariaDB return the number of **changed** rows, not matched rows. A
fenced `UPDATE` that writes values identical to the stored ones reports zero
even though the row is ours — for example a progress flush in the same second
as the claim, where `lease_until` does not move. Treating zero as lost
ownership aborts healthy jobs intermittently. Resolve the ambiguity with an
explicit ownership check after a zero-row update; it costs one `SELECT` only in
that rare case. The same trap applies to renewing a resource lock.

Renew the resource lock in the same call that renews the lease. A long
operation otherwise loses its lock mid-flight and a parallel job enters the
same resource.

## Retry ownership

Business retry belongs to the domain: classification, `maxAttempts` from the
type definition, backoff with jitter, and the idempotency check. Ordinary library
jobs use `attempts = 1`, but `CmsJobEnvelope` implements `RetryableJobInterface`
and permits infrastructure redelivery. With yii2-queue 2.3.8, setting attempts
to one without that envelope override silently acknowledges delivery attempt
two without invoking the runner. Delivery attempts must not consume business
attempts until the run is actually claimed.

The transport still redelivers after a process dies, which is the point. Make
redelivery of the same run id safe by deciding from the run's actual state:
finished means acknowledge and do nothing; running with a live lease means
another worker owns it; running with an expired lease means reclaim only when
the type is idempotent, otherwise `timed_out`, because the result of the
external call is unknown.

Cancelling a queued run only sets the business status. The transport message
stays in the queue and is acknowledged without work on delivery; reliable
removal from an arbitrary transport is not available.

## Requeue must publish exactly one message

Every `running → queued` transition creates exactly one new transport message,
in the same database transaction as the status change. There are five such
transitions and all of them must go through one code path: resource busy,
execution window closed, handler saved a cursor, business retry, and recovery
by the reaper.

Zero messages is the failure that hides: the row sits in `queued`, looks
perfectly normal in the admin list, and nothing will ever wake it. Two messages
runs the operation twice. Neither is caught by a happy-path test, so assert the
message count per run id explicitly, plus that the message delay matches the
row's `available_at` and that a second reap adds nothing.

Do not paper over this with a periodic republish of every queued row: it cannot
distinguish a row that lost its message from one whose message is still waiting,
so it manufactures duplicates.

A requeue preserves the run priority and uses the same type-derived TTR
(`timeout + leaseSeconds`) as initial publication, including recovery by the
reaper. Do not silently fall back to lane defaults on subsequent attempts.

A transition that finds the run fenced must publish nothing and change nothing.
Roll the transaction back and let the reaper decide later.

## Check the result of every fenced transition

`finish()` and the requeue path return whether the caller still owned the run.
Ignoring that return value is what turns fencing into decoration: a superseded
worker would still flip the status, publish a retry, raise the completion event
and send the user a notification about an operation another process is at that
moment running. Every call site must branch on the result.

The same applies to renewing the resource lock. Losing the lock is as
disqualifying as losing the run — from that moment a parallel job may already be
inside the resource — so a failed renewal must stop the handler immediately,
through the same mechanism as a lost run.

Fold attempt accounting into the same statement as the status change
(`attempt = attempt + 1`). A separate counter update leaves a window where
another worker claims the row in between and the attempt lands on the wrong
owner. It also removes the changed-rows ambiguity, because an arithmetic
expression always modifies the row.

## Direct `cli\Queue::run()` loses isolation

Isolation in `yii2-queue` lives in `cli\Command`, not in `cli\Queue`: the
command installs a `messageHandler` that spawns `queue/exec` in a child process
with a Symfony `Process` timeout equal to the TTR. Calling `run()` directly —
the natural way to keep the library's command out of the public contract —
silently executes jobs in-process, so a fatal error kills the worker, memory
accumulates between jobs, and TTR becomes advisory.

A package that keeps its own worker facade must therefore install its own
`messageHandler` and its own internal `exec` action. Enforced TTR is the part
that cannot be emulated cooperatively: a wedged handler never reaches a
cancellation check.

The parent generates a random attempt token before spawning the child and
passes it to the internal exec action. Claim/reclaim store that same token.
Hard-timeout finalization requires this original token, never a PID or the
token freshly read from the run. PIDs can collide across containers/hosts.
An absent token fails closed; deploy both sides together and restart workers.

Handle both a nonzero child exit and Symfony's `ProcessSignaledException`:
SIGKILL is an exception from `wait()`, not an ordinary returned exit code.
The recovery test sends real SIGKILL before and after claim; this checks the
signal path, not an actual out-of-memory condition.

If a child dies or times out before claim, acknowledge nothing: release its
DB message reservation with an explicit delay, conditioned on message id,
channel and delivery attempt. Do not merely move `cms_job_run.available_at`;
that does not schedule transport delivery. Return false to the driver so it
does not delete the released row. Already claimed crashes are recovered by
the lease reaper, under domain idempotency rules.

Count isolated deliveries in the parent, not via child-only AFTER_EXEC events.
Restore messageHandler, loopConfig and event listeners in finally after each
consume call; otherwise a later isolate=false call can still spawn children.

Transition events and the fenced status update are committed in one database
transaction. A stale worker must create neither a false error event nor a
completion event. Unsupported-envelope/unknown-type failures before claim
update queued rows only, with an affected-row check; they must never fail a
currently running owner.

## Unknown envelope version needs a terminal state

Refusing to execute a message whose envelope version is unknown is correct;
acknowledging it and returning is not. The message disappears and the run stays
`queued` forever with nothing left to deliver it. Record a visible terminal
failure on the run instead, so the operation shows up as failed rather than
as permanently pending.

## Overlap policy needs a database constraint

`skip` and `coalesce` promise that a second run will not start. A read-then-
insert check cannot keep that promise against two simultaneous pushes. Back it
with the unique index on the active dedup column, derive a dedup key from the
resource key when the caller supplies none, and treat the resulting integrity
violation as the normal "lost the race" outcome — returning null for `skip` and
`coalesce`, and inserting without the active marker for `queue` and `replace`.

## Transport migration ownership

Before switching from the original combined run/queue storage, stop producers,
drain or explicitly cancel legacy work, then stop old workers. The fencing
migration rejects queued/running rows before DDL: creating an empty transport
table would otherwise strand them. A pre-existing transport table without the
creation migration in history requires manual inspection, never silent adoption.

The package-only release test uses a disposable MariaDB instance and minimal
CMS FK tables. It covers fresh schema, old-history upgrade, unknown-table
collision, both empty and sx_ prefixes, enqueue rollback, and real isolated
workers on every configured lane. This is distinct from a clean Composer
dependency-resolution/install test or installing the whole CMS.

The release test's `--regression` mode passes its private database name to all
test processes through `SKEEKS_JOB_RELEASE_DB`. The fixture validates that name
and uses a fixed disposable DB host, bypassing site configuration in parent and
child alike. `SKEEKS_APP_ROOT` selects the dependency installation, not the DB.
Keep clean-public-dependency verification separate from local-candidate tests:
a scheduler bridge present in shared vendor may still be absent from releases.
The scheduler fixture does not prove its migrations or real storage uploads.

cms-job's consolidated migration owns `cms_queue`. Do not also register the
native yii2-queue migration namespace for this transport. Its valid native
migration chain is an alternative schema owner, not a broken migration format.

cms-job registers only its own migrationPath. Automatic discovery is off by
default; no native queue namespace or special CMS-controller override is needed.
Source loading and registration follow
[package-migrations.md](package-migrations.md); `cms/migrate` now uses original
paths and standard Yii namespace support without a runtime collector.

## Isolation constrains where job types may be declared

A job type registered at runtime exists only in the process that registered it.
The isolated child boots the application again and sees configuration only, so
such a type is unknown there and the run fails as `unknown_job_type`. Job types
belong in configuration; runtime registration is for tests, and those tests must
either disable isolation or use a configured type.

Use a separate test bootstrap/entry script that declares fixture types in both
parent and child. Assert the exact timeout cause and elapsed time, not just a
quick failed status: an unknown fixture type can make a broken timeout test
green. Run process-recovery fixtures on a private channel; old integration
tests using normal lanes require an empty test queue.

Resolve the child's entry script from the application root, not from
`$_SERVER['SCRIPT_FILENAME']` as `cli\Command` does. A worker started from
anything other than `yii` — a test, a maintenance script — would otherwise
re-execute that script instead of the job.

## The public console route is `cms-job/worker`

`cms-job/worker/queues` reports effective factory lane names and registry types,
including lanes without types. `--json` uses schema_version 1 with queues and
unconfigured_types; missing lane references return CONFIG (78). It does not
instantiate transports, query queue storage or report worker liveness. Only
allowlisted descriptive fields are printed, never transport secrets. max_ttr
is derived from type timeout + leaseSeconds, null for lanes without types.

`cms-job/worker` retains continuous listening by default. Shared hosting uses
`cms-job/worker/cron`: the same consumer and isolation, with forced once mode
and bounded defaults (100 deliveries, 50 seconds, 256 MiB). Zero selects the
cron default; positive overrides are allowed. Limits are checked between jobs,
not hard deadlines for the running operation. Hosting execution limits must
allow the current job to finish; long work still needs resumable chunks.

Cron holds a nonblocking local flock per lane in the controller's `cronLockPath`
(default `@root/console/runtime/cms-jobs/locks`). Contention skips consumption
with exit 0. Keep this private directory stable across releases; never unlink
live lock files. Descriptors are close-on-exec and released in finally. This
limits cron overlap on one server, not distributed processing or persistent
workers; do not enable both deployment modes unintentionally. No schema changes
or new domain reservation mechanism are involved. Cron requires PHP CLI and
child-process support; do not silently disable isolation or use a web hit.
Deployment examples and housekeeping remain in cms-job's `DEPLOYMENT.md`.

Per-site worker provisioning belongs to `cms-hosting`, not to `cms-job` or the
general backend shell. Its `admin-vps-site/queues` action composes standard
backend surfaces; package configuration discovery and OS worker liveness are
separate data sources. Run site PHP as the site user, never as the infrastructure
SSH/root user. Manager-created systemd/cron files must be namespaced by site and
channel, carry an ownership marker, and remain disableable when CLI discovery
breaks or a channel disappears from configuration. See cms-hosting's
`QUEUE-WORKERS.md` for the implementation boundary and acceptance-test limits.
Adopting an existing `skeeks-job@{channel}` instance records ownership in a
comment-only per-instance drop-in after validating effective systemd properties.
Adoption must not start, stop, reload or restart the process. Preserve the
original unit name; subsequent control uses that unit instead of provisioning a
second consumer. Never mark an entire shared template as owned by one site.

cms-hosting's existing-site queue policy is opt-in: no policy row means no
automated changes. Only AFTER_INSERT of a new SkeekS site seeds all/systemd;
migrations and updates never opt existing sites in. Policy edits and job publication share the same DB transaction;
reconciliation reads the latest revision under the same named lock used by
edits. A failed configuration read must never be interpreted as removed lanes.
Keep the control-plane `hosting-control` lane outside automatic site policies,
or switching its mode/stopping it can strand every subsequent reconciliation.
Separate desired policy, last attempted check and confirmed convergence in the
UI. The periodic publisher and an explicit after-deploy hook command are not
proof that an external deployment pipeline has installed that hook.

Register the module under the dashed id and keep the camelCase id working.
Worker routes end up in Supervisor and systemd unit files, so changing them
later breaks deployments silently. A comma-separated lane list is rejected with
the exact commands to run instead; accepting it and using the first name would
leave the remaining lanes unattended.
