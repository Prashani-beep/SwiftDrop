# Admin Module API Specifications (`/api/v1/admin`)

> **Module:** `internal/admin` | **Owners:** Members 7, 1 & 4 | **Status:** Baseline

---

## 1. Manage Delivery Zones
Creates or updates an operational delivery zone.

- **Method / Path:** `POST /api/v1/admin/zones`
- **Auth Required:** Bearer JWT (`ADMIN` only)
- **Headers:** `Authorization: Bearer <jwt>`

### Request Body
```json
{
  "zone_code": "ZONE_METRO_EAST",
  "name": "Metro East Delivery Zone",
  "base_surcharge": "3.50",
  "postal_code_prefixes": ["62701", "62702", "62703"]
}
```

### Success Response (`201 Created`)
```json
{
  "data": {
    "zone_id": "zn_01HA09876",
    "zone_code": "ZONE_METRO_EAST",
    "name": "Metro East Delivery Zone",
    "base_surcharge": "3.50",
    "active": true
  }
}
```

---

## 2. Generate Operational Report
Retrieves delivery volume, success rate, and SLA compliance metrics.

- **Method / Path:** `GET /api/v1/admin/reports/operational`
- **Auth Required:** Bearer JWT (`ADMIN` only)
- **Query Params:** `?from=2026-10-01&to=2026-10-21&zone=ZONE_METRO_EAST`

### Success Response (`200 OK`)
```json
{
  "data": {
    "period": { "from": "2026-10-01", "to": "2026-10-21" },
    "total_parcels": 1420,
    "delivered_successfully": 1380,
    "failed_attempts": 40,
    "first_attempt_delivery_rate": "97.2%",
    "average_delivery_time_hours": 3.8
  }
}
```
