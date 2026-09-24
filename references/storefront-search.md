# Storefront live search

`skeeks/cms-search` owns the `cmsSearch/suggest/index` JSON endpoint,
`StorefrontSuggest`, `SuggestQuery` and `LiveSearchAsset`. The Unify theme only
registers the asset with a `class_exists` guard; reusable search markup, CSS and
JavaScript must not move into the legacy theme. Existing full-search forms and
their configured query parameter remain the fallback.

The endpoint is GET-only, noindex and `private, no-store`. It returns bounded
groups rather than counts for the whole catalog. Terms are parameterized,
combined with AND across searchable columns, and capped in length and number.
Latin `x` and Cyrillic `х` in dimensions match symmetrically. Do not use `%`
from raw input as an unbounded SQL wildcard.

Sections and saved filters belong to the current site; filters require an active
parent section. Shop brands and collections are global and require an active product on the
current site. Products use the shop's existing product-type filter and the
configured content scope. Read prices through `ShopOrder::getProductPriceHelper`
and preserve price visibility and stock rules; never return base/purchase prices
as a substitute or share personalized results in cache.

For a read-only suggestion request, do not call `shop->shopUser`: its getter can
merge guest carts or create a saved user, and `ShopUser::getShopOrder` can save a
draft order. Read the existing site-scoped ShopUser/order from the authenticated
identity or guest session reference. If none exists, use an unsaved ShopOrder for
price calculation. Read session context first, close the session before catalog
queries, and avoid getters that reopen it later.

The browser waits 250 ms after input, cancels outdated fetches and checks request
revision before rendering. During refinement keep panel geometry stable but
make stale links inert. Text uses DOM text nodes; images and links accept only
HTTP(S). Keep full search available on errors and support arrows, Escape,
outside dismissal and a bounded scroll area on mobile.

Search original text and up to two deterministic Russian-to-Latin spellings.
Merge by group and entity ID, preserve original matches first and the per-group
limits. An existing Russian match must not suppress Latin brands or collections.
Return `highlightQuery` containing the searched spellings. If the original result
is empty, return the first successful spelling as `matchedQuery`; show that
substitution in suggestions. Full-results links and submission retain the original
query, since the full page now searches all spellings itself.
Brands use their `logo` relation, not `image`. Product matching includes brand name.

`SuggestQuery::applyProduct()` adds exact positive numeric product-ID matching
inside the existing site, visibility and content constraints, with ID matches
ranked before text/SKU matches. Both suggestions and full results use it. Do not
apply an ID `orWhere()` to the complete scoped query or match partial IDs with LIKE.

The shop full-results template uses `StorefrontSuggest::productQuery()` and the
same navigation providers through `page()`. `SuggestQuery::applyAll()` combines
original and transliterated conditions in SQL before LIMIT/OFFSET; original
matches rank first, then exact/prefix titles, priority and entity ID. Do not merge
independently paginated lists: that causes duplicates and missing records.
Navigation groups fetch eight plus one lookahead row and expose `nextPage` through
the GET-only `suggest/page` endpoint. Product results retain the theme's existing
cards, price/cart behavior and pager, with `triggerOffset=0` for manual loading.
`product-list.php` accepts optional `pagerOptions`; its default stays unchanged.
Keep navigation rendering/assets in cms-search and the shop template as adapter.

`cms-search/tests/suggest-query.php` exercises real MySQL matching using a uniquely
named connection-local temporary table; it never changes catalog rows. Also
verify public HTTP responses, rapid input, keyboard navigation and desktop/mobile
layout against the deployed site. A browser viewport test does not substitute
for a physical mobile keyboard test.
