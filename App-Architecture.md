# AskTabib — Software Architecture Description (SAD)

| Field | Value |
|-------|--------|
| **Document type** | Architecture Description (AD) |
| **Normative alignment** | Inspired by [ISO/IEC/IEEE 42010:2022](https://www.iso.org/standard/74393.html) (concepts, viewpoints, correspondence) |
| **Visualization method** | [C4 model](https://c4model.com/) (Context → Container → Component) |
| **Internal style** | Feature-first Clean Architecture + Cubit |
| **System of interest** | AskTabib dual-role telehealth Flutter client |
| **Flutter package** | `asktabib_patient` |
| **Mobile application id** | `com.asktabib.app` |
| **API base** | `https://asktabib.com/api/` |
| **Audience** | Engineering leads, architects, senior developers, security reviewers |
| **Owner** | Mobile Tech Lead |
| **Related docs** | [`ARCHITECTURE_DIAGRAMS.md`](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md), [`SECURITY_PERFORMANCE_ARCHITECTURE_IMPROVEMENTS.md`](./engineering%20Improvements/APP-IMPROVEMENT-REPORT.md) |

**Convention.** Behaviors described as *standard* in §7.5, §8 and §12 are the target posture following industry best practice. Verify each against the implementation during review; any deviation is recorded as a risk or an ADR.

---

## 1. Purpose and scope

This AD expresses the architecture of the AskTabib mobile client: stakeholders, concerns, viewpoints, views, and key decisions. It does not replace operational runbooks (Google Meet / FCM setup) or coding conventions.

**In scope:** client structure (feature modules, core), integration boundaries (Laravel API, Google OAuth / Calendar / Meet, FCM), cross-cutting concerns (session, role routing, outbound URL safety, push, on-device data, failure behavior).

**Out of scope:** Laravel internals (boundary only), store release processes, clinical content, legal compliance analysis (§12.2 states only the posture the client must support).

---

## 2. Framing: stakeholders, concerns, viewpoints

Entity of interest: the AskTabib Flutter client. Stakeholders: product, mobile engineering, backend, security, QA, operations.

### 2.1 Concerns

| ID | Concern |
|----|---------|
| C1 | Telehealth booking and consultation flow |
| C2 | Doctor operations (availability, finance, Google linking) |
| C3 | Secure joining of Google Meet sessions |
| C4 | Reliability of push delivery |
| C5 | Protection of tokens, secrets, and sensitive data on device |
| C6 | Evolvability and testability |
| C7 | Failure behavior (offline, expired sessions, unlinked accounts) |

### 2.2 Viewpoint ↔ concern correspondence

| Viewpoint | Section / diagram | Concerns |
|-----------|-------------------|----------|
| Context | §6.1 / D1 | C1, C2, C3 |
| Container | §6.2 / D2 | C1, C4, C5 |
| Component (modules) | §6.3 / D3 | C6 |
| Layering | §5 / D4 | C6 |
| Dynamic (Meet) | §7.3 / D5 | C2, C3, C4, C7 |
| Dynamic (push) | §7.4 / D6 | C3, C4 |
| Trust / security | §11 / D7 | C5 |
| Data | §12 / D9 | C5, C7 |
| Behavioral (roles) | §7.2 / D8 | C1, C2 |
| Failure | §7.5 | C7 |

**Correspondence rules:** (1) every external system in D1 appears in D2; (2) every module in D3 maps to a folder in §10; (3) every trust boundary in §11 appears in D7; (4) any ADR that changes structure updates the matching diagram in the same PR.

---

## 3. System identity and product context

| Role | Primary capabilities |
|------|----------------------|
| **Patient** | Discover doctors, book consultations, manage appointments, medical tips, join Meet when the session opens |
| **Doctor** | Home shell, availability, appointments (accept), finance/settlement, profile, connect Google Calendar/Meet, complete consultations |

Role selection after authentication is centralized (`role_home_route` / `AppRouter` guards). Doctor shell routes require a doctor role and a valid session token on the client; **server RBAC remains authoritative**.

| System | Responsibility |
|--------|----------------|
| Laravel API (`asktabib.com`) | Auth (Sanctum), appointments, doctor APIs, Google token exchange, Meet link creation, join-window enforcement, FCM dispatch, rate limiting |
| Google Cloud OAuth project | Native Sign-In + server-side code exchange |
| Firebase project | Device registration + FCM delivery |
| Google Calendar / Meet | Hosted video sessions |

---

## 4. Stakeholder concerns

| Stakeholder | Primary concerns | Architecture response |
|-------------|------------------|----------------------|
| Product | Stable booking + consultation UX | Feature modules; role-aware navigation |
| Mobile engineers | Predictable structure, low coupling | Dependency rule; GetIt DI |
| Backend engineers | Clear API contracts | `DioHelper`; repositories as anti-corruption layer |
| Security | Token theft, secret leakage, phishing URLs | Secure storage; secrets off-device; `SafeExternalUrl` |
| QA | Testable units, smoke paths | Domain isolation; §8.2 |
| Operations | Push and Meet reliability | Central FCM handlers; documented setup |

---

## 5. Architectural style

### 5.1 Feature-first Clean Architecture

```text
Presentation  →  Domain  ←  Data
     (UI/Cubit)   (entities, use cases, repo ports)
                  Data implements repository ports (Dio, models, Google SDK)
```

| Layer | Responsibility | Must not |
|-------|----------------|----------|
| **Presentation** | Widgets, pages, Cubit/state, localization helpers | Call Dio / parse JSON / hold OAuth secrets |
| **Domain** | Entities, abstract repositories, use cases | Depend on Flutter UI, Dio, Firebase SDKs |
| **Data** | Models, remote/local data sources, repository implementations | Leak into widgets |

**Reference module:** `lib/features/google_meet/`.

### 5.2 Cross-cutting `core/` and isolation

`lib/core/` holds infrastructure shared by two or more features: networking (`DioHelper`), config, theme, routing, FCM / local notifications / push payload and navigation, shared widgets, security utilities (`SafeExternalUrl`), age/password policies. Feature business rules do not belong in `core/`.

**Isolation rule:** features do not import each other's internals. **Documented exception (ADR-08):** `google_meet` uses `auth`'s `GetTokenUseCase`.

### 5.3 State and composition

| Concern | Choice | Rationale |
|---------|--------|-----------|
| UI state | Cubit (`flutter_bloc`) | Explicit status enums; immutable `copyWith` |
| DI | GetIt (`sl` in `injection_container.dart`) | Single composition root |
| Errors | `dartz` `Either<Failure, T>` | Predictable failure mapping at repository boundary |
| Localization | ARB (`en` / `ar` / `de`) | No hard-coded user strings in new UI |
| Design sizing | `flutter_screenutil` (390×844) | Consistent responsive baseline |

---

## 6. Logical building blocks (C4 mapping)

### 6.1 C4 Level 1 — System context
**People:** Patient, Doctor. **Systems:** AskTabib App, Backend, Google Identity/Calendar/Meet, Firebase/FCM.
→ [`ARCHITECTURE_DIAGRAMS.md` § D1](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d1-c4-system-context)

### 6.2 C4 Level 2 — Containers

| Container | Technology | Role |
|-----------|------------|------|
| Flutter UI shell | Flutter / Material | Screens, drawers, overlays |
| Application services | Cubits + use cases | Orchestration |
| Secure session store | FlutterSecureStorage | Sanctum token |
| Push stack | Firebase Messaging + local notifications | Token, foreground/background, meeting overlay |
| Google Sign-In SDK | `google_sign_in` | Obtain `serverAuthCode` only |
| HTTP client | Dio | REST to Laravel |

→ [`ARCHITECTURE_DIAGRAMS.md` § D2](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d2-c4-container)

### 6.3 C4 Level 3 — Components

| Module | Domain |
|--------|--------|
| `auth` | Login, signup, OTP, session, logout |
| `home` | Patient home, booking, profile, tips, appointments |
| `doctor` | Doctor shell: appointments, availability, finance, profile |
| `notifications` | In-app notification list |
| `google_meet` | Google Calendar / Meet linking |
| `debug` | Push diagnostics (debug builds only) |

Shared components in `core/`: router, Dio, FCM pipeline, URL safety, theme. The composition root (`injection_container.dart`) is a wiring file, not a component, and is omitted from D3.
→ [`ARCHITECTURE_DIAGRAMS.md` § D3](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d3-c4-component-flutter-client)

### 6.4 Per-feature slice
Canonical vertical slice and dependency rule: [`ARCHITECTURE_DIAGRAMS.md` § D4](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d4-clean-architecture-dependency-rule)

---

## 7. Runtime and integration views

### 7.1 Bootstrap sequence
1. Binding + `.env` load
2. System UI config
3. Firebase initialize
4. Register background FCM handler
5. GetIt `init()`
6. FCM + push handler initialize
7. `runApp(AskTabibApp)`

Push is wired after DI; Firebase before messaging.

### 7.2 Authentication and role home
1. Auth use cases obtain the Sanctum token → secure storage.
2. Role selects patient home (`/home`) or doctor home (`/doctor-home`).
3. Router guards refuse the doctor shell without doctor role + token.
4. Server enforces RBAC on every privileged API.

→ [`ARCHITECTURE_DIAGRAMS.md` § D8](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d8-dual-role-navigation-supporting)

### 7.3 Google Meet capability (critical path)
Meet is not embedded WebRTC.

1. **Linking (doctor):** native OAuth → `serverAuthCode` → `POST /google/mobile-callback` → Laravel stores Google tokens.
2. **Provisioning (backend):** triggered when the **doctor accepts the appointment**. Laravel creates the Calendar event with Meet conference data and stores the link server-side.
3. **Join window:** the appointment is *confirmed* on acceptance, but **joining is not possible until the scheduled start time**. Before that, the client shows a disabled Join action with the scheduled time. The server is authoritative: it releases `meet_link` (in the API response and in the push payload) only when the appointment is confirmed and the scheduled time has been reached.
4. **Delivery:** FCM `meeting_link_ready` + `meet_link` at the join window.
5. **Join:** allowlisted open of `meet.google.com`.

Client secret and token exchange stay only on Laravel.
→ [`ARCHITECTURE_DIAGRAMS.md` § D5](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d5-google-meet-end-to-end-mechanism-primary)

### 7.4 Push notification pipeline

| Component | Role |
|-----------|------|
| `FcmService` | Permission, token, refresh |
| `fcm_background_handler` | Isolate entry for background |
| `PushNotificationPayload` | Typed parsing of data keys |
| `PushNotificationHandler` | Foreground / opened routing |
| `LocalNotificationService` | Local display (foreground) |
| Meeting overlay / call controller | Full-screen "join Meet" experience |

**Message type (standard):** notification + data message, high priority. Background and terminated delivery is displayed by the OS (reliable on Android and iOS, where data-only messages are throttled); the data keys carry routing. Foreground messages are handled by `onMessage` and shown via local notification. Payload text is generic (no medical detail) and the link is never trusted without the allowlist check.
→ [`ARCHITECTURE_DIAGRAMS.md` § D6](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d6-fcm-push-and-meet-join)

### 7.5 Failure and offline behavior (standard)

| Situation | Client behavior |
|-----------|-----------------|
| **401 Unauthorized** | Clear secure storage and cached data, route to login with a "session expired" message. No silent retry loop. |
| **Network unavailable** | Connectivity detected; non-blocking offline banner. Failed actions show an error state with **Retry**. The session is **never** cleared because of a network error. |
| **Timeouts** | Connect/receive timeouts set on Dio. Idempotent GETs retried with exponential backoff (limited attempts). Writes (booking, accept, finance) are **not** auto-retried or queued; the user is told the action did not complete. |
| **Offline cached data** | Read-only display of already-loaded lists where available, clearly marked as possibly outdated. Sensitive/medical content is not persisted for offline use (§12). |
| **5xx / server error** | Generic localized error + Retry. No raw server messages shown. |
| **429 rate limited** | Message to wait; control is server-side. |
| **Join before the scheduled time** | Join disabled with the scheduled time; if the API is called early the server rejects and the client shows the same message. |
| **Doctor not linked to Google at acceptance** | Server refuses the acceptance with a specific error code; the doctor is prompted to connect Google Calendar first. |
| **Google refresh token revoked/expired** | Server marks the link invalid, the doctor is notified and prompted to re-link; affected appointments are flagged. |
| **Push permission denied** | Fallback to the in-app notification list and the appointment details screen, where Join is available at the scheduled time. |
| **Meet URL fails allowlist** | Rejected, not launched, generic error. |

---

## 8. Quality attributes and constraints

### 8.1 Targets

| Attribute | Target posture | Mechanisms |
|-----------|----------------|------------|
| **Security** | Target OWASP MASVS L1 (formal self-assessment pending) | Secure token storage; no OAuth secret in APK; Meet/media URL allowlists; role route gates; debug UI gated by `kDebugMode` |
| **Modularity** | Feature isolation | Feature folders; dependency rule; GetIt |
| **Reliability** | Push + Meet join resilient | Background handler; payload contract; WebView fallback for auth URL when returned |
| **Maintainability** | One pattern for new features | Template = `google_meet`; engineering guide |
| **Observability (dev)** | Debuggable push/Meet | Push debug page / logs in debug builds |
| **i18n** | AR / EN / DE | ARB + message-key localizers |
| **Performance** | Cold start ≤ 3 s and scrolling without sustained dropped frames on a mid-range reference device | Image cache discipline; shimmers; list patterns |

### 8.2 Test strategy (client)

| Level | Scope |
|-------|-------|
| Unit | Use cases, repositories (mocked data sources), payload parsing, `SafeExternalUrl`, join-window state logic |
| Cubit | State transitions per feature, including error/offline states |
| Widget / smoke | Login, role routing, appointment list, disabled Join before the scheduled time |
| Not automated | FCM delivery, Google Sign-In consent, actual Meet join (manual release checklist maintained under `docs/`) |

### 8.3 Hard constraints
- OAuth Web `client_secret` never ships in Flutter.
- Meet hosts restricted (`meet.google.com`).
- Medical/report media hosts restricted to AskTabib hosts.
- Dual Google Cloud projects: do not duplicate Android `(package + SHA-1)` OAuth clients across projects.

---

## 9. Key architecture decisions (ADR summary)

| ID | Decision | Status | Rationale |
|----|----------|--------|-----------|
| ADR-01 | Feature-first Clean Architecture | Accepted | Scales dual-role product; testable domain |
| ADR-02 | Cubit over full Bloc | Accepted | Lower ceremony for request/response UI |
| ADR-03 | GetIt composition root | Accepted | Explicit graph; matches codebase |
| ADR-04 | Native `serverAuthCode` for Meet linking | Accepted | Secret stays server-side; native UX |
| ADR-05 | Meet via Calendar conference, not in-app WebRTC | Accepted | Faster delivery, Google reliability, lower media-stack cost |
| ADR-06 | Centralized push payload + navigation | Accepted | Prevents divergent deep-link bugs |
| ADR-07 | Client role gates + server RBAC | Accepted | Defense in depth |
| ADR-08 | `google_meet` depends on `auth` token use case | Accepted exception | Needs the session token. **Follow-up:** expose a session port in `core/` and remove the cross-feature import |
| ADR-09 | Server-authoritative join window; link released only at the scheduled time | Accepted | Client clocks and UI cannot be trusted; prevents early or shared-link access |

---

## 10. Repository layout (as-built)

```text
lib/
  main.dart                 # bootstrap only
  app.dart                  # MaterialApp, global providers
  injection_container.dart  # GetIt registrations
  firebase_options.dart
  core/                     # shared infrastructure
  features/
    auth/
    home/
    doctor/
    notifications/
    google_meet/
    debug/
  l10n/
docs/                       # architecture & ops AD artifacts
test/
android/  ios/
```

Canonical feature skeleton:

```text
lib/features/<feature>/
  data/{data_sources,models,repositories}/
  domain/{entities,repositories,use_cases}/
  presentation/{cubits,pages,widgets,helper}/
```

---

## 11. Trust boundaries

| Boundary | What crosses | Control |
|----------|--------------|---------|
| Device ↔ Laravel | HTTPS + Bearer Sanctum | TLS; secure storage; Dio interceptors; server rate limiting |
| App ↔ Google Sign-In | Interactive OAuth; `serverAuthCode` | Platform OAuth clients; `serverClientId` = Web client |
| Laravel ↔ Google | Code exchange; Calendar API | Web client secret in server `.env` only |
| Firebase ↔ Device | FCM payload | Project `asktabib-eccde`; token sync to backend |
| App ↔ Browser / Meet app | Meet URL launch | `SafeExternalUrl` host allowlist |

→ [`ARCHITECTURE_DIAGRAMS.md` § D7](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d7-trust-boundaries--secret-placement)

---

## 12. Data view

### 12.1 On-device data (standard)

| Data | Location | Sensitivity | Lifecycle |
|------|----------|-------------|-----------|
| Sanctum token | FlutterSecureStorage | High | Deleted on logout and on 401 |
| Role / basic profile | In memory, restored from the server at session start | Medium | Cleared on logout |
| Non-sensitive preferences (language, theme) | SharedPreferences | Low | Kept across logout |
| FCM token | In memory / managed by the SDK | Low | Refreshed; upserted to backend; removed on logout |
| Image cache | Disk cache | Medium | Cleared on logout; size-capped |
| API responses with health context | Not persisted to disk | High | Held in memory only |
| Push payloads | Transient | Medium | Not persisted; no medical detail |

→ [`ARCHITECTURE_DIAGRAMS.md` § D9](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d9-on-device-data)

### 12.2 Data protection
Users are located in Germany and Syria. For EU users the client must support a GDPR-compatible posture for health data: data minimization on device, no medical detail in push payloads or logs, and deletion of local data on logout. Legal basis, retention, and processor agreements are out of scope of this document and belong to the product/legal owner.

---

## 13. Risks and residual gaps

| Risk | Impact | Mitigation / residual |
|------|--------|------------------------|
| OAuth misconfiguration (package/SHA-1) | Sign-In appears "canceled" | Documented checklist; SHA-1 on OAuth project only |
| iOS Google + APNs incomplete | Meet link / push broken on iOS | Tracked in Meet/notifications guide |
| Crafted navigation bypasses route gates | UI exposure only | Server RBAC mandatory |
| Stale FCM token after reinstall | Missed notifications | Token refresh + backend upsert |
| Doctor not linked / Google token revoked | Meet cannot be provisioned | §7.5; acceptance refused when unlinked; re-link prompt |
| Notification missed or permission denied | Late join | In-app list and appointment details show Join at the scheduled time |
| Packaging identity drift | Wrong Firebase/OAuth clients | `com.asktabib.app` as source of truth |

---

## 14. Conformance and evolution

This AD is current when:
1. Feature modules obey Presentation → Domain ← Data (exceptions only via ADR).
2. New privileged integrations (OAuth, payments, media) keep secrets off-device.
3. Diagrams in [`ARCHITECTURE_DIAGRAMS.md`](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md) are updated for material structural change.

**Change control:** treat material architecture changes as ADRs (append to §9) and update the C4 / sequence diagrams in the same PR when feasible.

---

**Owner:** Mobile Tech Lead. **Review cadence:** on major feature modules, auth/Meet/FCM changes, or packaging identity changes.

