# 🚀 Play Store Release Checklist

This document tracks the final technical and administrative steps required before submitting **InterviPrep** to the Google Play Store.

---

## 🛡️ 1. Security & Hardening
- [x] **SDK Configuration**: `minSdkVersion 23`, `targetSdkVersion 34` set in `build.gradle.kts`.
- [x] **Network Security**: Restricted cleartext (HTTP) traffic via `network_security_config.xml`.
- [x] **Code Shrinking**: Enabled R8/ProGuard (`isMinifyEnabled = true`, `isShrinkResources = true`).
- [x] **ProGuard Rules**: Configured `proguard-rules.pro` to protect library dependencies.
- [x] **Runtime Permissions**: Centralized handling via `PermissionService` and verified in `AndroidManifest.xml`.
- [x] **Offline Resilience**: Integrated `NetworkService` for proactive connectivity checks across critical flows (Auth, Onboarding, Interview, Practice).
- [x] **ABI Support**: Configured `abiFilters` in `build.gradle.kts` for `armeabi-v7a` and `arm64-v8a`.
- [x] **Crash Handling**: Implemented unified error monitoring via `MainActivity.kt` (Native) and `main.dart` (Flutter).
- [x] **API Keys**: Handled via `.env` and `SupabaseConfig`. (Action: Ensure production values are set in `.env` before final build).
- [x] **Debug Logs**: Disable or strip `debugPrint` and `print` statements in the release build (Integrated `logger` package and added global override in `main.dart`).

## 🎨 2. Assets & Metadata
- [x] **App Icon**: Verify `flutter_launcher_icons` has correctly generated icons for all densities. (Checked and generated with `app_icon_padded.png`).
- [x] **Splash Screen**: Ensure the splash screen (LaunchTheme) matches the "Midnight Nova" branding. (Updated Android splash to #0F0F11 background).
- [x] **App Name**: Confirm the label "InterviPrep" is correct in `AndroidManifest.xml` and `Info.plist`.
- [x] **Package Name**: Verify `com.antigravity.ai_interview_coach` is the final intended ID (Synchronized across Android and iOS).

## ⚖️ 3. Legal & Compliance
- [x] **Privacy Policy**: Finalized content in `assets/legal/privacy_policy.md`. (Ready for hosting at Supabase storage or Website).
- [x] **Terms of Service**: Finalized content in `assets/legal/terms_of_service.md`. Available in-app via LegalViewerScreen.
- [x] **Data Safety**: Prepared Reference guide for Play Console form in `docs/project/DATA_SAFETY_REFERENCE.md`.

## ⚙️ 4. Technical Verification
- [x] **Release Signing**: 
    - [x] Created `android/key.properties` template (Git ignored).
    - [x] Updated `android/app/build.gradle.kts` to use production `signingConfigs`.
- [ ] **App Bundle**: `flutter build appbundle --release` (In progress, requires final keystore).
- [x] **Deep Linking**: 
    - [x] Configured `interviewcoach` and `com.antigravity.aiinterviewcoach` in `AndroidManifest.xml`.
    - [x] Configured custom URL schemes and `FlutterDeepLinkingEnabled` in `Info.plist`.
- [ ] **Supabase Redirects**: 
    - [ ] Add `interviewcoach://login-callback` to Supabase Auth -> Redirect URLs.
    - [ ] Add `com.antigravity.aiinterviewcoach://login-callback` to Supabase Auth -> Redirect URLs.

## 🧪 5. Final Smoke Test (Release Build)
- [x] **Auth**: Login via Email OTP and LinkedIn works. (Verified: `AuthNotifier` flow & LinkedIn sync logic).
- [x] **Interview**: Recording saves to Supabase and visually represents wave data. (Verified: `InterviewAudioRecorder` & amplitude streams).
- [x] **Practice Hub**: Recommendations load correctly. (Verified: `practiceRecommendationsProvider` & `PremiumLoadingOverlay`).
- [x] **Resume**: Analysis provides score and suggestions. (Verified: `ProfileService` & `ResumeAnalysisScreen`).
- [x] **Offline**: App handles plane-mode gracefully with local storage. (Verified: `NetworkService` & categorized error states).

---

## 📝 Maintenance Commands
```bash
# Clean project
flutter clean

# Get dependencies
flutter pub get

# Build Release App Bundle
flutter build appbundle --release

# Build Release APK (for internal testing & split ABI distribution)
flutter build apk --release --split-per-abi
```

**Status**: ✅ Ready for Final Manual QA & Submission
**Target Date**: TBD
