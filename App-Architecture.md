# AskTabib — Software Architecture Description (SAD)

| Field | Value |
|-------|--------|
| **Document type** | Architecture Description (AD) |
| **Normative alignment** | [ISO/IEC/IEEE 42010:2022](https://www.iso.org/standard/74393.html) (concepts & structure) |
| **Visualization method** | [C4 model](https://c4model.com/) (Context → Container → Component) |
| **Internal style** | Feature-first Clean Architecture + Cubit |
| **System of interest** | AskTabib dual-role telehealth Flutter client |
| **Flutter package** | `asktabib_patient` |
| **Mobile application id** | `com.asktabib.app` |
| **API base** | `https://asktabib.com/api/` |
| **Audience** | Engineering leads, architects, senior developers, security reviewers |
| **Related docs** | [`ARCHITECTURE_DIAGRAMS.md`](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md),[`SECURITY_PERFORMANCE_ARCHITECTURE_IMPROVEMENTS.md`](./SECURITY_PERFORMANCE_ARCHITECTURE_IMPROVEMENTS.md) |

---

## 1. Purpose and scope

This Architecture Description (AD) expresses the **architecture of the AskTabib mobile client** as an engineered system: stakeholders, concerns, viewpoints, views, and key decisions. It does **not** replace operational runbooks (Google Meet / FCM setup) or coding conventions; those remain in the linked documents.

**In scope**

- Client-side structure (Flutter app, feature modules, core infrastructure)
- Integration boundaries with Laravel API, Google OAuth / Calendar / Meet, Firebase Cloud Messaging
- Cross-cutting concerns: auth session, role routing, outbound URL safety, push delivery
- Architectural qualities that govern evolution (modularity, security posture, testability)

**Out of scope**

- Full Laravel backend internal design (referenced only at the integration boundary)
- App Store / Play Console release processes
- Clinical protocol content

---

## 2. Normative framing (ISO/IEC/IEEE 42010)

Per ISO/IEC/IEEE 42010, an AD separates:

| Concept | Application in this document |
|---------|------------------------------|
| **Entity of interest** | AskTabib Flutter client (`com.asktabib.app`) |
| **Stakeholders** | Product, mobile engineering, backend, security, QA, operations |
| **Concerns** | Telehealth booking, doctor operations, secure Meet joining, reliability of push |
| **Viewpoints** | Context, Containers, Components (layers), Deployment / trust, Dynamic (runtime) |
| **Views** | Sections 4–8 below + diagrams in [`ARCHITECTURE_DIAGRAMS.md`](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md) |
| **Architecture decisions** | Section 9 (ADR-style rationale) |

Notation is intentionally **C4 + Mermaid**: notation-independent in spirit, tool-friendly for Git-hosted docs.

---

## 3. System identity and product context

AskTabib is a **dual-role telehealth client**:

| Role | Primary capabilities |
|------|----------------------|
| **Patient** | Discover doctors, book consultations, manage appointments, medical tips, join Meet when notified |
| **Doctor** | Home shell, availability, appointments, finance/settlement, self-profile, **connect Google Calendar/Meet**, complete consultations |

Role selection after authentication is centralized (`role_home_route` / `AppRouter` guards). Doctor shell routes require a doctor role **and** a valid session token on the client; **server RBAC remains authoritative**.

External systems that define the runtime landscape:

| System | Responsibility |
|--------|----------------|
| Laravel API (`asktabib.com`) | Auth (Sanctum), appointments, doctor APIs, Google token exchange & Meet link creation, FCM dispatch |
| Google Cloud OAuth project | Native Sign-In + server-side code exchange for Calendar / Meet |
| Firebase project | Device registration + FCM delivery |
| Google Meet / Calendar APIs | Hosted video sessions; Meet URLs consumed by the app via allowlisted launch |
---

## 4. Stakeholder concerns

| Stakeholder | Primary concerns | Architecture response |
|-------------|------------------|----------------------|
| Product | Stable booking + consultation UX | Feature modules per domain; role-aware navigation |
| Mobile engineers | Predictable structure, low coupling | Clean Architecture dependency rule; GetIt DI |
| Backend engineers | Clear API contracts | Dio via `DioHelper`; repositories as anti-corruption layer |
| Security | Token theft, OAuth secret leakage, phishing URLs | Secure storage; secrets off-device; `SafeExternalUrl` allowlists |
| QA | Testable units & smoke paths | Domain/use-case isolation; automated tests under `test/` |
| Operations | Push & Meet reliability | Centralized FCM handlers; documented Google/Firebase setup |

---

## 5. Architectural style

### 5.1 Feature-first Clean Architecture

The client follows **Clean Architecture** (dependency rule) organized by **feature**:

```text
Presentation  →  Domain  ←  Data
     (UI/Cubit)   (entities, use cases, repo ports)
                  Data implements repository ports (Dio, models, Google SDK)
```

| Layer | Responsibility | Must not |
|-------|----------------|----------|
| **Presentation** | Widgets, pages, Cubit/state, message localization helpers | Call Dio / parse JSON / hold OAuth secrets |
| **Domain** | Entities, abstract repositories, use cases | Depend on Flutter UI, Dio, Firebase SDKs |
| **Data** | Models, remote/local data sources, repository implementations | Leak into widgets |

**Reference module:** `lib/features/google_meet/` (canonical layout for new work).

### 5.2 Cross-cutting `core/`

`lib/core/` holds infrastructure shared by **two or more** features:

- Networking (`DioHelper`), config, theme, routing
- FCM / local notifications / push payload & navigation
- Shared widgets, security utilities (`SafeExternalUrl`), age/password policies

Feature business rules do **not** belong in `core/`.

### 5.3 State and composition

| Concern | Choice | Rationale |
|---------|--------|-----------|
| UI state | Cubit (`flutter_bloc`) | Explicit status enums; immutable `copyWith` |
| DI | GetIt (`sl` in `injection_container.dart`) | Single composition root; testable swaps |
| Errors | `dartz` `Either<Failure, T>` | Predictable failure mapping at repository boundary |
| Localization | ARB (`en` / `ar` / `de`) | No hard-coded user strings in new UI |
| Design sizing | `flutter_screenutil` (390×844) | Consistent responsive baseline |

---

## 6. Logical building blocks (C4 mapping)

### 6.1 C4 Level 1 — System context

**People:** Patient, Doctor  
**Software systems:** AskTabib App, AskTabib Backend, Google Identity/Calendar/Meet, Firebase/FCM  
**Relationships:** App authenticates and calls API; doctor links Google; backend creates Meet events; FCM notifies devices; app opens allowlisted Meet URLs.

→ Diagram: [`ARCHITECTURE_DIAGRAMS.md` § D1](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d1-c4-system-context)

### 6.2 C4 Level 2 — Containers

Inside the mobile client (logical containers / major runtime units):

| Container | Technology | Role |
|-----------|------------|------|
| Flutter UI shell | Flutter / Material | Screens, drawers, overlays |
| Application services | Cubits + use cases | Orchestration |
| Secure session store | FlutterSecureStorage | Sanctum token |
| Push stack | Firebase Messaging + local notifications | Token, foreground/background, meeting overlay |
| Google Sign-In SDK | `google_sign_in` | Obtain `serverAuthCode` only |
| HTTP client | Dio | REST to Laravel |

External containers: Laravel API, Google token endpoint, Google Calendar, FCM.

→ Diagram: [`ARCHITECTURE_DIAGRAMS.md` § D2](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d2-c4-container)

### 6.3 C4 Level 3 — Components (Flutter client)

Feature modules (current):

| Module | Domain |
|--------|--------|
| `auth` | Login, signup, OTP, session, logout |
| `home` | Patient home, booking, profile, tips, appointments |
| `doctor` | Doctor shell: appointments, availability, finance, profile |
| `notifications` | In-app notification list |
| `google_meet` | Google Calendar / Meet linking |
| `debug` | Push diagnostics (non-production oriented) |

Shared components in `core/`: router, Dio, FCM pipeline, URL safety, theme.

→ Diagram: [`ARCHITECTURE_DIAGRAMS.md` § D3](./ARCHITECTURE_DIAGRAMS.md#d3-c4-component-flutter-client)

### 6.4 Clean Architecture component (per feature)

Canonical vertical slice:

```text
pages / widgets
      ↓
   *Cubit
      ↓
  *UseCase
      ↓
*Repository (port)  ← implemented by →  *RepositoryImpl
                                              ↓
                                    *RemoteDataSource / *LocalDataSource
                                              ↓
                                         Dio / SDK / storage
```

→ Diagram: [`ARCHITECTURE_DIAGRAMS.md` § D4](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d4-clean-architecture-dependency-rule)

---

## 7. Runtime and integration views

### 7.1 Bootstrap sequence

Ordered startup (from `main.dart`):

1. Binding + `.env` load  
2. System UI config  
3. Firebase initialize  
4. Register background FCM handler  
5. GetIt `init()`  
6. FCM + push handler initialize  
7. `runApp(AskTabibApp)`

This order is intentional: push must be wired after DI, and Firebase before messaging.

### 7.2 Authentication & role home

1. Auth use cases obtain Sanctum token → secure storage.  
2. Role from session selects patient home (`/home`) or doctor home (`/doctor-home`).  
3. Router guards refuse doctor shell without doctor role + token.  
4. Server continues to enforce RBAC on every privileged API.

### 7.3 Google Meet capability (critical path)

Meet is **not** embedded WebRTC in-app. Architecture:

1. **Linking (doctor):** native OAuth → `serverAuthCode` → `POST /google/mobile-callback` → Laravel stores Google refresh/access tokens.  
2. **Provisioning (backend):** Calendar API creates event with Meet conference data.  
3. **Delivery:** FCM payload (`meeting_link_ready` + `meet_link`).  
4. **Join (patient/doctor):** allowlisted open of `meet.google.com` (+ local incoming-call UX when applicable).

Client secret and token exchange stay **only** on Laravel.  
→ Primary sequence diagram: [`ARCHITECTURE_DIAGRAMS.md` § D5](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d5-google-meet-end-to-end-mechanism-primary)  


### 7.4 Push notification pipeline

Centralized in `lib/core/services/`:

| Component | Role |
|-----------|------|
| `FcmService` | Permission, token, refresh |
| `fcm_background_handler` | Isolate entry for background |
| `PushNotificationPayload` | Typed parsing of data keys |
| `PushNotificationHandler` | Foreground / opened routing |
| `LocalNotificationService` | Local display |
| Meeting overlay / call controller | Full-screen “join Meet” experience |

→ Diagram: [`ARCHITECTURE_DIAGRAMS.md` § D6](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d6-fcm-push-and-meet-join)

---

## 8. Quality attributes and constraints

| Attribute | Target posture | Mechanisms |
|-----------|----------------|------------|
| **Security** | Moderate+ for client class | Secure token storage; no OAuth client secret in APK; Meet/media URL allowlists; role route gates; debug OAuth UI gated by `kDebugMode` |
| **Modularity** | Feature isolation | Feature folders; dependency rule; GetIt |
| **Reliability** | Push + Meet join resilient | Background handler; payload contract; fallback WebView for auth URL when returned |
| **Maintainability** | One pattern for new features | Template = `google_meet`; engineering guide |
| **Observability (dev)** | Debuggable push/Meet | Push debug page / logs in debug builds |
| **i18n** | AR / EN / DE | ARB + message-key localizers |
| **Performance** | Acceptable list/memory cost | Image cache discipline; shimmers; list patterns (see improvements report) |

**Hard constraints**

- OAuth Web `client_secret` never ships in Flutter.  
- Meet hosts restricted (`meet.google.com`).  
- Medical/report media hosts restricted to AskTabib hosts.  
- Dual Google Cloud projects: do not duplicate Android `(package + SHA-1)` OAuth clients across projects incorrectly.

---

## 9. Key architecture decisions (ADR summary)

| ID | Decision | Status | Rationale |
|----|----------|--------|-----------|
| ADR-01 | Feature-first Clean Architecture | Accepted | Scales dual-role product; enforces testable domain |
| ADR-02 | Cubit over full Bloc for most flows | Accepted | Lower ceremony; enough for request/response UI |
| ADR-03 | GetIt composition root | Accepted | Explicit graph; matches existing codebase |
| ADR-04 | Native `serverAuthCode` for Google Meet linking | Accepted | Keeps secret server-side; mobile-native UX |
| ADR-05 | Split Firebase (FCM) vs Google Cloud (OAuth) | Accepted | Separation of concerns; avoids misconfigured dual OAuth clients |
| ADR-06 | Meet via Google Calendar conference, not in-app WebRTC | Accepted | Faster delivery, Google reliability, lower media stack cost |
| ADR-07 | Centralized push payload + navigation | Accepted | Prevents divergent deep-link bugs |
| ADR-08 | Client role gates + server RBAC | Accepted | Defense in depth; UI not sole boundary |

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
| Device ↔ Laravel | HTTPS + Bearer Sanctum | TLS; token in secure storage; Dio interceptors |
| App ↔ Google Sign-In | Interactive OAuth; `serverAuthCode` | Platform OAuth clients; `serverClientId` = Web client |
| Laravel ↔ Google | Authorization code exchange; Calendar API | Web client secret in server `.env` only |
| Firebase ↔ Device | FCM data/notification | Project `asktabib-eccde`; token sync to backend |
| App ↔ Browser / Meet app | Meet URL launch | `SafeExternalUrl` host allowlist |

→ Diagram: [`ARCHITECTURE_DIAGRAMS.md` § D7](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md#d7-trust-boundaries--secret-placement)

---

## 12. Risks and residual gaps

| Risk | Impact | Mitigation / residual |
|------|--------|------------------------|
| OAuth misconfiguration (package/SHA-1) | Sign-In appears as “canceled” | Documented checklist; keep SHA-1 on OAuth project only |
| iOS Google + APNs incomplete | Meet link / push broken on iOS | Tracked in Meet/notifications guide |
| Client route gates bypassed by crafted navigation | UI exposure only | Server RBAC mandatory; keep gates maintained |
| Stale FCM token after reinstall | Missed Meet notifications | Token refresh + backend upsert |
| Dual-role packaging drift | Wrong Firebase/OAuth clients | Single `com.asktabib.app` applicationId as source of truth |

---

## 13. Conformance and evolution

**This AD is considered current when:**

1. Feature modules still obey Presentation → Domain ← Data.  
2. New privileged integrations (OAuth, payments, media) keep secrets off-device.  
3. Diagrams in [`ARCHITECTURE_DIAGRAMS.md`](./diagrams/APP_ARCHITECTURE_DIAGRAMS.md) are updated for material structural change.  


**Change control:** treat material architecture changes as ADRs (append to §9) and update the corresponding C4 / sequence diagrams in the same PR when feasible.

---

## 14. Document control

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2026-10-16 | Initial ISO 42010–aligned SAD for dual-role Flutter client |

**Authors / maintainers:** AskTabib mobile engineering  
**Review cadence:** on major feature modules, auth/Meet/FCM changes, or packaging identity changes
