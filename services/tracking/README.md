# Tracking API Service (`services/tracking`)

> **Service:** Read-Optimized Tracking API & Real-Time SSE Streams  
> **Primary Owner:** Member 5 — Tracking API & Real-Time Updates  
> **Deployment Mode:** Independent Deployable Go Service

---

## 1. Service Overview & Responsibilities
The `tracking` service is a standalone read-optimized projection service designed to serve heavy public tracking traffic and real-time live update streams without imposing read locks on the transactional Core API.

### Key Responsibilities:
- Public tracking lookup by tracking number (`SD-YYYYMMDD-XXXXX`).
- Private authenticated shipment tracking for customers and merchants.
- Server-Sent Events (SSE) streaming (`GET /api/v1/tracking/{number}/stream`).
- Public data masking and recipient privacy enforcement.
- Optional Redis Pub/Sub integration for horizontal multi-replica live stream fan-out.

---

## 2. Invariants & Rules
1. **Read-Only Model:** The tracking service NEVER mutates parcel lifecycle state. It is strictly a projection reader.
2. **Strict Privacy Masking:** Public tracking queries MUST NEVER return recipient full street addresses, phone numbers, sender full details, or proof-of-delivery photos.
3. **SSE Connection Health:** The SSE stream must emit periodic heartbeat pings (`:keepalive\n\n`) every 15 seconds to prevent proxy disconnections.

---

## 3. Changelog & Implementation Status
| Date | Change Summary | Author / Agent |
| :--- | :--- | :--- |
| 2026-10-21 | Initial service architecture baseline created | Architecture Setup |
