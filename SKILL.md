---
name: skeeks-cms-package-development
description: "Develop and evolve shared SkeekS CMS Composer packages such as skeeks/cms and skeeks/cms-backend. Use only for internal package engineering: public PHP contracts, reusable backend controllers, grids, filters, bulk and iframe actions, widgets, assets, migrations, package architecture and cross-project compatibility. Do not use for routine website content, CRM operations or project-only customization."
---

# SkeekS CMS package development

## Maintain this skill

The canonical source of this skill is the separate repository
`https://github.com/skeeks-cms/skeeks-cms-package-development`. On the SkeekS
Windows workstation its editable working copy is
`C:\SkeekS\dev\php\vendor\skeeks\skeeks-cms-package-development`.

When the user asks to extend or correct this skill, edit that repository.
Do not edit an installed or cached copy under the Codex user directory.

## Establish scope

Treat changes as shared framework work. They may affect every project that
installs the package.

Choose the owning package before editing:

- `skeeks/cms`: CMS models, domain behavior and administration;
- `skeeks/cms-backend`: reusable backend and cabinet controllers, actions,
  widgets, filters and presentation assets;
- `skeeks/cms-backend-admin`: administration-specific component wiring,
  theme subclass, header/footer/quick-access slots, guest/auth layouts,
  administration auth composition and admin-only integrations;
- `skeeks/cms-theme-unify-v2`: temporary compatibility package for old Unify
  layouts, assets and markup while they are being migrated; do not add new
  reusable UI, shell behavior or semantic renderers here;
- `skeeks/cms-mcp`: MCP/REST transports, tool contracts and API services;
- `skeeks/cms-oauth2-server`: OAuth resources, clients, codes and tokens.

The architectural target is one reusable `BackendComponent` foundation for
administration, UPA/client accounts and future role- or product-specific
cabinets. It owns the common shell and page-building rules. Administration
and the current UPA are consumers of this foundation, not separate UI
frameworks. A new site should receive a strong client account out of the box,
then customize it in the project through navigation, permissions, branding,
semantic theme values and exceptional product behavior rather than by forking
the shared shell or controller lifecycle.

Keep `BackendController` as the common controller foundation for cabinet
pages. Keep light/dark mode, the mode switcher and palette customizer on the
shared backend contract so Admin, UPA and future cabinets do not create
parallel implementations.

Customer cabinets consume `BackendAppAsset` and the standard backend shell
directly. `BackendTheme` supplies the comfortable cabinet sidebar by default;
the administration theme explicitly selects the dense default sidebar. Do not
restore `BackendCabinetAsset`, a global `cabinet.css` or a second cabinet shell.
Tune cabinet presentation through the shared `--sx-shell-*` variables and
semantic header/sidebar/footer slots; keep only brand values and exceptional
product screens in project assets.

`skeeks/crm` is a legacy package scheduled for removal. Do not add features,
UI migrations, compatibility work or other new changes there. Treat existing
uncommitted changes in that repository as a separate legacy workstream and do
not include them in new shared UI/theme stages. Put reusable replacement
contracts in their current owning packages, primarily `skeeks/cms` and
`skeeks/cms-backend`, after inspecting active non-CRM consumers.

Keep project-specific text, access rules and visual identity in the project.
Move behavior into a shared package only when at least two controllers,
projects or cabinet types can use the same contract.

For a reusable backend UI primitive, also require the boundary test documented
in `references/backend-ui-assets.md`: prove repeated semantics and states,
keep domain decisions in their owning package, and avoid widening the global
asset graph when an existing backend bundle already reaches every consumer.

## Work conservatively

1. Read the target package's `AGENTS.md` completely.
2. Inspect its Git status and preserve unrelated changes.
3. Use `ast-index` from the shared vendor root before raw search for PHP
   symbols, inheritance, usages and callers.
4. Inspect existing implementations and consumers before changing a public
   property, event, callback signature or default.
5. Keep existing administration backward-compatible. Prefer explicit opt-in
   for new presentation or behavior.
6. Put reusable PHP, markup, JS and structural CSS in the owning package.
7. Expose theme differences through semantic CSS variables instead of copying
   component CSS into every project.
8. New and migrated cabinet HTML uses only semantic `sx-*` classes and
   `data-sx-*` behavior hooks. Remove project-era `portal-*` and Unify
   `u-*`/`u-side-*`/`g-*` classes after moving their required presentation to
   the semantic contract.
9. Do not make Bootstrap classes part of new shared markup or a reusable
   backend contract. Prefer semantic `sx-*` hooks with CSS Grid or flex for
   new and deliberately migrated layouts. Existing local Bootstrap markup may
   remain until its screen is intentionally migrated.
10. Treat Bootstrap independence and complete Bootstrap removal as separate
    milestones. Prioritize migrating Admin pages from `sx-block`/`sx-panel` to
    `sx-surface`; remove Bootstrap layout utilities opportunistically from each
    touched screen, but keep explicit Yii/plugin behavior providers and
    compatibility adapters while they still have consumers. Do not make a
    zero-Bootstrap runtime a prerequisite for the surface migration. Schedule
    complete removal only after separate CSS and JavaScript usage audits prove
    that forms, dropdowns, modals and installed widgets have replacements.
11. Project theme CSS assigns shared `--sx-*` brand values directly; do not
   create a parallel project token graph that only aliases back to `--sx-*`.
12. Domain screens that are explicitly project-owned, such as the current
    `skeeks.com` GPD/store workflow, keep their established layout and
    project CSS. A namespace cleanup to `sx-gpd-*`/`sx-store-*` does not make
    that markup a reusable backend contract; promote only independently
    proven common primitives and do not redesign those screens incidentally.
13. Do not routinely clear published assets when keyed asset URLs already
    provide cache busting.

Do not update the shared vendor index unless the user explicitly asks.

## Use package references

Read the relevant reference completely before acting:

- For `BackendModelStandartController`, `BackendGridModelAction`, collection
  renderers, page actions, empty states, adaptive filters, bulk editing through
  standard iframe actions and theme tokens, read
  [references/backend-model-controller.md](references/backend-model-controller.md).
- For personal accounts and customer cabinets built on the SkeekS backend
  foundation, read both
  [references/backend-model-controller.md](references/backend-model-controller.md)
  and [references/customer-cabinets.md](references/customer-cabinets.md).
- For backend/UPA CSS and JS ownership, AssetBundle dependencies, payload
  budgets, icon providers and migration away from mandatory Unify assets, read
  [references/backend-ui-assets.md](references/backend-ui-assets.md).
- For new Admin/UPA containers, surface composition and migration away from
  `sx-block`/`sx-panel`, read
  [references/surfaces.md](references/surfaces.md).
- For standard in-cabinet web notifications, the header notification center,
  task notification rules and recipient selection, read
  [references/backend-notifications.md](references/backend-notifications.md).
- For the shared shop partner program, its package boundary, financial
  invariants, site scope and project integration, read
  [references/partner-program.md](references/partner-program.md).
- For reusable HTML/PDF reports, print settings, pagination, task-result
  formatting, attachments and default-template migrations, read
  [references/pdf-report-rendering.md](references/pdf-report-rendering.md).
- For shared lead ingestion, source adapters, idempotency, source payloads and
  source-submission links, read
  [references/lead-ingestion.md](references/lead-ingestion.md).

For a complete backend UI implementation or migration, also follow the
installed package runbook at `skeeks/cms-backend/BACKEND_UI_GUIDE.md`. Treat
it as the final checklist for presentation mode, entity cells, model cards,
drawers, conditional assets, themes, verification and project/package
boundaries.

Add future package knowledge as focused files under `references/`. Keep this
main workflow concise and do not duplicate the same contract in multiple
references. Record a mechanism only after implementing or verifying it.
Executable source remains the final source of truth.

## Validate shared changes

Verify in proportion to the blast radius:

1. Run `php -l` for every changed PHP file.
2. Run the narrowest available package tests or a deterministic runtime smoke
   test.
3. Exercise both old default behavior and the new opt-in behavior.
4. For UI work, test regular and privileged users, empty and populated data,
   active filters, simple controllers and multi-action controllers.
5. Check light and dark semantic variables when adding reusable CSS.
6. Recheck Git diff and do not stage or commit unrelated work.
