# Storefront structured data: listings and product cards

## Product only on a single product page

`schema.org/Product` (with `Offer`, `AggregateRating`, `Brand`) belongs to the
product card only: `theme-unify-shop` `views/modules/cms/content-element/product.php`
and `_product-price.php`. Google accepts Product rich results for one product
or its variants, not for a category or filter page. Do not wrap a listing in
`Product`/`AggregateOffer`/`AggregateRating`.

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
