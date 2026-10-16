# AskTabib Doctor — Google Meet Linking & Push Notifications Guide

This document explains how **Google Calendar / Google Meet linking** and **Firebase Cloud Messaging (FCM) notifications** work in the AskTabib Doctor Flutter app, which cloud projects/clients are used, the full runtime sequence, how to re-apply the setup in the future, and what must still be completed for **iOS**.

---

## 1. Architecture overview

| Concern | Approach |
|--------|----------|
| Google Meet / Calendar linking | Native mobile OAuth via `google_sign_in` → app obtains `serverAuthCode` → Laravel backend exchanges code for Google tokens |
| Push notifications | Firebase Cloud Messaging (`firebase_messaging`) |
| Web Google Sign-In (`serverAuthCode`) | **Not supported** in this app (`googleAuthWebUnsupported`) |

**Important split of cloud projects**

| Service | Cloud project | Why |
|--------|----------------|-----|
| Push notifications (FCM) | Firebase: **`asktabib-eccde`** | App registration + FCM delivery |
| Google OAuth (Calendar / Meet) | Google Cloud: **`linen-creek-491819-f4`** | Web + Android OAuth clients for token exchange |

The same Android app package is used by both.  
**Do not register the same `(package name + SHA-1)` Android OAuth client in two Google Cloud projects.**  
Firebase does **not** need the SHA-1 fingerprint for basic FCM. Keep SHA-1 only on the OAuth project (`linen-creek-491819-f4`).

---

## 2. App identifiers

| Platform | Identifier | Value |
|----------|------------|--------|
| Android `applicationId` / `namespace` | Package name | `com.asktabib.doctor` |
| iOS | Bundle ID | `com.asktabib.doctor` |
| API base URL | Backend | `https://asktabib.com/api/` |

Current debug signing SHA-1 (used while release builds still use the debug keystore):

```text
94:2E:43:E6:DE:87:21:71:6C:44:FF:4F:E4:F3:C3:0C:DA:5F:FC:7B
```

When you switch to a Play Store upload keystore, add that keystore’s **SHA-1** (and usually SHA-256) to the Google Cloud Android OAuth client as well.

---

## 3. Clients and projects in use

### 3.1 Firebase project (notifications)

| Field | Value |
|-------|--------|
| Project ID | `asktabib-eccde` |
| Project number | `905935166100` |
| Doctor Android app ID | `1:905935166100:android:4d23576b7a51ae77a53281` |
| Doctor package name | `com.asktabib.doctor` |
| Config file | `android/app/google-services.json` |
| Dart options | `lib/firebase_options.dart` |

Also present in the same `google-services.json` (other apps in the same Firebase project):

- `com.asktabib.app`
- `com.example.asktabib_doctor` (legacy; do not use for store builds)

### 3.2 Google Cloud project (OAuth / Meet)

| Field | Value |
|-------|--------|
| Project ID | `linen-creek-491819-f4` |
| Web OAuth Client ID (`serverClientId`) | `783425217776-h2khhelad60rjn20ll9mmemk0fjatfrv.apps.googleusercontent.com` |
| Web client secret | Backend only (Laravel `.env`) — **never** ship in Flutter |
| Backend redirect (web flow, if used) | `https://asktabib.com/api/google/callback` |
| Android OAuth client | Package `com.asktabib.doctor` + app signing SHA-1 |
| iOS OAuth client | Bundle ID `com.asktabib.doctor` (required for iOS Google Sign-In) |

### 3.3 Clients used by role

| Client type | Purpose | Where it lives |
|-------------|---------|----------------|
| Android OAuth client | Identifies the Android app to Google Sign-In (`package` + SHA-1) | Google Cloud `linen-creek-491819-f4` |
| iOS OAuth client | Identifies the iOS app (`bundle id`) | Google Cloud `linen-creek-491819-f4` |
| Web OAuth client | Used as `serverClientId` on mobile + backend code exchange | Google Cloud `linen-creek-491819-f4` |
| Firebase Android app | FCM registration for doctor app | Firebase `asktabib-eccde` |
| Firebase iOS app | FCM / APNs for doctor app | Firebase `asktabib-eccde` (must be configured — see iOS section) |

Installed / Desktop OAuth clients are **not** used for this native mobile flow.

---

## 4. Google Meet / Calendar linking — full sequence

### 4.1 Runtime sequence

1. Doctor logs into the AskTabib app (Sanctum token stored on device).
2. Doctor opens the home screen and taps **Connect Google Calendar**.
3. `GoogleAuthCubit.getAuthUrl()` runs.
4. `GetGoogleAuthUrlUseCase` → repository → `GoogleAuthRemoteDataSource.sendServerAuthCode()`.
5. Flutter initializes Google Sign-In:
   - Android/iOS: `serverClientId = Web Client ID`
   - Web: blocked
6. SDK flow:
   - `authenticate()`
   - `authorizeServer(scopes)` for Calendar scopes
   - returns `serverAuthCode`
7. App reads Sanctum bearer token (`GetTokenUseCase`).
8. App calls:

```http
POST https://asktabib.com/api/google/mobile-callback
Authorization: Bearer <sanctum_token>
Content-Type: application/json

{
  "code": "<serverAuthCode>"
}
```

9. Laravel validates Sanctum auth + **doctor** role, exchanges the code with Google using the **Web client ID + secret**, stores Google `access_token` / `refresh_token`, and can create Meet links later via Calendar APIs.
10. UI:
    - success without `auth_url` → show success message
    - success with `auth_url` → optional WebView fallback
    - failure → show error / diagnostic details

### 4.2 Scopes requested

```text
https://www.googleapis.com/auth/calendar
https://www.googleapis.com/auth/calendar.events
```

### 4.3 Main Flutter files

```text
lib/features/google_meet/data/data_sources/google_auth_remote_datasource.dart
lib/features/google_meet/data/repositories/google_auth_repository_impl.dart
lib/features/google_meet/data/models/google_auth_url_model.dart
lib/features/google_meet/domain/entities/google_auth_url_entity.dart
lib/features/google_meet/domain/repositories/google_auth_repository.dart
lib/features/google_meet/domain/use_cases/get_google_auth_url_usecase.dart
lib/features/google_meet/presentation/cubits/google_auth/google_auth_cubit.dart
lib/features/google_meet/presentation/cubits/google_auth/google_auth_state.dart
lib/features/google_meet/presentation/widgets/connect_google_button.dart
lib/features/google_meet/presentation/pages/google_auth_web_view_screen.dart
lib/features/google_meet/presentation/helper/google_message_localizer.dart
lib/injection_container.dart
lib/core/config/app_config.dart
```

### 4.4 Laravel files to verify

```text
routes/api.php
app/Http/Controllers/API/GoogleAuthController.php   # mobileCallback
app/Http/Middleware/RoleMiddleware.php
Google Calendar service / trait (token exchange + persistence)
.env  → GOOGLE_CLIENT_ID / GOOGLE_CLIENT_SECRET
```

Expected success response example:

```json
{
  "message": "Successfully authenticated with Google"
}
```

Common failure examples:

| Status | Meaning |
|--------|---------|
| 401 | Unauthenticated (missing/invalid Sanctum token) |
| 403 | Forbidden (user is not a doctor) |
| 400 | OAuth exchange failed |

### 4.5 Known pitfalls

- `GoogleSignInExceptionCode.canceled` on Android often means **OAuth misconfiguration** (wrong package/SHA-1/Web client), not a real user cancel.
- Package name mismatch between the APK and Google Cloud Android client breaks Sign-In.
- Calling `GET /google/mobile-callback` fails; backend expects **POST**.
- Non-doctor accounts receive **403**.

---

## 5. Push notifications — full sequence

### 5.1 Runtime sequence

1. App starts → Firebase initializes with `DefaultFirebaseOptions` / `google-services.json`.
2. Notification permission is requested (`FcmService`).
3. App obtains FCM device token and persists/refreshes it (token refresh listener).
4. Backend stores the doctor’s FCM token (via your existing API).
5. When an event occurs (for example meeting link ready), Laravel sends an FCM message using the Firebase Admin / HTTP v1 API for project **`asktabib-eccde`**.
6. App handles:
   - foreground
   - background (`firebaseMessagingBackgroundHandler`)
   - terminated / notification tap
7. For meeting-link payloads (`type = meeting_link_ready`), the app can show a local full-screen style alert / ringtone overlay and open the Meet URL.

### 5.2 Main Flutter files

```text
lib/main.dart
lib/firebase_options.dart
lib/core/services/fcm_service.dart
lib/core/services/fcm_background_handler.dart
lib/core/services/fcm_token_storage.dart
lib/core/services/push_notification_handler.dart
lib/core/services/push_notification_payload.dart
lib/core/services/push_notification_navigation.dart
lib/core/services/local_notification_service.dart
lib/core/services/meeting_link_call_controller.dart
lib/core/widgets/meeting_link_incoming_overlay.dart
android/app/google-services.json
android/app/src/main/AndroidManifest.xml
```

### 5.3 Meeting-link payload contract (app-side)

Typical data keys handled by the app include:

```text
type = meeting_link_ready
meet_link = <Google Meet URL>
title / body (optional)
```

### 5.4 Laravel / ops checklist for FCM

- Use a Firebase **service account** from project `asktabib-eccde`.
- Send to the latest FCM token saved from the doctor app.
- After reinstall / package change, the device gets a **new token** — backend must update it.

---

## 6. How to apply this setup in the future

### 6.1 New machine / new clone

1. Keep package/bundle id as `com.asktabib.doctor`.
2. Place the latest Firebase `google-services.json` in `android/app/`.
3. Confirm `lib/firebase_options.dart` Android `appId` matches the doctor Firebase app:
   - `1:905935166100:android:4d23576b7a51ae77a53281`
4. Confirm Flutter Web Client ID constant remains:

```text
783425217776-h2khhelad60rjn20ll9mmemk0fjatfrv.apps.googleusercontent.com
```

5. Confirm Laravel `.env` Web client ID/secret match that same Web client.
6. Build:

```bash
flutter pub get
flutter build apk --release
```

### 6.2 Changing signing key (Play Store)

1. Generate/upload keystore SHA-1 / SHA-256.
2. Add fingerprints to Google Cloud Android OAuth client in **`linen-creek-491819-f4`** only.
3. Do **not** try to add the same package+SHA-1 to Firebase if it already exists in the OAuth project.
4. Rebuild and retest Google linking + notifications.

### 6.3 Changing package name (avoid if already published)

If you must change package/bundle id:

1. Create a new Firebase Android/iOS app with the new id.
2. Create matching Android/iOS OAuth clients in `linen-creek-491819-f4`.
3. Replace `google-services.json` / `GoogleService-Info.plist`.
4. Update `applicationId`, iOS bundle id, and rebuild.
5. Update any store listings (package id cannot change after Play publish).

### 6.4 Replicating the Google linking mechanism in another app

1. Use `google_sign_in` with `serverClientId = Web Client ID`.
2. Request Calendar scopes and obtain `serverAuthCode`.
3. `POST /google/mobile-callback` with `{ "code": "..." }` + Bearer Sanctum token.
4. Backend exchanges code with Web client secret and stores tokens.
5. Restrict endpoint to doctor role if required.
6. Block unsupported web `serverAuthCode` flow with a clear error.

---

## 7. iOS — complete setup for Google Meet linking + notifications

Android is currently validated. iOS uses the **same Flutter code path** for Google Meet (`google_sign_in` + `POST /google/mobile-callback`) and the **same FCM handler code**, but Apple/Firebase/Google Console setup must be completed on a Mac.

**Current repo status (iOS):**

| Item | Status |
|------|--------|
| Bundle ID `com.asktabib.doctor` | Done in Xcode project |
| `UIBackgroundModes` includes `remote-notification` + `audio` | Done in `Info.plist` |
| Deep link scheme `asktabib` | Present |
| `GoogleService-Info.plist` in `ios/Runner/` | **Missing — must add** |
| Google Sign-In reversed client ID URL scheme | **Missing — must add** |
| APNs key uploaded to Firebase | **Must configure** |
| iOS OAuth client in `linen-creek-491819-f4` | **Must configure** |

---

### 7.1 Prerequisites

- macOS with latest stable **Xcode**
- Apple Developer Program account
- Physical iPhone (Push + Google Sign-In should be tested on device, not only Simulator)
- Access to:
  - Firebase project **`asktabib-eccde`**
  - Google Cloud project **`linen-creek-491819-f4`**
  - Apple Developer → Certificates, Identifiers & Profiles

---

### 7.2 Apple Developer — App ID & Push (notifications)

1. Open [Apple Developer → Identifiers](https://developer.apple.com/account/resources/identifiers/list).
2. Create/select App ID with Bundle ID:
   ```text
   com.asktabib.doctor
   ```
3. Enable capability:
   - **Push Notifications**
4. (Recommended) also enable:
   - Associated Domains (only if you use universal links later)
5. Create an **APNs Authentication Key**:
   - Keys → `+` → Apple Push Notifications service (APNs)
   - Download the `.p8` file **once**
   - Note:
     - Key ID
     - Team ID
6. Keep the `.p8` secure (backend/Firebase only; never commit to git if possible).

---

### 7.3 Firebase iOS app — notifications project `asktabib-eccde`

1. Firebase Console → project **`asktabib-eccde`** → Project settings → Your apps.
2. Add / select iOS app with Bundle ID:
   ```text
   com.asktabib.doctor
   ```
3. Download **`GoogleService-Info.plist`**.
4. Place it at:
   ```text
   ios/Runner/GoogleService-Info.plist
   ```
5. In Xcode, ensure the plist is added to the **Runner** target (Copy Bundle Resources).
6. Firebase → Project settings → **Cloud Messaging** → Apple app configuration:
   - Upload APNs Authentication Key (`.p8`)
   - Enter Key ID + Team ID
7. Align Dart Firebase options:
   - Preferred: run on Mac:
     ```bash
     flutterfire configure --project=asktabib-eccde
     ```
   - Or manually confirm `lib/firebase_options.dart` iOS values match the plist:
     - current placeholder app id in repo: `1:905935166100:ios:3b7a29a597c95ceca53281`
     - `iosBundleId: com.asktabib.doctor`
8. No SHA-1 is required for iOS FCM (APNs replaces Android SHA concerns).

**Do not** try to attach Android SHA-1 to Firebase for this iOS work.

---

### 7.4 Xcode project capabilities (notifications + Meet UX)

Open `ios/Runner.xcworkspace` (not `.xcodeproj`) and for **Runner** target:

1. **Signing & Capabilities**
   - Team selected
   - Bundle Identifier = `com.asktabib.doctor`
2. Add capability: **Push Notifications**
3. Add capability: **Background Modes** and enable:
   - Remote notifications
   - Audio, AirPlay, and Picture in Picture (already used for meeting-link ringtone/`audio` mode)
4. Confirm `Info.plist` already contains:

```xml
<key>UIBackgroundModes</key>
<array>
  <string>fetch</string>
  <string>remote-notification</string>
  <string>audio</string>
</array>
```

5. For notification permission prompts on iOS 10+, Flutter `firebase_messaging` / local notifications plugins request permission at runtime (`FcmService`). No extra native code is required beyond capabilities if plugins are registered (default Flutter setup).

Optional hardening in `AppDelegate.swift` (only if foreground presentation is incomplete on your iOS version):

```swift
import Flutter
import UIKit
import FirebaseCore
import FirebaseMessaging

@main
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    FirebaseApp.configure() // usually optional when using FlutterFire auto-init via plist
    GeneratedPluginRegistrant.register(with: self)
    UNUserNotificationCenter.current().delegate = self
    application.registerForRemoteNotifications()
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
```

Prefer testing first with the current AppDelegate; add native tweaks only if APNs token / foreground display fails.

---

### 7.5 Google Cloud — iOS OAuth client for Google Meet linking (`linen-creek-491819-f4`)

Google Meet linking on iOS still uses:

- Flutter `serverClientId` = **Web Client ID**
- Backend exchange with **Web Client secret**
- Plus an **iOS OAuth client** so Google can identify the iOS app

Steps:

1. Google Cloud Console → project **`linen-creek-491819-f4`**
2. APIs & Services → Credentials → Create Credentials → OAuth client ID
3. Application type: **iOS**
4. Bundle ID:
   ```text
   com.asktabib.doctor
   ```
5. Save and copy the **iOS Client ID**
6. Derive / copy the **reversed client ID** URL scheme  
   (from the iOS client or from `GoogleService-Info.plist` key `REVERSED_CLIENT_ID` if Firebase also created one)  
   Format looks like:
   ```text
   com.googleusercontent.apps.1234567890-abcdefg
   ```
7. Keep Web Client unchanged:
   ```text
   783425217776-h2khhelad60rjn20ll9mmemk0fjatfrv.apps.googleusercontent.com
   ```
   Flutter must continue using this as `serverClientId` (already in `google_auth_remote_datasource.dart`).

Enable APIs if not already enabled in the same project:

- Google Calendar API

---

### 7.6 Info.plist — URL schemes required for Google Sign-In (Meet linking)

Current `CFBundleURLTypes` only has deep link `asktabib`.  
Add a **second** URL type for Google Sign-In using the reversed client ID.

Example structure to end up with:

```xml
<key>CFBundleURLTypes</key>
<array>
  <!-- Existing AskTabib deep links -->
  <dict>
    <key>CFBundleTypeRole</key>
    <string>Editor</string>
    <key>CFBundleURLName</key>
    <string>com.asktabib.doctor</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>asktabib</string>
    </array>
  </dict>

  <!-- Required for Google Sign-In / Meet OAuth return -->
  <dict>
    <key>CFBundleTypeRole</key>
    <string>Editor</string>
    <key>CFBundleURLName</key>
    <string>google-sign-in</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>YOUR_REVERSED_CLIENT_ID</string>
    </array>
  </dict>
</array>
```

Replace `YOUR_REVERSED_CLIENT_ID` with the real value from Google Cloud / `GoogleService-Info.plist`.

Without this scheme, Google Sign-In on iOS typically fails to return to the app (often looks like cancel/interrupt).

---

### 7.7 Flutter side — no separate Meet flow for iOS

iOS uses the same sequence as Android:

1. `GoogleSignIn.initialize(serverClientId: WEB_CLIENT_ID)`
2. `authenticate()`
3. `authorizeServer([calendar, calendar.events])`
4. `POST /google/mobile-callback` with `{ code: serverAuthCode }` + Sanctum bearer token
5. Backend stores Google tokens and can create Meet links later

Checklist:

- [ ] Doctor account (route is `RoleMiddleware:doctor`)
- [ ] Same Web Client ID in Flutter + Laravel `.env`
- [ ] Calendar scopes unchanged
- [ ] Web platform remains blocked (`googleAuthWebUnsupported`)

No Laravel package-name change is required for iOS; backend already exchanges the Web client code.

---

### 7.8 Notifications on iOS — end-to-end sequence

1. App launches → Firebase initializes with iOS options / plist.
2. User grants notification permission (`FcmService.requestPermission`).
3. iOS registers for remote notifications → APNs device token.
4. Firebase maps APNs token → FCM token.
5. App reads FCM token and sends/stores it on Laravel (existing token sync path).
6. Laravel sends FCM via Firebase Admin for project **`asktabib-eccde`**.
7. Firebase delivers through APNs to the device.
8. App handles:
   - foreground
   - background
   - terminated / notification tap
9. If payload `type = meeting_link_ready`, open/join Meet link and use local notification / ringtone behavior.

Backend FCM notes for iOS:

- Prefer HTTP v1 API with service account from `asktabib-eccde`
- For alert display, include notification title/body and/or correct APNs payload fields
- After reinstall, FCM token changes — backend must update stored token

---

### 7.9 Privacy strings / App Store review notes

Already present:

- `NSPhotoLibraryUsageDescription`

Add/confirm if Apple review asks (as needed by features):

- Camera / Microphone (if in-app call/recording is added later)
- Any Sign in with Google disclosure text in App Privacy answers

For Google Calendar linking, document in App Review notes:

- Demo doctor account
- That the app requests Google Calendar scopes to create consultation Meet links

---

### 7.10 Build, install, and test plan (iOS)

On macOS:

```bash
cd ios
pod install
cd ..
flutter pub get
flutter clean
flutter run -d <iphone-udid>
# or release:
flutter build ipa
```

**Google Meet linking tests**

1. Login with a **doctor** account.
2. Tap Connect Google Calendar.
3. Complete Google account + consent for Calendar scopes.
4. Confirm backend success response (tokens saved).
5. Trigger a consultation Meet creation from backend and verify Meet link.

**Notification tests**

1. Allow notifications when prompted.
2. Log FCM token and confirm it is stored in Laravel.
3. Send a test push from Firebase Console to that token.
4. Verify receipt in:
   - foreground
   - background
   - killed state
5. Send `meeting_link_ready` payload with `meet_link` and verify open/join behavior.

---

### 7.11 iOS completion checklist (copy/paste)

#### A) Apple

- [ ] App ID `com.asktabib.doctor` exists
- [ ] Push Notifications enabled on App ID
- [ ] APNs Auth Key (`.p8`) created (Key ID + Team ID saved)

#### B) Firebase (`asktabib-eccde`) — notifications

- [ ] iOS app registered with bundle `com.asktabib.doctor`
- [ ] `ios/Runner/GoogleService-Info.plist` added to Runner target
- [ ] APNs key uploaded to Firebase Cloud Messaging
- [ ] `firebase_options.dart` iOS app id / bundle id match plist
- [ ] FCM token obtained on device and saved to backend

#### C) Google Cloud (`linen-creek-491819-f4`) — Meet OAuth

- [ ] iOS OAuth client created for `com.asktabib.doctor`
- [ ] Reversed client ID added to `Info.plist` URL schemes
- [ ] Web Client ID still used as Flutter `serverClientId`
- [ ] Laravel `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` match Web client
- [ ] Google Calendar API enabled

#### D) Xcode / app

- [ ] Opened via `ios/Runner.xcworkspace`
- [ ] Push Notifications capability enabled
- [ ] Background Modes: remote notifications (+ audio as needed)
- [ ] Signing team configured for device/TestFlight

#### E) QA

- [ ] Doctor Google Connect succeeds on real iPhone
- [ ] `/google/mobile-callback` returns success
- [ ] Push received (fg/bg/killed)
- [ ] `meeting_link_ready` opens Meet link

---

### 7.12 Common iOS failures

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Google Sign-In returns immediately / canceled | Missing reversed client ID URL scheme | Add scheme to `Info.plist` |
| No `serverAuthCode` | Wrong/missing Web `serverClientId` | Use Web client from `linen-creek` |
| 403 on mobile-callback | Non-doctor user | Use doctor account |
| No pushes at all | APNs key missing in Firebase | Upload `.p8` + Key ID + Team ID |
| Token null on iOS | Push capability / permission / plist missing | Fix Xcode capabilities + `GoogleService-Info.plist` |
| Works on Android only | iOS OAuth client / URL scheme incomplete | Finish sections 7.5–7.6 |

---

## 8. Quick verification matrix

| Check | Android | iOS |
|------|---------|-----|
| Package / Bundle = `com.asktabib.doctor` | Required | Required |
| Firebase app registered | Done (`...4d23576b...`) | Must confirm + plist |
| FCM token obtained & saved on backend | Test after install | Test after APNs setup |
| Android OAuth client package + SHA-1 in `linen-creek` | Required | N/A |
| iOS OAuth client bundle id in `linen-creek` | N/A | Required |
| Web client ID in Flutter + Laravel secret | Required | Required |
| `POST /google/mobile-callback` as doctor | Required | Required |
| SHA-1 added to Firebase | Not required for FCM | N/A |

---

## 9. Build commands

```bash
# Dependencies
flutter pub get

# Android release APK
flutter build apk --release
# Output: build/app/outputs/flutter-apk/app-release.apk

# Android App Bundle (Play Store)
flutter build appbundle --release

# iOS (macOS required)
flutter build ipa
```

After changing package name or Firebase config, uninstall the old app from the device before installing the new build.

---

## 10. Ownership summary

| Responsibility | System of record |
|----------------|------------------|
| Push delivery project | Firebase `asktabib-eccde` |
| Google Calendar/Meet OAuth project | Google Cloud `linen-creek-491819-f4` |
| Mobile app identity | `com.asktabib.doctor` |
| Token exchange & persistence | Laravel backend |
| Native Google account UI | Flutter `google_sign_in` |

If OAuth and FCM ever need to live under one Google Cloud project only, migrate carefully: one `(package + SHA-1)` Android OAuth client can exist in **only one** project at a time.

---

## 11. Machine-readable integration prompt (JSON)

Use the following prompt/spec to reproduce or audit the Google Calendar / Meet native linking flow:

```json
{
  "prompt_type": "google_oauth_calendar_meet_integration_spec",
  "project": "asktabib_doctor",
  "feature": "Google Calendar / Google Meet native mobile linking",
  "language": "en",
  "architecture_summary": {
    "flow_type": "Native Mobile OAuth (Android/iOS) + Backend token exchange",
    "mobile_sdk": "google_sign_in",
    "backend_role": "Exchange serverAuthCode for access_token/refresh_token and persist tokens",
    "web_support": "Not supported for serverAuthCode flow in this implementation"
  },
  "clients_used": {
    "android_oauth_client": {
      "purpose": "Identify the Android app to Google Sign-In",
      "package_name": "com.asktabib.doctor",
      "sha1_debug_current": "94:2E:43:E6:DE:87:21:71:6C:44:FF:4F:E4:F3:C3:0C:DA:5F:FC:7B",
      "notes": "Must match Google Cloud Console Android OAuth client (package + SHA1)."
    },
    "ios_oauth_client": {
      "purpose": "Identify the iOS app to Google Sign-In",
      "bundle_id": "com.asktabib.doctor",
      "notes": "Required when testing/shipping on iOS."
    },
    "web_oauth_client": {
      "purpose": "Used as serverClientId on mobile + used by backend for secure code exchange",
      "client_id": "783425217776-h2khhelad60rjn20ll9mmemk0fjatfrv.apps.googleusercontent.com",
      "project_id": "linen-creek-491819-f4",
      "redirect_uri_backend": "https://asktabib.com/api/google/callback",
      "client_secret_location": "Backend only (never in Flutter app)",
      "notes": "Installed/Desktop client is NOT used for this mobile native flow."
    }
  },
  "keys_and_config": {
    "flutter_google_web_client_id_constant": "783425217776-h2khhelad60rjn20ll9mmemk0fjatfrv.apps.googleusercontent.com",
    "google_sign_in_initialize": {
      "android_ios": {
        "clientId": null,
        "serverClientId": "WEB_CLIENT_ID"
      },
      "web": {
        "clientId": "WEB_CLIENT_ID",
        "serverClientId": null,
        "status": "Explicitly blocked with googleAuthWebUnsupported"
      }
    },
    "scopes": [
      "https://www.googleapis.com/auth/calendar",
      "https://www.googleapis.com/auth/calendar.events"
    ],
    "api_base_url": "https://asktabib.com/api/",
    "mobile_callback_endpoint": {
      "method": "POST",
      "path": "/google/mobile-callback",
      "auth": "Bearer Sanctum token",
      "body": {
        "code": "serverAuthCode from Google Sign-In"
      },
      "backend_route_guard": "auth:sanctum + RoleMiddleware:doctor"
    },
    "app_identifiers": {
      "android_application_id": "com.asktabib.doctor",
      "android_namespace": "com.asktabib.doctor",
      "ios_bundle_id": "com.asktabib.doctor"
    }
  },
  "sequence": [
    {
      "step": 1,
      "actor": "Doctor user",
      "action": "Logs into AskTabib app and opens Home screen"
    },
    {
      "step": 2,
      "actor": "UI",
      "action": "User taps Connect Google Calendar button",
      "file": "lib/features/google_meet/presentation/widgets/connect_google_button.dart"
    },
    {
      "step": 3,
      "actor": "Cubit",
      "action": "Calls getAuthUrl()",
      "file": "lib/features/google_meet/presentation/cubits/google_auth/google_auth_cubit.dart"
    },
    {
      "step": 4,
      "actor": "UseCase",
      "action": "Executes GetGoogleAuthUrlUseCase",
      "file": "lib/features/google_meet/domain/use_cases/get_google_auth_url_usecase.dart"
    },
    {
      "step": 5,
      "actor": "Repository",
      "action": "Calls remoteDataSource.sendServerAuthCode()",
      "file": "lib/features/google_meet/data/repositories/google_auth_repository_impl.dart"
    },
    {
      "step": 6,
      "actor": "DataSource",
      "action": "Initialize GoogleSignIn with serverClientId (Web Client ID)",
      "file": "lib/features/google_meet/data/data_sources/google_auth_remote_datasource.dart"
    },
    {
      "step": 7,
      "actor": "Google Sign-In SDK",
      "action": "authenticate() then authorizeServer(calendar scopes) and return serverAuthCode"
    },
    {
      "step": 8,
      "actor": "App",
      "action": "Read Sanctum token via GetTokenUseCase"
    },
    {
      "step": 9,
      "actor": "App",
      "action": "POST /google/mobile-callback with { code: serverAuthCode } and Bearer token"
    },
    {
      "step": 10,
      "actor": "Backend",
      "action": "Validate auth + doctor role, exchange code using Web client secret, save Google tokens",
      "controller": "GoogleAuthController@mobileCallback"
    },
    {
      "step": 11,
      "actor": "UI",
      "action": "If backend returns success without auth_url, show success snackbar. If auth_url exists, open WebView fallback."
    }
  ],
  "files_involved": {
    "flutter_feature_files": [
      "lib/features/google_meet/data/data_sources/google_auth_remote_datasource.dart",
      "lib/features/google_meet/data/repositories/google_auth_repository_impl.dart",
      "lib/features/google_meet/data/models/google_auth_url_model.dart",
      "lib/features/google_meet/domain/entities/google_auth_url_entity.dart",
      "lib/features/google_meet/domain/repositories/google_auth_repository.dart",
      "lib/features/google_meet/domain/use_cases/get_google_auth_url_usecase.dart",
      "lib/features/google_meet/presentation/cubits/google_auth/google_auth_cubit.dart",
      "lib/features/google_meet/presentation/cubits/google_auth/google_auth_state.dart",
      "lib/features/google_meet/presentation/widgets/connect_google_button.dart",
      "lib/features/google_meet/presentation/pages/google_auth_web_view_screen.dart",
      "lib/features/google_meet/presentation/helper/google_message_localizer.dart"
    ],
    "di_and_routing": [
      "lib/injection_container.dart",
      "lib/core/router/app_router.dart",
      "lib/core/dio/dio_helper.dart",
      "lib/core/config/app_config.dart"
    ],
    "platform_config": [
      "android/app/build.gradle.kts",
      "android/app/src/main/kotlin/com/asktabib/doctor/MainActivity.kt",
      "ios/Runner.xcodeproj/project.pbxproj",
      "web/index.html"
    ],
    "localization": [
      "lib/l10n/app_en.arb",
      "lib/l10n/app_ar.arb"
    ],
    "backend_files_reference": [
      "routes/api.php",
      "app/Http/Controllers/API/GoogleAuthController.php",
      "app/Http/Middleware/RoleMiddleware.php",
      "GoogleCalendarService trait (token exchange + token persistence)"
    ]
  },
  "critical_fixes_applied": [
    {
      "issue": "405 Method Not Allowed",
      "root_cause": "Flutter called GET /google/mobile-callback while backend expects POST",
      "fix": "Changed DioHelper.getData to DioHelper.postData with body {code}"
    },
    {
      "issue": "403 Forbidden",
      "root_cause": "Route guarded by RoleMiddleware:doctor; non-doctor users are rejected",
      "fix": "Use doctor account or temporarily adjust role middleware for testing"
    },
    {
      "issue": "Web assertion failures",
      "root_cause": "serverClientId is unsupported on Flutter Web google_sign_in",
      "fix": "Do not pass serverClientId on web; block web flow with googleAuthWebUnsupported"
    },
    {
      "issue": "OAuth package mismatch",
      "root_cause": "App used com.example... while Google Console expected com.asktabib.doctor",
      "fix": "Updated Android/iOS identifiers and regenerated SHA1"
    }
  ],
  "success_contract": {
    "backend_success_response_example": {
      "message": "Successfully authenticated with Google"
    },
    "ui_success_behavior": "Show localized success message (googleAuthSuccess) without requiring WebView"
  },
  "failure_contract_examples": {
    "unauthorized": {
      "status": 401,
      "body": { "error": "Unauthenticated" }
    },
    "forbidden": {
      "status": 403,
      "body": { "message": "Forbidden" }
    },
    "oauth_exchange_failed": {
      "status": 400,
      "body": {
        "error": "OAuth callback failed",
        "message": "<google_or_backend_error>"
      }
    }
  },
  "implementation_prompt_for_reuse": "Replicate this exact Google Calendar/Meet linking mechanism in a Flutter Clean Architecture app: use native Google Sign-In on Android/iOS with serverClientId set to the Web OAuth Client ID, request calendar + calendar.events scopes, obtain serverAuthCode, then POST it to /google/mobile-callback with Bearer Sanctum token. Backend must exchange the code using Web Client ID/secret, save tokens, and return success JSON. Ensure Android OAuth client package/SHA1 and iOS bundle ID match app identifiers. Do not use GET for mobile-callback. Restrict endpoint to doctor role if required. Block unsupported web serverAuthCode flow with a clear error."
}
```
