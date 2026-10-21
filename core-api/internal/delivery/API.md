# Delivery Records API Specifications (`/api/v1/deliveries`)

> **Module:** `internal/delivery` | **Owner:** Member 3 | **Status:** Baseline

---

## 1. Complete Delivery with Proof
Records a successful parcel delivery along with mandatory proof of delivery (photo or signature).

- **Method / Path:** `POST /api/v1/deliveries/{parcel_id}/complete`
- **Auth Required:** Bearer JWT (`COURIER`, `ADMIN`)
- **Headers:** 
  - `Authorization: Bearer <jwt>`
  - `Content-Type: application/json`

### Request Body
```json
{
  "proof_type": "PHOTO",
  "proof_url": "https://storage.swiftdrop.io/proofs/prf_photo_99214.jpg",
  "received_by_name": "Jane Smith",
  "notes": "Left on front porch behind pillar"
}
```

### Success Response (`200 OK`)
```json
{
  "data": {
    "delivery_id": "del_01HK9988776655",
    "parcel_id": "prcl_01HXYZ1234567890",
    "status": "DELIVERED",
    "delivered_at": "2026-10-21T11:15:00Z",
    "proof_verified": true
  }
}
```

### Error Responses
- `400 Bad Request`: `MISSING_DELIVERY_PROOF` — Proof URL or signature data missing.
- `403 Forbidden`: `NOT_ASSIGNED_COURIER` — Caller is not the assigned courier.
- `409 Conflict`: `INVALID_STATE_FOR_DELIVERY` — Parcel is not in `OUT_FOR_DELIVERY` status.

---

## 2. Record Failed Delivery Attempt
Logs an unsuccessful delivery attempt and sets the parcel to `ATTEMPT_FAILED`.

- **Method / Path:** `POST /api/v1/deliveries/{parcel_id}/failed`
- **Auth Required:** Bearer JWT (`COURIER`, `ADMIN`)
- **Headers:** `Authorization: Bearer <jwt>`

### Request Body
```json
{
  "reason_code": "RECIPIENT_UNAVAILABLE",
  "notes": "No answer at door, left delivery notice slip"
}
```

### Success Response (`200 OK`)
```json
{
  "data": {
    "attempt_number": 1,
    "max_attempts": 3,
    "status": "ATTEMPT_FAILED",
    "next_action": "REATTEMPT_SCHEDULED",
    "recorded_at": "2026-10-21T11:20:00Z"
  }
}
```
