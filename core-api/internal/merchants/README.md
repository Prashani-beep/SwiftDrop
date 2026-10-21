# Merchant Integration Module (`internal/merchants`)

> **Domain:** Merchant Profiles, B2B Tenant Management, API Keys & Webhook Configs  
> **Primary Owner:** Member 4 — Merchant Integration  
> **Clean Architecture Boundary:** Modular Monolith Internal Module

---

## 1. Module Overview & Responsibilities
The `merchants` module provides B2B capabilities for external merchant partners: managing merchant tenant accounts, generating secured API keys for programmatic parcel creation, and registering merchant webhook subscriptions.

### Key Responsibilities:
- Merchant registration, onboarding, and business profile management.
- API Key generation (hashed storage, prefix for identification: `sd_live_...`).
- Webhook subscription configuration (destination URL, event filters, shared secret).
- Destination URL security validation (blocking internal IPs, loopback, non-HTTPS).
- Coordinating with Member 8 (Messaging) who executes actual webhook delivery.

---

## 2. Domain Invariants & Business Rules
1. **SSRF Protection:** Webhook URLs must be validated server-side. Private/internal IPv4 ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`) and non-HTTP(S) schemes are strictly prohibited.
2. **Hashed API Secrets:** Merchant API keys must be hashed (SHA-256) upon generation. The plaintext key is shown to the merchant ONCE and never stored.
3. **Tenant Scoping:** All merchant operations must be strictly scoped to the authenticated `tenant_id`. No merchant may inspect or update another merchant's profile or webhooks.

---

## 3. Layer Breakdown
```
internal/merchants/
├── domain/                  # Merchant aggregate, ApiKey VO, WebhookSubscription entity, SSRFValidator
├── application/             # OnboardMerchant, CreateApiKey, RegisterWebhook use cases
├── infrastructure/          # PostgresMerchantRepository
└── transport/               # HTTP handlers (/api/v1/merchants/*) and DTOs
```

---

## 4. Database Ownership
- `merchants`: Tenant ID, company name, business registration number, approval status (`PENDING`, `APPROVED`, `SUSPENDED`).
- `merchant_api_keys`: Key ID, tenant ID, key prefix, key hash, created at, revoked at.
- `webhook_subscriptions`: Subscription ID, tenant ID, target URL, secret hash, subscribed events, is_active.

---

## 5. Changelog & Implementation Status
| Date | Change Summary | Author / Agent |
| :--- | :--- | :--- |
| 2026-10-21 | Initial module architecture baseline created | Architecture Setup |
