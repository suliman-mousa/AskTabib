# AskTabib — Architecture Diagrams Report

| Field | Value |
|-------|--------|
| **Document type** | Architecture views (diagram pack) |
| **Companion AD** | [`ARCHITECTURE_REPORT.md`](../App-Architecture.md) |
| **Standards** | ISO/IEC/IEEE 42010 viewpoints · C4 model · UML-style sequence (Mermaid) |
| **Audience** | Architects, senior engineers, onboarding, design reviews |
| **Rendering** | Mermaid (GitHub / VS Code / most Markdown previewers) |

---

## How to use this pack

| Priority | Diagram | Why it is "strong" |
|----------|---------|---------------------|
| **P0 (primary)** | **D5 — Google Meet end-to-end** | Highest business-critical integration: OAuth, backend tokens, Calendar Meet, join window, FCM, safe join |
| P1 | D1 System Context | Onboarding & stakeholder alignment |
| P1 | D2 Containers | Deployment / integration clarity |
| P1 | D7 Trust boundaries | Security reviews |
| P1 | D9 On-device data | Data-lifecycle review |
| P2 | D3 Components | Module map |
| P2 | D4 Clean Architecture | Coding standard visual |
| P2 | D6 FCM → Meet join | Notification ops |
| P3 | D8 Role routing | Dual-role navigation |

---

## D1. C4 System Context

**Viewpoint:** System Context (C4 L1)
**Concern:** Who uses AskTabib and which external systems it depends on.

```mermaid
C4Context
title AskTabib — System Context

Person(patient, "Patient", "Books consultations and joins Meet")
Person(doctor, "Doctor", "Manages schedule and connects Google Meet")

System(app, "AskTabib Mobile App", "Flutter dual-role client\ncom.asktabib.app")

System_Ext(api, "AskTabib Backend", "Laravel API + Sanctum\nasktabib.com")
System_Ext(googleId, "Google Identity & Calendar", "OAuth + Calendar/Meet APIs")
System_Ext(fcm, "Firebase Cloud Messaging", "Push delivery")
System_Ext(meet, "Google Meet", "Hosted video consultation")

Rel(patient, app, "Uses")
Rel(doctor, app, "Uses")
Rel(app, api, "HTTPS REST + Bearer token")
Rel(app, googleId, "Native Sign-In (serverAuthCode)", "doctor linking")
Rel(api, googleId, "Code exchange + create Meet event")
Rel(api, fcm, "Send notification")
Rel(fcm, app, "Push payload")
Rel(app, meet, "Open allowlisted Meet URL")
Rel(patient, meet, "Joins consultation")
Rel(doctor, meet, "Joins consultation")
```

**Reading guide:** The app never holds the Google OAuth client secret. Meet media is hosted by Google; the app only links accounts and opens URLs.

---

## D2. C4 Container

**Viewpoint:** Container (C4 L2)
**Concern:** Major runtime units and technology choices.

```mermaid
C4Container
title AskTabib — Container View (Mobile + Integrations)

Person(user, "Patient / Doctor", "")

System_Boundary(mobile, "AskTabib Mobile Client") {
  Container(ui, "Flutter UI Shell", "Flutter / Material", "Screens, drawers, overlays")
  Container(appLogic, "Application Layer", "Cubit + UseCases", "Orchestration & state")
  Container(session, "Secure Session Store", "FlutterSecureStorage", "Sanctum token")
  Container(push, "Push Stack", "FCM + local notifications", "Token, handlers, Meet ringtone UX")
  Container(gsi, "Google Sign-In SDK", "google_sign_in", "Obtains serverAuthCode only")
  Container(http, "HTTP Client", "Dio / DioHelper", "REST to Laravel")
}

System_Ext(api, "Laravel API", "Auth, appointments, Google callback, FCM dispatch")
System_Ext(google, "Google OAuth + Calendar", "Token exchange & Meet conference")
System_Ext(firebase, "Firebase", "FCM project asktabib-eccde")
System_Ext(meet, "Google Meet", "Video session")

Rel(user, ui, "Interacts")
Rel(ui, appLogic, "Events / state")
Rel(appLogic, session, "Read/write token")
Rel(appLogic, http, "API calls")
Rel(appLogic, gsi, "Doctor Connect Google")
Rel(appLogic, push, "Init & handle")
Rel(http, api, "HTTPS")
Rel(gsi, google, "OAuth consent")
Rel(api, google, "Exchange code / Calendar")
Rel(api, firebase, "Send FCM")
Rel(firebase, push, "Message")
Rel(ui, meet, "launchUrl allowlisted")
```

---

## D3. C4 Component (Flutter client)

**Viewpoint:** Component (C4 L3)
**Concern:** Feature modules and shared core.

The composition root (`injection_container.dart`) wires all modules and is omitted for readability. The `google_meet → auth` edge is a documented exception (ADR-08).

```mermaid
C4Component
title AskTabib Flutter Client — Component View

Container_Boundary(app, "lib/") {
  Component(core, "core", "Shared infra", "Dio, router, FCM, SafeExternalUrl, theme")
  Component(auth, "features/auth", "Auth", "Login, OTP, session use cases")
  Component(home, "features/home", "Patient", "Booking, tips, profile, appointments")
  Component(doctor, "features/doctor", "Doctor shell", "Availability, appointments, finance, profile")
  Component(notif, "features/notifications", "In-app list", "Notification feed")
  Component(meetFeat, "features/google_meet", "Meet linking", "GoogleAuthCubit + OAuth DS")
}

Rel(auth, core, "Uses networking & storage helpers")
Rel(home, core, "Uses")
Rel(doctor, core, "Uses")
Rel(notif, core, "Uses")
Rel(meetFeat, core, "Uses Dio")
Rel(meetFeat, auth, "GetTokenUseCase (ADR-08 exception)")
```

---

## D4. Clean Architecture dependency rule

**Viewpoint:** Code / layering
**Concern:** Enforceable dependency direction for every feature.

```mermaid
flowchart TB
  subgraph Presentation
    UI[Pages / Widgets]
    CUB[Cubit + State]
  end

  subgraph Domain
    UC[Use Cases]
    ENT[Entities]
    REPO_P[Repository Ports]
  end

  subgraph Data
    REPO_I[Repository Impl]
    DS[Data Sources]
    M[Models]
  end

  EXT[(Dio / SDKs / Secure Storage)]

  UI --> CUB
  CUB --> UC
  UC --> REPO_P
  UC --> ENT
  REPO_I -.implements.-> REPO_P
  REPO_I --> DS
  REPO_I --> M
  M --> ENT
  DS --> EXT

  style Domain fill:#e8f5e9
  style Presentation fill:#e3f2fd
  style Data fill:#fff3e0
```

**Rule:** arrows point inward toward Domain. UI never imports Dio data sources directly.

---

## D5. Google Meet end-to-end mechanism (PRIMARY)

**Viewpoint:** Dynamic / sequence
**Concern:** How an accepted appointment becomes a joinable Google Meet: doctor linking, provisioning on acceptance, the time-gated join window, push, and safe open.

```mermaid
sequenceDiagram
  autonumber
  actor Doctor
  actor Patient
  participant App as AskTabib App
  participant GSI as Google Sign-In SDK
  participant API as Laravel API
  participant Google as Google OAuth + Calendar
  participant FCM as Firebase FCM
  participant Meet as Google Meet

  Note over Doctor,Google: Phase A — Doctor links Calendar / Meet (one-time / re-link)
  Doctor->>App: Tap Connect Google Calendar
  App->>GSI: initialize(serverClientId = Web Client ID)
  GSI->>Doctor: Consent (Calendar scopes)
  GSI-->>App: serverAuthCode
  App->>API: POST /google/mobile-callback<br/>Bearer Sanctum + { code }
  API->>API: Validate auth + doctor role
  API->>Google: Exchange code (Web client ID + secret)
  Google-->>API: access_token + refresh_token
  API-->>App: Success (optional auth_url)
  Note over API: Tokens stored server-side only

  Note over Doctor,Meet: Phase B — Doctor accepts appointment → backend provisions Meet
  Doctor->>App: Accept appointment
  App->>API: Accept request
  alt Doctor linked and token valid
    API->>Google: Create Calendar event + conferenceData
    Google-->>API: meet_link (stored server-side)
    API-->>App: Appointment confirmed (no link released yet)
  else Not linked / refresh token invalid
    API-->>App: Refused with error code → prompt to link / re-link
  end

  Note over Patient,Meet: Phase C — Join window opens at the scheduled time
  Patient->>App: Open appointment before scheduled time
  App-->>Patient: Join disabled (shows scheduled time)
  API->>FCM: At scheduled time: notification + data<br/>type=meeting_link_ready
  FCM->>App: Push (fg / bg / terminated)
  App->>API: Fetch appointment (server releases meet_link only if confirmed AND time reached)
  App->>App: SafeExternalUrl allowlist<br/>host = meet.google.com
  App->>Meet: launchUrl (external / browser / Meet app)
  Patient->>Meet: Consultation
  Doctor->>Meet: Consultation
```

### D5 — companion flowchart (control & trust)

```mermaid
flowchart LR
  subgraph Device["Mobile device"]
    A[Doctor UI] --> B[google_sign_in]
    B --> C[serverAuthCode]
    C --> D[Dio POST mobile-callback]
    E[FCM handler] --> F[PushNotificationPayload]
    F --> G{meet_link host OK?}
    G -->|yes| H[Open Meet]
    G -->|no| I[Reject / ignore]
  end

  subgraph Server["Laravel trust zone"]
    D --> J[RoleMiddleware doctor]
    J --> K[OAuth code exchange]
    K --> L[(Google tokens DB)]
    L --> M[Calendar create Meet on accept]
    M --> W{Confirmed AND scheduled time reached?}
    W -->|yes| N[Release link + FCM send]
    W -->|no| X[Withhold link]
  end

  subgraph GoogleCloud["Google"]
    K --> O[Token endpoint]
    M --> P[Calendar API]
    H --> Q[Meet media]
  end

  N --> E
```

### D5 — failure paths

| Failure | Where detected | Expected outcome |
|---------|----------------|------------------|
| Doctor never linked Google | Backend (acceptance) | Acceptance refused with error code; doctor prompted to connect Google first |
| Refresh token revoked/expired (`invalid_grant`) | Backend (Phase B) | Link marked invalid; doctor notified to re-link; appointment flagged |
| Join attempted before the scheduled time | Client + backend | Client disables Join; backend rejects early requests and withholds `meet_link` |
| Non-doctor calls callback | Backend | 403 |
| Push not delivered / permission denied | Client / FCM | Link available from appointment details and in-app notifications at the scheduled time |
| `meet_link` host not allowlisted | Client (`SafeExternalUrl`) | Rejected, not launched |
| Network error during join fetch | Client | Offline banner + Retry; session untouched |

**Invariants encoded by D5**

1. `serverAuthCode` is ephemeral; long-lived Google tokens live only on Laravel.
2. Non-doctor callers receive API **403**.
3. The server, not the client, decides when the join window opens (ADR-09).
4. Join path never trusts arbitrary URLs, only allowlisted Meet hosts.

---

## D6. FCM push and Meet join

**Viewpoint:** Dynamic (notifications)
**Concern:** Lifecycle of a meeting-ready push inside the app.

Message type: notification + data, high priority. Background/terminated display is handled by the OS; foreground by `onMessage`.

```mermaid
sequenceDiagram
  participant API as Laravel
  participant FCM as Firebase
  participant OS as OS notification tray
  participant PS as FcmService / Handler
  participant PL as PushNotificationPayload
  participant LOC as LocalNotificationService
  participant NAV as Push Navigation / Overlay
  participant SAFE as SafeExternalUrl

  API->>FCM: HTTP v1 send (device token)
  alt App terminated / background
    FCM->>OS: Notification + data (generic text)
    Note over OS,PS: User taps notification → app opens with data
  else App foreground
    FCM->>PS: onMessage
    PS->>PL: Parse type / meet_link
    PL->>LOC: Local alert if needed
  end
  PS->>NAV: Route by type
  NAV->>SAFE: Validate meet_link
  SAFE-->>NAV: Allowed → launch
```

**Typical data keys:** `type = meeting_link_ready`, `meet_link = https://meet.google.com/...`. On iOS, delivery depends on APNs configuration (tracked as incomplete).

---

## D7. Trust boundaries & secret placement

**Viewpoint:** Security / deployment
**Concern:** Where secrets and tokens are allowed to exist.

```mermaid
flowchart TB
  subgraph PublicClient["PUBLIC — Mobile APK/IPA"]
    PKG[applicationId com.asktabib.app]
    WEBID[Google Web Client ID as serverClientId]
    SANCTUM[Sanctum token in Secure Storage]
    FCMTOK[FCM device token]
  end

  subgraph PrivateServer["PRIVATE — Laravel"]
    SECRET[GOOGLE_CLIENT_SECRET]
    GTOK[Google access + refresh tokens]
    SVC[Firebase service account]
  end

  subgraph GoogleSide["Google / Firebase Cloud"]
    OAUTH[OAuth clients Android/iOS/Web]
    CAL[Calendar + Meet]
    FB[FCM project]
  end

  WEBID -.->|identify server for code| OAUTH
  SECRET -->|code exchange| OAUTH
  GTOK --> CAL
  SVC --> FB
  FCMTOK -->|registered via API| PrivateServer
  SANCTUM -->|Authorization Bearer| PrivateServer

  style SECRET fill:#ffcdd2
  style GTOK fill:#ffcdd2
  style SVC fill:#ffcdd2
  style WEBID fill:#fff9c4
  style SANCTUM fill:#fff9c4
```

| Artifact | Allowed on device? |
|----------|-------------------|
| Google Web Client ID (`serverClientId`) | Yes (public identifier) |
| Google Client Secret | **No** |
| Google refresh token | **No** |
| Sanctum bearer token | Yes (secure storage only) |
| Firebase service account JSON | **No** |

---

## D8. Dual-role navigation (supporting)

**Viewpoint:** Application behavior
**Concern:** Patient vs doctor shell after auth.

```mermaid
flowchart TD
  SPLASH[Splash / session hydrate] --> AUTH{Valid Sanctum token?}
  AUTH -->|no| START[/start / login/]
  AUTH -->|yes| ROLE{Role?}
  ROLE -->|patient| HOME[/home — patient shell/]
  ROLE -->|doctor| GATE{canAccessDoctorHome?}
  GATE -->|yes| DHOME[/doctor-home/]
  GATE -->|no| HOME
  DHOME --> MEETUI[Connect Google Meet]
  HOME --> BOOK[Booking + appointments]
  NOTE[FCM meeting_link_ready] --> JOIN[Join Meet UX — both roles, at scheduled time]
  JOIN --> SAFE[SafeExternalUrl]
```

Client gates are UX only; the server enforces RBAC.

---

## D9. On-device data

**Viewpoint:** Data
**Concern:** What is stored on the device and when it is removed.

```mermaid
flowchart LR
  subgraph Device["On device"]
    SS[(Secure Storage<br/>Sanctum token)]
    MEM[(Memory only<br/>role, profile, API responses<br/>with health context)]
    CACHE[(Disk cache<br/>images, size-capped)]
    PREF[(SharedPreferences<br/>language, theme)]
    FT[FCM token]
  end
  LOGOUT[Logout / 401] -->|clears| SS
  LOGOUT -->|clears| MEM
  LOGOUT -->|clears| CACHE
  LOGOUT -->|removes| FT
  FT -->|upsert| API[Laravel API]
  PUSH[Push payload] -.->|generic text, no medical detail,<br/>not persisted| Device
```

---

## Diagram index (quick reference)

| ID | Title | C4 / kind | Priority |
|----|-------|-----------|----------|
| D1 | System Context | C4 L1 | P1 |
| D2 | Containers | C4 L2 | P1 |
| D3 | Flutter components | C4 L3 | P2 |
| D4 | Clean Architecture layers | Structural | P2 |
| **D5** | **Google Meet end-to-end** | **Sequence + control** | **P0** |
| D6 | FCM → Meet join | Sequence | P2 |
| D7 | Trust & secrets | Security | P1 |
| D8 | Role routing | Behavioral | P3 |
| D9 | On-device data | Data | P1 |

---

## Maintenance rule

When any of the following change, update **D5** (and usually D1/D2/D7/D9) in the same change set:

- OAuth callback path or payload
- Meet provisioning ownership or trigger, or the join-window rule
- FCM meeting payload contract or message type
- Allowlisted hosts for Meet / media
- Application id / OAuth client binding
- What is stored on the device

---
