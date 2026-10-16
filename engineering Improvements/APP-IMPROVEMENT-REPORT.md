# AskTabib — Security, Performance & Architecture Improvement Report

**Document type:** Engineering change report  
**Scope:** Client application improvements delivered in the recent hardening / optimization cycle  
**Audience:** Engineering leads, reviewers, release stakeholders  
**Language:** English  
**Application identity:** `com.asktabib.app` (Flutter package `asktabib_patient`)  
**Related docs:** [`ARCHITECTURE_REPORT.md`](./ARCHITECTURE_REPORT.md), [`ARCHITECTURE_DIAGRAMS.md`](./ARCHITECTURE_DIAGRAMS.md), [`ENGINEERING_BEST_PRACTICES.md`](./ENGINEERING_BEST_PRACTICES.md), [`GOOGLE_MEET_AND_NOTIFICATIONS.md`](./GOOGLE_MEET_AND_NOTIFICATIONS.md)

---

## 1. Executive summary

This report documents **only** the security, performance, and structural improvements that were intentionally implemented. Each change is recorded in a **Before → Reason → Result** format so reviewers can audit intent, risk reduction, and residual gaps.

The work targeted a dual-role telehealth Flutter client (patient + doctor) approaching store readiness. Shared infrastructure (auth, Dio, FCM, Meet) was hardened first; doctor-shell access control and outbound URL safety were tightened next; appearance-safe performance work reduced memory and list cost without redesigning UI.

| Domain | Pre-change posture | Post-change posture |
|--------|--------------------|---------------------|
| Security | Weak → Moderate | **Moderate+ (~7/10)** |
| Performance | Fair (~5–6/10) | **Good− (~7/10)** |
| Architecture / identity | Dual-role friction (`*.doctor` packaging patient UX) | **Neutral app id `com.asktabib.app`**; client route gates added |

---

## 2. Security improvements

### 2.1 Secrets and session storage

| | |
|--|--|
| **Before** | API guest key lived in source / loose config patterns; session tokens were not consistently treated as high-sensitivity secrets. |
| **Reason** | Mobile binaries are reverse-engineerable. Secrets in source increase leak surface; tokens on plain storage increase session theft risk on rooted / shared devices. |
| **Result** | API key loaded via `.env` / `--dart-define`; auth token stored with **FlutterSecureStorage** and hydrated at startup; debug-only verbose Dio logging. |

**Primary touchpoints:** `lib/core/config/app_config.dart`, `lib/features/auth/data/data_sources/auth_local_datsource.dart`, `lib/core/dio/dio_helper.dart`, `lib/main.dart`

---

### 2.2 Outbound URL allowlisting (Meet + medical reports)

| | |
|--|--|
| **Before** | Google Meet join used a host allowlist helper in some flows; doctor medical-report open used raw `launchUrl` on arbitrary absolute/relative URLs. |
| **Reason** | FCM and API payloads can be tainted. Opening untrusted URLs enables phishing and unexpected file hosts—especially sensitive for clinical attachments. |
| **Result** | `SafeExternalUrl` extended with `openAppMediaLink` / `isAllowedAppMediaLink` (AskTabib hosts only). Doctor appointment report open goes through the allowlist. Meet links remain restricted to `meet.google.com`. |

**Primary touchpoints:** `lib/core/utils/safe_external_url.dart`, `lib/features/doctor/presentation/pages/doctor_appointments_screen.dart`, `lib/core/services/push_notification_navigation.dart`

---

### 2.3 Role-based client route guards (doctor shell)

| | |
|--|--|
| **Before** | Post-login routing used role for *initial* home selection, but `AppRouter` opened `/doctor-home` without verifying role/token. FCM doctor appointment events could `pushNamedAndRemoveUntil` into the doctor shell without a role check. |
| **Reason** | Client UI is not a security boundary, but ungated doctor navigation creates unauthorized UI exposure, confused sessions, and easier abuse if a patient notification or deep navigation reaches doctor routes. Server RBAC remains mandatory. |
| **Result** | `canAccessDoctorHome(role, token)` gates doctor home. Non-doctors are redirected to `/home` or `/start`. FCM doctor *appointment* navigation refuses unless the cached session is a doctor with a token. **Incoming consultation call / full-screen Meet ringtone + Join for both roles was preserved intentionally.** |

**Primary touchpoints:** `lib/core/navigation/role_home_route.dart`, `lib/core/router/app_router.dart`, `lib/core/services/push_notification_navigation.dart`

---

### 2.4 Google Meet connect diagnostics in production

| | |
|--|--|
| **Before** | Connect Google Meet UI showed full API JSON, `auth_url`, status codes, and clipboard copy in all build modes (added earlier for debugging). |
| **Reason** | OAuth URLs and raw responses can leak configuration and session-adjacent material; end users do not need diagnostic payloads in release builds. |
| **Result** | Detailed response dialogs run only under `kDebugMode`. Release shows localized SnackBar messages and continues the WebView auth flow when an auth URL is present. |

**Primary touchpoints:** `lib/features/google_meet/presentation/widgets/connect_google_button.dart`

---

### 2.5 Password policy and upload limits

| | |
|--|--|
| **Before** | Weak / unclear client password rules; medical report attachments lacked a clear client size bound in the booking path. |
| **Reason** | Weak passwords increase account takeover risk. Unbounded uploads stress backends and increase abuse cost. |
| **Result** | Client `PasswordPolicy` (minimum length + letter + digit) with UX helper text; booking medical report limited to **3MB**. |

**Primary touchpoints:** `lib/core/utils/password_policy.dart`, auth field helpers, `lib/features/home/presentation/pages/book_consultation.dart`

---

### 2.6 Application identity alignment

| | |
|--|--|
| **Before** | Store / native IDs used `com.asktabib.doctor` while the product UX and Flutter package centered on patient + dual-role flows. |
| **Reason** | Package / listing mismatch confuses store review, OAuth clients, Firebase apps, and users. |
| **Result** | Native identity moved to **`com.asktabib.app`** (Android `applicationId` / `namespace`, iOS/macOS bundle IDs, Firebase Android app id selection, Meet expected-package hint). Operators must keep Google OAuth Android clients (package + SHA-1) and Firebase apps in sync. |

**Primary touchpoints:** `android/app/build.gradle.kts`, `MainActivity` package path, `ios/Runner*` / `macos`, `lib/firebase_options.dart`, `android/app/google-services.json`

---

### 2.7 Security residual risks (not closed in this cycle)

- Certificate pinning not implemented  
- Release minify / obfuscate not mandated in this report’s delivered set  
- Server-side RBAC remains the authoritative control for `/doctor/*`  
- iOS Firebase / OAuth console registration for `com.asktabib.app` must be verified operationally  
- Accessibility and comprehensive automated E2E are outside this security section  

---

## 3. Performance improvements

All items below were selected as **appearance-safe**: same visual design, lower runtime cost.

### 3.1 Network image decode sizing

| | |
|--|--|
| **Before** | `Image.network` / similar paths decoded full-resolution assets into memory for small UI slots (avatars, banners, tips header). |
| **Reason** | Unnecessary decode width/height inflates RAM and jank on mid-tier devices. |
| **Result** | `cacheWidth` / `cacheHeight` (and helpers) applied to doctor avatars, slider banners, and tips header based on display size / DPR. |

**Primary touchpoints:** `lib/core/utils/image_cache_utils.dart`, `doctor_item.dart`, `slider_item.dart`, `advise.dart`

---

### 3.2 List identity keys

| | |
|--|--|
| **Before** | Several doctor / appointment / notification lists lacked stable `ValueKey`s. |
| **Reason** | Without stable keys, Flutter rebuilds and state attachment are less efficient and more error-prone during scroll updates. |
| **Result** | `ValueKey`s keyed by entity id on major list items (doctors, appointments, notifications, tip items/sections). |

---

### 3.3 SVG network memory reuse

| | |
|--|--|
| **Before** | Specialty / remote SVGs could be re-fetched and re-parsed across rebuilds. |
| **Reason** | Specialty grids rebuild often; repeated network + parse work shows up as scroll cost. |
| **Result** | In-memory SVG bytes cache in `AppSvg` for network loads. |

**Primary touchpoints:** `lib/core/widgets/app_svg.dart`

---

### 3.4 Startup: deferred FCM initialization

| | |
|--|--|
| **Before** | Push / FCM initialization contended with first-frame UI work. |
| **Reason** | Cold start latency is dominated by work before the first interactive frame. |
| **Result** | `runApp` first; FCM / notification wiring deferred after the first frame. Call ringtone + full-screen join behavior retained. |

**Primary touchpoints:** `lib/main.dart`, push services

---

### 3.5 Dio timeouts and safe retries

| | |
|--|--|
| **Before** | Long timeouts (e.g. 60s); retry behavior risked duplicating non-idempotent POSTs. |
| **Reason** | Users wait too long on dead networks; retrying booking/OTP POSTs can create duplicate server side-effects. |
| **Result** | ~**20s** timeouts; automatic retries limited to **GET** (idempotent reads) with capped attempts. |

**Primary touchpoints:** `lib/core/dio/dio_helper.dart`

---

### 3.6 Doctor search pagination

| | |
|--|--|
| **Before** | Search results tended to load as a single unbounded client list despite API `page` / `per_page` / meta support. |
| **Reason** | Large result sets increase memory, parse time, and first-paint cost. |
| **Result** | Client search uses pagination + load-more on scroll; meta `page` / `per_page` wired through models and cubit state. |

**Primary touchpoints:** `home_remote_datasource.dart`, doctor search models/entities, `home_cubit.dart`, `all_doctors.dart`

---

### 3.7 Tips screen nested-scroll fix

| | |
|--|--|
| **Before** | Advise/tips used nested scrollables (`SingleChildScrollView` + shrink-wrapped `ListView`), building tip rows eagerly. |
| **Reason** | Nested scroll + full list build is a classic jank/memory pattern on content-heavy screens. |
| **Result** | Refactored to `CustomScrollView` + lazy `SliverList.separated` with tip/section keys; visual layout preserved. |

**Primary touchpoints:** `lib/features/home/presentation/pages/advise.dart`

---

### 3.8 Doctor list entrance animation removal (cost only)

| | |
|--|--|
| **Before** | Per-row `TweenAnimationBuilder` on doctor cards during scroll. |
| **Reason** | Per-item animation during scroll increases frame cost without changing steady-state design. |
| **Result** | Entrance animation removed from the doctor list path; card appearance unchanged at rest. |

---

### 3.9 Performance residual gaps

- Patient appointments / notifications pagination still weaker than doctor lists in places  
- Large asset compression and lazy tab construction remain optional follow-ups  
- Doctor profile image decode sizing and doctor list keys still have room to match patient-side parity  

---

## 4. Architecture / structural improvements (delivered)

This section covers structural changes that were **actually shipped**, not the full backlog (e.g. splitting the god `home` feature remains recommended but was not completed as a full refactor in this cycle).

### 4.1 Application ID / package alignment

See §2.6. This is both a product-identity and architecture packaging change: one binary, neutral application id, dual-role shells.

### 4.2 Centralized role home helpers

| | |
|--|--|
| **Before** | Role checks were informal / duplicated around login and splash. |
| **Reason** | Access rules for doctor shell must be a single reusable predicate to avoid drift between router and FCM. |
| **Result** | `isDoctorRole`, `canAccessDoctorHome`, `homeRouteForRole` live in `role_home_route.dart` and are reused by router + push navigation. |

### 4.3 Safe URL capability as shared core utility

| | |
|--|--|
| **Before** | Meet allowlisting existed; media/report open was ad hoc. |
| **Reason** | Outbound link policy belongs in `core`, not per-screen copy-paste. |
| **Result** | Expanded `SafeExternalUrl` API used by notifications and doctor appointments. |

### 4.4 Early medical disclaimer as shared presentation widget

| | |
|--|--|
| **Before** | Trust copy existed deep in profile / appointments / booking privacy, not on first patient surfaces. |
| **Reason** | Store and clinical trust require early disclosure that remote consults do not replace emergencies / visits. |
| **Result** | `EarlyMedicalDisclaimer` + l10n (`earlyMedicalDisclaimer`) on Start and Choose Specialities. *(Product/trust UX adjacent to architecture; included because it introduced a reusable core widget and ARB contract.)* |

---

## 5. Explicit non-goals of this cycle

The following were **discussed or recommended** but are **not** claimed as completed architecture rewrites in this report:

- Full split of `features/home` into smaller features  
- Modular GetIt registration per feature  
- go_router / typed route modules  
- Certificate pinning  
- Full Semantics / text-scale a11y program  
- Dual store listings / flavors (patient vs doctor apps)  

---

## 6. Verification notes for release engineering

1. Uninstall any previous install signed under `com.asktabib.doctor` before installing `com.asktabib.app`.  
2. Confirm Google Cloud Android OAuth clients use package `com.asktabib.app` + correct SHA-1 (debug and/or release).  
3. Confirm Firebase Android app for `com.asktabib.app` matches `google-services.json` / `firebase_options.dart`.  
4. Manually verify full-screen meeting call + ringtone + Join for **both** patient and doctor.  
5. Run automated suite: `flutter test` (see companion test report).  

---

## 7. Conclusion

The delivered work measurably improved **secret handling**, **session storage**, **doctor-shell gating**, **outbound URL safety**, **release hygiene for Meet diagnostics**, **startup/network/list performance**, and **application identity clarity**—without changing the product’s visual language.

Residual risk is concentrated in **server RBAC reliance**, **pinning/obfuscation**, **test depth beyond unit/widget smoke**, and **larger structural modularization** of the patient `home` surface. Those items should be tracked as follow-up epics, not assumed done by this report.

---

*End of report.*
