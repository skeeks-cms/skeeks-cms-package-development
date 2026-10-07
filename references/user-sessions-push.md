# Revocable web sessions and native push

`skeeks/cms` owns the opt-in `web/SessionUser`, `components/UserSessions`,
CmsUserSession and the shared Admin/UPA devices screen; see USER-SESSIONS.md.
Core must work without mobile tables, routes, jobs or Firebase dependencies.

`skeeks/cms-mobile` owns `components/MobilePush`, installations, provider
transports, native bridges and push controls. Read its README.md and
MOBILE-PUSH.md for setup and rollout. The dependency points from the extension
to CMS, never in reverse.

UserSessions exposes EVENT_REVOKED inside the revocation transaction. Its
event carries sessionIds, userId and realm, without credentials. Extensions
perform cleanup on the same DB connection; do not do external I/O here.
EVENT_RENDER_DETAILS lets extensions append trusted rendered HTML to an
authorized session card. Extension endpoints independently check ownership,
current authentication and CSRF.

Keep login authority separate from delivery addresses. A mobile installation
can change owners; it cannot authorize requests by possessing a push token.
The authenticated web session supplies the account. Installation secret proof
and a generation counter protect rebinding and pending delivery ownership.

Revocation must cover both active PHP sessions and Yii remember-me restoration.
Do not implement device logout merely by deleting a PHP session or push token.
The isolated `cms/tests/user-sessions.php` exercises real Yii login and
revocation without the mobile package or schema. The extension's
`tests/user-sessions-push.php` checks rebinding, cleanup rollback, token
conflicts, profile controls and the delivery handler.

Remember-me duration must not shorten an otherwise valid PHP session. Preserve
Yii idle timeout and absolute timeout semantics; renew both credential cookies.
Expired tracked sessions use the normal logout lifecycle; a logout veto cannot
retain revoked access. Bounded realm-scoped retention belongs to each owning
package and must not add a queue dependency to core CMS.

CMS authentication settings own the shared browser/WebView idle and absolute
lifetimes. Apply default/site policy after Cms initialization, never recursively
load Cms from User during identity restoration or use personal policy overrides.
Snapshot policy in each new login record; workers use its effective expires_at
without loading the web site's settings. Only foreground authenticated activity
extends idle expiry. Changing settings affects new logins, not cookie restoration;
explicit revocation is the mechanism for ending existing sessions immediately.

Worker identity lookup for CmsUser uses global ID and active status, independent
of the worker's current website. Delivery deduplication includes recipient and
installation generation. Token possession alone never permits reclaiming an
installation after local storage loss: the native client must rotate its token.

Use the same personal device actions/view in Admin and UPA. Actions require
the current account, realm and CSRF; knowing a row id is not authorization.
An OAuth client connection is a separate concept owned by cms-oauth2-server.

Domain packages choose recipients and call `mobilePush->enqueue()` with a
stable per-recipient event key and an internal route. The destination still
checks permissions when opened. Delivery outcomes belong to cms-mobile; execution,
leases, continuation and history use cms-job. Provider acceptance is not proof
of receipt or reading. Never introduce a second push queue or worker loop.
Mobile delivery jobs use visible history with 30-day retention, including
successful provider acceptance. They are event-triggered jobs, not periodic agents.

Native bridge configuration is a presentation setting, not an authentication
factor. Firebase server credentials stay in server deployment configuration;
Android google-services.json is client configuration. Do not place provider
keys, push tokens or installation secrets in job payloads or public DTOs.

cms-mobile accepts params['cms-mobile']['firebaseCredentialsFile'] as an explicit
server-side file path. Missing/invalid explicit credentials fail closed; absence
of the parameter allows Google ADC. The key itself is never stored in params.
The explicit JSON supplies project_id when the app has no explicit projectId.
Yii Config's injected $params is distinct from Yii::$app->params. The mobile
package explicitly maps its own params namespace into the application for both
web and console; verify credentials using the real application bootstrap.
Use the actual yii entry point when verifying worker settings: it can define
YII_ENV differently from a diagnostic script. Shared delivery configuration must
reach both environments; check enabled/realm and credential lookup in the worker
environment before considering provider authentication sufficient.
