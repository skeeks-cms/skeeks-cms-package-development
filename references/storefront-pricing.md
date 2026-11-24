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

The total shows kopecks only when there are any, on the server
(`asDecimal($amount, 2 or 0)`) and in the browser recalculation
(`Intl.NumberFormat` with 2 or 0 fraction digits): 16 164 and 6 981,70, the
same sum the cart shows (theme-unify-shop 2.2.1.27).

Basket and order lines store `shop_order_item.quantity` as `decimal(18,4)`
(cms-shop 3.2.7.47), like `shop_store_product` and `shop_bill_item`. With
`decimal(18,2)` two packs of 3.592 m² were saved as 7.18 m² and the cart total
differed from the card. Keep the exact pack multiple; round only money.

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
