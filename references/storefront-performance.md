# Storefront performance contracts

## Measure the whole request

Use separate Yii profiles for the initial HTML and each PJAX request, and compare
like-for-like guest/authenticated sessions. SQL duration is only part of request
time. A reduction in SQL count alone is not a measured page speedup. Normalize
SQL literals before saving diagnostics; never print sessions, cookies or raw
configuration from debug files.

## HTML compression

`skeeks/yii2-assets-auto-compress` owns `HtmlCompressor` and the Tyler formatter.
Read string input using an advancing byte offset. Copying the remaining document
for every line makes multiline HTML compression quadratic in document size.
Preserve existing whitespace, comment/extra flags, pre/textarea handling and
statistics behavior; verify output equivalence against the previous implementation.
The package's `tests/html-compressor.php` checks fixed cases and a large document.

## Product lists

`skeeks/cms-shop/helpers/ProductCardData` owns the snapshot of stock rows,
favorite flags and comparison flags for one rendered page/list. It queries only
the listed product IDs, uses the existing cart/shop-user owner relations, and
uses the current shop's stores. An empty store list intentionally preserves
`ShopProduct::getShopStoreProducts([])` semantics (no store restriction).

Never put this snapshot in shared cache or retain it across user/store changes
or mutations. Load a fresh snapshot for the next render. Do not replace the
unfiltered `shopStoreProducts` ActiveRecord relation with a filtered subset.
The snapshot keeps filtered stock separately and does not change price,
discount, offer selection or permission logic.

The existing `theme-unify-shop` catalog and product slider preload `image`,
`images`, `shopProduct.baseProductPrice` and `shopProduct.shopProductPrices`, then
pass the snapshot through Yii ListView `viewParams`. Individual/custom card
renders retain the old query path when no matching snapshot is supplied. Theme
updates tolerate an older cms-shop without ProductCardData; stock/flag batching
starts when the helper becomes available. Project-overridden list templates must
adopt the same preload/viewParams contract to obtain the same improvement.

Reuse ActiveDataProvider's current-page models for emptiness checks; do not run
`query->one()` before rendering the same list. Preserve query filters, ordering,
limits, page selection and existing price calculation.

## Verification

Run PHP 8.2 checks with the project's Composer autoloader, without bootstrapping
a production application:

- `cms-shop/tests/product-card-data.php <vendor/autoload.php>` uses an isolated
  in-memory SQLite fixture and real Yii SQL/owner relations. It compares batched
  results with individual queries, including users, guests, store scopes, absent
  and negative stock, empty lists and fresh reads after mutation.
- `theme-unify-shop/tests/product-lists.php <vendor/autoload.php>
  <cms-shop/tests/product-card-data.php>` runs the real shared list templates,
  ListView and ActiveDataProvider with isolated fixtures. Enable
  `short_open_tag=1`. The per-card markup is intercepted: this checks data flow,
  eager loading and pagination, not full visual rendering or real discount rules.

After deployment, measure actual page and PJAX profiles again. Do not present
microbenchmarks or fixture query counts as production before/after results.
