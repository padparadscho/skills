# Stronghold Pay REST API v2

## Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [Base URLs](#base-urls)
- [Response Format](#response-format)
- [API Endpoints](#api-endpoints)
  - [Customers](#customers)
  - [Customer Tokens](#customer-tokens)
  - [Payment Sources](#payment-sources)
  - [Charges](#charges)
  - [Tips](#tips)
  - [PayLinks](#paylinks)
- [Common Patterns](#common-patterns)

## Overview

The Stronghold Pay API v2 provides server-side control over payment operations. Use alongside the JS SDK or standalone for custom UX.

Full interactive API reference: https://docs.strongholdpay.com/docs/stronghold-pay/YXBpOjE5ODIyMjc1-api-reference-v2

## Authentication

All API requests require the `SH-SECRET-KEY` header with your secret API key.

```
SH-SECRET-KEY: sk_sandbox_sEGTb5Q9B8Pz-I5ZZ9dTKOko
```

- **Sandbox key**: `sk_sandbox_...` — for development/testing
- **Live key**: `sk_live_...` — for production (requires approval)

**Never expose secret keys in frontend code.** Only publishable keys (`pk_...`) belong on the client.

## Base URLs

| Environment    | Base URL                                                           |
| -------------- | ------------------------------------------------------------------ |
| Live / Sandbox | `https://api.strongholdpay.com`                                    |
| Mock Server    | `https://stoplight.io/mocks/strongholdpay/stronghold-pay/19822275` |

All endpoints are prefixed with `/v2/`.

## Response Format

All responses follow this structure:

```json
{
  "response_id": "resp_GMdYRc2lkKAgNv9S5qM39m1e",
  "time": "2024-01-15T12:00:00Z",
  "status_code": 200,
  "result": { ... }
}
```

Error responses:

```json
{
  "error": {
    "code": "invalid_charge_amount",
    "message": "Charge amount must be between $1.00 and $999.99, inclusive.",
    "type": "object_error"
  },
  "response_id": "resp_bdFOammVLNVRqxjawhgR-XjS",
  "time": "2024-01-15T12:00:00Z",
  "status_code": 400
}
```

## API Endpoints

### Customers

Manage customer records. Each customer can have multiple payment sources.

| Method   | Endpoint                      | Description       |
| -------- | ----------------------------- | ----------------- |
| `GET`    | `/v2/customers`               | List customers    |
| `POST`   | `/v2/customers`               | Create a customer |
| `GET`    | `/v2/customers/{customer_id}` | Get a customer    |
| `PUT`    | `/v2/customers/{customer_id}` | Update a customer |
| `DELETE` | `/v2/customers/{customer_id}` | Delete a customer |

**Create a customer**:

```bash
curl -X POST https://api.strongholdpay.com/v2/customers \
  -H 'SH-SECRET-KEY: sk_sandbox_...' \
  -H 'Content-Type: application/json' \
  -d '{
    "external_id": "user_123",
    "email": "customer@example.com",
    "name": "Jane Doe"
  }'
```

### Customer Tokens

Generate tokens for frontend SDK usage.

| Method | Endpoint                            | Description           |
| ------ | ----------------------------------- | --------------------- |
| `GET`  | `/v2/customers/{customer_id}/token` | Create customer token |

**Query parameter**: `is_external_id` (boolean) — set `true` if using external ID instead of Stronghold customer ID.

```bash
curl -X GET https://api.strongholdpay.com/v2/customers/{customer_id}/token \
  -H 'SH-SECRET-KEY: sk_sandbox_...' \
  -H 'Accept: application/json'
```

Response:

```json
{
  "result": {
    "token": "<jwt>",
    "expiry": "2024-01-16T00:00:00Z"
  }
}
```

### Payment Sources

Manage linked bank accounts for customers.

| Method   | Endpoint                                                          | Description             |
| -------- | ----------------------------------------------------------------- | ----------------------- |
| `GET`    | `/v2/customers/{customer_id}/payment-sources`                     | List payment sources    |
| `GET`    | `/v2/customers/{customer_id}/payment-sources/{payment_source_id}` | Get a payment source    |
| `DELETE` | `/v2/customers/{customer_id}/payment-sources/{payment_source_id}` | Remove a payment source |

Payment sources are typically created via the JS SDK `addPaymentSource` method or the `bank_link` PayLink, which handle the bank aggregator flow (Plaid/Yodlee).

### Charges

Create and manage charges (payments).

| Method | Endpoint                          | Description                  |
| ------ | --------------------------------- | ---------------------------- |
| `GET`  | `/v2/charges`                     | List charges                 |
| `POST` | `/v2/charges`                     | Create a charge              |
| `GET`  | `/v2/charges/{charge_id}`         | Get a charge                 |
| `POST` | `/v2/charges/{charge_id}/capture` | Capture an authorized charge |
| `POST` | `/v2/charges/{charge_id}/void`    | Void a charge                |
| `POST` | `/v2/charges/{charge_id}/refund`  | Refund a charge              |

**Create a charge (server-side)**:

```bash
curl -X POST https://api.strongholdpay.com/v2/charges \
  -H 'SH-SECRET-KEY: sk_sandbox_...' \
  -H 'Content-Type: application/json' \
  -d '{
    "customer_id": "customer_...",
    "payment_source_id": "payment_source_...",
    "amount": 4995,
    "currency": "usd",
    "type": "bank_debit_cnp",
    "authorize_only": false,
    "external_id": "order_456"
  }'
```

**Capture an authorized charge**:

```bash
curl -X POST https://api.strongholdpay.com/v2/charges/{charge_id}/capture \
  -H 'SH-SECRET-KEY: sk_sandbox_...'
```

### Tips

Create and manage tips associated with charges.

| Method | Endpoint                    | Description               |
| ------ | --------------------------- | ------------------------- |
| `GET`  | `/v2/tips`                  | List tips                 |
| `POST` | `/v2/tips`                  | Create a tip              |
| `GET`  | `/v2/tips/{tip_id}`         | Get a tip                 |
| `POST` | `/v2/tips/{tip_id}/capture` | Capture an authorized tip |
| `POST` | `/v2/tips/{tip_id}/void`    | Void a tip                |

### PayLinks

Create hosted payment page links. See [paylink.md](paylink.md) for full details.

| Method   | Endpoint              | Description      |
| -------- | --------------------- | ---------------- |
| `GET`    | `/v2/links`           | List PayLinks    |
| `POST`   | `/v2/links`           | Create a PayLink |
| `GET`    | `/v2/links/{link_id}` | Get a PayLink    |
| `DELETE` | `/v2/links/{link_id}` | Cancel a PayLink |

## Common Patterns

### Full Payment Flow (Backend + Frontend)

1. **Create customer** (backend): `POST /v2/customers`
2. **Generate token** (backend): `GET /v2/customers/{id}/token`
3. **Link payment source** (frontend): `strongholdPay.addPaymentSource(token, ...)`
4. **Create charge** (frontend or backend):
   - Frontend: `strongholdPay.charge(token, ...)` — shows authorization UI
   - Backend: `POST /v2/charges` — no UI (requires prior authorization)
5. **Check charge status** (backend): `GET /v2/charges/{charge_id}`
6. **Capture if authorize-only** (backend): `POST /v2/charges/{charge_id}/capture`

### Authorize-then-Capture Pattern

Use `authorize_only: true` when the final amount may change (e.g., adding tips):

1. Create charge with `authorize_only: true`
2. Optionally create a tip referencing the charge
3. Capture the charge: `POST /v2/charges/{charge_id}/capture`

### Re-authentication Flow

When a charge fails with `payment_source_login_required`:

1. Detect the error code in your backend
2. Pass the `paymentSourceId` to the frontend
3. Call `strongholdPay.updatePaymentSource(token, { paymentSourceId: '...' })`
4. Retry the charge after successful update
