# Storefront indexing: canonical, empty listings and sitemap

## Canonical

`skeeks/cms-seo` `CanUrl` adds `<link rel="canonical">` at the end of the page
only for controllers listed in `CmsSeoComponent::$canUrlEnableDefaultControllers`
and only when a page did not set its own canonical. The list includes
`cms/tree`, `cms/content-element`, `cms/saved-filter`, `cms/cms`,
`shop/brand` and `shop/collection`. A project can switch the mechanism off
with `'seo' => ['canUrl' => false]`; check the project config before
concluding that canonical is broken in a package.

`TreeController` sets an explicit canonical from `Tree::canonicalUrl`. A tree
with `redirect_tree_id` has the redirect target as its URL, so its canonical
points there as well.

## Empty listings

The listing itself decides emptiness: `theme-unify-shop`
`views/modules/cms/tree/catalog.php` (sections and saved filters) and
`views/modules/shop/brand/view.php` register meta robots `noindex, follow`
when the counted listing total is zero. Do not detect emptiness by the
«Товары не найдены!» text: several templates render an empty listing without
it.

The default listing hides out-of-stock products according to
`ShopSite::is_show_product_only_quantity`, applied by
`AvailabilityFiltersHandler`: `0` — no stock filter, `1` — stock in the
site's stores, `2` — stock in the site's and supplier stores (for the product
or its offers). A brand or filter with products can therefore be empty.

## Sitemap

`cms-seo` `SitemapGenerator` is shared by the native job
`seo.sitemap.generate` (`SitemapJobHandler`) and the CLI command
`seo/sitemap/generate`; production sites run it as the job
(`cms-job/worker/push seo.sitemap.generate`). It runs without a web session,
so it must not use `AvailabilityFiltersHandler`, which reads the session.

Brands are excluded when the brand has no products visible in the default
listing: active site elements without child offers, with the stock rule above.
Saved filters using the `catalog` view check their own category query, including
descendants and optional secondary bindings, plus the saved brand, country and
property selector. A brand's stock elsewhere must not populate an empty category
selection. Property selectors use the storefront EAV `property_id` + `value_enum`
predicate and accept a match on the product or its child content element
(`parent_content_element_id`, not `main_cce_id`). Apply the same default stock
rule to the selected parent product. Unknown selectors and custom views are
retained conservatively; a single matching visible product is sufficient.

The generator excludes inactive trees and saved filters whose parent is inactive
or explicitly non-indexable. For the `catalog` view, it checks active site
products across descendants and optional secondary bindings, excludes child
offers, and applies the same default stock rule. Other tree views are retained.
Console execution reads `is_join_second_trees` from the active `CmsTheme.config`
(PHP serialization, not JSON); `includeSecondaryTreeProducts` can explicitly
override it when project theme defaults differ from the stored settings.
Collections require an active collection and an active site product linked
through `shop_product2collection`. Their listing does not apply the catalog's
stock/offer exclusions, so the generator must not apply those exclusions either.

Published sitemap parts are persistent public output, not a JobContext temporary
workspace. Build complete XML in a locked staging directory, replace each part
at its stable public address, then publish the index last. This gives complete
individual XML responses, not an atomic snapshot across separate HTTP requests.
When a group/part disappears, retain its stable address with an empty urlset;
exclude it from the new index. This avoids cached-index 404s without keeping
obsolete page URLs or accumulating one public directory per run. Static XML
cache headers and migration of old versioned addresses belong to the web server;
the owning `cms-seo/SITEMAP.md` documents the publication and rollout contract.
## 404 page

`cms` routes errors to `cms/error/error`; `base\Controller::render` resolves
the view through the theme, so `theme-unify-shop`
`views/modules/cms/error/error.php` overrides the core
`@skeeks/cms/views/error/error` for shops. For 404 it renders navigation:
`ShopSite::catalogMainCmsTree` and its active children (without free-link
`redirect` sections), falling back to active level-1 sections when a site has
no catalog, plus a `contacts`/`kontakty` level-1 section and the site phone.
It keeps HTTP 404, registers meta robots `noindex, follow`, gets no canonical
(`cms/error` is not in the CanUrl list) and never redirects. Navigation
lookups are wrapped in `try/catch` so the error page cannot fail. Other status
codes render the core view.

## SEO templates

`{=minMoney}` in tree-type meta templates is replaced with the listing's
`lowPrice` (positive prices only). When the listing has no priced products the
phrase «цены/цена/стоимость от {=minMoney}» is removed instead of printing
«от 0 р.».
