# Parcel Management API Specifications (`/api/v1/parcels`)

> **Module:** `internal/parcels` | **Owner:** Member 2 | **Status:** Baseline

---

## 1. Book Parcel
Creates a new parcel booking for the authenticated sender (Customer or Merchant).

- **Method / Path:** `POST /api/v1/parcels`
- **Auth Required:** Bearer JWT (`CUSTOMER`, `MERCHANT`, `ADMIN`)
- **Headers:** 
  - `Authorization: Bearer <jwt>`
  - `Content-Type: application/json`
  - `Idempotency-Key: <uuid-v4>` (Mandatory)

### Request Body
```json
{
  "recipient_name": "Jane Smith",
  "recipient_phone": "+15559876543",
  "recipient_address": {
    "street": "742 Evergreen Terrace",
    "unit": "Apt 2B",
    "city": "Springfield",
    "state": "IL",
    "postal_code": "62704",
    "country": "USA"
  },
  "dimensions": {
    "weight_kg": 2.5,
    "length_cm": 30.0,
    "width_cm": 20.0,
    "height_cm": 15.0
  },
  "pickup_notes": "Ring doorbell twice"
}
```

### Success Response (`201 Created`)
```json
{
  "data": {
    "id": "prcl_01HXYZ1234567890",
    "tracking_number": "SD-20261021-04921",
    "status": "CREATED",
    "sender_id": "usr_01HABC1234567890",
    "recipient_name": "Jane Smith",
    "fee": {
      "currency": "USD",
      "amount": "14.50"
    },
    "created_at": "2026-10-21T09:00:00Z"
  }
}
```

### Error Responses
- `400 Bad Request`: `PARCEL_INVALID_DIMENSIONS` — Negative weight or dimensions exceed max limits.
- `401 Unauthorized`: Missing or invalid Bearer token.
- `409 Conflict`: `IDEMPOTENCY_KEY_PAYLOAD_MISMATCH` — Reusing idempotency key with different body.

---

## 2. Get Parcel Details
Retrieves authoritative parcel details and history for authorized parties.

- **Method / Path:** `GET /api/v1/parcels/{id}`
- **Auth Required:** Bearer JWT (`CUSTOMER` [own parcel], `COURIER` [assigned], `DISPATCHER`, `ADMIN`)
- **Headers:** `Authorization: Bearer <jwt>`

### Success Response (`200 OK`)
```json
{
  "data": {
    "id": "prcl_01HXYZ1234567890",
    "tracking_number": "SD-20261021-04921",
    "status": "OUT_FOR_DELIVERY",
    "sender_id": "usr_01HABC1234567890",
    "courier_id": "usr_courier_99",
    "recipient": {
      "name": "Jane Smith",
      "phone": "+15559876543",
      "address": "742 Evergreen Terrace, Springfield, IL"
    },
    "history": [
      {
        "status": "CREATED",
        "timestamp": "2026-10-21T09:00:00Z",
        "actor_role": "CUSTOMER"
      },
      {
        "status": "ASSIGNED",
        "timestamp": "2026-10-21T09:30:00Z",
        "actor_role": "DISPATCHER"
      },
      {
        "status": "OUT_FOR_DELIVERY",
        "timestamp": "2026-10-21T11:00:00Z",
        "actor_role": "COURIER"
      }
    ]
  }
}
```

### Error Responses
- `403 Forbidden`: `ACCESS_DENIED` — Requesting user is neither sender, recipient, assigned courier, nor dispatcher.
- `404 Not Found`: `PARCEL_NOT_FOUND` — Parcel ID does not exist.
