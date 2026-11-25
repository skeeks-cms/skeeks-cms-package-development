# Storefront structured data: listings and product cards

## Organization logo is an absolute link

`cms-theme-unify-v2` `views/modals.php` prints a hidden microdata
`Organization` on every page; `publisher` blocks of publications
(`content-element/default.php`) and the brand page (`theme-unify-shop`
`content-element/brand.php`) repeat it. `theme->logo` is usually a relative
`/uploads/...` path, and a `meta content` value is plain text that search
engines do not resolve against the page. Output the logo as
`<link itemprop="logo|image" href="Url::to($this->theme->logo, true)">`, skip
the tags when the logo is empty, and `Html::encode()` the site name, phone,
email and address (cms-theme-unify-v2 2.2.2.8, theme-unify-shop 2.2.1.29).
Test: `cms-theme-unify-v2/tests/organization-markup.php <vendor/autoload.php>`.

## Product only on a single product page

`schema.org/Product` (with `Offer`, `AggregateRating`, `Brand`) belongs to the
product card only: `theme-unify-shop` `views/modules/cms/content-element/product.php`
and `_product-price.php`. Google accepts Product rich results for one product
or its variants, not for a category or filter page. Do not wrap a listing in
`Product`/`AggregateOffer`/`AggregateRating`.

Product `description` (Yandex requires it for product snippets): the card takes
`productDescriptionShort`, otherwise `productDescriptionFull` (both getters
already fall back to the main/parent product), strips HTML including
`script`/`style`, decodes entities, collapses whitespace and cuts to 500
characters at a word boundary (theme-unify-shop 2.2.1.22). Since 2.2.1.28 it is
never empty: next come the product's collections (`shopProduct->collections`,
for an offer without its own — those of `shopProductWhithOffers`),
`description_short` then `description_full`; last, a line built from `seoName`
with «Бренд», «Коллекция», «Страна» and «Цена указана за 1 <measure symbol>»
(the same unit wording the Yandex feed uses when a product has no description).
Supplier-imported products usually have no own text but a described collection.
Test: `theme-unify-shop/tests/product-description.php <vendor/autoload.php>`.

`ShopComponent::getAgregateCategoryData()` (`cms-shop`) still returns
`offerCount`, `lowPrice`, `highPrice` and the rating keys (`reviewCount`,
`ratingValue`, `bestRating`, `worsRating`). The theme uses only `offerCount`
and `lowPrice` (`{=minMoney}`, listing totals); project overrides of
`catalog.php` may still read the other keys, so keep them unchanged.

## Listing: CollectionPage + ItemList

`views/modules/cms/tree/catalog.php` (sections and saved filters, since
theme-unify-shop 2.2.1.14) outputs one JSON-LD `CollectionPage`:

- `url` and `@id` (`<url>#webpage`) from the saved filter or the tree;
- `name` from `$model->seoName`; `description` equals the page meta
  description (the tree-type template when it was applied, otherwise the
  model field — `SavedFilterController` copies filter texts into the tree);
- `mainEntity` `ItemList` of the products actually rendered on the page, with
  `position` continuing across pagination and `numberOfItems` = listing total.

Handshake: before rendering the list view, `catalog.php` sets
`$this->params['sxCatalogLdItems'] = []`; `views/products/product-list.php`
fills it with `url`/`name` of `$dataProvider->getModels()` only while it is an
empty array (first list of the page). Do not read the data provider from
`catalog.php` before the list view renders: `product-list.php` changes its page
size, and the collections view mode rewrites its query. In collections mode
and in project overrides of `product-list.php` the ItemList is simply absent.

The block is skipped for an empty listing and when the section/filter
description already contains its own `CollectionPage` JSON-LD: hand-written
SEO markup in content takes priority, the theme must not duplicate it.

The brand page `views/modules/shop/brand/view.php` (since theme-unify-shop
2.2.1.25; before it wrapped the brand in `NewsArticle` microdata) follows the
same contract: one JSON-LD `CollectionPage` with `@id` `<brand url>#webpage`,
`description` read from the meta description registered by `BrandController`,
`about` = `Brand` (name, logo) and the same `sxCatalogLdItems` handshake around
`product-list` (no ItemList in collections mode). It is skipped when the brand
has no products or its description already has `CollectionPage`.
`NewsArticle` stays only on real publications.

## Offer availability and AggregateRating on the product card

Since theme-unify-shop 2.2.1.27 `_product-price.php` has no hard-coded
`availability`. `helpers\ProductAvailability::forSite($shopStoreProducts)` gives
one decision for both the `Offer` markup and the visible status, and
`_product-availability.php` renders them together inside the `Offer`:

- the site keeps no stores (`shop->allStores` empty): stock is not confirmed,
  neither markup nor status;
- positive quantity: `InStock`, «В наличии» (supplier stores only: with the note
  «на центральном складе», the wording of `_product-quantity.php`);
- no quantity, `shopSite->is_show_product_no_quantity`: `BackOrder`, «Под заказ»;
- otherwise `OutOfStock`, «Нет в наличии». It replaces the former «Товара нет»
  rows that depended on `is_show_quantity_product`.

Never set availability from the buy button alone and never default to InStock.

`Offer` `price` is the final unit price the visitor sees as «Ваша цена»:
`$priceHelper->minMoney` (shop discounts applied), formatted with two decimals,
and `priceCurrency` from the same `Money` (theme-unify-shop 2.2.1.28). Never
`basePrice`: Yandex Webmaster «Товары» compares the markup with the visible
price and the product feed (which exports the guest discounted price plus
`oldprice`), and a base price on discounted items is reported as an error.
Test: `theme-unify-shop/tests/product-offer-price.php` with `short_open_tag=1`.

`AggregateRating` in `product.php` and the visible block `_product-rating.php`
(rendered at the top of `_product-price.php`, so all layouts get it) share one
condition: `rating_value > 0 && rating_count > 0`, same values, `bestRating`
from `shopSite->max_product_rating_value`. The count is called «оценки», not
reviews. `shopSite->is_generate_product_rating` fills empty ratings with
generated values (range in shop settings); the theme cannot tell them from
real ones, so the owner decides per site whether to keep generation on.

Test: `theme-unify-shop/tests/product-availability.php <vendor/autoload.php>`
with `short_open_tag=1`.
