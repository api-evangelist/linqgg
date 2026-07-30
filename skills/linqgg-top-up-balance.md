---
name: linqgg-top-up-balance
description: Take money from a player through hosted checkout, native card, Apple Pay or Brazil Pix, and credit their LinQ game account.
api: LinQ Wallet Public API
generated: '2026-07-19'
method: generated
source: https://docs.linq.gg/modules/money
operations:
  - linq.geo.operations.v1.LocationService.IsOperationAllowed
  - linq.money.payments.v1.PaymentsService.IsLimitReached
  - linq.money.payments.v1.PaymentsService.GetPaymentProfile
  - linq.money.payments.v1.PaymentsService.NewReplenishOrder
  - linq.money.payments.v1.PaymentsService.CreatePixOrder
  - linq.money.payments.v1.PaymentsService.GetOrderStatus
  - linq.money.payments.v1.NativePaymentsService.GetCardPaymentConfig
  - linq.money.payments.v1.NativePaymentsService.GetApplePayConfig
  - linq.money.payments.v1.NativePaymentsService.GetPaymentSources
  - linq.money.payments.v1.NativePaymentsService.MakePayment
---

# Top up a player's LinQ balance

Every top-up is an **Order**. You create the order, settle it, then confirm its
status. Amounts are integers in coins throughout — `$5.00` is `500`.

## 1. Gate the operation

Call `LocationService.IsOperationAllowed` (`linq.geo.operations.v1`) with
`operation: depositing` and the device `coordinates` (latitude and longitude are
required). Proceed only if `allowed` is true. Coordinate checks gate individual
money operations; an IP-level ban blocks the app entirely but still permits demo
or free play.

Then call `PaymentsService.IsLimitReached` with `operation: depositing` and the
intended `amount`. If `is_reached` is true, stop and show the player how much
`remains` against the current `limit`. Anonymous profiles are capped at a total
deposit of 20.00 — offer wallet sign-in or `AuthUserService.SaveGameUser` to
lift it.

Optionally call `PaymentsService.GetPaymentProfile` (passing `app_store_country`
when you have it) to learn the real-money `currency` and `country` that apply.

## 2. Create the replenishment order

```
const payload = await service.newReplenishOrder({
  amount: 500,            // coins
  reference: 'any',       // your own correlation id
}, getAuthorization(user.walletToken ?? user.accessToken));
```

The response is an `OrderStatusResponse` carrying `id`, `status`, `amount`,
`reference` and — for replenish orders — a `checkout` link.

This call is naturally idempotent: if the order has not yet been processed, a
repeat request returns the **existing order with the same id** rather than
creating a duplicate. Do not generate a second order to retry.

## 3. Settle it — pick one route

### Hosted checkout

Open `checkout` in a webview. See `components/linqgg-components.yml` for the
UniWebView handoff: register the `window.uniwebview` flag on `OnPageFinished`,
then read the `completion` message with its `success` argument on
`OnMessageReceived`. Treat that message as a hint only — always re-read the order
status in step 4.

### Native card

Call `NativePaymentsService.GetCardPaymentConfig` with the `order_id` to get the
TokenEx and Kount configuration. Collect card number, CVV, cardholder name,
expiry and billing address in your own UI, then hand them to the Unity SDK:

```
var order = await LinqSDK.CheckoutByOrdinaryCard(order, details, address);
```

Never send captured card data to your own backend. The SDK tokenizes on device
via TokenEx; routing it through your servers breaks the PCI DSS posture LinQ
documents. Server-side, the equivalent is
`NativePaymentsService.MakePayment` with `card_tokenex_payment` holding the
TokenEx `token` and `token_hmac`, never a raw PAN.

Billing address requires country (ISO 3166-1 alpha-2), region (US and BR) and
zip. Wrap the call in try/catch — the SDK throws `InvalidOperationException`
on failure rather than returning an error object.

### Apple Pay

Call `NativePaymentsService.GetApplePayConfig` with the `order_id`, then:

```
var order = await LinqSDK.CheckoutByApplePayCard(order);
```

Handle all three exceptions or the game will hang:
`PaymentFailureException` (authorized but failed at the provider — show a failure
message), `PaymentDiscardException` (player closed the sheet — treat as
cancellation), `PaymentUnknownException` (Apple Pay unsupported, including the
Unity Editor — fall back to ordinary card).

### Saved card

Call `NativePaymentsService.GetPaymentSources` with the `order_id` to list stored
instruments, then `GetSavedCardPaymentConfig` with `order_id` and `saved_card_id`,
and settle with `MakePayment` using `saved_card_payment`.

### Brazil Pix

Use `PaymentsService.CreatePixOrder` instead of `NewReplenishOrder`. It nests the
order params and optional `pix_params` (`tax_id` — the Brazilian CPF — and a
`BillingAddress` with `country: 'BR'` and a two-letter state `region`).

```
const result = await service.createPixOrder({
  order: { amount: 500, reference: '1234' },
  pixParams: { taxId: '...', address: { country: 'BR', region: 'RJ', ... } },
}, getAuthorization(user.walletToken ?? user.accessToken));
```

Display `result.pixParams.code` to the player. Anonymous players must have full
name and email saved via `AuthUserService.SaveGameUser` before a code can be
generated.

## 4. Handle 3DS

If `PaymentResponse.script_3ds` is set, open it in a webview to run the issuer
challenge before treating the payment as settled. If `retry_order_id` is set
(Apple Pay), retry against that order id.

## 5. Confirm the outcome

Always re-read the order. Webview and SDK signals are not authoritative.

```
const payload = await service.getOrderStatus({ id: orderId },
  getAuthorization(user.walletToken ?? user.accessToken));
if (payload.response.status === 'completed') {
  // apply the internal game operation
}
```

Status machine: `pending` -> `awaiting` -> `processing` -> `accepted` ->
`completed`. Terminal failures are `declined` (payment provider rejected it),
`cancelled` (player cancelled) and `terminated` (system cancelled). Only
`completed` should trigger crediting the player in your game.

LinQ publishes no decline reason codes — `declined` is the whole signal. Show a
generic failure and let the player start a new order.

For Pix, poll `GetOrderStatus` after a delay; there is no callback or webhook.

## Testing

Against `services.stage.galactica.games`:

- Ordinary card: `4242 4242 4242 4242`, exp `12/2027`, holder `CARD HOLDER`, CVV `123`.
- Apple Pay sandbox: `5204 2452 5000 1488`, exp `11/2022`, CVV `111` (needs an
  Apple Pay sandbox tester account; transactions fail at the provider).
- Set the cardholder name to `NOFRAUD` to skip Kount anti-fraud checks when the
  native modules are unavailable (Android or the Unity Editor).
- Pix cannot complete automatically in stage — drive the order with
  `HelpersService.UpdateOrder` to `COMPLETED` or `DECLINED`.
