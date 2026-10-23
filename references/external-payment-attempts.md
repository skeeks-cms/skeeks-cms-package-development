# External payment attempts

A merchant payment session and a shop order are different identities. Keep an
attempt's external identity durable before sending a creating HTTP request.
After a lost response, query that same identity before deciding whether to
create anything; provider idempotency prevents a second remote session for it.

Serialize interactive retries by the shop order with a cross-process mutex.
Keep provider HTTP outside database transactions. A short transaction may
retire a confirmed terminal attempt and create its successor; commit that local
identity before the remote creation request. Keep the retired attempt and its
external identity for history and later provider callbacks.

Only an authoritative provider status may authorize replacement. A local link
TTL, an API outage or an unknown/malformed response is not proof that payment
cannot still complete. A pending cooling period remains pending. Recheck local
payment state after the provider lookup before creating a successor, and never
replace an authorized, captured, refunded or otherwise settled attempt.

Verified by `cms-shop-paysystem-yapay/tests/payment-retry.php`: remote status
recovery, durable identity before POST, provider duplicate response, terminal
rotation, unchanged cart, paid-state changes during lookup, pending cooling
period and the mutex acquisition boundary. Provider-specific status names and
retry behavior belong to that package's README.