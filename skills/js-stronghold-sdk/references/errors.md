# Stronghold Pay Error Reference

## Table of Contents

- [Error Response Format](#error-response-format)
- [Error Object Fields](#error-object-fields)
- [Error Types](#error-types)
- [Error Code List](#error-code-list)
  - [API Errors](#api-errors)
  - [Auth Errors](#auth-errors)
  - [Invalid Request Errors](#invalid-request-errors)
  - [Object Errors](#object-errors)
  - [Validation Errors](#validation-errors)

## Error Response Format

HTTP 2xx = success, 4xx = client error, 5xx = server error. All 4xx errors include a programmatic error code.

```json
{
  "error": {
    "code": "invalid_charge_amount",
    "message": "Charge amount must be between $1.00 and $999.99, inclusive.",
    "type": "object_error"
  },
  "response_id": "resp_bdFOammVLNVRqxjawhgR-XjS",
  "time": "2021-01-13T03:52:00Z",
  "status_code": 400
}
```

## Error Object Fields

| Field       | Type              | Description                                                                |
| ----------- | ----------------- | -------------------------------------------------------------------------- |
| `type`      | string            | Broad error category. Safe for programmatic use                            |
| `code`      | string            | Specific error code. Safe for programmatic use                             |
| `message`   | string (optional) | Human-readable description. **Not safe** for programmatic use (may change) |
| `attribute` | string (optional) | Path to the attribute that caused the error (usually request body field)   |
| `reference` | string (optional) | Additional context about the error                                         |

## Error Types

| Type                    | Description                                   |
| ----------------------- | --------------------------------------------- |
| `api_error`             | Server-side or temporary problems             |
| `auth_error`            | Authentication/authorization failures         |
| `invalid_request_error` | Wrong endpoint usage or invalid parameters    |
| `object_error`          | Business logic errors related to object state |
| `validation_error`      | Field validation failures                     |

## Error Code List

### API Errors

| Code                      | Description                                  |
| ------------------------- | -------------------------------------------- |
| `server_error`            | Stronghold was unable to process the request |
| `merchant_software_error` | Problem accessing merchant software          |

### Auth Errors

| Code                     | Description                                               |
| ------------------------ | --------------------------------------------------------- |
| `invalid_api_key`        | API key is not valid (wrong value or scope)               |
| `live_not_approved`      | Secret key is valid but live environment not yet approved |
| `invalid_customer_token` | Customer token is wrong or expired                        |

### Invalid Request Errors

| Code           | Description                             |
| -------------- | --------------------------------------- |
| `not_found`    | Referenced object not found             |
| `invalid_id`   | Object ID is not valid                  |
| `sandbox_only` | Request is for sandbox environment only |

### Object Errors

| Code                               | Description                                | Action                     |
| ---------------------------------- | ------------------------------------------ | -------------------------- |
| `invalid_operation`                | Operation on the object is invalid         | Check object state         |
| `payment_source_already_exists`    | Duplicate payment source                   | Use existing source        |
| `payment_source_login_required`    | Customer must re-authenticate              | Call `updatePaymentSource` |
| `payment_source_unavailable`       | Source temporarily unavailable             | Retry later                |
| `payment_source_login_unavailable` | Auth to source currently unavailable       | Retry later                |
| `payment_source_inactive`          | Source has been deactivated                | Link a new source          |
| `payment_source_action_required`   | Customer action needed before access       | Prompt customer            |
| `insufficient_balance`             | Insufficient funds in payment source       | Notify customer            |
| `customer_blocked`                 | Customer is blocked                        | Contact support            |
| `pay_link_canceled`                | PayLink was canceled                       | Create a new PayLink       |
| `pay_link_expired`                 | PayLink has expired                        | Create a new PayLink       |
| `pay_link_already_used`            | PayLink already consumed                   | Create a new PayLink       |
| `pay_link_charge_amount_modified`  | Associated charge was modified             | Create a new PayLink       |
| `invalid_charge_amount`            | Amount outside permitted range ($1.00–max) | Fix amount                 |
| `invalid_tip_amount`               | Tip amount outside permitted range         | Fix amount                 |
| `charge_tip_already_created`       | Tip already exists for this charge         | —                          |
| `charge_blocked_exceeds_limit`     | Stronghold spending limit exceeded         | Contact support            |

### Validation Errors

| Code            | Description                              |
| --------------- | ---------------------------------------- |
| `missing_field` | Required field is missing                |
| `invalid_field` | Field value is invalid                   |
| `value_taken`   | Value already exists (unique constraint) |

## Handling `payment_source_login_required`

This is the most common error requiring user action. When received:

1. Detect `error.code === 'payment_source_login_required'` in your error handler
2. Extract the `paymentSourceId` from the error context or your records
3. Call `strongholdPay.updatePaymentSource(customerToken, { paymentSourceId: '...' })`
4. Retry the original charge after the customer successfully re-authenticates
