# Admin & Operations Module (`internal/admin`)

> **Domain:** Delivery Zones, System Configuration, Operational Reports & Backoffice Oversight  
> **Primary Owners:** Member 7 (Admin Frontend & Gateway), Member 1 (Security), Member 4 (Merchant Approval)  
> **Clean Architecture Boundary:** Modular Monolith Internal Module

---

## 1. Module Overview & Responsibilities
The `admin` module handles system-wide backoffice operations and configuration that transcend individual domain contexts.

### Key Responsibilities:
- Geographic delivery zone administration (postal code mappings, zone polygons, surcharge rules).
- System-wide operational reports (delivery success rates, SLA violations, courier utilization).
- Cross-domain audit trail inspection for compliance and incident investigations.
- Merchant account approval/rejection workflows in coordination with `internal/merchants`.

---

## 2. Domain Invariants & Business Rules
1. **Strict Role Guard:** All endpoints in this module REQUIRE the `ADMIN` role. No customer, merchant, or courier may access administrative queries or configurations.
2. **Immutable Audit Logs:** Audit logs and historical transition records can be queried but CANNOT be updated or deleted by any administrative user.
3. **Zone Uniqueness:** Postal codes may not be assigned to multiple conflicting operational delivery zones.

---

## 3. Layer Breakdown
```
internal/admin/
├── domain/                  # Zone entity, SystemConfig VO, AuditQueryFilter, domain errors
├── application/             # ManageZones, ApproveMerchant, GenerateOperationalReport use cases
├── infrastructure/          # PostgresAdminRepository, PostgresReportingQuery
└── transport/               # HTTP handlers (/api/v1/admin/*) and DTOs
```

---

## 4. Database Ownership
- `delivery_zones`: Zone ID, zone code, name, base surcharge, active status.
- `zone_postal_codes`: Postal code prefix to zone mapping.
- `system_settings`: Key-value configuration store for runtime operational toggles.

---

## 5. Changelog & Implementation Status
| Date | Change Summary | Author / Agent |
| :--- | :--- | :--- |
| 2026-10-21 | Initial module architecture baseline created | Architecture Setup |
