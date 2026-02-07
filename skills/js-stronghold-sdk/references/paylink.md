# PayLink Reference

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Creating PayLinks](#creating-paylinks)
- [PayLink Types](#paylink-types)
  - [checkout](#checkout)
  - [bank_link](#bank_link)
  - [tipping](#tipping)
- [Callbacks](#callbacks)

## Overview

PayLinks are hosted payment pages that handle the entire payment UI. No frontend SDK integration required — redirect customers to a Stronghold-hosted URL. Ideal when you don't want to build custom payment UI.

API endpoint: `POST /v2/links`
Auth: `SH-SECRET-KEY` header

## How It Works

1. Create a PayLink via the REST API for a specific customer
2. Receive a URL in the response (e.g., `https://strongholdpay.com/l/WGge7VDl`)
3. Redirect the customer to that URL
4. Customer completes the action on the hosted page
5. Customer is redirected to your `success_url` or `exit_url`

PayLinks are customer-specific and expire (typically 24 hours after creation).

## Creating PayLinks

```bash
curl -X POST https://api.strongholdpay.com/v2/links \
  -H 'SH-SECRET-KEY: sk_sandbox_...' \
  -H 'Content-Type: application/json' \
  -d '{ ... }'
```

All PayLinks share this base structure:

```json
{
  "type": "checkout|bank_link|tipping",
  "customer_id": "customer_...",
  "callbacks": {
    "success_url": "https://yoursite.com/success/",
    "exit_url": "https://yoursite.com/cancel/"
  }
}
```

Response includes:

```json
{
  "result": {
    "id": "O5DzVkud",
    "type": "checkout",
    "url": "https://strongholdpay.com/l/O5DzVkud",
    "status": "created",
    "expires_at": "2024-01-16T12:00:00Z",
    "has_expired": false,
    "environment": "sandbox",
    "customer": { "id": "customer_..." }
  }
}
```

## PayLink Types

### checkout

Full checkout flow with order details, itemized cart, optional tipping.

```json
{
  "type": "checkout",
  "customer_id": "customer_-dr2n7sN5hGAuLlA2dtuNhR8",
  "order": {
    "total_amount": 5900,
    "tax_amount": 400,
    "items": [
      {
        "name": "Item 1",
        "description": "Description 1",
        "quantity": 2,
        "total_amount": 4000,
        "image_url": "https://example.com/item1.png"
      },
      {
        "name": "Item 2",
        "description": "Description 2",
        "quantity": 3,
        "total_amount": 1500,
        "image_url": "https://example.com/item2.png"
      }
    ]
  },
  "tip": {
    "beneficiary_name": "Joe"
  },
  "callbacks": {
    "success_url": "https://yoursite.com/success/",
    "exit_url": "https://yoursite.com/cancel/"
  }
}
```

Response adds computed fields:

```json
{
  "order": {
    "total_amount": 5900,
    "tax_amount": 400,
    "sub_amount": 5500,
    "convenience_fee": 0,
    "items": [
      { "name": "Item 1", "quantity": 2, "total_amount": 4000, "price": 2000 },
      { "name": "Item 2", "quantity": 3, "total_amount": 1500, "price": 500 }
    ]
  }
}
```

### bank_link

Allows a customer to link a new bank account (payment source) without any charge.

```json
{
  "type": "bank_link",
  "customer_id": "customer_h.KlK0X8xtXQ1xbmfyJyyahQ",
  "callbacks": {
    "success_url": "https://yoursite.com/success/",
    "exit_url": "https://yoursite.com/cancel/"
  }
}
```

### tipping

Allows a customer to add a tip on top of a previously created charge.

```json
{
  "type": "tipping",
  "customer_id": "customer_h.KlK0X8xtXQ1xbmfyJyyahQ",
  "charge_id": "charge_JbN2.-uFKjLexI0UYeNY6mWX",
  "tip": {
    "beneficiary_name": "Joe"
  },
  "callbacks": {
    "success_url": "https://yoursite.com/success/",
    "exit_url": "https://yoursite.com/cancel/"
  }
}
```

## Callbacks

Every PayLink requires a `callbacks` object:

| Field         | Description                                             |
| ------------- | ------------------------------------------------------- |
| `success_url` | URL to redirect customer to after successful completion |
| `exit_url`    | URL to redirect customer to if they exit/cancel         |

Sandbox test credentials for bank linking within PayLinks:

| Aggregator | Username              | Password      |
| ---------- | --------------------- | ------------- |
| Plaid      | `user_good`           | `pass_good`   |
| Yodlee     | `YodTest.site16441.2` | `site16441.2` |
