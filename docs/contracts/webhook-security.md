# SwiftDrop — Webhook Security & SSRF Defense Contract

> **Authority:** Fulfills SRS SEC-010, SEC-011, and Engineering Standards §8 (External Integrations & Webhook Security).  
> **Owning Contexts:** Merchant Integration (`core-api/internal/merchants/`, Member 4) & Event Worker (`services/messaging/worker/`, Member 8).

---

## 1. Overview & Threat Model

SwiftDrop allows merchants to register outbound HTTP webhook endpoints to receive real-time notifications about parcel lifecycle milestones (`parcel.created`, `parcel.picked_up`, `parcel.in_transit`, `parcel.delivered`, `parcel.delivery.failed`).

Because outbound webhooks make HTTP requests to user-supplied URLs, they present critical security risks:
1. **Server-Side Request Forgery (SSRF):** Malicious endpoints attempting to access cloud metadata instances (`http://169.254.169.254/`), internal Kubernetes services (`http://core-api:8080/`), or internal database ports.
2. **DNS Rebinding Attacks:** Hostnames that resolve to a public IP during registration but resolve to an internal IP when the delivery worker fires.
3. **Replay & Tampering Attacks:** Eavesdroppers capturing and resubmitting webhook events to merchant backends.

---

## 2. Cryptographic Signature Scheme (HMAC-SHA256)

Every outbound webhook request from SwiftDrop includes authentication headers allowing the merchant to verify payload authenticity and integrity.

### 2.1 Request Headers
```http
POST /webhooks/swiftdrop HTTP/1.1
Host: merchant-store.com
Content-Type: application/json; charset=utf-8
X-SwiftDrop-Delivery-ID: whd_01HXYZ987654321
X-SwiftDrop-Timestamp: 1729512000
X-SwiftDrop-Signature: t=1729512000,v1=9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
User-Agent: SwiftDrop-Webhook-Delivery/1.0
```

### 2.2 Signature Calculation Procedure
1. Extract the current Unix epoch timestamp in seconds (e.g. `1729512000`).
2. Construct the **signature payload** by concatenating the timestamp, a period (`.`), and the raw JSON request body bytes:
   $$\text{signed\_payload} = \text{timestamp} + "." + \text{raw\_body\_bytes}$$
3. Compute the **HMAC-SHA256** hash of this string using the merchant's endpoint signing secret.
4. Format the `X-SwiftDrop-Signature` header as:
   $$t=\text{timestamp},v1=\text{hex\_encoded\_hash}$$

### 2.3 Replay Attack Prevention
Merchants must verify that the `X-SwiftDrop-Timestamp` header is within **300 seconds (5 minutes)** of the current server time:
$$| \text{CurrentTime} - \text{Timestamp} | \le 300\text{ seconds}$$
If the timestamp exceeds this delta, the merchant backend must reject the request.

---

## 3. Server-Side Request Forgery (SSRF) Prevention

The Event Worker's HTTP client enforces zero-trust outbound networking through a **hardened HTTP transport**:

```
Merchant URL ──► Custom DNS Resolver ──► CIDR Blacklist Check ──► Direct IP Dial (TLS SNI Preserved)
```

### 3.1 Prohibited Destination Ranges (CIDR Blocklist)
Outbound webhooks are strictly rejected if DNS resolves to any of the following IP ranges:

| Target Range | Description | Threat Mitigated |
| :--- | :--- | :--- |
| `0.0.0.0/8` | "This network" (RFC 1122) | Local routing misuse |
| `127.0.0.0/8` | IPv4 Loopback | Host localhost access |
| `10.0.0.0/8` | RFC 1918 Private | Internal VPC / Cluster network probing |
| `172.16.0.0/12` | RFC 1918 Private | Docker bridge / Pod network scanning |
| `192.168.0.0/16` | RFC 1918 Private | Local LAN resources |
| `169.254.0.0/16` | Link-Local IPv4 | Cloud instance metadata (`169.254.169.254`) |
| `::1/128` | IPv6 Loopback | Host localhost access |
| `fe80::/10` | IPv6 Link-Local | Local link interception |
| `fc00::/7` | IPv6 Unique Local Address | Internal cluster networks |

### 3.2 DNS Rebinding Defense: Safe Dialing in Go
To eliminate DNS rebinding (Time-Of-Check to Time-Of-Use / TOCTOU), the HTTP client **resolves DNS once and dials the verified IP directly**:

```go
// Enforced inside services/messaging/worker/webhook/client.go
func SafeDialContext(ctx context.Context, network, addr string) (net.Conn, error) {
    host, port, err := net.SplitHostPort(addr)
    if err != nil {
        return nil, err
    }
    
    ips, err := net.DefaultResolver.LookupIP(ctx, "ip", host)
    if err != nil {
        return nil, fmt.Errorf("DNS resolution failed: %w", err)
    }

    for _, ip := range ips {
        if isBlacklistedIP(ip) {
            return nil, fmt.Errorf("SSRF defense: blocked target IP %s", ip)
        }
    }

    // Connect to the first validated IP directly
    target := net.JoinHostPort(ips[0].String(), port)
    dialer := &net.Dialer{Timeout: 5 * time.Second}
    return dialer.DialContext(ctx, network, target)
}
```

### 3.3 Protocol & Port Constraints
- Only **HTTPS** URLs (`https://...`) are permitted in production. Plain HTTP is disallowed unless running under the `local` environment profile.
- Non-standard ports (e.g. `http://example.com:5432/`) are blocked. Only port `443` (and `80` locally) is permitted.
- Redirects (`HTTP 301, 302, 307`) are **NOT followed** automatically, preventing open redirect SSRF bypasses.

---

## 4. Delivery Retry Policy & Dead Letter Queue (DLQ)

When a merchant endpoint fails or returns non-2xx status, the Event Worker executes a **bounded exponential backoff** with random jitter:

| Attempt | Backoff Delay | Cumulative Elapsed | Failure Action |
| :--- | :--- | :--- | :--- |
| **Attempt 1** | Immediate | 0s | First attempt |
| **Attempt 2** | 30 Seconds $\pm 5\text{s}$ | ~30s | First retry |
| **Attempt 3** | 2 Minutes $\pm 15\text{s}$ | ~2m 30s | Second retry |
| **Attempt 4** | 10 Minutes $\pm 60\text{s}$ | ~12m 30s | Third retry |
| **Attempt 5** | 30 Minutes $\pm 180\text{s}$ | ~42m 30s | Final retry |
| **Exhaustion** | Permanent Failure | ~45m | Routed to DLQ; Endpoint marked `DEGRADED` |

- **Timeout:** Each HTTP request has a strict **5.0-second socket timeout**.
- **Payload Truncation:** For error logging, response bodies from merchant servers are truncated to 500 characters to prevent log flooding or out-of-memory spikes.

---

## 5. Merchant Verification Reference Implementation

### Python Example
```python
import hmac
import hashlib
import time

def verify_swiftdrop_webhook(payload_bytes: bytes, sig_header: str, secret: str) -> bool:
    try:
        parts = dict(item.split("=") for item in sig_header.split(","))
        timestamp = int(parts["t"])
        signature = parts["v1"]
    except (ValueError, KeyError):
        return False

    # Check replay attack window (5 minutes)
    if abs(time.time() - timestamp) > 300:
        return False

    signed_content = f"{timestamp}.".encode("utf-8") + payload_bytes
    expected_sig = hmac.new(
        secret.encode("utf-8"),
        signed_content,
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(expected_sig, signature)
```
