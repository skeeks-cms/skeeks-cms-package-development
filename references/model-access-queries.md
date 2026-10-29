# Model access queries

When a controller checks a loaded model with a scoped query and `exists()`,
qualify its primary-key condition with the model table name (or the query's
explicit alias). A permission scope may add relation joins only for ordinary
users; a bare `id` can therefore work for administrators but fail for the
users who need the scope.

When a visibility scope accepts an explicit identity, propagate it through
every nested company/client/other scoped subquery. A parameterless nested scope
would silently use the current viewer instead of the selected identity.
For privileged "available to employee" filters, retain the viewer's mandatory
scope and add the selected identity's scope with AND; authorize selecting another
identity on the server. Lists, aggregates and direct-ID reads must keep the same
visibility boundary. Combine free-text alternatives inside one AND condition;
top-level orWhere() can undo previously applied permissions or filters.
Verified consumers: ShopPaymentQuery, AdminPaymentController and the MCP payment
list/statistics service; regression coverage compares complete ID sets under
the viewer and explicit worker and rejects a worker selecting an administrator.

Keep model access and collection visibility aligned. Fix SQL qualification
without removing the permission scope or replacing it with an administrator
role check. Verify both an allowed model and a model outside the visible set
under an ordinary identity, plus the administrator path.

Entity notification recipient selection must reuse that same visibility scope
with each recipient's explicit identity. When visibility depends on child rows
(such as submitted contacts), persist those rows before evaluating recipients.
Use correlated EXISTS evidence to avoid multiplying parent rows in grids and
counts; keep matching read-only. Verified consumer: CmsLeadQuery and
CmsLead::availableManagerIds(), covered by cms-lead-company-access.php.

A `forManager()` scope must not `joinWith()` many-to-many relations to filter
visibility: each matching junction row duplicates the parent, so lists, counts
and pagination differ from distinct records. Filter the primary key, qualified
with `getPrimaryTableName()`, by `IN` subqueries over the junction tables (or
EXISTS), and pass the explicit identity into nested scopes. Consumers that need
a relation alias for their own filter join it themselves after the scope.
Verified consumer: `CmsContractorQuery::forManager()` (skeeks/cms 6.4.9.46),
checked locally for admin, a scoped worker (row count equals distinct count)
and a worker without clients; bill/payment forms add their own
`companies`/`users` joins.

Verified consumer: `cms-hosting/AdminDnsZoneController::getModel()`, whose
manager scope joins the related deal for non-administrators.

`CmsTaskQuery::forManager($user)` and `CmsDealQuery::forManager($user)` pass
the explicit user into their nested project/company/client scopes (skeeks/cms,
2026-10-23); before, a check "may this employee see the task/deal" made for
another employee silently used the current viewer. Verified consumer:
`cms-communication` `CrmRecords::visibleQuery()` (letter visibility through a
linked task or deal, choosing a responsible person among employees who may
read a letter).

## Employee hierarchy (department heads)

"Subordinates" of an employee have one rule in the core:
`CmsDepartment::subordinateIds($leaderId)` (skeeks/cms 6.4.9.73). For every
department the employee heads it walks all nested departments (`pid` tree) and
returns their workers (`cms_department2worker`) plus the heads of the nested
departments, without the employee. Sibling and parent branches are never
included. `User::getSubordinates()` is built on it, so every `forManager()`
scope that merges `$user->subordinates` (companies, clients, projects, calls,
leads, tasks, proposals) and the worker card rules follow the same tree. Do not
query `cms_department`/`cms_department2worker` for hierarchy elsewhere.

What a head may do with records of subordinates is decided by the existing
checks, not by a second hierarchy rule:

- tasks: `CmsTaskQuery::forManager()` includes tasks whose executor or creator
  is the viewer or a subordinate; `CmsTaskRule` reuses that scope, so a head
  opens and edits them;
- leads: `CmsLeadQuery::forManager()` shows leads executed by subordinates,
  and a head works on them (`CrmAccess::isLeadWorker()`, see
  crm-access-levels.md, owner's decision 29.10.2026).

Leaders of an employee (whom to ask: access requests of П2а) are
`CmsDepartment::leaderIdsOf($workerId)`, the exact inverse of
`subordinateIds()` on the same tree (skeeks/cms 6.4.9.75); do not walk
departments upward elsewhere.

Employee pickers that filter by an executor use `CmsUserQuery::managedBy()`:
all workers for an administrator, otherwise the viewer and subordinates. Pickers
that assign an executor keep all workers. Coverage:
`cms/tests/cms-department-subordinates.php`.
