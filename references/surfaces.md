# Backend surfaces

`skeeks/cms-backend` owns the single surface contract for new Admin, UPA and
customer-cabinet interfaces.

## Canonical implementation

Use `skeeks\cms\backend\widgets\BackendSurfaceWidget` for a structured
container. It registers `BackendUiAsset` and accepts:

- `title`, `hint`, `actions`, `content`, `footer`;
- `raised`, `clip`, `responsive`;
- `headerBordered`, `bodyFlush`, `footerStretch`;
- root and slot HTML options: `options`, `headerOptions`, `titleOptions`,
  `hintOptions`, `actionsOptions`, `bodyOptions`, `footerOptions`;
- `tag` and `titleTag` when semantic HTML requires another element.

The widget supports both `::widget()` and `::begin()` / `::end()`. Titles and
hints are plain text and encoded. Actions, body and footer are trusted HTML,
arrays of fragments, or closures owned by the view/package.

Use direct `sx-surface` markup only for a genuinely low-level composition such
as `sx-metrics`, a table/list shell or a component whose semantic structure
does not match the widget. Reusable missing behavior belongs in the widget and
global `BackendUiAsset` CSS, not in a project-specific card shell.

## CSS contract

Canonical structure uses:

- `sx-surface` with `--raised`, `--clip`, `--padded`, `--compact` and
  `--responsive` modifiers;
- `sx-surface__header`, `__heading`, `__title`, `__hint`, `__actions`,
  `__body` and `__footer` slots;
- `__header--bordered`, `__body--flush` and `__footer--stretch` slot modifiers;
- semantic `--sx-surface-*` tokens from `BackendUiAsset`.

Project themes may override surface variables for brand geometry and palette.
They must not duplicate the complete surface, header, footer, shadow or mobile
stacking implementation.

## Compatibility boundary

Do not emit `sx-block` or `sx-panel` in new or deliberately migrated Admin/UPA
interfaces. They live only in deprecated `BackendBlockAsset` and
`BackendPanelAsset` compatibility bundles depending on `BackendUiAsset`; never
add either as a dependency of a new component. The UPA shell must stay free of
both compatibility bundles; the Admin shell may load `BackendBlockAsset` while
old administration views remain. Remove these adapters only after searches
confirm that all installed consumers emit canonical `BackendSurfaceWidget` or
`sx-surface` markup.

When migrating a screen, remove project-era panel/card selectors after their
remaining product-specific layout has been separated from the global surface
contract. Verify populated and empty content, light and dark themes, desktop
and narrow viewports, focus visibility and horizontal overflow.
