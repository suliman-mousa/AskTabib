# AskTabib Admin — Architecture Diagrams

| Field | Value |
|--------|--------|
| **System** | AskTabib Admin Dashboard |
| **Notation** | Mermaid (C4-inspired + sequence + deployment) |
| **Companion** | [architecture-report.md](../Web-Admin-Architecture.md) |
| **Date** | 16 October 2026 |
| **How to read** | Render in GitHub, VS Code Mermaid preview, or any Mermaid-capable viewer |

This pack holds the **highest-leverage** views only: context, containers, frontend structure, auth, FCM, HTTP contract, deployment, and CI. Detail text lives in the architecture report.

---

## D1 — System context (C4 Level 1)

Who uses the admin console, and which external systems it depends on.

```mermaid
C4Context
  title AskTabib Admin — System Context

  Person(admin, "Admin operator", "Germany / Syria ops")
  System(spa, "AskTabib Admin SPA", "Vite React dashboard at /dashboard/")
  System_Ext(api, "AskTabib API", "Laravel auth, RBAC, domain APIs")
  System_Ext(fcm, "Firebase Cloud Messaging", "Web push fan-out")
  System_Ext(pusher, "Pusher", "Optional Echo channels")

  Rel(admin, spa, "Uses", "HTTPS")
  Rel(spa, api, "REST Bearer + x-api-key", "HTTPS")
  Rel(spa, fcm, "Token + onMessage / SW", "HTTPS")
  Rel(api, fcm, "Triggers admin booking push", "Server")
  Rel(spa, pusher, "Optional realtime", "WSS")
```

If the C4 plugin is unavailable in your viewer, use this equivalent:

```mermaid
flowchart LR
  Admin["Admin operator"]
  SPA["AskTabib Admin SPA<br/>/dashboard/"]
  API["AskTabib API<br/>Laravel"]
  FCM["Firebase Cloud Messaging"]
  Pusher["Pusher / Echo<br/>optional"]

  Admin -->|HTTPS| SPA
  SPA -->|REST Bearer + x-api-key| API
  SPA -->|token · onMessage · SW| FCM
  API -->|triggers booking push| FCM
  SPA -.->|WSS optional| Pusher
```

---

## D2 — Container view (C4 Level 2)

Deployable and runtime units inside the admin boundary.

```mermaid
flowchart TB
  subgraph Browser["Browser runtime"]
    SPA["React SPA<br/>features · hooks · stores"]
    SW["Service Worker<br/>firebase-messaging-sw.js"]
    LS["localStorage<br/>accessToken · authUser · fcmToken"]
  end

  subgraph Edge["Delivery edge"]
    NGX["nginx:alpine<br/>/dashboard/ + CSP headers"]
  end

  subgraph Build["Build pipeline"]
    VITE["Vite build<br/>VITE_* baked in"]
    CI["GitHub Actions<br/>lint · test · e2e"]
  end

  API["AskTabib API"]
  FCM["FCM"]

  CI --> VITE --> NGX
  NGX -->|static assets| SPA
  SPA --> LS
  SPA <-->|postMessage / push| SW
  SPA -->|Axios| API
  SW -->|system notification| AdminUI["OS notification UI"]
  SPA --> FCM
  SW --> FCM
```

---

## D3 — Frontend layered structure

Internal layering of the SPA (strongest map for onboarding and reviews).

```mermaid
flowchart TB
  subgraph Presentation
    Pages["features/*/pages"]
    Layout["layouts/Sidebar · DashboardLayout"]
    SharedUI["components/* · guards"]
  end

  subgraph Application
    Hooks["hooks/useApi · useFcmListener · …"]
    Stores["store/use*Store Zustand"]
  end

  subgraph Infrastructure
    Axios["services/axiosInstance"]
    Endpoints["services/apiEndpoints"]
    FCMSvc["services/fcm · firebase"]
    Echo["services/echo"]
  end

  subgraph CrossCutting
    Auth["utils/auth · sessionCleanup"]
    TZ["utils/timezone"]
    I18n["app/i18n"]
    Secure["fcmPayload · loginThrottle · imageFile · safeUrl"]
  end

  Pages --> Hooks
  Pages --> Stores
  Layout --> SharedUI
  SharedUI --> Auth
  Hooks --> Axios
  Hooks --> FCMSvc
  Hooks --> Echo
  Axios --> Endpoints
  Axios --> Auth
  Axios --> TZ
  Axios --> I18n
  FCMSvc --> Secure
```

---

## D4 — Feature and route map

Domain modules and how they hang off the router.

```mermaid
flowchart LR
  App["App.jsx"] --> Routes["routes.jsx<br/>lazy + Suspense"]
  Routes --> Guest["GuestRoute /login"]
  Routes --> Prot["ProtectedRoute"]

  Guest --> AuthF["features/auth"]
  Prot --> Home["home"]
  Prot --> Appt["appointments"]
  Prot --> Docs["doctors"]
  Prot --> Spec["specialties"]
  Prot --> Tips["tips"]
  Prot --> Ads["ads"]
  Prot --> Ann["announcements"]
  Prot --> Fin["finance"]

  Home --> Shell["DashboardLayout"]
  Appt --> Shell
  Docs --> Shell
```

---

## D5 — Admin authentication sequence

The admission path that prevents non-admin 403 loops.

```mermaid
sequenceDiagram
  autonumber
  actor Admin
  participant Login as LoginPage
  participant API as AskTabib API
  participant Auth as auth.js
  participant Store as useUserStore
  participant Router as AppRoutes

  Admin->>Login: phone + password
  Login->>Login: loginThrottle check
  Login->>API: POST /api/login
  API-->>Login: token + role (+ user)
  alt role is not admin
    Login->>Login: reject · no persistence
    Login-->>Admin: admin-only error
  else role === admin
    Login->>Auth: establishSession(token, user)
    Auth->>Auth: localStorage accessToken + authUser
    Auth->>Store: setUser(profile)
    Login->>Router: navigate /home
    Router->>Auth: ProtectedRoute → isAuthenticated()
    Auth-->>Router: true
    Router-->>Admin: dashboard shell
  end
```

---

## D6 — FCM booking alert sequence (strongest realtime view)

Foreground and background paths for `admin_booking_received`.

```mermaid
sequenceDiagram
  autonumber
  participant API as AskTabib API
  participant FCM as Firebase FCM
  participant SW as Messaging SW
  participant SPA as Admin SPA
  participant UI as Toast · Bell · Appointments

  Note over SPA,SW: Bootstrap once per session
  SPA->>SW: register scope /dashboard/
  SPA->>FCM: getToken(VAPID)
  SPA->>API: POST /api/device/token

  API->>FCM: send admin_booking_received
  alt SPA in foreground
    FCM->>SPA: onMessage
    SPA->>SPA: sanitize / allowlist payload
    SPA->>UI: toast + bell + refresh list
  else SPA in background / closed tab
    FCM->>SW: push event
    SW->>SW: validate payload
    SW-->>AdminOS: system notification
    AdminOS->>SW: notificationclick
    SW->>SPA: open /appointments/new
  end
```

---

## D7 — HTTP request contract (interceptor pipeline)

Every authenticated call inherits the same headers and query contract.

```mermaid
flowchart LR
  Page["Page / useApi"] --> AX["axiosInstance"]
  AX --> R1["Attach Bearer"]
  R1 --> R2["Attach x-api-key"]
  R2 --> R3["params.lang = ar|en"]
  R3 --> R4["params.timezone =<br/>mapDeviceTimezone()"]
  R4 --> API["AskTabib API"]
  API -->|200 envelope| OK["useApi success path"]
  API -->|401| Clean["sessionCleanup → login"]
  API -->|403 / errors| Err["fieldErrors / message"]
```

Timezone mapping (inline decision table):

```mermaid
flowchart TD
  Dev["Intl device IANA zone"] --> Map{"prefix?"}
  Map -->|Europe/*| BER["Europe/Berlin"]
  Map -->|Asia/*| DAM["Asia/Damascus"]
  Map -->|other / empty| DEF["Asia/Damascus default"]
  BER --> Q["query timezone"]
  DAM --> Q
  DEF --> Q
```

---

## D8 — Deployment topology

From CI to browser edge.

```mermaid
flowchart TB
  subgraph CI["GitHub Actions"]
    A1["npm ci"]
    A2["lint · typecheck"]
    A3["vitest coverage"]
    A4["playwright e2e"]
    A1 --> A2 --> A3 --> A4
  end

  subgraph Docker["Docker multi-stage"]
    B1["node:20-alpine<br/>npm run build"]
    B2["nginx:alpine<br/>dist → /usr/share/nginx/html/dashboard"]
    B1 --> B2
  end

  subgraph Runtime["Production edge"]
    N["nginx :80"]
    N -->|302| D["/dashboard/"]
    D -->|SPA fallback| H["index.html"]
  end

  A4 -.->|release artifact / image| B1
  B2 --> N
  H --> Browser["Admin browser"]
  Browser --> API["AskTabib API"]
  Browser --> FCM["FCM"]
```

---

## D9 — Security trust boundaries

Where trust starts and stops (use in security reviews).

```mermaid
flowchart TB
  subgraph TB1["Trust boundary: Browser"]
    UI["UI gates · throttle · payload allowlist"]
    CSP["nginx CSP · nosniff · frame deny"]
  end

  subgraph TB2["Trust boundary: API"]
    AuthN["Authentication"]
    AuthZ["RBAC admin routes"]
    RL["Rate limits"]
  end

  subgraph TB3["Trust boundary: Firebase"]
    Push["Push delivery"]
    Pub["Public web config + VAPID only"]
  end

  UI -->|Bearer| AuthN
  AuthN --> AuthZ
  UI --> Push
  CSP -.-> UI
  Pub --> UI

  Note1["Never: Service Account in SPA"]
  Note2["Never: localStorage role as sole authz"]
```

---

## D10 — Quality pipeline

How confidence is produced before merge.

```mermaid
flowchart LR
  PR["Pull request"] --> QA["CI quality job"]
  QA --> Lint["ESLint"]
  QA --> TC["tsc --noEmit"]
  QA --> Unit["Vitest + MSW + coverage"]
  QA --> E2E["Playwright<br/>Chromium · Firefox · WebKit"]
  Unit --> Art1["coverage artifact"]
  E2E --> Art2["playwright-report"]
  Lint --> Gate{"all green?"}
  TC --> Gate
  Unit --> Gate
  E2E --> Gate
  Gate -->|yes| Merge["Merge-ready"]
  Gate -->|no| Fix["Fix · re-run"]
```

---

## Index — when to use which diagram

| ID | Use when… |
|----|-----------|
| **D1** | Explaining the product to stakeholders or new engineers |
| **D2** | Discussing hosting, SW, or env boundaries |
| **D3** | Code reviews / “where does this change go?” |
| **D4** | Adding a feature module or route |
| **D5** | Debugging login, role, or session bugs |
| **D6** | Debugging push / booking alert latency |
| **D7** | API contract or timezone / lang mismatches |
| **D8** | Docker, nginx, or release discussions |
| **D9** | Security review or threat modelling |
| **D10** | CI failures or test-strategy decisions |

---

## Maintenance

Update a diagram in the same PR that changes the behaviour it describes. Prefer editing Mermaid here over attaching binary images so the diagrams stay reviewable in git.
