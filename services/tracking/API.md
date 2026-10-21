# Tracking API Specifications (`/api/v1/tracking`)

> **Service:** `services/tracking` | **Owner:** Member 5 | **Status:** Baseline

---

## 1. Public Tracking Lookup
Returns sanitized milestone history for any valid tracking number.

- **Method / Path:** `GET /api/v1/tracking/public/{tracking_number}`
- **Auth Required:** None (Public)
- **Headers:** `Accept: application/json`

### Success Response (`200 OK`)
```json
{
  "data": {
    "tracking_number": "SD-20261021-04921",
    "status": "OUT_FOR_DELIVERY",
    "status_description": "Your parcel is out for delivery with our courier.",
    "destination": {
      "city": "Springfield",
      "state": "IL"
    },
    "estimated_delivery": "2026-10-21T14:00:00Z",
    "milestones": [
      {
        "status": "CREATED",
        "description": "Shipment booked and label generated",
        "timestamp": "2026-10-21T09:00:00Z"
      },
      {
        "status": "IN_TRANSIT",
        "description": "Arrived at Regional Distribution Hub",
        "timestamp": "2026-10-21T10:15:00Z"
      },
      {
        "status": "OUT_FOR_DELIVERY",
        "description": "Out for final delivery",
        "timestamp": "2026-10-21T11:00:00Z"
      }
    ]
  }
}
```

### Error Responses
- `404 Not Found`: `TRACKING_NOT_FOUND` — Invalid tracking number.
- `429 Too Many Requests`: Rate limit exceeded for public IP.

---

## 2. Real-Time Tracking Stream (Server-Sent Events)
Streams real-time status changes and location milestones to client browsers.

- **Method / Path:** `GET /api/v1/tracking/{tracking_number}/stream`
- **Auth Required:** Optional (Sanitized for public, detailed if Bearer JWT provided)
- **Headers:** `Accept: text/event-stream`

### Stream Payload Event (`event: status_update`)
```
event: status_update
id: evt_01HXYZ9988776655
data: {"tracking_number":"SD-20261021-04921","new_status":"DELIVERED","timestamp":"2026-10-21T11:15:00Z"}
```
