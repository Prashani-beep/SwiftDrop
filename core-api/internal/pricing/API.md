# Pricing Module API Specifications (`/api/v1/pricing`)

> **Module:** `internal/pricing` | **Owner:** Member 2 | **Status:** Baseline

---

## 1. Estimate Delivery Fee
Calculates the estimated shipping fee based on weight, dimensions, and zones before booking.

- **Method / Path:** `POST /api/v1/pricing/estimate`
- **Auth Required:** Optional (Public with rate limit, or Bearer JWT)
- **Headers:** `Content-Type: application/json`

### Request Body
```json
{
  "pickup_postal_code": "62701",
  "delivery_postal_code": "62704",
  "weight_kg": 2.5,
  "length_cm": 30.0,
  "width_cm": 20.0,
  "height_cm": 15.0,
  "service_tier": "STANDARD"
}
```

### Success Response (`200 OK`)
```json
{
  "data": {
    "currency": "USD",
    "base_fee": "8.00",
    "weight_fee": "4.50",
    "distance_fee": "2.00",
    "total_fee": "14.50",
    "billable_weight_kg": 2.5,
    "volumetric_weight_kg": 1.8,
    "breakdown": {
      "pricing_tier": "DOMESTIC_STANDARD",
      "zone_code": "ZONE_1"
    }
  }
}
```

### Error Responses
- `400 Bad Request`: `PRICING_INVALID_INPUT` — Non-positive dimension or invalid postal code.
