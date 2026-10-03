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

Verified consumer: `cms-hosting/AdminDnsZoneController::getModel()`, whose
manager scope joins the related deal for non-administrators.
