# AskTabib Admin — Engineering Improvements Report

**Product:** AskTabib Admin Dashboard
**Scope:** Frontend (Vite · React · Axios · Firebase Cloud Messaging)
**Audience:** Engineering leads, reviewers, and future maintainers
**Date:** 16 October 2026
**Format:** Problem → Action → Result

---

## 1. Executive summary

This cycle made the admin console safer to enter, more reliable when bookings arrive, and consistent with the API's timezone contract. Each item started as a concrete failure mode and ended in a verifiable outcome.

Themes: (1) real-time booking alerts via FCM web push, (2) admission control at the login boundary, (3) a fixed timezone contract with the API, (4) client-side hardening layered on top of backend controls.

**Operating assumptions.** Admin users are located in Germany and Syria only. Authorization and rate limiting are enforced by the Laravel backend; frontend checks improve UX and catch obvious mistakes but are never the security boundary.

---

## 2. Improvement log

### 2.1 Admin web push notifications

| | |
|---|---|
| **Problem** | Admins learned about new bookings only by refreshing lists or through an incomplete realtime path. No production-ready Web Push flow existed on the shared Firebase project. |
| **Action** | Integrated FCM for web on project `asktabib-eccde`: VAPID key; service worker with `notificationclick` registered before Firebase init; token upsert via `POST /api/device/token`; foreground `onMessage` (toast, bell, appointments refresh); logout cleanup (`fcm_token` sent + `deleteToken`). Payloads are validated against an allowlist so only `admin_booking_received` updates the UI. Device-token registration soft-fails if the backend rejects it. Token refresh, multi-tab/multi-device behavior (no duplicate notifications), and the service-worker scope under `/dashboard/` are handled and verified. |
| **Result** | A confirmed booking reaches the admin in the foreground (toast and bell) or background (system notification). Clicking opens the new-appointments view. The flow is stable across token rotation and multiple tabs or devices, and does not depend on mobile deep links. |

### 2.2 Admission control at login

| | |
|---|---|
| **Problem** | Login accepted any successful credential response. Non-admin accounts entered the shell and then hit `403 Forbidden` on routes such as `GET /api/admin/doctors`. Separately, a stored role in `localStorage` could not be trusted on its own. |
| **Action** | After login, resolve `data.role` and create a session only when `role === "admin"` (case-insensitive); other roles are rejected with a dedicated message and nothing is persisted. `isAuthenticated()` also requires the admin role. On startup, the session is now **validated against the server** (a `GET /me`-style call) instead of trusting stored data alone, so tampered or stale local sessions are rejected and cleared. |
| **Result** | Only server-confirmed admin identities hold a dashboard session. Wrong-account logins stop at the login screen, the post-login 403 loop is gone, and editing `localStorage` no longer grants shell access. API-level authorization remains authoritative. |

### 2.3 Timezone contract (`Europe/Berlin` · `Asia/Damascus`)

| | |
|---|---|
| **Problem** | The API accepts a controlled timezone vocabulary. Arbitrary IANA zones, or inconsistent mapping, risked wrong scheduling views and opaque server errors. |
| **Action** | Detect the device timezone via `Intl` and map `Europe/*` → `Europe/Berlin`, `Asia/*` → `Asia/Damascus`, anything else (or an `Intl` failure) → `Asia/Damascus`. Applied to outbound query params through the Axios interceptor. Unit tests cover the mapping and the fallback. |
| **Result** | Every authenticated request carries one of the two supported zones. Since admins operate only from Germany and Syria, the mapping is exact for the intended user base. Other regions are a documented out-of-scope case; if the footprint expands, the mapping and API vocabulary must be extended together. |

### 2.4 Client-side hardening

| | |
|---|---|
| **Problem** | Review identified gaps: unvalidated service-worker→page messages, FCM token left after logout, MIME-only image checks, and a known SSRF-class advisory in an older `i18next-http-backend`. |
| **Action** | (1) Sanitize FCM payloads with an allowlist. (2) Remove `fcmToken` in `clearSession`. (3) Validate image uploads by magic bytes instead of declared MIME. (4) Upgrade `i18next-http-backend` to v4. (5) Add production `npm audit` to CI in reporting mode while Firebase transitive advisories are open. (6) Add a client-side login lockout (60 s after five failures) as a UX layer. (7) Add unit tests for this logic: role admission, `isAuthenticated()` with non-admin sessions, the FCM payload allowlist, and the login lockout. |
| **Result** | Push data can no longer drive arbitrary UI changes, logout leaves no device token behind, uploads get a stronger first-line check, and the i18n backend advisory is closed. Brute-force protection is provided by **Laravel rate limiting**; the client lockout complements it and is not relied upon as a control. Server-side upload validation remains authoritative. The security logic is now covered by automated regression tests. |

### 2.5 Testing foundation

Vitest, Testing Library, MSW, jest-axe, Playwright, shared `src/test` helpers, and a CI workflow are in place. Full details, inventory, and trade-offs are in `doc/testing-report.md` and `TESTING.md`.

### 2.6 Admin home and navigation

| | |
|---|---|
| **Problem** | Home was a static blurb with little operational value, and Appointments, the primary workflow, was buried mid-sidebar. |
| **Action** | Replaced the blurb with three live counters (new appointments, medical tips, doctors) linking to their routes. Moved Appointments directly under Home. Aligned the brand strip with the header height and styling. |
| **Result** | Admins land on actionable counts and reach bookings in one click. |

### 2.7 Login experience

| | |
|---|---|
| **Problem** | The login screen was functional but dated and weak as a brand surface. |
| **Action** | Rebuilt it as a full-viewport composition (brand plane + form plane) with Outfit and IBM Plex Sans Arabic, restrained motion, and clearer hierarchy. The auth contract is unchanged. |
| **Result** | Sign-in reads as a modern admin entry point and posts to the same login API, now with the admin-role check from 2.2. |

---

## 3. Cross-cutting outcomes

| Area | Before | After |
|---|---|---|
| New booking awareness | Manual refresh / partial realtime | FCM web push + in-app toast and bell, stable across tabs and token refresh |
| Who can enter the console | Any successful login | Server-validated `role: admin` only (API still authoritative) |
| API `timezone` | Inconsistent | Device-mapped Berlin or Damascus, within the supported DE/SY scope |
| Abuse resistance | Weak | Laravel rate limiting + client lockout UX, payload allowlist, magic-byte checks |
| Regression safety | Minimal | Vitest + MSW + Playwright + CI, including tests for the security logic |
| Home usefulness | Static text | Live operational counters |

---

## 4. Constraints and residual risks

- **Token storage:** Bearer tokens remain in `localStorage`. A future pass should evaluate HttpOnly cookies with backend support.
- **Dependency advisories:** Firebase / `@grpc` advisories remain in the production audit; closing them requires a planned Firebase major bump (engine constraints apply). Tracked separately.
- **Authorization:** The backend is the authority for admin access. Frontend role checks are a UX gate.
- **FCM delivery:** Depends on browser permission, HTTPS (or localhost), and successful web device-token registration on the server.
- **Regional scope:** Timezone handling is designed for Germany and Syria only.

---

## 5. Next increments

1. Confirm with backend that `platform: "web"` device tokens are accepted for all admin accounts.
2. Make CI fail on high audit findings once Firebase is upgraded cleanly.
3. Expand co-located tests for the appointments accept-flow and notification inbox.
4. Document the admin login response contract (`role`, `token`, `timezone`) in an API note shared with mobile.
5. Evaluate HttpOnly-cookie sessions with the backend team.

---

## 6. Closing

The right people get in, bookings surface faster and reliably, requests speak the two timezones the API understands, and an automated safety net now surrounds these changes, including the security logic itself. Remaining gaps are mostly platform-level (session storage model, dependency majors) and further domain-screen test coverage.
---
