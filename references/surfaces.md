# Backend surfaces

`skeeks/cms-backend` owns the single surface contract for new Admin, UPA and
customer-cabinet interfaces.

The long-term goal is for every ordinary backend page to be composed from the
same small set of semantic elements, with `sx-surface` as the canonical
container. Administration, UPA and future cabinets must share this page
grammar even when their navigation, permissions, density and branding differ.

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
- `sx-surface-stack` for a vertical group of sibling surfaces with canonical
  spacing;
- semantic `--sx-surface-*` tokens from `BackendUiAsset`.

Project themes may override surface variables for brand geometry and palette.
They must not duplicate the complete surface, header, footer, shadow or mobile
stacking implementation.

## Compatibility boundary

Do not emit `sx-block` or `sx-panel` in new or deliberately migrated Admin/UPA
interfaces. They live only in deprecated `BackendBlockAsset` and
`BackendPanelAsset` compatibility bundles depending on `BackendUiAsset`; never
add either as a dependency of a new component.

The UPA/client shell is already outside this compatibility boundary and must
stay free of `sx-block`, `sx-panel` and both compatibility bundles. The
standard administration shell also no longer loads `BackendBlockAsset` through
`BackendAdminAppAsset`; an installed legacy view that still emits `sx-block`
must register the compatibility asset explicitly. Administration-only
`AdminPanelWidget`/`AdminPanelAsset` remain explicit compatibility entry points
for external historical `sx-panel` consumers through `BackendPanelAsset`; no
standard Admin dashboard or hosting shell loads them. Do not treat these
adapters as new primitives or copy them into another cabinet.

Migrate administration incrementally, screen by screen. Each touched screen
must replace its `sx-block`/`sx-panel` composition with
`BackendSurfaceWidget` or canonical `sx-surface` slots and stop registering
the obsolete compatibility bundle when it has no remaining consumer. Keep the
deprecated asset classes functional as explicit compatibility entry points;
remove them only after repository and rendered DOM checks confirm that no
installed consumer remains.

When migrating a screen, remove project-era panel/card selectors after their
remaining product-specific layout has been separated from the global surface
contract. Verify populated and empty content, light and dark themes, desktop
and narrow viewports, focus visibility and horizontal overflow. Do not add
Bootstrap grid or utility classes to the replacement shared contract; use
semantic slots with CSS Grid/flex where layout is needed.
