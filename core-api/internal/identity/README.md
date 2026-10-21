# Identity, Access & Security Module (`internal/identity`)

> **Domain:** Authentication, Authorization, User Accounts & RBAC  
> **Primary Owner:** Member 1 — Identity, Access & Security  
> **Clean Architecture Boundary:** Modular Monolith Internal Module

---

## 1. Module Overview & Responsibilities
The `identity` module serves as the authoritative security boundary for SwiftDrop. It manages user credentials, authentication lifecycles (email/password and Google OAuth 2.0 / OIDC), JWT token issuance/validation, and role-based access control (RBAC).

### Key Responsibilities:
- User registration, local password hashing (Argon2id/bcrypt), and login authentication.
- Google OAuth 2.0 / OpenID Connect callback verification (issuer, audience, nonce, signature).
- Stateless JWT issuance with standard claims (`sub`, `email`, `role`, `tenant_id`, `exp`).
- Reusable authentication middleware (`AuthMiddleware`) and authorization guards (`RequireRole`, `RequireTenant`).
- Tenant isolation helpers ensuring customers access only their shipments and merchants access only their tenant data.

---

## 2. Domain Invariants & Business Rules
1. **Password Safety:** Passwords must be hashed using Argon2id or bcrypt before storage; plaintext passwords must NEVER be persisted or logged.
2. **Role Immutability:** Clients cannot assign their own roles. Only administrators may assign or promote users to `COURIER`, `DISPATCHER`, or `ADMIN`.
3. **Tenant Scoping:** Users with role `MERCHANT` must possess a valid, non-null `tenant_id`.
4. **Token Expiry:** Access tokens must expire within 15 minutes. Refresh tokens must be revocable.
5. **Masked Failures:** Authentication failures must return generic messages (`INVALID_CREDENTIALS`) to prevent account enumeration.

---

## 3. Layer Breakdown
```
internal/identity/
├── domain/                  # User entity, Role value object, Password hash VO, domain errors
├── application/             # RegisterUser, LoginUser, VerifyGoogleToken, RefreshToken use cases
├── infrastructure/          # PostgresUserRepository, BcryptHasher, JWTService
└── transport/               # HTTP handlers (/api/v1/auth/*, /api/v1/users/*) and DTO mappers
```

---

## 4. Database Ownership
- `users`: User identity, email, password hash, role (`CUSTOMER`, `MERCHANT`, `COURIER`, `DISPATCHER`, `ADMIN`), active status.
- `user_profiles`: Name, contact phone, default delivery address.
- `refresh_tokens`: Hashed token family, user ID, expiry, revoked status.

---

## 5. Changelog & Implementation Status
| Date | Change Summary | Author / Agent |
| :--- | :--- | :--- |
| 2026-10-21 | Initial module architecture baseline created | Architecture Setup |
