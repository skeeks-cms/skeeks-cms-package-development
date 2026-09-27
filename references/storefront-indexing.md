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

Brands and saved filters bound to a brand (`shop_brand_id`) are excluded when
the brand has no products visible in the default listing: active site
elements without child offers, with the same stock rule as above. The check is
conservative — when visibility cannot be determined, nothing is excluded, and
filters by properties or country stay in the sitemap (their empty pages carry
`noindex`).

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
