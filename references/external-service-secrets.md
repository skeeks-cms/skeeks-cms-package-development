# External-service secrets

Shared packages must not store API keys or other reusable credentials in their
database tables. A record may store a stable source name such as
`api_key_param`; the corresponding secret value belongs to project-owned
configuration or the process environment.

When the package already owns an application component, prefer an explicit,
write-only secret map on that component. The project loads real values from its
uncommitted or otherwise protected params source and passes only the required
entries into the component. Domain models resolve the stored source name
through that component instead of depending directly on global
`Yii::$app->params`.

Keep environment lookup as a supported deployment source. A legacy
`Yii::$app->params` fallback may remain during a compatibility period, but new
project integrations should use the owning component. Never expose the secret
map through model attributes, admin forms, debug output, exception messages or
public component getters. Tests should prove that component-provided values are
resolved while the application params array remains free of those secrets.

Exception: credentials that administrators enter for records created in the UI
(mailbox logins of `skeeks/cms-communication`) cannot live in project params.
Store them encrypted (`Yii::$app->security->encryptByKey`) in a dedicated
column, with the key supplied to the owning component from protected params or
the environment as above. Keep the column out of `fields()`, render password
inputs always empty (an empty value keeps the stored secret), decrypt only in
transport code, and fail closed with a configuration message when the key is
missing or cannot decrypt. `cms-communication/tests/accounts.php` verifies it.
