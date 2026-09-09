# Web application bootstrap from console commands

Some console workflows construct a temporary `yii\web\Application` to inspect
web routes, controllers, permissions or backend menus. The synthetic
application must receive the real web entry script through
`components.request.scriptFile` before it is constructed. It also needs a
deterministic `components.request.scriptUrl` fallback because CLI server
variables describe the `yii` script, not the web entry point.

Yii derives `@webroot` from the request script file during web application
initialization. Under CLI, the process script is normally the project-level
`yii` file, so leaving `scriptFile` implicit points `@webroot` at the project
root instead of `frontend/web`. Asset publication can then fail or write to the
wrong directory. Do not require every project to override
`assetManager.basePath` to compensate.

Preserve explicit project values. For the standard SkeekS application layout,
use `ROOT_DIR . '/frontend/web/index.php'` as the `scriptFile` fallback only
when the file exists, and use `/index.php` as the internal `scriptUrl` fallback.
Keep the correction inside the shared console workflow that creates the
synthetic web application.
