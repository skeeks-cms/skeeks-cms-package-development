# Storefront ecommerce (Yandex Metrica dataLayer)

## Ownership

`skeeks/cms-shop` `src/assets/classes/Shop.js` raises storefront events on
`sx.Shop` (`detail`, `add`, `remove`, `purchase`, `buyOneClick`). The theme
turns them into `dataLayer` pushes: `skeeks/theme-unify-shop`
`src/assets/src/js/classes/Shop.js`. Site-level analytics code (goals in the
SEO component's counters code) subscribes to the same `sx.Shop` events instead
of parsing pages or duplicating ecommerce payloads.

## One purchase source

The order-finish page is the single source of `purchase`.
`theme-unify-shop` `views/modules/shop/order/finish.php` triggers it only when
the session flash `order` equals the displayed order id. The flash is consumed
on the first view, so reopening the page does not resend the purchase.

`CartController::actionBuyOneClick` recalculates and saves the order before
responding, sets the same flash and returns `redirect` to the order page.
`createAjaxBuyOneClick` therefore triggers `purchase` from the AJAX response
only when the response has no `redirect`; otherwise the finish page sends it.
Firing from both places produced duplicate purchases, and a push made right
before navigation can be lost.

The theme `purchase` handler additionally skips events without `order.id`,
sends one purchase per order id per tab (`sessionStorage`), passes `revenue`
as a number and warns in the console when the amount is not positive. Do not
drop a purchase because of a zero amount: it hides the defect and loses the
conversion.

## Product lines

Build `purchase` products from `ShopOrderItem`, not from the current catalog
price: `ShopComponent::productDataForJsEvent()` supplies id, name, category
and brand; the unit price is `moneyWithDiscount` (`amount` is the price before
the per-unit `discount_amount`).

Metrica accepts only integer quantities and truncates fractional ones, so
product revenue no longer matches the order. For a fractional quantity (m²,
kg) send one line: `quantity: 1`, `price`: the line total, and the real
quantity with `measure_name` in `variant`. Integer quantities keep the unit
price. `actionField.revenue` stays the saved order amount.

## Verification

Check logic without sending data to Metrica by temporarily replacing
`dataLayer.push` in the browser and triggering `sx.Shop.trigger('purchase', …)`;
remove the `sessionStorage` keys afterwards. A real end-to-end check needs an
order, which also lands in the site's statistics and notifications: mark it as
a test order and get the owner's consent.
