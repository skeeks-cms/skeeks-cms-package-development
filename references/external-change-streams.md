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
