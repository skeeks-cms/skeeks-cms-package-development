# Storefront delivery integration

The shop delivery checkout has three distinct steps: the carrier callback
provides a tariff, the delivery form stores its amount, and submission refreshes
the order totals. Verify all three steps and reload the cart when changing a
carrier adapter; an API response alone does not prove the checkout price works.

The carrier-specific callback schema belongs in its delivery package. For
`skeeks/cms-shop-delivery-cdek`, see its `README.md` for the widget contract.
Keep optional price calculation compatible with configurations where the
delivery form does not render a price input.

An opt-in calculated delivery amount must also be gated in the checkout
model's money resolver. Hiding its input alone leaves an older calculated
amount in the order data and can override the fixed delivery price.

## Automatic delivery price lifecycle

The common contract lives in `skeeks/cms-shop` (introduced in 3.2.7.26).
`ShopOrder` already emits `EVENT_BEFORE_RECALCULATE` and
`EVENT_AFTER_RECALCULATE`; do not add parallel cart events for each carrier.
`ShopOrderItem` calls order recalculation after insert, update and delete.
Inside that lifecycle `ShopOrder::refreshDeliveryCalculation($force)` invokes
the active checkout adapter, clears a populated item relation before reading
the current cart, and stores updated delivery data without recursively saving.
The surrounding order/item lifecycle persists the amounts and quote together.

New calculated delivery adapters extend `DeliveryCheckoutModel` and implement:

- `supportsAutomaticCalculation()`: explicit opt-in; inherited false preserves
  existing/fixed-price adapters.
- `refreshDeliveryPrice($force = false)`: derive inputs from server-side order
  data, retain destination and chosen tariff, update money on success, clear
  stale money and set `deliveryCalculationError` on failure, return a boolean.
  Force must bypass quote reuse.
- A freshness key covering every price input used by the provider (items,
  quantities, weight/dimensions, origin, destination, tariff, currency and any
  declared value/discount/configuration affecting that provider). Use bounded
  freshness, not an indefinite saved amount. Do not cache failed quotes as a
  valid zero price. A real zero-price quote is valid.

`loadStoredDeliveryData()` restores server-owned hash/time/error from persisted
order data; ordinary checkout `load()` must not accept these metadata fields.
`getStoredDeliveryData()` omits runtime order/handler/delivery object references.
Do not make quote metadata safe for mass assignment in an adapter's rules.
Keep carrier request formats, credential handling, parcel conversion and API
timeouts in the carrier package. Do not trust a price posted by its widget.

`CartController` refreshes calculated delivery on opening a draft cart and
forces verification before checkout. On failure checkout stops. If the price
changed during final verification, it persists the new cart total and requests
buyer review/retry before creating the order. Completed orders are not requoted.
New adapters (including Boxberry) implement this contract; inheriting it does
not automatically provide that carrier's API calculator.

Order AJAX JSON exposes `deliveryCalculation.error` and `calculatedAt`.
It also exposes `inputHash`, the opaque server-owned calculation fingerprint.
Carrier maps that embed cart inputs at initialization must invalidate their
displayed tariff list when this fingerprint changes, including while hidden
and before a pickup point is selected. A correct server total does not make an
already rendered provider map fresh. Keep this refresh inside the carrier
widget and retain fixed-price opt-out.
Invalidate hidden maps without rebuilding them. Defer provider scripts/points
until the buyer opens the map, and unload its iframe after confirming a point;
server-side price refresh must remain independent of that map's presence.
When upgrading a carrier widget, update its server protocol at the same time.
Prefer provider-supported loading by visible bounds over fetching all pickup
points; marker loading must stay independent of tariff/quote recalculation.
An explicit initial map centre can avoid geocoding at startup. Validate both
coordinates together and convert their order to the provider's contract;
camera settings must never change the shipping origin/destination or quote.
Map access, text geocoding and carrier authorization are separate dependencies.
Checkout widgets must update their calculation feedback from the response even
when the theme updates totals without rendering the delivery widget again.
Scope response handling to the current order and delivery, insert error text
as text (not HTML), and clear a stale error after a successful calculation.

Multiple carrier delivery methods can render checkout widgets simultaneously
in hidden tabs. Scope input selectors and document event namespaces to each
widget; one adapter instance must not unregister another's cart listener.
Unload a map already built when its delivery tab is deactivated, not just when
a destination is confirmed. Gate lazy requests on the active delivery.

Courier adapters must verify the provider city identifier, store the complete
address in handler data and map it to standard order delivery fields in
`modifyOrder()`. Include destination and sender/recipient modes in freshness
keys and validate allowed modes server-side. Separate a disabled presentation
select from the canonical submitted tariff input; otherwise concurrent form
saves omit the chosen tariff. Reject asynchronous options for an unsaved or
changed address, even if the response belongs to the current order. Carrier
quote calculation does not imply waybill creation or courier booking.

Verify quantity increases/decreases, add/remove, reopen, expired quotes,
destination/tariff/config changes, final forced verification, failures and
unavailable tariffs, valid zero prices, fixed-mode opt-out and completed-order
stability. Prefer isolated tests; live cart checks do not require placing orders.

When several delivery methods share a checkout-model class, a rendered widget
must match both the model class and delivery ID. Otherwise inactive forms
receive the active method's state and endpoints.
