# AI Assistant Service (`services/ai-assistant`)

> **Service:** Python / FastAPI AI Assistant & Model Context Protocol (MCP) Server  
> **Primary Owner:** Member 10 — AI Assistant/MCP & Cross-Workflow Quality  
> **Deployment Mode:** Independent Python/FastAPI Container

---

## 1. Service Overview & Responsibilities
The `ai-assistant` service provides natural language interaction for parcel tracking, delivery fee calculation, and customer support queries using an LLM.

### Key Responsibilities:
- FastAPI conversational endpoints and tool registry.
- Controlled Model Context Protocol (MCP) tool execution.
- Strictly calling approved Core API and Tracking API REST endpoints.
- Validating all tool arguments server-side with Pydantic schemas.
- Enforcing user/tenant authorization context on all tool executions.

---

## 2. Invariants & Rules
1. **Zero Database Access:** The AI service must NEVER connect directly to PostgreSQL or execute database queries. It interacts strictly via approved HTTP APIs.
2. **Read-Only Baseline:** Core assistant tools (`track_parcel`, `estimate_shipping_fee`) are strictly read-only. The assistant cannot book parcels or modify status without explicit user confirmation.
3. **No Hallucinated Facts:** If an API returns `404 Not Found`, the assistant must state that the shipment was not found; it must never invent parcel statuses or fake delivery dates.

---

## 3. Changelog & Implementation Status
| Date | Change Summary | Author / Agent |
| :--- | :--- | :--- |
| 2026-10-21 | Initial service architecture baseline created | Architecture Setup |
