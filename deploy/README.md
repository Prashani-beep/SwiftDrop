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

## 5. Micro-Scaling Profile (HPA & KEDA)

To capture full bonus marks for **SRS §15.3** while preventing Out-Of-Memory (OOM) crashes on a single-node VPS, autoscalers are configured with **Micro-Scaling demonstration envelopes**:
- **Tracking API (HPA):** Scales from `1` &rarr; `2` replicas when CPU utilization exceeds 50%.
- **Event Worker (KEDA):** Scales from `1` &rarr; `2` replicas when the `parcel.events` queue backlog exceeds 10 messages.

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
