# InterviPrep — Google AdMob Production Readiness Checklist
> Audited: 2026-03-09 | Status: **NOT YET production-ready — 6 blockers, 4 non-blockers**

---

## Quick Summary

| Category | Item | Status |
|---|---|---|
| 🔴 P0 | Replace test App IDs (Android + iOS) | ❌ Missing |
| 🔴 P0 | Replace test Ad Unit IDs in `.env` | ❌ Missing |
| 🔴 P0 | Wire real Rewarded Ad SDK into Daily Drill gate | ❌ Simulated |
| 🔴 P0 | Register physical test device IDs | ❌ `'EMULATOR'` only |
| 🟡 P1 | Add server-side ad verification to `grant-drill-ad-bonus` EF | ⚠️ Client-trusted |
| 🟡 P1 | Wire Firebase Analytics to `AdService._logEvent()` | ⚠️ Stub (logs only) |
| 🟢 P2 | Call `logUpgradeAfterAd()` after IAP success | ⚠️ Never called |
| 🟢 P2 | Add SKAdNetwork entries for all Google ad partners | ⚠️ Partial (4 only) |
| ✅ Done | AdMob SDK initialized correctly | ✅ |
| ✅ Done | UMP consent flow (GDPR/CCPA) | ✅ |
| ✅ Done | Banner + Native + Rewarded widgets | ✅ |
| ✅ Done | PRO/ELITE ad gate (`adReadyProvider`) | ✅ |
| ✅ Done | Session frequency cap (max 3 impressions) | ✅ |
| ✅ Done | Android `AndroidManifest.xml` App ID entry | ✅ |
| ✅ Done | iOS `Info.plist` App ID + ATT keys | ✅ |

---

## 🔴 P0 — Blockers (App will be REJECTED in stores without these)

---

### P0-1 · Replace AdMob App IDs (Android + iOS)

**What:** The App ID embedded in the native configs is Google's global test App ID.
Using it in a published app violates AdMob policy and will result in account suspension.

**Android** — `android/app/src/main/AndroidManifest.xml` line 78:
```xml
<!-- CURRENT (TEST - must change) -->
android:value="ca-app-pub-3940256099942544~3347511713"

<!-- REPLACE WITH your real AdMob App ID from AdMob dashboard -->
android:value="ca-app-pub-XXXXXXXXXXXXXXXX~XXXXXXXXXX"
```

**iOS** — `ios/Runner/Info.plist` line 96:
```xml
<!-- CURRENT (TEST - must change) -->
<string>ca-app-pub-3940256099942544~1458002511</string>

<!-- REPLACE WITH your real AdMob App ID -->
<string>ca-app-pub-XXXXXXXXXXXXXXXX~YYYYYYYYYY</string>
```

**Where to get:** [AdMob Dashboard](https://apps.admob.com) → Apps → Your App → App settings → App ID

---

### P0-2 · Replace Ad Unit IDs in `.env`

**What:** All 8 ad unit IDs in `.env` are Google's public test IDs.
They serve test ads to anyone. Production builds must use your own registered ad units.

**Current `.env` (all TEST IDs — replace every line below):**
```env
ADMOB_ANDROID_BANNER_ID=ca-app-pub-3940256099942544/6300978111      ← test
ADMOB_ANDROID_REWARDED_ID=ca-app-pub-3940256099942544/5224354917    ← test
ADMOB_ANDROID_NATIVE_ID=ca-app-pub-3940256099942544/2247696110      ← test
ADMOB_ANDROID_INTERSTITIAL_ID=ca-app-pub-3940256099942544/1033173712 ← test (unused)
ADMOB_IOS_BANNER_ID=ca-app-pub-3940256099942544/2934735716          ← test
ADMOB_IOS_REWARDED_ID=ca-app-pub-3940256099942544/1712485313        ← test
ADMOB_IOS_NATIVE_ID=ca-app-pub-3940256099942544/3986624511          ← test
ADMOB_IOS_INTERSTITIAL_ID=ca-app-pub-3940256099942544/4411468910    ← test (unused)
```

**Where to create:** AdMob Dashboard → Your App → Ad units → **Create ad unit**
Create one for each type: Banner (Android), Banner (iOS), Rewarded (Android), Rewarded (iOS), Native (Android), Native (iOS).

> ⚠️ The `INTERSTITIAL_ID` vars exist in `.env` but are not currently read anywhere in Dart code — safe to ignore for now.

---

### P0-3 · Wire Real Rewarded Ad into Daily Drill Gate

**What:** `DailyDrillNotifier.watchAdForBonus()` uses a `Future.delayed(5s)` to simulate a rewarded ad.
In production this lets users get free bonus questions without ever watching a real ad — zero ad revenue.

**File:** `lib/features/daily_drill/providers/daily_drill_providers.dart` lines 219–246

**Current (simulated):**
```dart
Future<bool> watchAdForBonus() async {
  state = state.copyWith(isWatchingAd: true);
  // ← SIMULATED AD (5 s). No real ad shown. No ad revenue.
  await Future.delayed(const Duration(seconds: 5));
  await _ref.read(aiServiceProvider).grantDrillAdBonus();
  ...
}
```

**Required (production):**
```dart
Future<bool> watchAdForBonus() async {
  state = state.copyWith(isWatchingAd: true);

  final adService = _ref.read(adServiceProvider);
  bool rewarded = false;

  // Show pre-loaded rewarded ad — blocks until dismissed
  adService.showRewardedAd(
    onRewarded: () => rewarded = true,
    onNotReady: () {
      // Ad not available → either deny bonus OR grant with a grace policy
      state = state.copyWith(isWatchingAd: false, error: 'Ad not available. Try again shortly.');
    },
  );

  if (!rewarded) return false;

  // Only grant bonus after confirmed watch
  await _ref.read(aiServiceProvider).grantDrillAdBonus();
  ...
}
```

> ⚠️ The `_DrillInterstitialAdGate` widget already shows a countdown UI — it just needs the underlying call to use the real SDK. The `AdService.showRewardedAd()` is already fully implemented with all callbacks. This is roughly a **30-minute code change**.

---

### P0-4 · Register Real Physical Test Device IDs

**What:** The SDK is currently configured with `['EMULATOR']` as the only test device ID.
On **real physical devices** during QA, live ads will be served and **impressions counted against your AdMob account** if you accidentally click them during testing.

**File:** `lib/core/services/ad_service.dart` lines 103–107
```dart
if (kDebugMode) {
  await MobileAds.instance.updateRequestConfiguration(
    RequestConfiguration(testDeviceIds: ['EMULATOR']),  // ← add real device hashes
  );
}
```

**How to get device ID:**
1. Run the app in debug mode on your physical device
2. Check logcat for: `Use RequestConfiguration.Builder.setTestDeviceIds()` with the device hash
3. Add that hash to the list:
```dart
testDeviceIds: ['EMULATOR', 'YOUR_ANDROID_DEVICE_HASH', 'YOUR_IOS_DEVICE_HASH'],
```

---

## 🟡 P1 — Important (Revenue & Policy risk)

---

### P1-1 · Add Server-Side Ad Verification to `grant-drill-ad-bonus` EF

**What:** The `grant-drill-ad-bonus` Edge Function currently grants a bonus question to
any authenticated user who calls it — without verifying they actually watched an ad.
A malicious user can call the EF directly (via curl, Postman, etc.) to get unlimited questions for free.

**Current EF** (`supabase/functions/grant-drill-ad-bonus/index.ts`):
```typescript
// Any authenticated POST → grants bonus. No ad proof required.
const { data } = await supabase.rpc('grant_drill_ad_bonus', { p_user_id: userId });
```

**Fix — SSV (Server-Side Verification):**
AdMob Rewarded Ads support SSV callbacks where Google's servers call your server directly to confirm a real ad was watched. Steps:

1. In AdMob Dashboard → Ad units → your Rewarded unit → SSV → enable SSV
2. Set callback URL to: `https://qogderhgxrhyuptimxsi.supabase.co/functions/v1/grant-drill-ad-bonus`
3. Verify the `key_id` + `signature` from Google's GET callback in the EF before granting bonus
4. Remove direct POST grant from the EF (or keep as fallback with rate limiting)

Reference: [AdMob SSV docs](https://developers.google.com/admob/android/ssv)

---

### P1-2 · Wire Firebase Analytics to `AdService._logEvent()`

**What:** `AdService._logEvent()` is a stub that only calls `AppLogger.d()`. No actual analytics events are sent to Firebase Analytics. You're flying blind on ad performance.

**File:** `lib/core/services/ad_service.dart` line 250:
```dart
void _logEvent(String name, Map<String, Object> params) {
  AppLogger.d('[AdService][analytics] $name $params');
  // TODO: FirebaseAnalytics.instance.logEvent(name: name, parameters: params);
}
```

**Fix** — uncomment and wire:
```dart
import 'package:firebase_analytics/firebase_analytics.dart';

void _logEvent(String name, Map<String, Object> params) {
  AppLogger.d('[AdService][analytics] $name $params');
  FirebaseAnalytics.instance.logEvent(name: name, parameters: params);
}
```

`firebase_analytics: ^11.0.3` is **already in `pubspec.yaml`** — no new dependency needed.

**Events currently tracked (will fire once wired):**

| Event | Trigger |
|---|---|
| `ad_impression` | Banner or Rewarded ad shown |
| `rewarded_ad_completed` | User watched full rewarded ad |
| `ad_skipped` | User dismissed rewarded ad early |
| `free_user_upgrade_after_ad` | User upgrades after seeing ads |

---

## 🟢 P2 — Polish (won't block release, but improves revenue tracking)

---

### P2-1 · Call `logUpgradeAfterAd()` After Successful IAP

**What:** `AdService.logUpgradeAfterAd()` is implemented but never called anywhere.
It records how many ad impressions a user saw before converting to PRO — critical for optimizing ad pressure.

**File:** `lib/core/services/iap/purchase_service.dart` (or wherever IAP success is handled)

**Add after successful PRO purchase:**
```dart
// After subscription confirmed
ref.read(adServiceProvider).logUpgradeAfterAd();
```

---

### P2-2 · Complete SKAdNetwork Entries for iOS (currently 4, need ~20)

**What:** `ios/Runner/Info.plist` has only 4 SKAdNetwork IDs. Google's full list for AdMob
requires ~20 entries for accurate attribution on iOS 14+. Missing entries reduce measurable
ROAS and may affect ad fill rates from some demand partners.

**Fix:** Replace the 4 existing entries with the full list from:
[developers.google.com/admob/ios/3p-skadnetworks](https://developers.google.com/admob/ios/3p-skadnetworks)

---

## ✅ Already Production-Ready

| Component | File | Details |
|---|---|---|
| AdMob SDK initialization | `ad_service.dart` | Lazy, UMP-first, mobile-only guard |
| UMP consent flow | `ump_consent_service.dart` | Full GDPR/CCPA with fail-open |
| Banner widget | `ad_widgets.dart` | Theme-aware, 2s fallback, session cap |
| Native card widget | `ad_widgets.dart` | Template-based, dark/light support |
| Rewarded button widget | `ad_widgets.dart` | Pre-loaded, fail-open, `onNotReady` |
| PRO/ELITE ad gate | `ad_manager_provider.dart` | SDK stays cold for paid users |
| Session frequency cap | `ad_service.dart` | Max 3 impressions; 30min interstitial gap |
| Rewarded ad pre-loading | `ad_service.dart` | Pre-loads next rewarded immediately |
| A/B test framework | `ad_ab_test_service.dart` | Deterministic 50/50 UUID bucketing |
| iOS `Info.plist` ATT string | `Info.plist` | `NSUserTrackingUsageDescription` present |
| Platform-only guard | `ad_service.dart` | Web/Desktop excluded via `kIsWeb` |
| `AdBannerWidget` in Drill | `daily_drill_screen.dart` | After answer reveal, FREE only |
| `AdNativeCard` on Home | `home_screen.dart` | Between sections, self-hides for PRO |
| `RewardedAdButton` on Report | `interview_details_screen.dart` | Transcript gate, fail-open |
| Eager SDK warm-up | `home_screen.dart initState` | `adManagerProvider.future` on login |

---

## Production Go-Live Sequence

Complete in this order to avoid store rejection:

```
1. [P0] Create production AdMob account → get real App IDs
2. [P0] Create 6 ad units in AdMob dashboard (Banner×2, Rewarded×2, Native×2)
3. [P0] Update AndroidManifest.xml + Info.plist with real App IDs
4. [P0] Update .env with real Ad Unit IDs
5. [P0] Wire real RewardedAd SDK into watchAdForBonus()
6. [P0] Add physical device hashes to testDeviceIds list
7. [P1] Wire Firebase Analytics to _logEvent()
8. [P1] Enable AdMob SSV for Rewarded unit + update grant-drill-ad-bonus EF
9. [P2] Call logUpgradeAfterAd() on IAP success
10. [P2] Update SKAdNetwork entries in Info.plist from full Google list
11. [QA] Test on real Android + iOS physical devices in debug mode
12. [QA] Confirm test ads load (banner/native/rewarded) before going ENVIRONMENT=production
13. [QA] Change ENVIRONMENT=production in .env
14. [SHIP] Submit to Play Store / App Store
```

---

## Estimated Effort

| Priority | Items | Effort |
|---|---|---|
| P0 (blockers) | 4 items | ~3–4 hours |
| P1 (important) | 2 items | ~2–3 hours |
| P2 (polish) | 2 items | ~1 hour |
| **Total** | **8 items** | **~6–8 hours** |
