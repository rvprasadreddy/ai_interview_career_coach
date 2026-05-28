# Android Fingerprints

This document contains the certificate fingerprints for this project's Android application. These are required for Google OAuth, Firebase, and other secure services.

## Debug Fingerprints
**Generated on:** 2026-03-14

### SHA-1
`A7:5F:4A:7B:99:7C:69:3F:FC:FB:C4:FE:F2:93:1C:F2:12:29:40:24`
*Used for: Google Cloud Console (Android Client ID), Supabase Authentication verification.*

### SHA-256
`11:B3:9B:BD:08:BC:30:35:98:6E:64:2B:36:E0:5D:0A:3C:0A:E9:B1:11:EA:8C:96:12:C9:D9:37:1B:CD:1A:0F`
*Used for: Android App Links (Digital Asset Links), Firebase App Check.*

---

## Configuration Checklist
1. [ ] **Google Cloud Console**: Add the **SHA-1** to your **Android Client ID**. Ensure the package name is `com.antigravity.ai_interview_coach`.
2. [ ] **Supabase Dashboard**: Copy the **Web Client ID** from Google Cloud and paste it into the **Google Client ID (Web)** field in Supabase Auth settings.
3. [ ] **Firebase Console**: (Optional) Add these fingerprints to your Android app settings in the Firebase console to enable Phone Auth or App Check.

---

> [!NOTE]
> These are **DEBUG** fingerprints. When you create a production release, you will need to retrieve the **Release SHA-1** (either from your local upload keystore or the Google Play Console "App Signing" section) and add it to the Google Cloud Console as well.
