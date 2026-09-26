# Receiving external change streams

The native job's execution cursor and the external protocol's committed cursor
have different owners. JobContext progress/cursor writes are buffered by the
reporter. A receiver promising durable receipt must commit its protocol cursor
and the corresponding domain state in the same database transaction; a buffered
job cursor alone is not an acknowledgement of durable receipt.

Perform HTTP outside that transaction. After acquiring the connection-state
lock, compare the phase, generation and cursor used by the request with the
current row before accepting the response. Reject a stale response instead of
overwriting a concurrently committed position. Native resource/dedup locks remain
the normal execution coordinator; the database comparison protects the data
boundary even across interrupted attempts.

Keep received and applied state explicit. A successful receipt job proves that
metadata is durable, not that another subsystem applied it. Compact per-entity
source revisions and unresolved-state markers are domain state, not a second
execution queue. Scheduling, attempt limits, cancellation, leases, worker
restarts and job history stay in cms-job. Never put transport credentials in
job payloads, cursors, results or exception text.

Verified by cms-shop/tests/gpd-catalog-receiver.php on disposable MariaDB: rollback
after item writes, an intervening cursor commit during a network request,
resumable native chunks over 10000 entities and cancellation before receipt.
Domain API semantics and rollout are documented by the owning package.

Test both the domain transport interface and the actual HTTP request formatter.
A fake returning valid pages cannot detect a cursor serialized into a GET JSON
body while the server reads query parameters. cms-shop's wire regression uses
Yii Request::prepare with MockTransport: GET cursor/limit are in the URL and its
body is empty; POST batch IDs remain JSON. Keep these tests independent of live
credentials and never dump prepared request headers for diagnostics.

Keep per-run receipt counters separate from the persistent catalog totals. A
subsequent empty poll must not report the initial snapshot as newly received.
Use the reporter's progress_message for human-readable completion text and keep
technical stage identifiers as a fallback. The shared job history grid prefers
that escaped message; domain handlers own its wording and must distinguish
received updates/revocations from applied shop changes.

When a site has one configured external catalog, derive its binding from the
native run's site instead of requiring a repeated connection name in every
schedule payload. Configuration simplification must retain the durable row ID
and cursor; reject ambiguous bindings and changed credential fingerprints.
During migration, empty and legacy payloads must share the same resource lock.
Verified in cms-shop's receiver tests, including site isolation and cursor retention.
Application acknowledgements belong in the transaction that writes the destination
entity. Recheck the received revision after locking its state, and reject a stale
response; never acknowledge pending/unordered absence as deletion. External media
preparation is outside that transaction and may leave unreferenced files after failure.
CMS may delete an unreferenced storage record while replacing media, so cached IDs
must be revalidated before reuse. Verified by cms-shop CatalogApplier regression and
rollback checks using real CMS catalog models. Domain policies remain in cms-shop docs.
Independent streams may reuse a receipt/application engine only with explicit,
allowlisted storage bindings and separate durable cursors. Adding another stream
must not create ambiguous source-connection rows in the original stream's table.
When reference and product writers touch the same destination models/media, use
the same native resource key with different dedup keys. Missing dependencies are
retryable domain state and must leave the applied revision unacknowledged.
Verified by cms-shop's two-prefix reference receiver/application regression.

Stable code dictionaries can use full-list endpoints instead of a change journal.
For insert-only synchronization, check the destination code before preparing media
and recheck inside the write transaction. An unknown required code is a pending
dependency, never permission to acknowledge an incomplete entity. Verified by
cms-shop's stable-reference model integration test (local values/media preserved).
When adding per-entity synchronization to previously editable imported records,
separate the new-record default from the one-time existing-record backfill. Protect
existing linked records without changing their content, and never repeat that
backfill in a worker. Check protection before media preparation and again before
writing; acknowledge the protected version so dependent entities can use the local
record. Verified by CMS tree migration and cms-shop category model tests.
When an imported model derives fields from a configurable handler during beforeSave, inspect behavior ordering: serialization may replace the handler settings array before that callback. Initialize the handler while settings are structured and set its relevant properties before saving. Verified by cms-shop's real property-model regression; model-specific repair remains owned by cms-shop.

Keep source publication and destination application in their respective owners.
A reusable shop receiver may wake its native apply job after durable receipt;
it must not register a consuming project's JSON publisher or server tables.
Project bootstrap listeners can coalesce ActiveRecord saves at request/action
completion and wake native jobs only outside the write transaction. They are
latency optimizations, not durable capture: bulk SQL bypasses AR events, so retain
transactional source markers and periodic recovery. Check committed pending work
before dispatch and preserve resource/dedup keys; a queue wakeup failure must not
undo an already successful content save. Verified by the market wakeup regression
and cms-shop/tests/gpd-apply-dispatch.php.
Source discovery must respect the publisher's lock order. Even INSERT IGNORE for
an existing source row, or a SELECT in a MySQL/MariaDB trigger, can wait on the
source lock while the saving transaction already owns membership rows. Discover
missing source rows from the committed dirty journal outside the membership
transaction instead. A two-connection regression must hold a source row locked
and prove another membership insertion can still commit.
