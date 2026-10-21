# Merchant Module API Specifications (`/api/v1/merchants`)

> **Module:** `internal/merchants` | **Owner:** Member 4 | **Status:** Baseline

---

## 1. Register Webhook Subscription
Registers an HTTPS destination URL to receive parcel status change webhooks.

- **Method / Path:** `POST /api/v1/merchants/webhooks`
- **Auth Required:** Bearer JWT or API Key (`MERCHANT`, `ADMIN`)
- **Headers:** `Authorization: Bearer <jwt>`

### Request Body
```json
{
  "target_url": "https://api.merchantstore.com/webhooks/swiftdrop",
  "events": ["parcel.status_changed.v1", "parcel.delivered.v1"],
  "secret": "whsec_998877665544332211"
}
```

### Success Response (`201 Created`)
```json
{
  "data": {
    "subscription_id": "wh_sub_01HXYZ776655",
    "target_url": "https://api.merchantstore.com/webhooks/swiftdrop",
    "events": ["parcel.status_changed.v1", "parcel.delivered.v1"],
    "is_active": true,
    "created_at": "2026-10-21T10:00:00Z"
  }
}
```

### Error Responses
- `400 Bad Request`: `SSRF_PROHIBITED_URL` — Target URL resolves to an internal/private IP address.
- `422 Unprocessable`: Unsupported event type specified.

---

## 2. Generate API Key
Generates a new programmatic API key for the merchant.

- **Method / Path:** `POST /api/v1/merchants/api-keys`
- **Auth Required:** Bearer JWT (`MERCHANT` admin user only)
- **Headers:** `Authorization: Bearer <jwt>`

### Request Body
```json
{
  "label": "Production Server Key"
}
```

### Success Response (`201 Created`)
```json
{
  "data": {
    "key_id": "key_01HM998877",
    "label": "Production Server Key",
    "api_key": "sd_live_a1b2c3d4e5f67890123456789abcdef",
    "note": "Save this key now. It will not be shown again."
  }
}
```
