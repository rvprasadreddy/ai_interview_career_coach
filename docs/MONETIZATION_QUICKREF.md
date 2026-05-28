# Monetization Engine — Developer Quick Reference (v2)

> Full documentation: `docs/MONETIZATION_ENGINE.md`

---

## Single Import

```dart
import 'package:ai_interview_coach/core/access/access.dart';
```

---

## FeatureGate — Tier Gating (Does the tier allow it?)

Use `FeatureGate` when a feature is gated behind a subscription tier (FREE / PRO / ELITE).

### Gate #1: Silent hide (most common)
```dart
FeatureGate(
  featureKey: FeatureKeys.feedbackDeepdive,
  child: const FeedbackSection(),
)
```

### Gate #2: Show upgrade CTA when locked
```dart
FeatureGate(
  featureKey: FeatureKeys.readinessRadar,
  child: const RadarChart(),
  lockedBuilder: (ctx) => const UpgradePromptCard(
    title: 'Readiness Radar is Pro',
    subtitle: 'Upgrade to see your full performance breakdown.',
  ),
)
```

### Gate #3: Teaser overlay (dim + lock icon)
```dart
FeatureGate(
  featureKey: FeatureKeys.resumeAtsDetailed,
  blurMode: FeatureBlurMode.frostedGlass,
  child: const FullAtsReport(),
)
```

### Gate #4: Programmatic check (in controllers)
```dart
final access = ref.read(accessStateProvider);
if (access.canAccess(FeatureKeys.audioRecording)) { ... }
```

---

## UsageLimitGuard — Quota Gating (Has the quota been hit?)

Use `UsageLimitGuard` when the user's **tier allows** the feature but they have a **usage cap**.
The two layers are separate — you often use both.

### Guard #1: Block entirely when quota hit (default)
```dart
UsageLimitGuard(
  ruleKey: UsageLimitKeys.mockInterviews,
  child: StartInterviewButton(),
)
```

### Guard #2: Show inline banner with upgrade CTA
```dart
UsageLimitGuard(
  ruleKey: UsageLimitKeys.dailyDrillQuestions,
  mode: UsageLimitMode.banner,
  child: DrillStartCard(),
)
```

### Guard #3: Usage counter chip above widget
```dart
// Shows "1 left" / "0 left" chip above the button
UsageLimitGuard(
  ruleKey: UsageLimitKeys.mockInterviews,
  mode: UsageLimitMode.counter,
  child: StartInterviewButton(),
)
```

### Guard #4: Frosted-glass quota overlay
```dart
UsageLimitGuard(
  ruleKey: UsageLimitKeys.mockInterviews,
  mode: UsageLimitMode.overlay,
  child: InterviewCard(),
)
```

### Guard #5: Programmatic quota check (in controllers)
```dart
final access = ref.read(accessStateProvider);
final limit  = access.getUsageLimit(UsageLimitKeys.mockInterviews);

if (limit.isAtLimit) {
  // Show paywall / block navigation
}

print(limit.used);       // e.g. 1
print(limit.limit);      // e.g. 1 (or -1 = unlimited)
print(limit.remaining);  // e.g. 0
print(limit.window);     // 'weekly' | 'daily' | 'session'
print(limit.isUnlimited);// true for PRO
```

### Guard #6: Combined FeatureGate + UsageLimitGuard
```dart
// Step 1: Gate on tier (is the feature unlocked?)
FeatureGate(
  featureKey: FeatureKeys.weeklyMock,
  blurMode: FeatureBlurMode.inlineBanner,
  child: // Step 2: Gate on quota (is the cap hit?)
    UsageLimitGuard(
      ruleKey: UsageLimitKeys.mockInterviews,
      mode: UsageLimitMode.banner,
      child: StartInterviewButton(),
    ),
)
```

---

## Refresh After Purchase / Subscription Change

```dart
await ref.read(accessProvider.notifier).refresh();
```

---

## In-App Purchases (IAP) — Checkout Flow

Use `purchaseProvider` to manage the checkout state and `PaywallScreen` for the UI.

### Step 1: Open the Paywall
```dart
PaywallScreen.show(context, reason: 'Unlock Resume ATS');
```

### Step 2: Custom Buy Button (Manual)
```dart
final iap = ref.watch(purchaseProvider);
final proPlan = iap.products.firstWhere((p) => p.id == IapProductIds.proMonthly);

ElevatedButton(
  onPressed: iap.status == IapStatus.pending 
    ? null 
    : () => ref.read(purchaseProvider.notifier).purchase(proPlan),
  child: iap.status == IapStatus.pending 
    ? const CircularProgressIndicator()
    : const Text('Upgrade to Pro'),
)
```

### Step 3: Restore Purchases
```dart
ref.read(purchaseProvider.notifier).restore();
```

---

## Google Ads — Rewarded Ads

Use `adServiceProvider` directly for rewarded flows (e.g., unlocking a one-time bonus).

### showRewardedAd
```dart
final adService = ref.read(adServiceProvider);

adService.showRewardedAd(
  onRewarded: () {
    // Grant the user their bonus/reward here
    ref.read(dailyDrillProvider.notifier).grantBonusReveal();
  },
  onNotReady: () {
    // Ad wasn't loaded — show an error or alternative
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('Ads are not available right now.'))
    );
  },
);
```

---

## FeatureKeys Reference (v2)

### FREE tier
| Constant | Feature Key | Description |
|----------|-------------|-------------|
| `FeatureKeys.weeklyMock` | `mock.weekly` | 1 interview/week, 3 questions max |
| `FeatureKeys.dailyDrillBasic` | `daily_drill.basic` | 1 daily drill question/day |
| `FeatureKeys.resumeAtsBasic` | `resume.ats.basic` | Basic ATS score |
| `FeatureKeys.aiPersonalityPro` | `ai.personality.pro` | Professional AI personality |
| `FeatureKeys.readinessBasic` | `readiness.basic` | Limited readiness score |
| `FeatureKeys.feedbackSummary` | `feedback.summary` | High-level AI summary |
| `FeatureKeys.learningBasic` | `learning.basic` | Basic learning paths |

### PRO tier
| Constant | Feature Key | Description |
|----------|-------------|-------------|
| `FeatureKeys.mockUnlimited` | `mock.unlimited` | Unlimited interviews |
| `FeatureKeys.followupsAdaptive` | `ai.followups.adaptive` | Adaptive follow-ups |
| `FeatureKeys.aiPersonalityAll` | `ai.personality.all` | All AI personalities |
| `FeatureKeys.audioRecording` | `audio.recording` | Audio recording + replay |
| `FeatureKeys.transcriptFull` | `transcript.full` | Full transcript |
| `FeatureKeys.feedbackDeepdive` | `feedback.deepdive` | Deep-dive per-question feedback |
| `FeatureKeys.resumeAtsDetailed` | `resume.ats.detailed` | Full ATS report |
| `FeatureKeys.resumeAiRecommend` | `resume.ai_recommendations` | AI resume recommendations |
| `FeatureKeys.performanceAdvanced` | `performance.advanced` | Advanced analytics |
| `FeatureKeys.readinessRadar` | `readiness.radar` | Full radar chart |
| `FeatureKeys.dailyDrillUnlimited` | `daily_drill.unlimited` | Unlimited daily drill |
| `FeatureKeys.learningAdaptive` | `learning.adaptive` | Adaptive roadmap |
| `FeatureKeys.adsDisable` | `ads.disable` | Ad-free experience |

### ELITE tier / Add-ons
| Constant | Feature Key | Description |
|----------|-------------|-------------|
| `FeatureKeys.mockCompanySpecific` | `mock.company_specific` | Company-specific packs |
| `FeatureKeys.mockJdTailored` | `mock.jd_tailored` | JD-based interview tailoring |
| `FeatureKeys.reviewHuman` | `review.human` | Human coach review |

---

## UsageLimitKeys Reference

| Constant | Rule Key | Window | FREE Limit | PRO Limit |
|----------|----------|--------|------------|-----------|
| `UsageLimitKeys.mockInterviews` | `mock.interviews` | weekly | 1 | unlimited |
| `UsageLimitKeys.mockQuestions` | `mock.questions` | session | 3 | unlimited |
| `UsageLimitKeys.dailyDrillQuestions` | `daily_drill.questions` | daily | 1 | unlimited |

---

## Providers Summary

| Provider | Type | Use Case |
|----------|------|----------|
| `accessProvider` | `AsyncValue<UserAccessState>` | Full async state (loading/error/data) |
| `accessStateProvider` | `UserAccessState` | Sync access, returns `loading` while fetching |
| `adsEnabledProvider` | `bool` | Direct ad flag check |
| `userTierProvider` | `String` | Current tier string ("FREE"/"PRO"/"ELITE") |
| `isTrialActiveProvider` | `bool` | Whether trial is in progress |

---

## UserAccessState API

```dart
final state = ref.watch(accessStateProvider);

// Tier checks
state.tier           // → "FREE" | "PRO" | "ELITE"
state.isFree         // → bool
state.isPro          // → bool
state.isElite        // → bool
state.isPaidUser     // → tier == PRO || ELITE
state.isTrialActive  // → bool
state.shouldShowAds  // → bool (same as adsEnabled)
state.addonsActive   // → List<String>

// Feature access (use FeatureGate in UI instead)
state.canAccess(FeatureKeys.feedbackDeepdive) // → bool

// Usage limits (use UsageLimitGuard in UI instead)
state.getUsageLimit(UsageLimitKeys.mockInterviews)          // → UsageLimitEntry
state.isAtLimit(UsageLimitKeys.dailyDrillQuestions)         // → bool
state.remainingUsage(UsageLimitKeys.mockInterviews)         // → int (-1 = unlimited)
state.currentUsage(UsageLimitKeys.dailyDrillQuestions)      // → int
```

---

## Supabase Project Info

| Key | Value |
|-----|-------|
| Project ID | `qogderhgxrhyuptimxsi` |
| Region | `ap-south-1` (Mumbai) |
| Edge Function | `resolve-user-access` (v2) |
| Function auth | `verify_jwt: true` |

---

## Database Functions (server-side only)

| Function | Caller | Purpose |
|----------|--------|---------|
| `can_start_interview(user_id)` | authenticated | Check weekly interview quota before starting |
| `can_do_daily_drill(user_id)` | authenticated | Check daily drill quota |
| `increment_interview_usage(user_id)` | service_role only | Atomic increment at interview creation |
| `increment_daily_drill_usage(user_id)` | service_role only | Atomic increment at drill start |

---

## Don't Do This ❌

```dart
// ❌ Hardcoded tier check
if (ref.read(userTierProvider) == 'PRO') { ... }

// ❌ Raw string feature key
FeatureGate(featureKey: 'feedback.deepdive', child: ...)

// ❌ Direct canAccess without FeatureGate in UI
if (state.featuresEnabled.contains('mock.unlimited')) { ... }

// ❌ Checking usage without UsageLimitGuard in UI
if (state.usageLimits['mock.interviews']?.remaining == 0) { ... }
```

## Do This ✅

```dart
// ✅ Tier gate via widget
FeatureGate(featureKey: FeatureKeys.feedbackDeepdive, child: ...)

// ✅ Quota gate via widget
UsageLimitGuard(ruleKey: UsageLimitKeys.mockInterviews, child: ...)

// ✅ Programmatic tier check (in controller/notifier only)
if (access.canAccess(FeatureKeys.audioRecording)) { ... }

// ✅ Programmatic quota check (in controller/notifier only)
if (access.isAtLimit(UsageLimitKeys.mockInterviews)) { ... }

// ✅ Ad gate via widget
AdGate(child: BannerAdWidget())
```
