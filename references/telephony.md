# Telephony live-call isolation

The backend telephony widget is personal to the authenticated employee. Every
live endpoint that reads or changes a call must scope the query to both the
employee's configured provider and either:

- `cms_worker_user_id` equal to the authenticated user's ID; or
- `provider_user_num` equal to that user's unique extension at the provider.

Never fall back from that scope to the provider's latest active call. Providers
are shared by multiple employees, so a provider-only query leaks another
employee's caller, company, contact, phone number and call status.

Apply the same ownership scope to polling by call ID. For cancellation, resolve
an owned call before invoking the provider handler; possession of a provider
call ID is not authorization. Register an outgoing call with its employee and
extension immediately after the provider accepts it, so ownership checks work
before webhook delivery.
