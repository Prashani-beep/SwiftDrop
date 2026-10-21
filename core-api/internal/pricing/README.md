# Pricing Module (`internal/pricing`)

> **Domain:** Dynamic Fee Estimation, Tariffs & Volumetric Weight  
> **Primary Owner:** Member 2 — Parcel Management & Pricing  
> **Clean Architecture Boundary:** Modular Monolith Internal Module

---

## 1. Module Overview & Responsibilities
The `pricing` module calculates delivery fees based on parcel weight, dimensions (volumetric weight), distance/zones, and delivery priority.

### Key Responsibilities:
- Tariff lookup and rule evaluation.
- Billable weight calculation: $\max(\text{actual\_weight}, \text{volumetric\_weight})$.
- Base fee + distance surcharge + weight tier fee estimation.
- In-process domain service interface consumed by `internal/parcels`.
- Public and authenticated fee estimation endpoints.

---

## 2. Domain Invariants & Business Rules
1. **Server Authority:** The frontend must NEVER calculate authoritative delivery fees. All fees must be calculated and validated by the backend.
2. **Exact Decimal Monetary Types:** Money values must be represented using exact integer minor units (e.g., cents in USD) or arbitrary-precision decimals; binary floating-point types (`float32`/`float64`) are strictly prohibited.
3. **Deterministic Calculations:** Given identical inputs (dimensions, weight, pickup/delivery zone, speed), the fee estimation must produce identical results.
4. **Volumetric Divisor:** Standard domestic divisor is 5000: $\text{volumetric\_kg} = \frac{L \times W \times H \text{ (cm)}}{5000}$.

---

## 3. Layer Breakdown
```
internal/pricing/
├── domain/                  # Money VO, Tariff entity, FeeCalculationPolicy, domain errors
├── application/             # CalculateFee use case & PricingCalculator interface
├── infrastructure/          # PostgresTariffRepository
└── transport/               # HTTP handler (/api/v1/pricing/estimate) and DTOs
```

---

## 4. Changelog & Implementation Status
| Date | Change Summary | Author / Agent |
| :--- | :--- | :--- |
| 2026-10-21 | Initial module architecture baseline created | Architecture Setup |
