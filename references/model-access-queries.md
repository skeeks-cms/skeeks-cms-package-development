# Model access queries

When a controller checks a loaded model with a scoped query and `exists()`,
qualify its primary-key condition with the model table name (or the query's
explicit alias). A permission scope may add relation joins only for ordinary
users; a bare `id` can therefore work for administrators but fail for the
users who need the scope.

Keep model access and collection visibility aligned. Fix SQL qualification
without removing the permission scope or replacing it with an administrator
role check. Verify both an allowed model and a model outside the visible set
under an ordinary identity, plus the administrator path.

Verified consumer: `cms-hosting/AdminDnsZoneController::getModel()`, whose
manager scope joins the related deal for non-administrators.
