# SwiftDrop Infrastructure & Deployment Tier (`deploy/`)

> **Primary Ownership:**  
> - **Member 9:** DevOps, Kubernetes Manifests, Helm Charts, CI/CD & Observability  
> - **Member 7:** API Gateway & Edge Routing Policies (`deploy/gateway/`)  
> - **Member 8:** Outbox Reconciliation CronJob & Event Worker Deployment  
> **Target Environment:** K3s Lightweight Kubernetes on single-node VPS / Cloud Provider

---

## 1. Workload Architecture & Classification

To guarantee data safety and prevent volume mount conflicts, cluster workloads are strictly separated into **Stateless** and **Stateful** paradigms:

```
SwiftDrop Kubernetes Cluster (Namespace: `swiftdrop`)
├── Edge & Gateway Tier:
│   └── Traefik Ingress Controller (Deployment, NodePort / HostPort, Cloudflare Real-IP trust)
├── Stateless Workloads (Deployments):
│   ├── core-api (Go Modular Monolith: :8080)
│   ├── tracking-api (Go SSE Service: :8081 — Autoscaled via HPA)
│   ├── ai-assistant (FastAPI Service: :8000)
│   ├── web (React/TS Unified SPA: :3000)
│   ├── outbox-relay (Go Transactional Outbox Publisher daemon)
│   └── event-worker (Go RabbitMQ Consumer worker — Autoscaled via KEDA)
├── Scheduled Batch Workloads (CronJob):
│   └── swiftdrop-outbox-reconciler (CronJob: `*/10 * * * *` — Sweeps stuck outbox entries & delivery timeouts)
└── Stateful Workloads (StatefulSets):
    ├── postgres (PostgreSQL 16 StatefulSet + headless Service + 10Gi PVC on `local-path`)
    └── rabbitmq (RabbitMQ 3.13 StatefulSet + headless Service + 5Gi PVC on `local-path`)
```

---

## 2. Directory Layout

```
deploy/
├── README.md                            # Living deployment documentation (This file)
├── gateway/                             # Edge gateway and reverse proxy (Member 7)
│   ├── README.md                        # Gateway architecture and local Docker run guide
│   └── traefik/
│       ├── traefik.yml                  # Static configuration (Cloudflare trusted IPs, entrypoints)
│       └── dynamic-routes.yml           # Dynamic routing, CORS, upload limits & rate limits
└── kubernetes/                          # Kubernetes resource manifests (Member 9)
    ├── postgres-statefulset.yml         # Authoritative PostgreSQL 16 StatefulSet & headless service
    ├── rabbitmq-statefulset.yml         # RabbitMQ 3.13 event broker StatefulSet & headless service
    ├── cronjob-reconciliation.yml       # Periodic outbox sweeper & delivery audit CronJob
    └── autoscaling/                     # Bonus criteria autoscaling profiles (SRS §15.3)
        ├── hpa-tracking.yml             # CPU-based HorizontalPodAutoscaler for Tracking API
        └── keda-event-worker.yml        # RabbitMQ queue depth ScaledObject for Event Worker
```

---

## 3. Stateful Workloads: Why `StatefulSet`?

As specified in **SRS §11.2** (*"do not assume a Deployment alone makes stateful data safe"*), in-cluster PostgreSQL and RabbitMQ are deployed as **StatefulSets** rather than standard Deployments:
1. **Deterministic Identity:** Guaranteed ordinal names (`postgres-0`, `rabbitmq-0`) and permanent internal DNS hostnames (`postgres-0.postgres-headless.swiftdrop.svc.cluster.local`).
2. **Dedicated Storage via `volumeClaimTemplates`:** Bound to K3s's built-in `local-path` storage class.
3. **No Multi-Attach Deadlocks:** Prevents volume attachment race conditions when pods are rescheduled or restarted on node reboot.

---

## 4. Cloudflare + Traefik Real-IP Trust

Because Cloudflare sits in front of the cluster edge, Traefik is configured with `forwardedHeaders.trustedIPs` in `deploy/gateway/traefik/traefik.yml` covering all official Cloudflare IPv4 CIDR blocks.
- **Without trusted IPs:** Traefik would attribute all inbound traffic to Cloudflare's edge IPs, causing the 30 req/min rate limiter on `/api/v1/tracking` to lock out all users globally.
- **With trusted IPs:** Traefik restores the true client IP from `CF-Connecting-IP` / `X-Forwarded-For`, isolating rate-limiting buckets per visitor.

---

## 5. Workload Scalability Rules & Autoscaling Profiles (HPA & KEDA)

To capture full bonus marks for **SRS §15.3** while preventing Out-Of-Memory (OOM) crashes and database connection saturation on a single-node VPS, cluster workloads adhere to strict scalability profiles:

### 5.1 Scalable Workloads
1. **Core API (HorizontalPodAutoscaler):**
   - **Metrics:** Scales between `2` and `4` replicas when CPU utilization exceeds 70%.
   - **Stateless Invariant:** Pods store zero session state in memory and zero files on local disk (shared S3/MinIO for attachments).
   - **Connection Pool Ceiling:** Per-pod DB pool is tuned such that: $\text{Replicas}_{\text{max}} \times \text{MaxOpenConns} \le \text{PostgreSQL MaxConnections}$ (e.g. $4 \times 20 = 80 \le 100$).
   - **Graceful Termination:** Listens for `SIGTERM` and allows a 20-second grace period for in-flight requests and database transactions to commit cleanly.
2. **Tracking API (HorizontalPodAutoscaler):**
   - **Metrics:** Scales from `1` &rarr; `2` (or `2` &rarr; `4`) replicas when CPU exceeds 50% or active HTTP/2 connections peak.
   - **Realtime Fan-Out Invariant:** When $>1$ replica is active, live Server-Sent Events (SSE) streams are broadcasted across pods via **Redis Pub/Sub**.
   - **Edge Proxy Safety:** Emits `X-Accel-Buffering: no` and sends 15-second keepalive pings (`: ping\n\n`) to prevent Cloudflare/Traefik stream termination.
3. **Event Worker (KEDA ScaledObject):**
   - **Metrics:** Scales from `1` &rarr; `3` replicas based on the RabbitMQ `events` queue backlog (`queueLength > 10`).
   - **Consumer Idempotency Invariant:** Every consumer MUST verify incoming event IDs against the authoritative `idempotency_keys` table before executing side-effects (SMS/email notifications, merchant webhooks). Duplicate message deliveries must be safely acknowledged without duplicate actions.
4. **Web Frontend (HorizontalPodAutoscaler):**
   - **Metrics:** Scales from `2` &rarr; `3` replicas based on traffic volume. Stateless Nginx container serving immutable hashed JS/CSS assets.

### 5.2 Non-Scalable Singletons & Stateful Systems
1. **Outbox Relay Daemon (STRICT SINGLETON — `replicas: 1`):**
   - **PROHIBITION:** The Outbox Relay MUST NEVER be scaled horizontally. A single poller guarantees transactional outbox row ordering and prevents database lock contention or duplicate publishing to RabbitMQ.
2. **PostgreSQL 16 & RabbitMQ 3.13 (`StatefulSet`):**
   - Primary database and message broker run as single-instance StatefulSets bound to dedicated `local-path` PersistentVolumeClaims. They must NEVER be scaled via standard Deployment replicas.
3. **Outbox Reconciler (`CronJob`):**
   - Scheduled periodic job (`*/10 * * * *`) configured with `concurrencyPolicy: Forbid` to prevent concurrent overlapping executions.


---

## 6. Local / Cluster Deployment Commands

```bash
# 1. Create the SwiftDrop namespace
kubectl create namespace swiftdrop

# 2. Deploy Stateful Persistence (Postgres & RabbitMQ)
kubectl apply -f deploy/kubernetes/postgres-statefulset.yml
kubectl apply -f deploy/kubernetes/rabbitmq-statefulset.yml

# 3. Deploy Periodic Reconciliation CronJob
kubectl apply -f deploy/kubernetes/cronjob-reconciliation.yml

# 4. Deploy Autoscaling Profiles (Optional / Demo)
kubectl apply -f deploy/kubernetes/autoscaling/hpa-tracking.yml
kubectl apply -f deploy/kubernetes/autoscaling/keda-event-worker.yml
```
