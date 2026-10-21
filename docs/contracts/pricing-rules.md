# SwiftDrop — Pricing Engine Rules & Fee Calculation Contract

> **Authority:** Resolves SRS §13.2 (Decision #2: Pricing formula & currency), §17.2 (Open Decision #2), and FR-PAR-002/003.  
> **Owning Context:** Pricing Context (`core-api/internal/pricing/`)  
> **Primary Owner:** Member 2 (Parcel Management & Pricing)

---

## 1. Currency & Financial Representation

In strict adherence to **SRS §7.2** and **Engineering Standards §5.2**:
- **Authoritative Currency:** **Sri Lankan Rupee (LKR)**.
- **Internal Storage:** All financial amounts are stored as **64-bit signed integers (`bigint`) representing minor currency units (cents)**.
  - Example: `LKR 450.00` is represented as integer `45000`.
  - Example: `LKR 1,250.50` is represented as integer `125050`.
- **Prohibition:** Floating-point data types (`float32`, `float64`) are **STRICTLY PROHIBITED** in database columns and application domain models for financial computations.
- **Transport Representation:** In JSON API payloads, fee objects provide both integer cents and formatted decimal strings:
  ```json
  "fee": {
    "currency": "LKR",
    "amount_cents": 45000,
    "formatted": "LKR 450.00"
  }
  ```

---

## 2. Billable Weight & Volumetric Calculation

To account for low-density packages consuming significant vehicle volume, SwiftDrop employs industry-standard volumetric weight pricing.

### 2.1 Volumetric Weight Formula
$$\text{Volumetric Weight (kg)} = \frac{\text{Length (cm)} \times \text{Width (cm)} \times \text{Height (cm)}}{5000}$$

- The volumetric divisor is standardized at **5000 $\text{cm}^3/\text{kg}$**.
- Dimensions are rounded to 2 decimal places before division.

### 2.2 Chargeable / Billable Weight
$$\text{Billable Weight} = \max(\text{Actual Weight (kg)}, \text{Volumetric Weight (kg)})$$

---

## 3. Dynamic Tariff Calculation Formula

The total delivery fee is determined by:
1. Origin and destination zone pair
2. Billable weight
3. Service level multiplier (Standard vs. Express vs. Same-Day)

### 3.1 Mathematical Fee Formula

$$\text{Base Charge} = \text{ZoneBaseFee}$$

$$\text{Weight Surcharge} = \max(0, \lceil \text{Billable Weight} - 1.0 \rceil) \times \text{PerKgRate}$$

$$\text{Subtotal} = \text{Base Charge} + \text{Weight Surcharge}$$

$$\text{Total Fee (Cents)} = \text{Subtotal} \times \text{ServiceLevelMultiplier}$$

### 3.2 Service Level Multipliers

| Service Level | Multiplier | SLA Delivery Window | Description |
| :--- | :--- | :--- | :--- |
| `STANDARD` | **1.0x** | 2–3 Business Days | Standard road delivery network |
| `EXPRESS` | **1.5x** | Next Business Day | Priority sortation and direct routing |
| `SAME_DAY` | **2.0x** | Within 6 Hours (booked before 12:00 PM) | Direct courier dispatch within intra-zone limits |

---

## 4. Default Tariff Schedule (Baseline Rule Set v1)

Zone classification baseline:
- `ZONE_COLOMBO_METRO`: Colombo 01–15 & immediate suburbs
- `ZONE_GREATER_WESTERN`: Gampaha, Kalutara, outer Western Province
- `ZONE_OUTSTATION`: Central, Southern, Northern, Eastern, and North-Western provinces

| Zone Pair | Zone Classification | Base Fee (First 1 kg) | Additional Rate (Per Additional kg) |
| :--- | :--- | :--- | :--- |
| Metro $\rightarrow$ Metro | Intra-Zone Metro | **LKR 350.00** (`35000` cents) | **LKR 80.00** (`8000` cents) |
| Metro $\rightarrow$ Greater Western | Inter-Zone Western | **LKR 450.00** (`45000` cents) | **LKR 100.00** (`10000` cents) |
| Metro $\rightarrow$ Outstation | Long Haul | **LKR 650.00** (`65000` cents) | **LKR 150.00** (`15000` cents) |
| Outstation $\rightarrow$ Outstation | Cross-Province | **LKR 750.00** (`75000` cents) | **LKR 180.00** (`18000` cents) |

---

## 5. Concrete Calculation Examples

### Example A: Heavy Intra-Zone Package
- **Origin / Dest:** Colombo 03 $\rightarrow$ Colombo 07 (`Metro -> Metro`)
- **Dimensions:** 20 cm $\times$ 15 cm $\times$ 10 cm ($3,000\text{ cm}^3 \div 5000 = 0.60\text{ kg}$ volumetric)
- **Actual Weight:** 3.4 kg
- **Billable Weight:** $\max(3.4, 0.6) = 3.4\text{ kg} \rightarrow 4\text{ kg}$ billable band (1 base + 3 additional)
- **Service Level:** `STANDARD` (1.0x)
- **Calculation:**
  - Base Fee (first 1 kg): `35000` cents (LKR 350.00)
  - Additional Weight: $3 \times 8000 = 24000$ cents (LKR 240.00)
  - Total Fee: $35000 + 24000 = \mathbf{59000}\text{ cents}$ (**LKR 590.00**)

### Example B: Bulky Express Outstation Package
- **Origin / Dest:** Colombo 03 $\rightarrow$ Kandy (`Metro -> Outstation`)
- **Dimensions:** 50 cm $\times$ 40 cm $\times$ 30 cm ($60,000\text{ cm}^3 \div 5000 = 12.0\text{ kg}$ volumetric)
- **Actual Weight:** 2.0 kg
- **Billable Weight:** $\max(2.0, 12.0) = 12.0\text{ kg}$ (1 base + 11 additional)
- **Service Level:** `EXPRESS` (1.5x)
- **Calculation:**
  - Base Fee: `65000` cents
  - Additional Weight: $11 \times 15000 = 165000$ cents
  - Subtotal: $65000 + 165000 = 230000$ cents
  - Multiplier: $230000 \times 1.5 = \mathbf{345000}\text{ cents}$ (**LKR 3,450.00**)

---

## 6. Quote Expiration & Booking Snapshot Contract

1. **Quote Generation (`POST /api/v1/pricing/quote`):**
   - Returns quote record valid for **15 minutes** (`expires_at = NOW() + 15m`).
   - Includes cryptographic or database-backed quote UUID and version of the tariff rule applied.
2. **Booking Immutability:**
   - When a booking is submitted via `POST /api/v1/parcels`, the client passes `quote_id`.
   - Core API verifies `expires_at > NOW()`.
   - The accepted price (`price_cents`) is copied permanently onto the `parcels` row.
   - Subsequent changes to pricing rules or tariff tables have **zero effect** on existing booked parcels.
