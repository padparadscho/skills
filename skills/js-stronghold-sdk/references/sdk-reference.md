# Stronghold.Pay.JS SDK Reference

## Table of Contents

- [Initialization](#initialization)
- [Customer Token](#customer-token)
- [Callback Functions](#callback-functions)
- [SDK Methods](#sdk-methods)
  - [addPaymentSource](#addpaymentsource)
  - [updatePaymentSource](#updatepaymentsource)
  - [charge](#charge)
  - [tip](#tip)
- [Arguments Reference](#arguments-reference)

## Initialization

Include scripts in `<head>` (order matters — do not load asynchronously):

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.4.1/jquery.min.js"></script>
<script src="https://api.strongholdpay.com/v2/js"></script>
```

Instantiate the client:

```js
const strongholdPay = Stronghold.Pay({
  publishableKey: "pk_sandbox_...", // matches environment
  environment: "sandbox", // 'sandbox' or 'live'
  integrationId: "integration_...", // unique per integrator
});
```

**Important**: If dynamically loading scripts, ensure ordering is preserved (default may be async).

## Customer Token

Required for all SDK method calls. Generated server-side via REST API:

```
GET /v2/customers/{customer_id}/token
Header: SH-SECRET-KEY: sk_sandbox_...
```

- 12-hour lifespan
- Customer-specific
- Not intended for storage — request a new one per session

## Callback Functions

All SDK methods accept these callbacks in the options object:

| Callback    | Signature         | Description                                                              |
| ----------- | ----------------- | ------------------------------------------------------------------------ |
| `onSuccess` | `onSuccess(data)` | Action completed successfully. `data` type depends on method (see below) |
| `onExit`    | `onExit()`        | Customer closed/exited the drop-in UI                                    |
| `onError`   | `onError(error)`  | Error occurred. Returns an Error object (see errors reference)           |
| `onReady`   | `onReady()`       | Drop-in UI is loaded and visible to customer                             |

### onSuccess return types

| Method                | Returns               | Description                  |
| --------------------- | --------------------- | ---------------------------- |
| `addPaymentSource`    | Payment Source object | Newly created payment source |
| `updatePaymentSource` | Payment Source object | Updated payment source       |
| `charge`              | Charge object         | Newly created charge         |
| `tip`                 | Tip object            | Newly created tip            |

## SDK Methods

### addPaymentSource

Opens the payment source linking drop-in UI. Customer connects a bank account via Plaid/Yodlee. The payment source is saved on Stronghold's servers.

```js
strongholdPay.addPaymentSource(customerToken, {
  onSuccess: function (paymentSource) {
    console.log("Linked:", paymentSource.id);
  },
  onExit: function () {
    console.log("User exited");
  },
  onError: function (err) {
    console.error("Error:", err.code);
  },
});
```

### updatePaymentSource

Updates credentials for an existing payment source when the financial institution requires re-authentication. Use when the API returns `payment_source_login_required`.

**Required option**: `paymentSourceId`

```js
strongholdPay.updatePaymentSource(customerToken, {
  paymentSourceId: "payment_source_GyaKTRI1RqDyvTRMNo4UFOak",
  onSuccess: function (paymentSource) {
    console.log("Updated:", paymentSource.id);
  },
  onExit: function () {},
  onError: function (err) {},
});
```

### charge

Opens the charge authorization drop-in. Customer reviews and authorizes a payment.

**Required option**: `charge` object
**Optional**: `authorizeOnly` (default `false`), `tip` object

```js
strongholdPay.charge(customerToken, {
  authorizeOnly: false,
  charge: {
    type: "bank_debit_cnp",
    amount: 4995, // $49.95 in cents
    currency: "usd",
    paymentSourceId: "payment_source_XpPw9ylVlqRcun-bFWp6dkY8",
    externalId: "e34685512545", // optional
  },
  tip: {
    // optional — associate a tip with the charge
    amount: 300, // $3.00 in cents
    beneficiaryName: "John",
    details: {
      // optional
      displayMessage: "Order made by John",
      drawerId: "",
    },
  },
  onSuccess: function (charge) {
    // Check charge status via API from backend
    console.log("Charge created:", charge.id);
  },
  onExit: function () {},
  onError: function (err) {},
});
```

**Note**: Authorization and capture are not guaranteed. Use the charge ID to check status via the REST API from the backend.

When `authorizeOnly: true`, the charge reaches `authorized` state. Use the Capture Charge API endpoint to capture it later.

### tip

Creates a standalone tip. Requires a prior charge — the tip must reference it.

**Required option**: `tip` object (with `chargeId`)
**Optional**: `authorizeOnly` (default `false`)

```js
strongholdPay.tip(customerToken, {
  authorizeOnly: false,
  tip: {
    amount: 300, // $3.00 in cents
    currency: "usd",
    paymentSourceId: "payment_source_XpPw9ylVlqRcun-bFWp6dkY8",
    chargeId: "charge_...", // required — original charge
    beneficiaryName: "John",
    details: {
      // optional
      displayMessage: "Tip for great service",
      drawerId: "",
    },
  },
  onSuccess: function (tip) {
    console.log("Tip created:", tip.id);
  },
  onExit: function () {},
  onError: function (err) {},
});
```

## Arguments Reference

| Argument                     | Type    | Description                                                         |
| ---------------------------- | ------- | ------------------------------------------------------------------- |
| `customerToken`              | string  | JWT from customer token API                                         |
| `options`                    | object  | Method-specific options + callbacks                                 |
| `paymentSourceId`            | string  | ID of a customer's payment source                                   |
| `authorizeOnly`              | boolean | `true` = authorize only (no capture); `false` = capture immediately |
| `charge`                     | object  | Charge details                                                      |
| `charge.type`                | string  | Charge type (e.g., `bank_debit_cnp`)                                |
| `charge.amount`              | number  | Amount in cents (e.g., `4995` = $49.95)                             |
| `charge.currency`            | string  | Currency code (e.g., `usd`)                                         |
| `charge.paymentSourceId`     | string  | Payment source for the charge                                       |
| `charge.externalId`          | string  | Optional external reference ID                                      |
| `tip`                        | object  | Tip details                                                         |
| `tip.amount`                 | number  | Amount in cents                                                     |
| `tip.currency`               | string  | Currency code                                                       |
| `tip.beneficiaryName`        | string  | Name of tip recipient                                               |
| `tip.chargeId`               | string  | Original charge ID (required for standalone tip)                    |
| `tip.paymentSourceId`        | string  | Payment source for the tip                                          |
| `tip.details`                | object  | Optional tip display details                                        |
| `tip.details.displayMessage` | string  | Custom message during tipping flow                                  |
| `tip.details.drawerId`       | string  | Drawer identifier                                                   |
