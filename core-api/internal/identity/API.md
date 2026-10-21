# Identity Module API Specifications (`/api/v1/auth`, `/api/v1/users`)

> **Module:** `internal/identity` | **Owner:** Member 1 | **Status:** Baseline

---

## 1. Register Customer Account
Creates a new customer account with email and password.

- **Method / Path:** `POST /api/v1/auth/register`
- **Auth Required:** None (Public)
- **Headers:** `Content-Type: application/json`

### Request Body
```json
{
  "email": "customer@example.com",
  "password": "SecurePassword123!",
  "full_name": "Alice Johnson",
  "phone": "+15551234567"
}
```

### Success Response (`201 Created`)
```json
{
  "data": {
    "user_id": "usr_01HABC1234567890",
    "email": "customer@example.com",
    "full_name": "Alice Johnson",
    "role": "CUSTOMER",
    "created_at": "2026-10-21T08:30:00Z"
  }
}
```

### Error Responses
- `400 Bad Request`: Validation failure (malformed email, weak password).
- `409 Conflict`: `AUTH_EMAIL_ALREADY_EXISTS` — Email is already registered.

---

## 2. Login User
Authenticates credentials and returns a signed JWT access token.

- **Method / Path:** `POST /api/v1/auth/login`
- **Auth Required:** None (Public)
- **Headers:** `Content-Type: application/json`

### Request Body
```json
{
  "email": "customer@example.com",
  "password": "SecurePassword123!"
}
```

### Success Response (`200 OK`)
```json
{
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "Bearer",
    "expires_in": 900,
    "user": {
      "id": "usr_01HABC1234567890",
      "email": "customer@example.com",
      "role": "CUSTOMER",
      "tenant_id": null
    }
  }
}
```

### Error Responses
- `401 Unauthorized`: `AUTH_INVALID_CREDENTIALS` — Incorrect email or password.
- `429 Too Many Requests`: Rate limit exceeded on login attempts.

---

## 3. Google OAuth 2.0 / OIDC Callback
Exchanges Google ID token/authorization code for a SwiftDrop JWT session.

- **Method / Path:** `POST /api/v1/auth/google`
- **Auth Required:** None (Public)
- **Headers:** `Content-Type: application/json`

### Request Body
```json
{
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjEy..."
}
```

### Success Response (`200 OK`)
```json
{
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "Bearer",
    "expires_in": 900,
    "user": {
      "id": "usr_01HABC1234567890",
      "email": "customer@gmail.com",
      "role": "CUSTOMER"
    }
  }
}
```
