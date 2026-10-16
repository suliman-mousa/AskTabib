# AskTabib — Automated Test Report

**Document type:** Test engineering report  
**Scope:** Automated tests introduced for critical client logic and smoke coverage  
**Audience:** Engineering leads, QA, release stakeholders  
**Language:** English  
**Command of record:** `flutter test`  
**Result of record:** **24 / 24 passed** (executed during delivery of this suite)

---

## 1. Executive summary

Prior to this work, the repository effectively had **no meaningful automated coverage** (default Flutter counter `widget_test` only). A focused suite was added to lock in the highest-risk logic from the recent security and trust work:

- Role-based doctor shell access rules  
- Outbound URL allowlisting (Meet + medical media)  
- Password policy  
- Early medical disclaimer localization (EN/AR)  
- A compact smoke pack for store-prep confidence  

This is **not** a full E2E / device lab program. It is a **professional baseline**: fast, deterministic unit + widget tests plus a smoke file that must stay green before release builds.

| Metric | Value |
|--------|--------|
| Total tests | 24 |
| Passed | 24 |
| Failed | 0 |
| Framework | `flutter_test` |
| External device required | No (for this suite) |

---

## 2. Testing strategy

### 2.1 What we chose to automate

| Layer | Why |
|-------|-----|
| **Unit** | Pure functions / policies with high security impact and zero UI flakiness |
| **Widget** | Verifies early medical disclaimer renders in EN and AR with real l10n delegates |
| **Smoke** | Tiny cross-cutting assertions that mirror “must not regress before store” |

### 2.2 What we intentionally left manual

| Area | Reason |
|------|--------|
| Google Sign-In / Meet OAuth | Requires real Google Cloud clients, SHA-1, and interactive UI |
| FCM + full-screen call + ringtone | Requires device notifications, audio, and OS permission state |
| Booking against live API | Needs authenticated backend + non-deterministic data |
| Release signing / Play App Signing SHA | Console / keystore operational checks |

Manual QA remains mandatory for those surfaces.

---

## 3. Test inventory

### 3.1 Unit — role routing & doctor access

**File:** `test/core/navigation/role_home_route_test.dart`  
**SUT:** `lib/core/navigation/role_home_route.dart`

| Case group | Assertions |
|------------|------------|
| `isDoctorRole` | Accepts `doctor` case-insensitively; rejects patient/guest/null/empty |
| `canAccessDoctorHome` | Doctor + token allowed; doctor without token blocked; patient with token blocked; null role blocked |
| `homeRouteForRole` | Doctor → `/doctor-home`; others → `/home` |

**Engineering value:** Guards the client policy behind Route Guard + FCM navigation against silent regressions.

---

### 3.2 Unit — SafeExternalUrl

**File:** `test/core/utils/safe_external_url_test.dart`  
**SUT:** `lib/core/utils/safe_external_url.dart`

| Case group | Assertions |
|------------|------------|
| Meet links | `meet.google.com` allowed; foreign hosts rejected; empty/invalid rejected |
| App media / reports | Absolute AskTabib hosts allowed; relative paths resolve to `https://asktabib.com/...`; foreign + Meet hosts rejected for media |
| Notification links | Meet + AskTabib allowed; unknown hosts rejected |

**Engineering value:** Prevents reintroduction of unrestricted `launchUrl` behavior for clinical attachments and notification deep links.

---

### 3.3 Unit — PasswordPolicy

**File:** `test/core/utils/password_policy_test.dart`  
**SUT:** `lib/core/utils/password_policy.dart`

| Case group | Assertions |
|------------|------------|
| Empty / short | Empty and below minimum length fail with correct enum |
| Composition | Letters-only and digits-only fail |
| Valid | Mixed letter+digit (≥8) accepted; Unicode letter + digit accepted |

**Engineering value:** Locks the client-side signup/reset strength contract (server must still enforce independently).

---

### 3.4 Widget — Early medical disclaimer

**File:** `test/core/widgets/early_medical_disclaimer_test.dart`  
**SUT:** `lib/core/widgets/early_medical_disclaimer.dart`

| Case | Assertions |
|------|------------|
| English locale | Finds EN disclaimer copy |
| Arabic locale | Finds AR disclaimer copy |

**Harness notes:** Uses `ScreenUtilInit`, `AppLocalizations` delegates, and `ThemeData` with `AppColors.light` extension—matching production theming/l10n dependencies without booting the full app graph (Firebase/GetIt).

**Engineering value:** Protects the early trust disclosure added on Start / specialties surfaces.

---

### 3.5 Smoke pack

**File:** `test/smoke_test.dart`

| Smoke check | Intent |
|-------------|--------|
| Role routing separation | Doctor and patient shells stay distinct; patient cannot pass `canAccessDoctorHome` |
| Media host allowlist | AskTabib allowed; evil host rejected |
| Password policy | Weak passwords rejected; strong accepted |

**Engineering value:** One file reviewers can skim; fails fast if a critical security invariant regresses.

---

## 4. Removed / replaced legacy test

| Before | After |
|--------|--------|
| `test/widget_test.dart` — default Flutter counter smoke (not wired to AskTabib) | Deleted and replaced by domain-relevant unit/widget/smoke tests |

---

## 5. How to run

From the project root:

```bash
flutter test
```

Run a single file:

```bash
flutter test test/smoke_test.dart
flutter test test/core/navigation/role_home_route_test.dart
```

Recommended CI gate (minimum):

```bash
flutter test --reporter expanded
```

---

## 6. Coverage interpretation (honest)

| Statement | Truth |
|-----------|--------|
| “We have automated tests now” | **Yes** — 24 focused tests |
| “Security logic for role + URLs + passwords is locked” | **Yes** at unit level |
| “Full app E2E is covered” | **No** |
| “Meet / FCM / OAuth are covered” | **No** — manual |
| “Store release is automatically proven safe” | **No** — tests reduce regression risk; they do not replace device/store checklists |

Approximate confidence gain:

- **High** for pure policy regressions (role, URL allowlist, password)  
- **Medium** for early disclaimer presence  
- **Low/None** for platform integrations until instrumented integration tests or device farms are added  

---

## 7. Recommended next test increments

Ordered by ROI for this codebase:

1. **Widget / router smoke** for `canAccessDoctorHome` redirect behavior with a fake `AuthLocalDataSource` (requires light DI test harness).  
2. **Unit tests** for `PushNotificationPayload` classification (`isIncomingConsultationCall` vs doctor appointment events)—protects call UI vs doctor-shell navigation split.  
3. **Golden or widget** smoke for Start page containing disclaimer + primary CTAs (asset-heavy; may need `TestWidgetsFlutterBinding` asset bundle setup).  
4. **Integration test** (optional) for login → role home with mocked Dio.  
5. Keep **manual protocol** for Meet join, full-screen intent, and OAuth after any package/SHA change.

---

## 8. Traceability to product risks

| Product / security risk | Test that mitigates regression |
|-------------------------|--------------------------------|
| Patient opens doctor shell | `role_home_route_test` + `smoke_test` |
| Malicious report URL open | `safe_external_url_test` + `smoke_test` |
| Weak passwords accepted on client | `password_policy_test` + `smoke_test` |
| Early medical disclaimer removed/lost in EN/AR | `early_medical_disclaimer_test` |
| Meet host policy loosened accidentally | `safe_external_url_test` (Meet cases) |

---

## 9. Conclusion

The delivered suite establishes a **professional, low-flake baseline** aligned with the security and trust changes already shipped. It is appropriate as a **pre-merge / pre-release gate** for policy logic, and insufficient alone as proof of end-to-end telehealth readiness.

Treat `flutter test` green as a **necessary** condition for release candidates, not a **sufficient** one: always complement with device verification of Meet, FCM call UI, and OAuth after identity or signing changes.

---

### Appendix A — File map

```
test/
  smoke_test.dart
  core/
    navigation/role_home_route_test.dart
    utils/safe_external_url_test.dart
    utils/password_policy_test.dart
    widgets/early_medical_disclaimer_test.dart
```

### Appendix B — Execution record

| Item | Value |
|------|--------|
| Command | `flutter test` |
| Outcome | All tests passed (`+24`) |
| Environment | Local Flutter SDK on Windows (developer workstation) |

---

*End of report.*
