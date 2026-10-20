# Storefront price for a selected quantity

In theme-unify-shop, the total block in `_product-price.php` uses
`ProductPriceHelper::minMoney`: the same final discounted unit price that the
main price of the product card shows. `ShopProduct::minProductPrice` and
`ProductPriceHelper::minPrice` hold the price before discount rules are
applied; never use them for this total or for the `data-amount` of the
browser-side recalculation.

`Money::mul()`/`multiply()` mutate the object. For the total, clone `minMoney`
first, then multiply the copy by `measure_ratio_min`. Keep the unit price
unchanged for later consumers of the same helper.

Verification: `theme-unify-shop/tests/product-price-total.php <vendor/autoload.php>`
with `short_open_tag=1`. It executes the real template block and checks a
discounted, a regular and a zero price, a fractional quantity, `data-amount`
and that the source `Money` objects stay unchanged. Set the application's
BCMath precision, for example `-d bcmath.scale=8`; `Money::mul` itself uses
the current global BCMath scale.

The total for the selected quantity and its `data-amount` are output only when
`$isShowPrice === true`: they follow the same price visibility setting and
availability check as the main price of the card. The existence of a price
alone does not permit showing it publicly. The total test covers both
visibility states; with a hidden price the block emits no HTML or JS.
