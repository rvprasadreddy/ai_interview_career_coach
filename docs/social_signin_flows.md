# Social Sign-In Flow Documentation
### InterviPrep — Apple & Google Authentication

> **Last Updated:** May 2026  
> **Files:** `lib/features/auth/services/auth_service.dart`, `lib/features/auth/providers/auth_provider.dart`, `lib/features/auth/screens/login_screen.dart`

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Apple Sign-In Flow](#2-apple-sign-in-flow)
3. [Google Sign-In Flow](#3-google-sign-in-flow)
4. [Shared Post-Login Pipeline](#4-shared-post-login-pipeline)
5. [Nonce Strategy — Side by Side](#5-nonce-strategy--side-by-side)
6. [Error Handling Reference](#6-error-handling-reference)
7. [iOS & Android Configuration](#7-ios--android-configuration)
8. [Supabase Dashboard Settings](#8-supabase-dashboard-settings)
9. [Key Maintenance Notes](#9-key-maintenance-notes)

---

## 1. Architecture Overview

The authentication pipeline has **three layers**:

```
UI Layer  (login_screen.dart)
  Button tap → calls AuthNotifier method
  Device-lock precheck before showing sheet
  Manages local button loading spinner

State Layer  (auth_provider.dart / AuthNotifier)
  Manages global AuthState (isLoading, error)
  Delegates SDK calls to AuthService
  Handles Supabase auth state change events
  Calls profile sync after successful login

Service Layer  (auth_service.dart / AuthService)
  Direct calls to Apple / Google native SDKs
  Exchanges tokens with Supabase
  Syncs user profile to database
  All error logging → Firebase Crashlytics
```

---

## 2. Apple Sign-In Flow

> **Platform:** iOS only (button hidden on Android)
> **SDK:** `sign_in_with_apple` Flutter package
> **Backend:** Supabase `signInWithIdToken`

### 2.1 Step-by-Step Flow

```
User taps "Continue with Apple"
         |
         v
[login_screen.dart] _handleAppleLogin()
  1. Check network connectivity — show snackbar if offline
  2. precheckDeviceLock() — block if device belongs to a different account
  3. Call AuthNotifier.signInWithApple()
  4. Reset _isAppleLoading in finally block (always runs)
         |
         v
[auth_provider.dart] AuthNotifier.signInWithApple()
  1. Set state: isLoading=true, error=null
  2. Call AuthService.signInWithApple()
  3a. If result['canceled']==true → isLoading=false, return (no error)
  3b. If success → call syncAppleProfile(), log Analytics event
  3c. If throws → set error in state, isLoading=false
  4. _handleAuthChange fires asynchronously → loads profile → isAuthenticated=true
         |
         v
[auth_service.dart] AuthService.signInWithApple()
  1. Generate rawNonce (32-char secure random string)
  2. Compute hashedNonce = SHA-256(rawNonce)
  3. Call SignInWithApple.getAppleIDCredential(nonce: hashedNonce)
     → Apple embeds hashedNonce inside the id_token
  4. Extract identityToken from credential
  5. Call Supabase.signInWithIdToken(idToken, nonce: rawNonce)
     → Supabase SHA-256s rawNonce and verifies against token
  6. Return Map { given_name, family_name, email, user_identifier }
         |
         v
[auth_provider.dart] _handleAuthChange(signedIn event)
  1. Detect isAppleAuth via appMetadata['provider'] == 'apple'
  2. Call syncUserToDatabase(user, 'apple')
     → Device-lock enforcement
     → New user: upsert to users table
     → Activate 3-day Elite trial via Edge Function
     → Upsert device mapping in user_devices table
  3. Call _loadUserProfile() → set isAuthenticated=true
```

### 2.2 Nonce Security Design

```
App                        Apple                      Supabase
 |                            |                           |
 |-- rawNonce (random) -->    |                           |
 |   hashedNonce=SHA256(raw)  |                           |
 |                            |                           |
 |-- getAppleIDCredential --> |                           |
 |   (nonce: hashedNonce)     |                           |
 |                            | embeds hashedNonce        |
 |                            | inside id_token           |
 |<-- identityToken ----------|                           |
 |                            |                           |
 |-- signInWithIdToken -------+-------------------------> |
 |   (idToken, nonce: rawNonce)                          |
 |                            |  SHA256(rawNonce)         |
 |                            |  == token nonce? YES      |
 |<-- Supabase Session -------+---------------------------|
```

**Why this matters:** Without the nonce, a stolen `id_token` could be replayed against any Supabase instance. The nonce ties the token to this specific login request.

### 2.3 First Login vs. Repeat Login

| Data | First Login | Repeat Login |
|---|---|---|
| `given_name` | Provided by Apple | null — Apple never sends again |
| `family_name` | Provided by Apple | null |
| `email` | Provided (real or relay) | null |
| `user_identifier` | Always present | Always present |

**Critical:** `syncAppleProfile()` is called immediately after successful login while `appleCredential` data is still in memory. On repeat logins, the name fallback chain is:

```
Apple name → userMetadata['full_name'] → email prefix → 'User'
```

### 2.4 Cancellation Handling

The Apple SDK throws `SignInWithAppleAuthorizationException` with `code=canceled` when the user dismisses the sheet. This is **not an error** — handled as a sentinel return value:

```dart
return {'canceled': true}; // no throw, no error dialog shown
```

The `AuthNotifier` detects this and silently resets `isLoading=false`.

---

## 3. Google Sign-In Flow

> **Platform:** iOS and Android
> **SDK:** `google_sign_in` Flutter package
> **Backend:** Supabase `signInWithIdToken`

### 3.1 Step-by-Step Flow

```
User taps "Continue with Google"
         |
         v
[login_screen.dart] _handleGoogleLogin()
  1. Check network connectivity
  2. precheckDeviceLock() — block if device belongs to different account
  3. Call AuthNotifier.signInWithGoogle()
  4. If returns false (cancelled) → reset _isGoogleLoading
         |
         v
[auth_provider.dart] AuthNotifier.signInWithGoogle()
  1. Set state: isLoading=true, error=null
  2. Call AuthService.signInWithGoogle()
  3a. If returns false (cancelled) → isLoading=false, return false
  3b. If returns true → loading stays ON (auth event will complete it)
  3c. If throws → set error in state, isLoading=false
         |
         v
[auth_service.dart] AuthService.signInWithGoogle()
  1. Create GoogleSignIn(serverClientId: googleWebClientId)
     NOTE: clientId intentionally omitted (see Nonce section below)
  2. Call googleSignIn.signOut() — clear previous cached account
  3. Call googleSignIn.signIn() — shows account picker
  4. If user == null → return false (cancelled)
  5. Get googleAuth (idToken + accessToken)
  6. Call Supabase.signInWithIdToken(
       provider: google,
       idToken: idToken,
       accessToken: accessToken
       // NO nonce — Skip nonce checks enabled in Supabase dashboard
     )
  7. Return true
         |
         v
[auth_provider.dart] _handleAuthChange(signedIn event)
  1. Detect isGoogleAuth via appMetadata['provider'] == 'google'
  2. Call syncGoogleProfile(user)
     → Calls 'google-profile-sync' Supabase Edge Function
     → Edge Function handles new vs. returning user logic
     → Also calls syncUserToDatabase(user, 'google')
        → Device-lock enforcement
        → New user: upsert to users table
        → Activate 3-day Elite trial
        → Upsert device mapping in user_devices table
  3. Log Analytics: logLogin('google'), logSignUp('google') if new
  4. Call _loadUserProfile() → set isAuthenticated=true
```

### 3.2 Nonce Strategy — Why Google Skips It

> **IMPORTANT — Do not change this without understanding the root cause**

Google's native SDK on **iOS** has a specific behavior:
- If `clientId` is provided to `GoogleSignIn(clientId: ...)`, the SDK **automatically embeds an internal nonce** inside the `id_token`
- This nonce is **not exposed** to the Flutter app — we cannot read it
- If we pass NO nonce to Supabase but the token contains one, Supabase rejects with HTTP 400: `"Passed nonce and nonce in id_token should either both exist or not"`

**Resolution applied:**
1. `clientId` is intentionally **omitted** from `GoogleSignIn(...)` — only `serverClientId` is used
2. In **Supabase Dashboard → Auth → Providers → Google → "Skip nonce checks" is ENABLED**
3. Supabase still fully validates: token signature, audience (`aud`), and expiry (`exp`)

This is the officially recommended approach in Supabase docs for native Google Sign-In.
See: https://supabase.com/docs/guides/auth/social-login/auth-google

### 3.3 Why `signOut()` is Called Before `signIn()`

```dart
await googleSignIn.signOut(); // pre-sign-in cleanup
```

Without this, the Google SDK reuses the previously cached account silently (no picker shown). Calling `signOut()` first forces the account selection dialog to appear on every login attempt.

---

## 4. Shared Post-Login Pipeline

Both Apple and Google go through `syncUserToDatabase()` after login:

```
syncUserToDatabase(user, provider)
  |
  |-- 1. Get deviceId from DeviceUtils.getDeviceId()
  |
  |-- 2. Check device ownership (user_devices table)
  |      If device is registered to a DIFFERENT email:
  |      → throw AuthException (device-limit) and force sign out
  |      [Whitelisted emails: 'support@interviprep.ai' bypass this check]
  |
  |-- 3. Check user_subscriptions table
  |      If no subscription OR free user never used trial:
  |      → Upsert user in 'users' table
  |      → Call 'activate-trial' Edge Function (3-day Elite trial)
  |      Else:
  |      → Just update 'updated_at' timestamp
  |
  └-- 4. Upsert device mapping in user_devices table
         (onConflict: user_id, device_id)
```

---

## 5. Nonce Strategy — Side by Side

| | Apple Sign-In | Google Sign-In |
|---|---|---|
| **Nonce used?** | YES — mandatory | NO — intentionally omitted |
| **Who generates nonce?** | App (Random.secure()) | N/A |
| **Sent to provider** | SHA256(rawNonce) | Nothing |
| **Sent to Supabase** | rawNonce (Supabase re-hashes) | Nothing |
| **Supabase "Skip nonce checks"** | DISABLED (not present) | ENABLED |
| **Why the difference?** | Apple exposes the nonce; we control both sides | Google SDK embeds its own internal nonce on iOS that we cannot read |
| **Security level** | Full anti-replay protection | Signature + audience + expiry still verified |

---

## 6. Error Handling Reference

### Apple Error Codes

| AuthorizationErrorCode | Common Cause | User Message |
|---|---|---|
| `canceled` | User dismissed sheet | Silent — no error shown |
| `failed` | Entitlement missing, bundle-ID mismatch | "Apple Sign-In failed. Check device configuration." |
| `invalidResponse` | Malformed Apple credential | "Apple returned an invalid response." |
| `notHandled` | System could not process request | "Apple Sign-In could not be completed." |
| `notInteractive` | MDM policy / device restriction | "Apple Sign-In is unavailable on this device." |
| `unknown` (1000) | iCloud not signed in, Apple server down | "Unexpected error. Ensure iCloud is signed in." |

All non-cancellation errors → **Firebase Crashlytics** via `AppLogger.e(message, error, StackTrace.current)`.

### Google Error Handling

| Condition | Handling |
|---|---|
| User cancelled (null account) | Returns `false` — no error shown |
| `idToken == null` | Throws Exception |
| Supabase rejection | Caught → `_getErrorMessage()` in AuthNotifier |
| Network failure | Caught by generic error handler |

### Shared Error Message Mapping (`_getErrorMessage`)

| Raw Error | User-Friendly Message |
|---|---|
| `nonce` / `both exist or not` | "Sign-in failed. Please try again." |
| `device_id_unique` / `duplicate key` | "This device is already linked to another account." |
| `device-limit` status code | Device policy message |
| `socketexception` / `failed host lookup` | "Connection error. Check your internet." |
| `too many requests` / `rate limit` | "Too many attempts. Wait a minute." |

---

## 7. iOS & Android Configuration

### iOS Required Files

#### `ios/Runner/Runner.entitlements`
```xml
<!-- Sign In with Apple — REQUIRED -->
<key>com.apple.developer.applesignin</key>
<array>
  <string>Default</string>
</array>

<!-- Associated Domains — for Supabase deep link callback -->
<key>com.apple.developer.associated-domains</key>
<array>
  <string>applinks:YOUR_PROJECT.supabase.co</string>
</array>
```

#### Bundle Identifier (project.pbxproj)
```
PRODUCT_BUNDLE_IDENTIFIER = com.antigravity.aiinterviewcoach;
```

> This MUST exactly match the Client ID registered in Supabase and Apple Developer Center.

#### `ios/Runner/Info.plist` — URL Schemes
```xml
<!-- OAuth deep-link callback scheme -->
<string>com.antigravity.aiinterviewcoach</string>

<!-- Google Sign-In reverse client ID scheme -->
<string>com.googleusercontent.apps.30088253105-ej2besf6mcrlunncrg8knme1fu7485np</string>
```

### Android — Google Sign-In

Google Sign-In on Android uses `serverClientId` (web client ID from Firebase):
```dart
GoogleSignIn(serverClientId: DefaultFirebaseOptions.googleWebClientId)
```
No special Android manifest changes needed beyond standard Firebase `google-services.json` setup.

> Apple Sign-In does NOT exist on Android — the button is hidden when `defaultTargetPlatform != TargetPlatform.iOS`.

---

## 8. Supabase Dashboard Settings

### Apple Provider (Auth → Providers → Apple)

| Setting | Value | Notes |
|---|---|---|
| Enable Sign in with Apple | ON | |
| Client IDs | `com.antigravity.aiinterviewcoach` | Must match bundle ID exactly |
| Secret Key (.p8) | Configured | **Expires every 6 months** |
| Allow users without an email | ON | Required for Apple's private email relay |
| Skip nonce checks | OFF / not present | App sends nonce; Supabase must verify it |

### Google Provider (Auth → Providers → Google)

| Setting | Value | Notes |
|---|---|---|
| Enable Google | ON | |
| Client ID | Web client ID from Firebase | |
| **Skip nonce checks** | **ENABLED** | Critical — do not disable |

---

## 9. Key Maintenance Notes

### Apple Secret Key Expiry (Every 6 Months)

Apple `.p8` OAuth keys expire every 6 months. When expired:
- Native iOS Apple Sign-In continues to work (uses `signInWithIdToken`, not OAuth)
- Web-based Apple OAuth breaks

**To renew:**
1. Go to Apple Developer Center → Certificates, Identifiers & Profiles → Keys
2. Find your key → check creation date
3. If expired: create a new key, download `.p8`
4. Paste new key into Supabase → Auth → Providers → Apple → Secret Key

### Device Lock Policy

Both Apple and Google logins enforce **1 account per physical device**:
- Enforced in `syncUserToDatabase()` via the `user_devices` table
- Whitelisted email `support@interviprep.ai` bypasses this (App Store reviewer)
- Violations throw `AuthException(statusCode: 'device-limit')` and force sign-out

### Reviewer Bypass

```dart
static const List<String> _deviceRestrictionWhitelist = [
  'support@interviprep.ai',
];
```

This account skips device registration checks for App Store / Play Store review.
Do not add more emails here without a controlled process.

### Trial Activation

All new social sign-ins (Apple + Google) automatically receive a **3-day Elite trial** via the `activate-trial` Supabase Edge Function. Fires inside `syncUserToDatabase()` when `existingSubscription == null`.

### Analytics Events

| Event | Apple | Google |
|---|---|---|
| `logLogin(method)` | `'apple'` | `'google'` |
| `logSignUp(method)` | Not logged separately | `'google'` if `is_new_user == true` |

---

*Generated from codebase — May 2026. Update this document whenever the auth flow changes.*
