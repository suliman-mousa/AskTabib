# AskTabib Admin — Software Architecture Report

| Field | Value |
|--------|--------|
| **System** | AskTabib Admin Dashboard (`asktabib-admin`) |
| **Document type** | Software Architecture Description (SAD) |
| **Viewpoint** | C4 + layered frontend + deployment |
| **Scope** | Web SPA under `/dashboard/`; Laravel API and Firebase are external systems |
| **Audience** | Engineering leads, architects, maintainers, onboarding |
| **Date** | 16 October 2026 |
| **Related** | [testing-report.md](./testing-report.md) · [improvements-report.md](./engineering Improvements/Web-Admin-Improvements-Report.md) · [diagrams-report.md](./diagrams/Web-Admin-Diagrams.md) |

---

## 1. Purpose

This document describes how the admin console is structured, how it talks to the platform, and which boundaries must not be blurred. It is written for engineers who need to change the system safely—not as a product brochure.

Architectural style: **feature-sliced SPA** with a thin shared kernel (HTTP, auth, i18n, realtime). Authorization and business rules live on the **Laravel API**; the browser enforces UX gates and defence-in-depth only.

---

## 2. System context

AskTabib Admin is the browser console for platform operators. Admins authenticate, manage doctors and content, review appointments, and receive realtime booking alerts.

| Actor / system | Role |
|----------------|------|
| **Admin user** | Human operator (Germany / Syria operating assumption) |
| **Admin SPA** | This repository — Vite · React 19 · Axios · Zustand · i18next · FCM |
| **AskTabib API** | Laravel backend — auth, RBAC, domain APIs |
| **Firebase Cloud Messaging** | Web push delivery (`asktabib-eccde`) |
| **Pusher / Echo** *(optional path)* | Channel-based notifications when configured |

The SPA is served at **`/dashboard/`** (Vite `base` + React Router basename). Root `/` redirects into that path at the nginx edge.

---

## 3. Quality attributes (drivers)

Decisions in this codebase are driven by the following, in rough priority order:

1. **Correct admission** — only `role === admin` sessions enter the shell; API remains authoritative.
2. **Operational reliability** — booking alerts must reach the admin in foreground and background.
3. **Contract fidelity** — every API call carries `lang` and a clamped `timezone` (`Europe/Berlin` \| `Asia/Damascus`).
4. **Maintainability** — feature folders, lazy routes, centralized endpoints and interceptors.
5. **Deployability** — multi-stage Docker → static assets under nginx with security headers.
6. **Verifiability** — Vitest + MSW + Playwright + CI quality gate.

---

## 4. Logical architecture

### 4.1 Layered view

```
┌─────────────────────────────────────────────────────────────┐
│  Presentation   features/* pages · layouts · shared UI       │
├─────────────────────────────────────────────────────────────┤
│  Application    hooks (useApi, useFcmListener, …) · stores   │
├─────────────────────────────────────────────────────────────┤
│  Infrastructure axios · firebase · fcm · echo · apiEndpoints │
├─────────────────────────────────────────────────────────────┤
│  Cross-cutting  auth · timezone · i18n · session · throttle  │
└─────────────────────────────────────────────────────────────┘
```

| Layer | Responsibility | Typical location |
|-------|----------------|------------------|
| Presentation | Screens, forms, navigation shell | `src/features/**`, `src/layouts`, `src/components` |
| Application | Request orchestration, listeners, client state | `src/hooks`, `src/store` |
| Infrastructure | HTTP client, FCM, Echo, endpoint map | `src/services` |
| Cross-cutting | Session, role, timezone, payload validation | `src/utils`, `src/app/i18n` |

There is **no Redux and no TanStack Query**. Server data is fetched per page/hook via `useApi` → `axiosInstance`. Client UI state (user, toasts, notifications, loading) lives in **Zustand** stores without a provider tree.

### 4.2 Source topology

| Path | Intent |
|------|--------|
| `src/main.jsx` | Bootstrap: i18n, Firebase side-effect, hydrate user store, mount `App` |
| `src/app/` | Shell: `App`, routes, i18n bootstrap |
| `src/features/{auth,home,doctors,appointments,…}` | Domain UI modules (pages + local components) |
| `src/layouts/` | Dashboard chrome (sidebar, header) |
| `src/components/` | Shared UI and route guards |
| `src/services/` | Axios, endpoints, Firebase, FCM, Echo |
| `src/hooks/` | Reusable application behaviour |
| `src/store/` | Zustand stores |
| `src/utils/` | Pure helpers and policy clamps |
| `public/` | Static assets, locales, `firebase-messaging-sw.js` |
| `e2e/` · `src/test/` | Playwright and Vitest/MSW harness |
| `Dockerfile` · `nginx.conf` | Build and edge serving |

### 4.3 Feature modules (bounded UI areas)

| Module | Primary capability |
|--------|-------------------|
| **auth** | Login, throttle, admin role gate, session establish |
| **home** | Landing dashboard (three operational containers) |
| **appointments** | Tabbed admin appointment queues; FCM deep-link target |
| **doctors** | CRUD, schedules, per-doctor finance |
| **specialties** | Specialty catalogue admin |
| **tips** | Medical tips content |
| **ads** | Advertisement management |
| **announcements** | Broadcast announcements |
| **finance** | Platform financial summary |

Routes are **lazy-loaded** (`React.lazy` + `Suspense`) so the initial bundle stays lean; vendor/realtime chunks are split further in Vite `manualChunks`.

---

## 5. Runtime behaviour

### 5.1 Application bootstrap

1. Load CSS and initialize i18n (HTTP backend for `public/locales/{ar,en}`).
2. Initialize Firebase app module (side-effect import).
3. If a non-expired admin session exists in `localStorage`, hydrate `useUserStore`.
4. Mount `App` inside `StrictMode` → `ErrorBoundary` → `BrowserRouter` (basename from `BASE_URL`) → global hosts (scroll, notifications/toasts, loading overlay) → `AppRoutes`.

### 5.2 Routing and admission

| Concern | Mechanism |
|---------|-----------|
| Guest-only | `GuestRoute` around `/login` |
| Authenticated | `ProtectedRoute` → `isAuthenticated()` |
| Session rules | Bearer present, JWT `exp` not past (when JWT-shaped), stored role is admin |
| Unknown paths | Redirect to `/home` or `/login` |

Login creates a session **only** when the API returns a token **and** `role === "admin"` (case-insensitive). Non-admin credentials are rejected without persistence.

### 5.3 HTTP pipeline

All domain traffic goes through a single Axios instance:

| Step | Behaviour |
|------|-----------|
| Request | `Authorization: Bearer …`, `x-api-key`, query `lang` (`ar`\|`en`), query `timezone` (mapped device zone) |
| Response success | Returned to `useApi` which interprets `success` / `errors` envelopes |
| Response `401` | Session cleanup + redirect to login (except login/logout URLs) |

`useApi` also manages local/global loading, abort-safe error mapping, and XSS-safe message filtering.

### 5.4 Timezone contract

Device IANA zone → API zone:

| Device continent prefix | API `timezone` |
|-------------------------|----------------|
| `Europe/*` | `Europe/Berlin` |
| `Asia/*` | `Asia/Damascus` |
| Other / unknown | `Asia/Damascus` (default) |

Mapping is centralized in `src/utils/timezone.js` and applied on **every** request via the Axios interceptor—pages must not invent their own zone strings.

### 5.5 Realtime: FCM (primary) and Echo (secondary)

**FCM web push** is the production path for admin booking alerts:

1. Register service worker at `{BASE_URL}firebase-messaging-sw.js` with scope `/dashboard/`.
2. Request notification permission; obtain token with VAPID key.
3. Upsert token via `POST /api/device/token` (soft-fail on backend rejection).
4. Foreground: `onMessage` → validated payload → toast, bell, appointments refresh.
5. Background: SW shows system notification; click navigates to new appointments.
6. Logout: notify backend / delete token; clear stored `fcmToken`.

Payloads are allowlisted (`admin_booking_received` and related sanitization). The SPA must not trust arbitrary SW postMessages.

**Laravel Echo / Pusher** remains available for channel-style listeners when env keys are present; it is not a substitute for FCM push delivery.

---

## 6. State model

| Store | Concern |
|-------|---------|
| `useUserStore` | Current admin profile (hydrated from session) |
| `useToastStore` | Ephemeral UI toasts |
| `useNotificationStore` | In-app notification list / unread signals |
| `useLoadingStore` | Global overlay counter for long requests |

Session secrets and profile cache: `localStorage` keys `accessToken`, `authUser`, `fcmToken` (legacy `token` cleared on write). Tokens in the browser are a known residual risk; HttpOnly cookies would be a backend-led evolution.

---

## 7. Cross-cutting policies

| Policy | Implementation |
|--------|----------------|
| Admin-only shell | Login role check + `isAuthenticated()` |
| Login abuse dampening | Client throttle (`loginThrottle`) — UX; API rate limits are authoritative |
| Upload safety | Magic-byte image validation before send |
| URL safety | `toSafeHttpUrl` for outbound links |
| FCM trust boundary | `fcmPayload` sanitize / allowlist |
| i18n + RTL | i18next; `applyDocumentDirection` on language change |
| Error containment | Root `ErrorBoundary` |
| CSP / headers | nginx `Content-Security-Policy` and related hardened headers |

---

## 8. Deployment architecture

```
[ CI: lint · typecheck · unit · e2e ]
              │
              ▼
[ Docker multi-stage ]
  node:20-alpine  →  npm ci · vite build (env baked at build time)
  nginx:alpine    →  dist → /usr/share/nginx/html/dashboard
              │
              ▼
[ Browser ]  GET /dashboard/…  →  SPA + SW
              │
              ├── HTTPS → AskTabib API
              └── FCM / (optional) Pusher WSS
```

| Concern | Choice |
|---------|--------|
| Build | Vite 7, esbuild minify, hashed assets under `assets/js|css` |
| Base path | `/dashboard/` |
| Runtime config | `VITE_*` variables inlined at build (API base, API key, Firebase, Pusher, …) |
| Edge | nginx SPA fallback `try_files` → `index.html`; `/` → `302 /dashboard/` |
| Quality gate | GitHub Actions: audit (advisory), lint, typecheck, coverage, Playwright |

---

## 9. Security architecture (frontend view)

```
Trust boundary A — Browser
  · UI gates, payload validation, CSP, no Service Account secrets

Trust boundary B — AskTabib API
  · Authentication, authorization, rate limits, persistence

Trust boundary C — Firebase
  · Push fan-out; client holds only public Firebase config + VAPID
```

**Hard rules for this repo:**

- Never embed Firebase Service Account JSON or private keys in the frontend.
- Never treat `localStorage` role alone as authorization for sensitive operations.
- Never widen nginx `script-src` casually; tune `connect-src` / `img-src` to real hosts before production.

---

## 10. Testing architecture (summary)

| Layer | Tool | Role |
|-------|------|------|
| Unit / component | Vitest · Testing Library · jest-axe | Utils, hooks, shared UI |
| HTTP in unit tests | MSW v2 | Real Axios against fake HTTP |
| E2E | Playwright (Chromium, Firefox, WebKit) | Critical admin login path |
| CI | `.github/workflows/ci.yml` | Automated gate + artifacts |

Full detail: [testing-report.md](./testing-report.md).

---

## 11. Architectural decisions (ADR-style)

| ID | Decision | Rationale |
|----|----------|-----------|
| ADR-01 | Feature folders + shared kernel | Scales domain UI without a heavyweight framework |
| ADR-02 | Zustand over Redux/Query | Small surface area; no provider tax; fits JSX-first app |
| ADR-03 | Single Axios instance + interceptors | Guarantees auth, lang, timezone on every call |
| ADR-04 | Lazy routes + manual chunks | Faster first paint under `/dashboard/` |
| ADR-05 | FCM as admin alert channel | Matches mobile platform push; works foreground/background |
| ADR-06 | Admin role gate at login | Stops non-admin 403 loops at the boundary |
| ADR-07 | Berlin/Damascus timezone clamp | Matches API contract for operator geographies |
| ADR-08 | Docker + nginx static | Simple, cacheable, header-hardened delivery |

---

## 12. Constraints and residuals

| Item | Note |
|------|------|
| Token storage | Bearer in `localStorage` — XSS remains the primary client risk |
| Frontend role check | Defence-in-depth only; API RBAC is the authority |
| Firebase audit advisories | Production `npm audit` may continue-on-error until upstream upgrades |
| Echo vs FCM | Prefer FCM for booking push; keep Echo env optional |
| TypeScript | Tooling/`allowJs` hygiene, not full product typing |

---

## 13. Evolution guidance

When extending the system:

1. Add domain UI under `src/features/<domain>/`; keep HTTP paths in `apiEndpoints.js`.
2. Prefer `useApi` + interceptors over ad-hoc `fetch`/`axios` calls.
3. Do not pass raw device IANA zones to the API—use `getDeviceTimezone()`.
4. Treat any new push payload type as untrusted until allowlisted in `fcmPayload`.
5. Protect new routes with `ProtectedRoute`; keep login as the only guest entry.
6. Extend automated tests at the layer that fails cheapest (unit before E2E).

Visual counterparts for this document live in [diagrams-report.md](./diagrams/Web-Admin-Diagrams.md).
