# 🎯 InterviPrep: Product & Technical Gaps Audit
**Target Version**: 1.0.0 (Production Roadmap)  
**Last Audit**: February 25, 2026

This document tracks identified gaps between the current implementation and a production-grade "Gold Standard" release.

---

## 1. 🔍 Observability & Monitoring
*Critical for identifying and fixing issues before users complain.*

- [x] **Crash Reporting (Crashlytics)**: Unified `AppLogger` integration that bridges all logs and exceptions to Firebase in production.
- [x] **Analytics (Firebase)**: `AnalyticsService` implemented and integrated into Auth flow. Ready for custom event tagging.
- [ ] **Latency Tracking**: No telemetry for Edge Function performance. We need to track the average time taken for AI to generate questions/feedback.

## 2. 🛡️ Compliance & Safety (App Store Ready)
*Failure to address these can lead to rejection or legal liability.*

- [x] **Account Deletion Flow** (iOS Mandate): Provide a clear path in Profile/Settings to permanently delete account and all data.
- [ ] **Data Portability (GDPR)**: No way for users to export their transcripts or performance reports.
- [x] **Accessible Legal Links**: Privacy Policy and Terms of Service must be 1-2 clicks away from the main UI.
- [x] **Content Moderation**: Implemented dual-layer safety check (Local Keyword Filter + OpenAI Moderation API) for user transcripts and AI-generated content.

## 3. 💳 Monetization & IAP Edge Cases
*Ensuring revenue streams are robust and compliant.*

- [x] **Pending Purchase Handling**: `PurchaseService` now detects and notifies users of "Pending" status (e.g. cash at store).
- [x] **Subscription Renewal UI**: Profile screen displays "Next Billing Date" and provides a direct "Manage Subscription" button.
- [x] **Trial Countdown**: Visual "Trial Ending Soon" banner on Home Screen tracks 7-day period.
- [x] **Add-on Consumption**: Entitlements are server-verified and reactive; multi-device sync handled via `accessProvider` refreshes.

## 4. 🎙️ Interview Feature Resilience
*Hardening the core "AI Interview" experience.*

- [x] **Pause/Resume Support**: Active interview screen now supports pausing STT/TTS and timers via a premium blur overlay.
- [x] **Auto-Save/Recovery**: mid-interview state is persisted to secure storage; "Resume Session" banner allows recovery from crashes.
- [ ] **Offline Practice Hub**: Cache "Interview Tips" and "Question Library" locally so users can read them without internet.
- [x] **STT Accuracy Guard**: Implemented real-time confidence tracking with visual warnings for low-clarity audio.
- [x] **AI Voice Optimization**: Auto-detects device hardware capabilities to adjust TTS speech rate for stability on older devices.

## 🚀 App Lifecycle & Maintenance
*Keeping the ecosystem healthy.*

- [x] **Force Update Mechanism**: Implemented `app_config` table in Supabase and `SystemGuard` in-app to block old versions.
- [x] **Maintenance Mode**: Remote flag in `app_config` triggers a global "Back Soon" screen via real-time Supabase streams.
- [x] **Global Connectivity Banner**: App-wide overlay implemented using `connectivity_plus` to warn users when offline.

## 🎨 UX & Polish
- [x] **Social Auth Diversity**: Add **Apple Sign-In** (Required for iOS) and **Google Sign-In**.
- [x] **Interactive Onboarding**: A 3-slide "Value Prop" walkthrough before the login screen.
- [x] **Haptic Feedback**: Add subtle haptics for "Answer Submitted" or "Interview Completed" for a premium feel.
