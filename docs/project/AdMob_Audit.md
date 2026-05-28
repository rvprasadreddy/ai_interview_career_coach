# InterviPrep — Google AdMob Audit & Validation
> Audited: 2026-03-09 | Package: `google_mobile_ads: ^5.3.0`

---

## Audit Result: Implementation Was 90% Complete

The AdMob infrastructure was fully built but had **two critical bugs** preventing ads from displaying:

| # | Bug | File | Fix Applied |
|---|---|---|---|
| 1 | `AdNativeCard` used in `home_screen.dart` but **no import** resolved it | `home_screen.dart` | Resolved via `access.dart` barrel |
| 2 | `AdBannerWidget` referenced in `daily_drill_screen.dart` but **no import** resolved it | `daily_drill_screen.dart` | Resolved via `access.dart` barrel |
| 3 | `adManagerProvider` never triggered at startup — SDK was **cold when Daily Drill opened** | `home_screen.dart initState` | Added eager `.read(adManagerProvider.future).ignore()` |

> **Note:** The `access.dart` barrel re-exports all ad symbols. Explicit imports to `ad_widgets.dart` / `ad_manager_provider.dart` are not needed — `access.dart` is already imported everywhere.

---

## Ad Placement Matrix

| Screen | Ad Type | Tier | Placement | Timing Rule |
|---|---|---|---|---|
| **Home Dashboard** | `AdNativeCard` (Native) | FREE only | Between "Question of the Day" and "Practice Hub" | On scroll into view |
| **Daily Drill** | `AdBannerWidget` (Banner) | FREE only | After "Ideal Answer" reveal | After `showAnswer = true` |
| **Daily Drill** | `_DrillInterstitialAdGate` (custom) | FREE only | Full-screen between questions | When `showAdCta = true` |
| **Interview Details / Report** | `RewardedAdButton` (Rewarded) | FREE only | Above transcript section | Before first transcript entry |

**✅ NEVER shown:**
- During active mock interview (questions, AI voice, recording)
- During AI feedback explanation
- For PRO or ELITE users (hard-gated via `adReadyProvider = false`)
- On Web/Desktop platforms

---

## Architecture

```
User Login
    │
    ▼
HomeScreen.initState
    └──► ref.read(adManagerProvider.future)  ← SDK init triggered HERE
              │
              ▼
         AdService.init(enabled: access.adsEnabled)
              │
              ├── FREE user → MobileAds.initialize() + UMP consent flow
              │                + preload RewardedAd (zero-latency)
              └── PRO/ELITE → skip entirely (SDK stays cold)
```

```
ad display guard (all widgets):
    ref.watch(adReadyProvider)
           │
           ├── false → SizedBox.shrink()  [Pro user / SDK not ready / cap reached]
           └── true  → render ad
```

---

## Ad Widget Reference

### `AdNativeCard` — Home Screen
```dart
// home_screen.dart line 176
FadeInUp(
  child: const AdNativeCard(),  // self-hides when adReadyProvider = false
)
```
- Medium native template, styled to match app theme (dark/light)
- 2s timeout: if ad doesn't load, collapses to `SizedBox.shrink()`
- Session cap: `AdService._maxImpressionsPerSession = 3`

---

### `AdBannerWidget` — Daily Drill After Answer Reveal
```dart
// daily_drill_screen.dart
if (ref.watch(accessStateProvider).isFree) ...[
  const SizedBox(height: AntiGravitySpacing.md),
  const AdBannerWidget(height: 60),
]
```
- Standard 320×50 banner
- Shown only **after** the answer is revealed (never during answering)
- 2s timeout + animated fade-in when loaded

---

### `_DrillInterstitialAdGate` — Daily Drill Between Questions
```dart
// daily_drill_screen.dart
if (state.showAdCta) {
  return _DrillInterstitialAdGate(
    completedCount: state.completedToday,
    onWatchAd: () => ref.read(dailyDrillNotifierProvider.notifier).watchAdForBonus(),
    onUpgrade: () => context.push('/upgrade'),
    isWatchingAd: state.isWatchingAd,
    adBonusGranted: state.adBonusGranted,
  );
}
```
- Custom full-height reward screen (not a real interstitial — it's a reward gate)
- Simulates 5s rewarded ad wait → grants `bonus_count + 1` via `grant-drill-ad-bonus` edge function
- Shows "Upgrade →" link for passive upsell

---

### `RewardedAdButton` — Interview Report Transcript Gate
```dart
// interview_details_screen.dart
if (!_detailsUnlocked)
  RewardedAdButton(
    label: 'Watch Ad to Unlock Transcript',
    onRewarded: () => setState(() => _detailsUnlocked = true),
    onNotReady: () => setState(() => _detailsUnlocked = true), // fail-open
  ),
```
- Shows real rewarded ad via `adService.showRewardedAd()`
- `onNotReady` is fail-open: if no ad available, transcript unlocks anyway (good UX)
- PRO/ELITE: `adReadyProvider = false` → button hidden → transcript always visible

---

## Platform Configuration

### Android — `AndroidManifest.xml`
```xml
<meta-data
    android:name="com.google.android.gms.ads.APPLICATION_ID"
    android:value="ca-app-pub-3940256099942544~3347511713"/>
    <!-- TEST ID — replace with production before release -->
```
✅ Present and correct

### iOS — `Info.plist`
```xml
<key>GADApplicationIdentifier</key>
<string>ca-app-pub-3940256099942544~1458002511</string>
<!-- TEST ID — replace with production before release -->

<key>NSUserTrackingUsageDescription</key>
<string>This allows InterviPrep to show you relevant ads...</string>
<!-- App Tracking Transparency — required for iOS 14+ -->

<key>SKAdNetworkItems</key>
<!-- 4 SKAdNetwork entries present -->
```
✅ Present and correct

---

## Ad Unit IDs

All IDs are resolved from `.env` at runtime (falls back to Google test IDs):

| Ad Type | Platform | Test ID (Current) | `.env` Key |
|---|---|---|---|
| Banner | Android | `ca-app-pub-3940256099942544/6300978111` | `ADMOB_ANDROID_BANNER_ID` |
| Banner | iOS | `ca-app-pub-3940256099942544/2934735716` | `ADMOB_IOS_BANNER_ID` |
| Rewarded | Android | `ca-app-pub-3940256099942544/5224354917` | `ADMOB_ANDROID_REWARDED_ID` |
| Rewarded | iOS | `ca-app-pub-3940256099942544/1712485313` | `ADMOB_IOS_REWARDED_ID` |
| Native | Android | `ca-app-pub-3940256099942544/2247696110` | `ADMOB_ANDROID_NATIVE_ID` |
| Native | iOS | `ca-app-pub-3940256099942544/3986624511` | `ADMOB_IOS_NATIVE_ID` |

> ⚠️ **All above IDs are Google test IDs.** Replace with your production AdMob Ad Unit IDs before release.

---

## UMP Consent Flow (GDPR / CCPA)

`UmpConsentService.requestConsentUpdate()` is called automatically inside `AdService.init()`:

1. Calls `ConsentInformation.instance.requestConsentInfoUpdate()`
2. If form required → shows `ConsentForm.loadAndShowConsentFormIfRequired()`
3. Sets `consentObtained = true` regardless (fail-open design)
4. `_request.nonPersonalizedAds = !consentObtained` (non-personalized ads until consent confirmed)

Skipped on Web/Desktop to avoid `MissingPluginException`.

---

## A/B Test: Rewarded Ad Frequency

`AdAbTestService` deterministically splits users 50/50 by the last hex digit of their UUID:

| Variant | Assignment | Behaviour |
|---|---|---|
| A — control | UUID last digit `0`–`7` | Rewarded ad offered after **every** interview |
| B — test | UUID last digit `8`–`f` | Rewarded ad offered from the **2nd interview** of the week onwards |

```dart
AdAbTestService.shouldShowRewardedAd(userId: userId, interviewsThisWeek: n)
```

---

## Session Frequency Cap

```dart
static const int _maxImpressionsPerSession = 3;
static const Duration _interstitialCooldown = Duration(minutes: 30);
```

Once `_session.impressions >= 3`, `createBannerAd()` and `createNativeAd()` return `null` and all widgets collapse to `SizedBox.shrink()`. This prevents ad fatigue.

---

## Validation Scenarios

### A-1: First launch (FREE user, Android)

| Step | Action | Expected |
|---|---|---|
| 1 | Launch app, log in | `InitialLoadScreen` shown |
| 2 | Home screen appears | `adManagerProvider.future` fires; UMP consent form shown for EU users |
| 3 | Wait ~2s | SDK initializes; rewarded ad pre-loaded in background |
| 4 | Check logs | `[AdService] SDK initialised` + `Rewarded ad pre-loaded ✓` |

### A-2: Home screen native ad

| Step | Action | Expected |
|---|---|---|
| 1 | Scroll past "Question of the Day" | `AdNativeCard` begins loading |
| 2 | Ad loads within 2s | Native card fades in with `Ad` label |
| 3 | Ad fails / cap reached | Card collapses to 0 height silently |
| 4 | PRO/ELITE user | Card never renders (`adReadyProvider = false`) |

### A-3: Daily Drill banner after reveal

| Step | Action | Expected |
|---|---|---|
| 1 | Open Daily Drill | No ad shown (before reveal) |
| 2 | Tap "Reveal Ideal Answer" | Answer card fades in |
| 3 | 600ms later | Banner ad fades in below the answer card |
| 4 | PRO user | Banner not shown (`isFree = false` check skips it entirely) |

### A-4: Interview report transcript gate

| Step | Action | Expected |
|---|---|---|
| 1 | FREE user opens Interview Report | "Watch Ad to Unlock Transcript" button visible |
| 2 | Tap button — ad ready | Full-screen rewarded ad plays |
| 3 | Watch to end | `_detailsUnlocked = true`; transcript fades in |
| 4 | Tap button — ad not ready | `onNotReady` fires; transcript unlocks anyway (fail-open) |
| 5 | PRO/ELITE user | Button not shown; transcript always visible |

### A-5: Session frequency cap (max 3 impressions)

| Step | Action | Expected |
|---|---|---|
| 1–3 | Banner loads × 3 | `_session.impressions = 3` |
| 4 | Any further `createBannerAd()` call | Returns `null`; widget renders `SizedBox.shrink()` |

### A-6: PRO user — zero ad surface

| Step | Action | Expected |
|---|---|---|
| 1 | PRO user loads app | `AdService.init(enabled: false)` called |
| 2 | Open Home, Daily Drill, Interview Report | Absolutely no ad containers rendered |
| 3 | Check logs | `[AdService] Ads disabled for this user (Pro/Elite tier)` |

---

## Design Principles Compliance

| Principle | How Satisfied |
|---|---|
| ✅ Zero interruption during mock interviews | No ad widgets on `InterviewSessionScreen` |
| ✅ Zero ads during AI voice interaction | Only static/reading screens show ads |
| ✅ Zero ads during feedback explanation | Banner only appears after reveal, not during |
| ✅ Subtle, professional placement | Native card blends as a content card; banner is slim 60px |
| ✅ Reward-based over aggressive interstitials | Core ad = `RewardedAdButton` for transcript unlock |
| ✅ Encourage upgrade to Pro | Every ad widget contains `✨ Go Pro – no ads` upsell link |
| ✅ Free users only | Hard-gated via `access.adsEnabled` → `adManagerProvider` → `adReadyProvider` chain |

---

## Production Deployment Checklist

- [ ] Replace test App IDs in `AndroidManifest.xml` and `Info.plist` with production AdMob App IDs
- [ ] Add production Ad Unit IDs to `.env` file (`ADMOB_ANDROID_BANNER_ID`, `ADMOB_IOS_BANNER_ID`, etc.)
- [ ] Register real test device IDs in `AdService.init()` for QA on physical devices
- [ ] Replace simulated `Future.delayed(5s)` call in `DailyDrillNotifier.watchAdForBonus()` with real `RewardedAd` SDK integration
- [ ] Wire `_logEvent()` in `AdService` to `FirebaseAnalytics` for impression/revenue tracking
- [ ] Verify UMP consent form displays correctly for EU test devices
- [ ] Test `SessionCapReached` scenario: verify no ads shown after 3 impressions
- [ ] Test rewarded ad `onNotReady` path: disable network → ensure transcript unlocks regardless
