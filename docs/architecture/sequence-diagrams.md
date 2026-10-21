# SwiftDrop — Architecture Sequence Diagrams

> **Authority:** Fulfills SRS §17.1 (Required Deliverables: Sequence & Activity Diagrams) and reflects Domain Boundaries (`domain-boundaries.md`).  
> **Audience:** All SwiftDrop Developers, AI Coding Agents, and Solution Architects.

---

## 1. Customer Parcel Booking Flow

Covers customer price calculation, interactive booking confirmation, atomic transaction persistence, and asynchronous notification dispatch.

```mermaid
sequenceDiagram
    autonumber
    actor Customer as Customer (Web Portal)
    participant Gateway as Traefik Ingress Gateway
    participant CoreAPI as Core API (Modular Monolith)
    participant DB as PostgreSQL
    participant Relay as Outbox Relay Daemon
    participant Broker as RabbitMQ Broker
    participant Worker as Event Worker Service
    participant Notify as External SMS/Email Provider

    Customer->>Gateway: POST /api/v1/pricing/quote (Weight, Dims, Zones)
    Gateway->>CoreAPI: Forward Quote Request
    CoreAPI->>CoreAPI: Pricing Engine calculates fee (LKR cents)
    CoreAPI->>DB: INSERT into pricing_quotes
    CoreAPI-->>Customer: Return Quote (Quote ID, Amount, Expiry 15m)

    Customer->>Gateway: POST /api/v1/parcels/book (Quote ID, Recipient, Address)
    Gateway->>CoreAPI: Forward Booking Request (with JWT)
    
    activate CoreAPI
    CoreAPI->>DB: BEGIN Transaction
    CoreAPI->>DB: Validate Quote & Verify Not Expired
    CoreAPI->>DB: INSERT INTO parcels (Status: CREATED, Tracking Number)
    CoreAPI->>DB: INSERT INTO parcel_addresses (Pickup, Dropoff)
    CoreAPI->>DB: INSERT INTO parcel_status_history (to_status: CREATED)
    CoreAPI->>DB: INSERT INTO outbox_events (event_type: parcel.created)
    CoreAPI->>DB: COMMIT Transaction
    deactivate CoreAPI

    CoreAPI-->>Customer: 201 Created (Tracking Number: SD-20261021-XXXXX)

    Note over Relay,DB: Asynchronous Outbox Pipeline
    loop Every 500ms
        Relay->>DB: SELECT * FROM outbox_events WHERE status = 'PENDING' FOR UPDATE SKIP LOCKED
        Relay->>Broker: Publish AMQP parcel.created (Publisher Confirms)
        Broker-->>Relay: Ack
        Relay->>DB: UPDATE outbox_events SET status = 'PUBLISHED'
    end

    Broker->>Worker: Consume parcel.created
    Worker->>Worker: Check Idempotency Key
    Worker->>Notify: Send Booking Confirmation (SMS/Email)
    Notify-->>Worker: 200 OK (Provider Message ID)
    Worker->>DB: INSERT INTO notification_deliveries (Status: SENT)
    Worker->>Broker: AMQP Ack
```

---

## 2. B2B Merchant API Order Creation & Webhook Delivery

Covers automated merchant order submission via API key authentication, rate limiting, and webhook event callback.

```mermaid
sequenceDiagram
    autonumber
    actor MerchantSys as Merchant ERP / Store
    participant Gateway as Traefik API Gateway
    participant CoreAPI as Core API (Modular Monolith)
    participant DB as PostgreSQL
    participant Relay as Outbox Relay Daemon
    participant Broker as RabbitMQ Broker
    participant Worker as Webhook Dispatch Worker
    actor MerchantWebhook as Merchant Webhook Endpoint

    MerchantSys->>Gateway: POST /api/v1/merchants/parcels<br/>Header: X-API-Key: sd_live_a1b2c3d4_...<br/>Header: Idempotency-Key: uuid-9988
    
    Gateway->>Gateway: Check Edge Rate Limit (Merchant Quota)
    Gateway->>CoreAPI: Forward Request

    activate CoreAPI
    CoreAPI->>DB: Query merchant_api_credentials by prefix 'a1b2c3d4'
    CoreAPI->>CoreAPI: Verify Argon2id/SHA256 Key Hash & Scopes
    CoreAPI->>DB: Check idempotency_keys for uuid-9988
    
    alt Idempotency Key Already Exists
        CoreAPI-->>MerchantSys: 200 OK (Cached Saved Response)
    else New Request
        CoreAPI->>DB: BEGIN Transaction
        CoreAPI->>DB: INSERT INTO parcels (Status: CREATED, merchant_id)
        CoreAPI->>DB: INSERT INTO parcel_status_history
        CoreAPI->>DB: INSERT INTO outbox_events (event_type: parcel.created)
        CoreAPI->>DB: INSERT INTO idempotency_keys (Key, RequestHash, Response)
        CoreAPI->>DB: COMMIT Transaction
        CoreAPI-->>MerchantSys: 201 Created (Parcel Details, Tracking Number)
    end
    deactivate CoreAPI

    Note over Relay,Broker: Event Published to RabbitMQ
    Relay->>Broker: Publish parcel.created

    Broker->>Worker: Consume parcel.created
    Worker->>DB: Query webhook_endpoints for Merchant & Subscribed Event
    
    activate Worker
    Worker->>Worker: Generate HMAC-SHA256 Signature Header<br/>(X-SwiftDrop-Signature, Timestamp)
    Worker->>Worker: Validate URL (SSRF Defense: Disallow RFC1918 / 169.254.169.254)
    Worker->>MerchantWebhook: POST Event Payload (Timeout: 5s)
    
    alt Webhook Returns 2xx
        MerchantWebhook-->>Worker: 200 OK
        Worker->>DB: INSERT INTO webhook_deliveries (Status: SUCCESS)
        Worker->>Broker: AMQP Ack
    else Webhook Returns 5xx / Timeout
        MerchantWebhook-->>Worker: Error or Timeout
        Worker->>DB: INSERT INTO webhook_deliveries (Status: PENDING, NextRetry: +30s)
        Worker->>Broker: AMQP Nack / Re-queue with Backoff
    end
    deactivate Worker
```

---

## 3. Courier Delivery Milestone & Real-Time SSE Fan-Out

Covers courier proof-of-delivery upload, domain transition validation, real-time tracking SSE projection, and customer notifications.

```mermaid
sequenceDiagram
    autonumber
    actor Courier as Courier Mobile Client
    participant Gateway as Traefik API Gateway
    participant CoreAPI as Core API
    participant DB as PostgreSQL
    participant Relay as Outbox Relay
    participant Broker as RabbitMQ
    participant TrackingSvc as Tracking API Service
    participant WebClient as Customer Live Tracking (Browser)

    WebClient->>Gateway: GET /api/v1/tracking/stream/SD-20261021-99812
    Gateway->>TrackingSvc: Establish SSE Connection (HTTP/2 Streaming)
    TrackingSvc-->>WebClient: 200 OK (event: connected, current milestone)

    Courier->>Gateway: POST /api/v1/courier/parcels/{id}/deliver<br/>(Proof Photo, Signature, GPS)
    Gateway->>CoreAPI: Forward Request (with Courier JWT)

    activate CoreAPI
    CoreAPI->>DB: BEGIN Transaction
    CoreAPI->>DB: SELECT * FROM parcels WHERE id = $1 FOR UPDATE
    CoreAPI->>CoreAPI: Aggregate checks invariants:<br/>- Is courier assigned?<br/>- Is status OUT_FOR_DELIVERY?<br/>- Is proof attached?
    CoreAPI->>DB: INSERT INTO delivery_attempts (Outcome: SUCCESS, ProofUrl, GPS)
    CoreAPI->>DB: UPDATE parcels SET current_status = 'DELIVERED', version = version + 1
    CoreAPI->>DB: INSERT INTO parcel_status_history (to_status: DELIVERED)
    CoreAPI->>DB: INSERT INTO outbox_events (event_type: parcel.delivered)
    CoreAPI->>DB: COMMIT Transaction
    deactivate CoreAPI

    CoreAPI-->>Courier: 200 OK (Status: DELIVERED)

    Relay->>Broker: Publish AMQP parcel.delivered

    par SSE Live Update Fan-Out
        Broker->>TrackingSvc: AMQP Event: parcel.delivered
        TrackingSvc->>TrackingSvc: Sanitize Payload (Strip PII, recipient phone/name)
        TrackingSvc-->>WebClient: SSE event: milestone_updated<br/>{ status: "DELIVERED", timestamp: "..." }
    and Customer Push Notification
        Broker->>CoreAPI: AMQP Event: parcel.delivered
        CoreAPI->>DB: Record and queue Customer Delivery Confirmation SMS
    end
```

---

## 4. Google OAuth 2.0 / OpenID Connect (OIDC) Authentication

Covers secure identity federation, CSRF prevention via state/nonce, account linking, and issuance of JWT tokens and refresh cookies.

```mermaid
sequenceDiagram
    autonumber
    actor User as User (Browser)
    participant CoreAPI as Core API Auth Service
    participant Google as Google Identity Provider
    participant DB as PostgreSQL

    User->>CoreAPI: GET /api/v1/auth/oauth/google
    CoreAPI->>CoreAPI: Generate cryptographically random State & Nonce
    CoreAPI-->>User: 302 Redirect to accounts.google.com<br/>Set-Cookie: oauth_state=...; HttpOnly; Secure; SameSite=Lax

    User->>Google: Authenticate & Authorize SwiftDrop Scopes (email, profile)
    Google-->>User: 302 Redirect to /api/v1/auth/oauth/google/callback?code=...&state=...

    User->>CoreAPI: GET /api/v1/auth/oauth/google/callback?code=...&state=...<br/>Cookie: oauth_state=...
    
    activate CoreAPI
    CoreAPI->>CoreAPI: Verify returned state matches oauth_state cookie (CSRF check)
    CoreAPI->>Google: POST /token (Exchange auth code for ID Token & Access Token)
    Google-->>CoreAPI: 200 OK (id_token containing sub, email, email_verified)
    CoreAPI->>CoreAPI: Verify Google ID token signature, issuer, and nonce
    
    CoreAPI->>DB: SELECT * FROM oauth_identities WHERE provider = 'GOOGLE' AND provider_subject = sub
    
    alt Identity Exists
        CoreAPI->>DB: SELECT * FROM users WHERE id = identity.user_id
    else New Federated Identity
        CoreAPI->>DB: SELECT * FROM users WHERE email = google.email
        alt Local User Exists
            CoreAPI->>DB: INSERT INTO oauth_identities (user_id, 'GOOGLE', sub)
        else Brand New Account
            CoreAPI->>DB: INSERT INTO users (email, display_name, status: ACTIVE)
            CoreAPI->>DB: INSERT INTO user_roles (user_id, role_id: 'CUSTOMER')
            CoreAPI->>DB: INSERT INTO oauth_identities (user_id, 'GOOGLE', sub)
        end
    end

    CoreAPI->>CoreAPI: Mint RS256 JWT Access Token (15m expiry)
    CoreAPI->>DB: INSERT INTO user_sessions / refresh_tokens (token_hash, 7d expiry)
    
    CoreAPI-->>User: 302 Redirect to /portal/dashboard<br/>Set-Cookie: swiftdrop_refresh_token=...; HttpOnly; Secure; SameSite=Strict<br/>Response Body: { accessToken: "..." }
    deactivate CoreAPI
```

---

## 5. Transactional Outbox → RabbitMQ Publisher Pipeline

Illustrates zero message loss, at-least-once delivery, and consumer-side idempotency.

```mermaid
sequenceDiagram
    autonumber
    participant App as Core API Use Case
    participant DB as PostgreSQL (Outbox Table)
    participant Relay as Outbox Relay Daemon
    participant Broker as RabbitMQ (Durable Exchange)
    participant Consumer as Event Consumer Worker

    Note over App,DB: Phase 1: Atomic Database Commit
    activate App
    App->>DB: BEGIN Transaction
    App->>DB: Mutate Domain Aggregate (e.g. Parcel, Dispatch)
    App->>DB: INSERT INTO outbox_events (AggregateID, EventType, Payload, Status: 'PENDING')
    App->>DB: COMMIT Transaction
    deactivate App

    Note over Relay,Broker: Phase 2: Reliable Publisher Loop
    activate Relay
    Relay->>DB: SELECT * FROM outbox_events WHERE status = 'PENDING'<br/>ORDER BY created_at ASC LIMIT 50 FOR UPDATE SKIP LOCKED
    Relay->>Broker: Publish Message (DeliveryMode: Persistent, MessageID: EventUUID)
    Broker-->>Relay: Publisher Confirm (Ack received from RabbitMQ)
    Relay->>DB: UPDATE outbox_events SET status = 'PUBLISHED', published_at = NOW()<br/>WHERE id IN (...)
    deactivate Relay

    Note over Broker,Consumer: Phase 3: At-Least-Once Consumer Execution
    Broker->>Consumer: Push Message
    activate Consumer
    Consumer->>DB: SELECT 1 FROM processed_events WHERE event_id = $1
    
    alt Event Already Processed (Duplicate Delivery)
        Consumer->>Broker: AMQP Ack (Drop Duplicate Safely)
    else First Time Processing
        Consumer->>Consumer: Execute Business Action (e.g. Send SMS / Trigger Projection)
        Consumer->>DB: INSERT INTO processed_events (event_id, processed_at)
        Consumer->>Broker: AMQP Ack
    end
    deactivate Consumer
```
