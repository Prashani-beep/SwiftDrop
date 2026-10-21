# SwiftDrop Unified Web Application (`web/`)

> **Application Type:** Single-Page Application (SPA)  
> **Tech Stack:** React 18+, Vite, TypeScript, Tailwind CSS, TanStack Query, React Router v6  
> **Primary Ownership:**  
> - **Member 6:** Customer & Merchant Portal (`src/features/customer-merchant/`)  
> - **Member 7:** Dispatcher Board & Admin Backoffice (`src/features/dispatcher-admin/`)  
> - **Shared:** Application Shell, Auth & Design System (`src/app/`, `src/shared/`)

---

## 1. Architectural Blueprint & Rationale

SwiftDrop adopts a **Unified Single-Page Application (Option B)** architecture rather than splitting the client into multiple separate repositories or deployments.

### Why Unified SPA?
1. **Single Deployment & Port:** Built once into static assets and served via Traefik Ingress or containerized reverse proxy on a single port (e.g. `3000`). No complex multi-port routing or cross-origin overhead.
2. **Unified Authentication & RBAC Shell:** Single login screen (`/auth/login`). Upon JWT authentication, the user is redirected dynamically based on their role (`ROLE_CUSTOMER`, `ROLE_MERCHANT` &rarr; `/portal/*` vs `ROLE_DISPATCHER`, `ROLE_ADMIN`, `ROLE_COURIER` &rarr; `/ops/*`).
3. **Shared Design System:** A unified component system (buttons, modals, form inputs, status chips, notification toasts) ensures consistent brand visual standards across all portals.
4. **Feature-Sliced Isolation:** Code is isolated into dedicated feature folders (`customer-merchant` vs `dispatcher-admin`). Member 6 and Member 7 work in decoupled directories without merge conflicts.
5. **Route-Based Code Splitting:** Vite dynamically splits bundles at route boundaries (`React.lazy`). Customers never download the dispatcher board or admin reporting code.

---

## 2. Directory Structure

```
web/
├── README.md                            # Living frontend documentation (This file)
├── package.json                         # Dependencies and build scripts
├── vite.config.ts                       # Vite bundling configuration & path aliases
├── tsconfig.json                        # Strict TypeScript compiler options
├── tailwind.config.js                   # Design tokens, color palette & typography
├── public/                              # Static public assets (favicons, logos)
└── src/
    ├── app/                             # Core application shell & providers
    │   ├── routes/                      # Route definitions with lazy-loaded views
    │   ├── providers/                   # AuthProvider, QueryClientProvider, ThemeProvider
    │   ├── layouts/                     # PortalLayout, OpsLayout, PublicLayout
    │   └── guards/                      # RequireAuth, RoleGuard, GuestOnly
    ├── shared/                          # Cross-cutting UI components & utilities
    │   ├── components/                  # Button, Modal, Input, DataTable, Badge
    │   ├── api/                         # Axios/Fetch client, interceptors, error handler
    │   ├── hooks/                       # useAuth, useDebounce, useSSE, useNotification
    │   ├── types/                       # Shared models (User, Role, ApiEnvelope, ParcelStatus)
    │   └── utils/                       # Currency formatters, date helpers, validators
    └── features/
        ├── customer-merchant/           # Owned by Member 6
        │   ├── booking/                 # Multi-step parcel booking wizard & fee estimation
        │   ├── tracking/                # Interactive tracking view & SSE live updates
        │   ├── orders/                  # Merchant bulk order creation & CSV upload
        │   ├── address-book/            # Saved addresses & recipient directory
        │   └── settings/                # Merchant profile & webhook registration
        └── dispatcher-admin/            # Owned by Member 7
            ├── dispatch-board/          # Live courier assignment & unassigned parcel board
            ├── fleet/                   # Courier status, vehicle capacity & shift management
            ├── courier-mobile/          # Mobile-first courier task list & POD capture
            ├── zones/                   # Admin delivery zone configuration & polygon maps
            └── reports/                 # Operational metrics, SLA charts & audit logs
```

---

## 3. Role-Based Routing & Access Control Matrix

The router guards ensure strict authorization before any protected view is mounted:

| Path Pattern | Allowed Roles | Feature Owner | Description |
| :--- | :--- | :---: | :--- |
| `/auth/login` | *Public / Guest* | Shared | Unified authentication screen (email & password / SSO). |
| `/auth/register` | *Public / Guest* | Member 6 | Customer self-registration & Merchant onboarding application. |
| `/track/:trackingNumber` | *Public* | Member 6 | Sanitized public shipment tracker (SSE real-time milestone stream). |
| `/portal/dashboard` | `ROLE_CUSTOMER`, `ROLE_MERCHANT` | Member 6 | Customer shipment overview, recent activity, quick booking action. |
| `/portal/book` | `ROLE_CUSTOMER`, `ROLE_MERCHANT` | Member 6 | Multi-step parcel booking wizard with live fee estimation. |
| `/portal/merchants/orders` | `ROLE_MERCHANT` | Member 6 | Merchant B2B batch order ingestion and label generation. |
| `/ops/dispatch` | `ROLE_DISPATCHER`, `ROLE_ADMIN` | Member 7 | Interactive dispatch board: parcel queues, courier map, assignment. |
| `/ops/couriers` | `ROLE_DISPATCHER`, `ROLE_ADMIN` | Member 7 | Active courier fleet management, load monitoring, availability toggle. |
| `/ops/zones` | `ROLE_ADMIN` | Member 7 | Delivery zone geometry, base tariffs, and operational boundary setup. |
| `/ops/reports` | `ROLE_ADMIN` | Member 7 | SLA reports, courier performance metrics, and audit log explorer. |
| `/courier/tasks` | `ROLE_COURIER` | Member 7 | Mobile-responsive courier route, pickup/delivery tasks, barcode scanner. |
| `/courier/tasks/:id/pod`| `ROLE_COURIER` | Member 7 | Proof-of-delivery capture (camera photo upload, recipient signature). |

---

## 4. Shared API Client & Authentication Guarantees

All backend HTTP requests flow through a central client configured in `src/shared/api/`:
- **Base URL:** Driven by `VITE_API_BASE_URL` (defaulting to `/api/v1` through the reverse proxy).
- **JWT Injection:** Outgoing requests automatically inject `Authorization: Bearer <accessToken>`.
- **Token Refresh Interceptor:** On receiving `401 Unauthorized`, the client attempts a transparent token refresh using the HTTP-only refresh cookie before retrying the failed request.
- **Tenant Context:** For merchant accounts, `X-Tenant-ID` is passed when applicable to enforce multi-tenant isolation.
- **Zero Client-Side Calculation:** Frontend components **never calculate delivery fees or determine valid status transitions independently**. All prices and statuses are fetched authoritatively from Core API endpoints.

---

## 5. Development & Verification Commands

```bash
# Navigate to web frontend workspace
cd web

# Install dependencies
npm install

# Run Vite development server (HMR enabled on http://localhost:3000)
npm run dev

# Run TypeScript type-checking & ESLint
npm run lint

# Build production bundle with minification and gzip optimization
npm run build

# Preview production build locally
npm run preview
```

---

## 6. Living Documentation Maintenance Policy
- **Member 6** must update this document whenever new customer or merchant routes, workflows, or views are introduced.
- **Member 7** must update this document whenever new dispatcher boards, courier mobile workflows, or admin views are added.
- **Zero Drift:** Any change affecting routing, shared API contracts, or role guards must be documented here synchronously.
