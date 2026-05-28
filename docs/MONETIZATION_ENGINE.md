# InterviPrep — Monetization Engine

**Version:** 2.0  
**Updated:** 2026-02-25  
**Status:** ✅ Production — Deployed to Supabase (project: `qogderhgxrhyuptimxsi`)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Business Rules](#2-business-rules)
3. [Architecture Diagram](#3-architecture-diagram)
4. [Database Schema](#4-database-schema)
5. [Row Level Security Policies](#5-row-level-security-rls-policies)
6. [Edge Function — resolve-user-access](#6-edge-function--resolve-user-access)
7. [Access Resolution Algorithm](#7-access-resolution-algorithm)
8. [Flutter Architecture](#8-flutter-architecture)
9. [File Reference Map](#9-file-reference-map)
10. [Usage Guide — How to Gate Features](#10-usage-guide--how-to-gate-features)
11. [Usage Guide — How to Gate Ads](#11-usage-guide--how-to-gate-ads)
12. [In-App Purchase (IAP) Implementation](#12-in-app-purchase-iap-implementation)
13. [Google Ads Configuration & lifecycle](#13-google-ads-configuration--lifecycle)
14. [Extending the System](#14-extending-the-system)
15. [Android Build Configuration](#15-android-build-configuration)

---

## 1. Overview

InterviPrep uses a **server-driven monetization engine** where all access decisions are computed on the Supabase backend and delivered to Flutter as a resolved state object. The Flutter app contains **zero business logic** about plans, tiers, trial periods, or ad rules. It only reads the computed result.

**Key design principles:**
- All feature gating logic lives in one place: the `resolve-user-access` Edge Function
- Flutter widgets consume feature flags (strings), never tier names (e.g., avoid `if tier == 'PRO'`)  
- Adding a new feature requires no UI file changes — only a new DB row and a new constant
- Trial, subscription, and add-on logic is fully composable and independently managed

---

## 2. Business Rules

### Subscription Tiers

| Tier  | Monthly Price | Quarterly Price | Ads | Features |
|-------|---------------|-----------------|-----|----------|
| FREE  | $0            | $0              | ✅ Shown | FREE features only |
| PRO   | ₹899 (~$10)   | ₹2299 (~$28)    | ❌ Hidden | FREE + PRO features |
| ELITE | ₹1799 (~$22)  | ₹4499 (~$55)    | ❌ Hidden | FREE + PRO + ELITE features |

### Trial Rules

- Every new user receives a **3-day ELITE trial** automatically on account creation
- Trial gives the user **ELITE-level features** during the period
- Trial **never removes ads** — FREE users on trial still see ads
- Trial is recorded in `user_trials` and is **immutable** (used once per user, ever)

### Add-on Rules

- Users can purchase add-on packages independently of their subscription
- Add-ons **merge features on top** of the subscription
- An add-on with `disables_ads = true` removes ads for that user
- Add-ons can have expiry dates or be lifetime (no expiry)

### Ad Display Matrix

| User State                      | Ads Shown? |
|---------------------------------|------------|
| FREE (no add-ons)               | ✅ Yes |
| FREE + Trial active             | ✅ Yes (trial never removes ads) |
| FREE + `ads_free` add-on active | ❌ No |
| PRO subscription                | ❌ No |
| ELITE subscription              | ❌ No |

---

## 3. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     FLUTTER APP                             │
│                                                             │
│  App startup                                                │
│  └─▶ accessProvider (AsyncNotifier)                         │
│       └─▶ AccessService.resolveAccess()                     │
│            └─▶ [HTTP POST] resolve-user-access              │
│                                                             │
│  Widget tree consumes:                                      │
│  ┌─────────────────┐  ┌──────────────────┐                  │
│  │   FeatureGate   │  │     AdGate       │                  │
│  │ featureKey: str │  │ child: adWidget  │                  │
│  │ child: widget   │  │                  │                  │
│  └────────┬────────┘  └────────┬─────────┘                  │
│           │                    │                             │
│           └──── reads ─────────┘                             │
│                     │                                        │
│             accessStateProvider                              │
│             (UserAccessState)                                │
└───────────────────────┬─────────────────────────────────────┘
                        │ JWT-authenticated POST
┌───────────────────────▼─────────────────────────────────────┐
│              SUPABASE EDGE FUNCTION                          │
│              resolve-user-access                             │
│                                                             │
│  1. Verify JWT → get user_id                                 │
│  2. Parallel fetch:                                          │
│     ├── features (all active feature keys + tier mapping)    │
│     ├── user_subscriptions (plan, status, expiry)            │
│     ├── user_trials (trial expiry)                           │
│     └── user_addons (active add-ons + their features/ads)    │
│  3. Run access resolution algorithm                          │
│  4. Return AccessResponse JSON                               │
└───────────────────────┬─────────────────────────────────────┘
                        │ service_role queries (bypass RLS)
┌───────────────────────▼─────────────────────────────────────┐
│                   SUPABASE POSTGRES                          │
│                                                             │
│  subscription_plans  ──▶  features                          │
│  user_subscriptions  ──▶  (tier mapping)                    │
│  user_trials                                                │
│  addons  ──▶  addon_features  ──▶  features                 │
│  user_addons ──▶ addons                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Database Schema

All tables live in the `public` schema.

### `subscription_plans`
Catalog of available subscription tiers. Seeded with FREE, PRO, ELITE.

```sql
CREATE TABLE public.subscription_plans (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          TEXT NOT NULL UNIQUE,       -- "FREE" | "PRO" | "ELITE"
  display_name  TEXT NOT NULL,
  price_monthly NUMERIC(10,2) NOT NULL DEFAULT 0,
  price_yearly  NUMERIC(10,2) NOT NULL DEFAULT 0,
  description   TEXT,
  is_active     BOOLEAN NOT NULL DEFAULT true,
  sort_order    INT NOT NULL DEFAULT 0,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Seeded rows:**
| name  | price_monthly | price_yearly |
|-------|---------------|--------------|
| FREE  | 0             | 0            |
| PRO   | 9.99          | 99.99        |
| ELITE | 19.99         | 199.99       |

---

### `features`
The single source of truth for all feature keys and their minimum required tier.

```sql
CREATE TABLE public.features (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  feature_key    TEXT NOT NULL UNIQUE,     -- e.g. "resume_ats"
  display_name   TEXT NOT NULL,
  description    TEXT,
  min_plan_name  TEXT NOT NULL REFERENCES public.subscription_plans(name),
  is_active      BOOLEAN NOT NULL DEFAULT true,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Seeded feature keys:**

| feature_key             | min_plan | Description |
|-------------------------|----------|-------------|
| `basic_interview`       | FREE     | Standard AI interview sessions |
| `interview_history`     | FREE     | View past interview sessions |
| `daily_drill`           | FREE     | Daily practice questions |
| `basic_profile`         | FREE     | Create and manage profile |
| `resume_ats`            | PRO      | AI-powered ATS resume analysis |
| `performance_trends`    | PRO      | Track score trends over time |
| `voice_settings`        | PRO      | Custom AI voice and accent settings |
| `practice_hub`          | PRO      | Curated video and article library |
| `learning_resources`    | PRO      | Structured learning pathways |
| `interview_feedback`    | PRO      | In-depth per-question AI feedback |
| `advanced_analytics`    | ELITE    | Deep behavioral and skill analytics |
| `ai_custom_personality` | ELITE    | Customize AI interviewer personality |
| `unlimited_interviews`  | ELITE    | Unlimited AI mock interview sessions |
| `linkedin_sync`         | ELITE    | Sync profile from LinkedIn |
| `custom_interview_types`| ELITE    | Specialized interview type config |
| `elite_practice_content`| ELITE    | Premium curated content library |

---

### `user_subscriptions`
One row per user. Tracks their active plan.

```sql
CREATE TABLE public.user_subscriptions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  plan_name     TEXT NOT NULL REFERENCES public.subscription_plans(name),
  status        TEXT NOT NULL DEFAULT 'active'  -- 'active'|'cancelled'|'expired'|'paused'
                CHECK (status IN ('active', 'cancelled', 'expired', 'paused')),
  started_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at    TIMESTAMPTZ,               -- NULL = non-expiring
  cancelled_at  TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (user_id)
);
```

> **Note:** Auto-populated with a FREE subscription on user signup via the `provision_free_subscription` trigger.

---

### `user_trials`
One row per user. Immutable after creation. Tracks the 3-day ELITE trial.

```sql
CREATE TABLE public.user_trials (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  started_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at    TIMESTAMPTZ NOT NULL DEFAULT (now() + INTERVAL '3 days'),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (user_id)
);
```

> **Note:** Auto-populated on signup via the same trigger. Trials cannot be re-granted via this table — one per user.

---

### `addons`
Catalog of purchasable add-on packages.

```sql
CREATE TABLE public.addons (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name           TEXT NOT NULL UNIQUE,     -- e.g. "ads_free"
  display_name   TEXT NOT NULL,
  description    TEXT,
  price          NUMERIC(10,2) NOT NULL DEFAULT 0,
  disables_ads   BOOLEAN NOT NULL DEFAULT false,
  is_active      BOOLEAN NOT NULL DEFAULT true,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Seeded add-ons:**

| name         | price | disables_ads | Description |
|--------------|-------|--------------|-------------|
| `ads_free`   | 4.99  | ✅ true       | Remove all ads permanently |
| `elite_tools`| 14.99 | ❌ false      | Unlock Elite-tier features without upgrading |

---

### `addon_features`
Maps which feature keys each add-on grants. M:N join table.

```sql
CREATE TABLE public.addon_features (
  addon_id    UUID NOT NULL REFERENCES public.addons(id) ON DELETE CASCADE,
  feature_key TEXT NOT NULL REFERENCES public.features(feature_key),
  PRIMARY KEY (addon_id, feature_key)
);
```

---

### `user_addons`
Tracks which add-ons a user has purchased.

```sql
CREATE TABLE public.user_addons (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  addon_id     UUID NOT NULL REFERENCES public.addons(id),
  status       TEXT NOT NULL DEFAULT 'active'   -- 'active'|'expired'|'refunded'
               CHECK (status IN ('active', 'expired', 'refunded')),
  purchased_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at   TIMESTAMPTZ,                     -- NULL = lifetime
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (user_id, addon_id)
);
```

### Auto-provision Trigger

On every new `auth.users` insert, the `provision_free_subscription` trigger (SECURITY DEFINER) creates:
1. A `user_subscriptions` row with `plan_name = 'FREE'`
2. A `user_trials` row with `expires_at = now() + 3 days`

---

## 5. Row Level Security (RLS) Policies

All tables have RLS enabled. Here is a summary:

| Table                | SELECT | INSERT | UPDATE | DELETE |
|----------------------|--------|--------|--------|--------|
| `subscription_plans` | Public (active only) | Blocked | Blocked | Blocked |
| `features`           | Public (active only) | Blocked | Blocked | Blocked |
| `addons`             | Public (active only) | Blocked | Blocked | Blocked |
| `addon_features`     | Public | Blocked | Blocked | Blocked |
| `user_subscriptions` | Owner only | Owner only | Owner only | Blocked |
| `user_trials`        | Owner only | Owner only | Blocked (immutable) | Blocked |
| `user_addons`        | Owner only | Blocked (webhook only) | Blocked | Blocked |

> **Important:** The `resolve-user-access` Edge Function uses the **service_role key** to bypass RLS on all reads. This is intentional — the function runs in a trusted server environment.

---

## 6. Edge Function — `resolve-user-access`

**Slug:** `resolve-user-access`  
**Method:** `POST`  
**Auth:** JWT required (`verify_jwt = true`)  
**Location:** `supabase/functions/resolve-user-access/index.ts`  
**Live URL:** `https://qogderhgxrhyuptimxsi.supabase.co/functions/v1/resolve-user-access`

### Response Shape

```typescript
interface AccessResponse {
  tier: string;                    // "FREE" | "PRO" | "ELITE"
  is_trial_active: boolean;
  trial_expires_at: string | null; // ISO 8601
  subscription_expires_at: string | null;
  features_enabled: string[];      // array of feature_key strings
  features_locked: string[];       // array of feature_key strings
  ads_enabled: boolean;
  addons_active: string[];         // array of addon name strings
}
```

### Example Response (FREE user on trial)

```json
{
  "tier": "FREE",
  "is_trial_active": true,
  "trial_expires_at": "2026-02-25T07:00:00Z",
  "subscription_expires_at": null,
  "features_enabled": [
    "basic_interview", "interview_history", "daily_drill", "basic_profile",
    "resume_ats", "performance_trends", "voice_settings", "practice_hub",
    "learning_resources", "interview_feedback", "advanced_analytics",
    "ai_custom_personality", "unlimited_interviews", "linkedin_sync",
    "custom_interview_types", "elite_practice_content"
  ],
  "features_locked": [],
  "ads_enabled": true,
  "addons_active": []
}
```

### Performance

The function issues **4 parallel database queries** via `Promise.all()` to minimize latency:
1. All active features from `features`
2. User's subscription from `user_subscriptions`
3. User's trial from `user_trials`
4. User's active add-ons + their granted features from `user_addons → addons → addon_features`

---

## 7. Access Resolution Algorithm

This is the exact logic executed server-side. The Flutter app never re-implements this.

```
INPUTS:
  subscription: { plan_name, status, expires_at }
  trial: { expires_at } | null
  addons: [ { name, disables_ads, features: [feature_key] } ]
  all_features: [ { feature_key, min_plan_name } ]

PLAN_RANK = { FREE: 0, PRO: 1, ELITE: 2 }

STEP 1: Determine effective subscription tier
  IF subscription.status == 'active' AND (expires_at is null OR expires_at > now()):
    effectiveTier = subscription.plan_name
  ELSE:
    effectiveTier = 'FREE'
  effectiveTierRank = PLAN_RANK[effectiveTier]

STEP 2: Determine trial boost
  isTrialActive = (trial != null AND trial.expires_at > now())
  IF isTrialActive AND effectiveTier == 'FREE':
    trialTierRank = PLAN_RANK['ELITE']   ← trial boosts features to ELITE
  ELSE:
    trialTierRank = effectiveTierRank    ← no boost (paid users aren't on trial)

STEP 3: Collect add-on grants
  validAddons = addons where status == 'active' AND (expires_at null OR > now())
  addonGrantedFeatures = Set of all feature_keys from validAddonRows
  addonDisablesAds = any(addon.disables_ads for addon in validAddons)

STEP 4: Resolve each feature
  FOR each feature in all_features:
    featureRank = PLAN_RANK[feature.min_plan_name]
    grantedByTier  = featureRank <= trialTierRank
    grantedByAddon = feature.feature_key IN addonGrantedFeatures
    IF grantedByTier OR grantedByAddon:
      → featuresEnabled.add(feature.feature_key)
    ELSE:
      → featuresLocked.add(feature.feature_key)

STEP 5: Resolve ads
  adsEnabled = (effectiveTierRank < PLAN_RANK['PRO']) AND NOT addonDisablesAds
  NOTE: Trial does NOT affect ads. Only effectiveTier and add-ons do.

RETURN AccessResponse
```

---

## 8. Flutter Architecture

### Layer Overview

```
lib/core/
├── access/
│   └── access.dart              ← Barrel export (single import for everything)
├── constants/
│   └── feature_keys.dart        ← Compile-time string constants for all features
├── models/
│   └── user_access_state.dart   ← Immutable data model
├── services/
│   └── access_service.dart      ← Edge Function HTTP caller
├── providers/
│   └── access_provider.dart     ← Riverpod state management
└── widgets/
    ├── feature_gate.dart        ← FeatureGate, LockedOverlayGate, UpgradePromptCard
    └── ad_gate.dart             ← AdGate, AdGateBuilder, AdSlotPlaceholder
├── services/
│   └── iap/
│       ├── iap_product_ids.dart ← Store ID manifest
│       ├── iap_models.dart      ← IapState, PurchaseResult
│       └── purchase_service.dart ← InAppPurchase wrapper
├── providers/
│   ├── purchase_provider.dart   ← Purchase stream manager
│   ├── ad_manager_provider.dart ← Ad lifecycle coordinator
│   └── ...
```

---

### `UserAccessState` (Model)

**File:** `lib/core/models/user_access_state.dart`

The immutable snapshot returned from the Edge Function and held in state.

```dart
class UserAccessState {
  final String tier;                   // "FREE" | "PRO" | "ELITE"
  final bool isTrialActive;
  final DateTime? trialExpiresAt;
  final DateTime? subscriptionExpiresAt;
  final Set<String> featuresEnabled;   // Use canAccess(), not this directly
  final Set<String> featuresLocked;
  final bool adsEnabled;
  final List<String> addonsActive;

  // Primary access query methods — use these in code
  bool canAccess(String featureKey);   // ← USE THIS for feature checks
  bool get shouldShowAds;              // ← USE THIS for ad checks
  bool get isPaidUser;
  bool get isElite;
  bool get isPro;
  bool get isFree;

  // Safe fallback during loading
  static const UserAccessState loading;
}
```

---

### `FeatureKeys` (Constants)

**File:** `lib/core/constants/feature_keys.dart`

Compile-time constants mirroring the `features.feature_key` column in Supabase. **Never use raw strings** in feature gate checks.

```dart
abstract final class FeatureKeys {
  // FREE tier
  static const String basicInterview    = 'basic_interview';
  static const String interviewHistory  = 'interview_history';
  static const String dailyDrill        = 'daily_drill';
  static const String basicProfile      = 'basic_profile';

  // PRO tier
  static const String resumeAts         = 'resume_ats';
  static const String performanceTrends = 'performance_trends';
  static const String voiceSettings     = 'voice_settings';
  static const String practiceHub       = 'practice_hub';
  static const String learningResources = 'learning_resources';
  static const String interviewFeedback = 'interview_feedback';

  // ELITE tier
  static const String advancedAnalytics    = 'advanced_analytics';
  static const String aiCustomPersonality  = 'ai_custom_personality';
  static const String unlimitedInterviews  = 'unlimited_interviews';
  static const String linkedinSync         = 'linkedin_sync';
  static const String customInterviewTypes = 'custom_interview_types';
  static const String elitePracticeContent = 'elite_practice_content';
}

abstract final class AddonKeys {
  static const String adsFree    = 'ads_free';
  static const String eliteTools = 'elite_tools';
}
```

---

### `AccessProvider` (State Management)

**File:** `lib/core/providers/access_provider.dart`

```dart
// Primary async provider — handles loading/error states
final accessProvider = AsyncNotifierProvider<AccessNotifier, UserAccessState>(...)

// Convenience sync provider — returns UserAccessState.loading while fetching
final accessStateProvider = Provider<UserAccessState>(...)

// Derived convenience providers
final adsEnabledProvider    = Provider<bool>(...)
final userTierProvider      = Provider<String>(...)
final isTrialActiveProvider = Provider<bool>(...)
```

**Key behaviours:**
- Automatically re-fetches when `authStateProvider` changes (login/logout)
- `refresh()` method forces a re-fetch after purchases
- Unauthenticated users get a fully locked state (no features, ads on)

---

### `AccessService`

**File:** `lib/core/services/access_service.dart`

Single-responsibility: calls the Edge Function and parses the response. No state, no caching.

```dart
class AccessService {
  Future<UserAccessState> resolveAccess() async { ... }
}

class AccessServiceException implements Exception {
  final String message;
  final int? statusCode;
}
```

---

### App Startup Warm-up

**File:** `lib/app.dart`

The root `App` widget eagerly subscribes to `accessProvider`:

```dart
// In App.build():
ref.watch(accessProvider); // warm-up: resolves in parallel with first screen render
```

This ensures the access state is resolved during the initial load screen, not lazily when the first `FeatureGate` is encountered.

---

## 9. File Reference Map

| File | Purpose |
|------|---------|
| `lib/core/access/access.dart` | Barrel export — single import for all access system components |
| `lib/core/models/user_access_state.dart` | Immutable access state model |
| `lib/core/constants/feature_keys.dart` | All feature key constants |
| `lib/core/services/access_service.dart` | Edge Function HTTP client |
| `lib/core/providers/access_provider.dart` | Riverpod AsyncNotifier + convenience providers |
| `lib/core/widgets/feature_gate.dart` | FeatureGate, LockedOverlayGate, UpgradePromptCard |
| `lib/core/widgets/ad_gate.dart` | AdGate, AdGateBuilder, AdSlotPlaceholder |
| `lib/core/services/iap/purchase_service.dart` | Core IAP lifecycle manager |
| `lib/core/providers/purchase_provider.dart` | Riverpod notifier for IAP state |
| `lib/features/subscription/screens/paywall_screen.dart` | Store UI / Checkout flow |
| `supabase/functions/resolve-user-access/index.ts` | Access resolution (local mirror) |
| `supabase/functions/verify-purchase/index.ts` | Receipt verification logic |

**Supabase (Cloud):**
| Resource | Details |
|----------|---------|
| Project ID | `qogderhgxrhyuptimxsi` |
| Edge Function | `resolve-user-access` (Access resolution) |
| Edge Function | `verify-purchase` (Receipt verification) |
| DB Tables | `subscription_plans`, `features`, `user_subscriptions`, `user_trials`, `addons`, `addon_features`, `user_addons`, `iap_purchases` |
| Trigger | `provision_free_subscription` on `auth.users` insert |

---

## 10. Usage Guide — How to Gate Features

### Import (single line)

```dart
import 'package:ai_interview_coach/core/access/access.dart';
```

### Option A: Silent Gate (hide the widget if locked)

The simplest pattern. If the user doesn't have access, nothing renders.

```dart
FeatureGate(
  featureKey: FeatureKeys.resumeAts,
  child: const ResumeAtsButton(),
)
```

### Option B: Custom Locked Widget (show upgrade CTA)

```dart
FeatureGate(
  featureKey: FeatureKeys.advancedAnalytics,
  child: const AdvancedAnalyticsChart(),
  lockedBuilder: (context) => const UpgradePromptCard(
    title: 'Advanced Analytics is an Elite Feature',
    subtitle: 'Upgrade to Elite to unlock deep performance insights.',
    ctaLabel: 'Upgrade to Elite',
    onUpgradeTap: () => context.push('/upgrade'),
  ),
)
```

### Option C: Teaser Overlay (show dimmed content with lock icon)

Great for subscription page teasers where you want users to see what they're missing.

```dart
FeatureGate(
  featureKey: FeatureKeys.elitePracticeContent,
  showLockedOverlay: true,
  child: const ElitePracticeCard(),  // shown at 35% opacity with lock icon
)
```

### Option D: Programmatic check (in logic/controllers)

```dart
final access = ref.read(accessStateProvider);
if (access.canAccess(FeatureKeys.unlimitedInterviews)) {
  // Start unlimited interview
} else {
  // Show upgrade prompt
}
```

### Option E: After a purchase — refresh access

```dart
// After payment webhook confirms, or after your purchase flow completes:
await ref.read(accessProvider.notifier).refresh();
```

---

## 11. Usage Guide — How to Gate Ads

Wrap any ad widget with `AdGate`. The gate reads the server-resolved `adsEnabled` flag.

### Banner Ad

```dart
AdGate(
  child: BannerAdWidget(adUnitId: AdUnits.homeScreenBanner),
)
```

### Interstitial Ad Trigger

```dart
AdGate(
  child: InterstitialAdTrigger(onShow: () { ... }),
)
```

### Ad with custom replacement (e.g., promotional content for paid users)

```dart
AdGate(
  replacementWidget: const PremiumUserPromoCard(),
  child: BannerAdWidget(adUnitId: AdUnits.practiceHubBanner),
)
```

### Builder variant (when you need BuildContext inside the slot)

```dart
AdGateBuilder(
  builder: (context) {
    final screenWidth = MediaQuery.of(context).size.width;
    return BannerAdWidget(width: screenWidth);
  },
)
```

### Layout shift prevention during loading

When access state is loading, `AdGate` renders `SizedBox.shrink()`. To prevent layout
shift, reserve space explicitly:

```dart
// Reserve the banner slot height even while loading
const AdSlotPlaceholder(height: 60)
// … then replace with the actual AdGate once loaded
AdGate(child: BannerAdWidget())
```

---

## 12. In-App Purchase (IAP) Implementation

InterviPrep uses the `in_app_purchase` plugin for cross-platform billing. Crucially, the system enforces **Server-Side Verification** — the app never trusts the local store result.

### Implementation Workflow

1.  **Boot Phase**: `PurchaseService` initializes on app start (warmed up in `app.dart`). It queries both Google Play and App Store for current localized prices.
2.  **Purchase Initiation**: User clicks a buy button. `PurchaseService` triggers the OS billing sheet.
3.  **Update Stream**: All updates (Pending -> Purchased/Error) are caught by a long-lived stream listener in `PurchaseService`. 
4.  **Verification**: 
    - On successful payment, the app sends the receipt (JSON on Android, base64 on iOS) to the `verify-purchase` Supabase Edge Function.
    - The Edge Function validates the receipt with the respective store API.
    - Once verified, the backend updates `user_subscriptions` or `user_addons`.
    - Entitlement records are duplicated into the `iap_purchases` table for audit.
5.  **Completion**: 
    - After the backend returns 200 OK, the app calls `completePurchase()` on the OS billing SDK to consume/finalize the transaction.
    - `accessProvider` is invalidated, triggering a global features sync.

### Key Files
- `lib/core/services/iap/purchase_service.dart`: The brain of the IAP flow.
- `lib/core/providers/purchase_provider.dart`: Riverpod state manager for the paywall.
- `lib/features/subscription/screens/paywall_screen.dart`: The multi-plan checkout UI.

### Critical Safety: Network Failure during Verification
If a user pays but the network fails before the app can reach the `verify-purchase` function, the app **does not** call `completePurchase()`. The OS billing SDK will keep the purchase in its local queue and re-trigger the update stream next time the app opens, ensuring no "lost" sales.

---

## 13. Google Ads Configuration & Lifecycle

Ads are managed via the `google_mobile_ads` plugin and coordinated by the `AdManagerProvider`.

### Zero-Footprint for Paid Users
To maintain performance and privacy for premium users, the Google Ads SDK is **never** initialized if the `UserAccessState` indicates a Pro or Elite tier. The `AdManagerProvider` watches the access state and only calls `AdService.init()` if `adsEnabled` is true.

### Session Frequency Caps
To prevent ad fatigue, the `AdService` enforces a session-level impression cap of **3 ads** (banners, interstitials, or rewarded). Once reached, `createBannerAd()` returns null, and `AdGate` automatically hides the slot.

### Rewarded Ad Pre-loading
Rewarded ads are pre-loaded immediately after SDK initialization and re-loaded as soon as one is shown. This ensures a zero-latency experience when a user opts to watch an ad for a reward (e.g., unlocking a question revealed in Daily Drill).

### Consent Management (UMP)
Complies with GDPR and CCPA by integrating the User Messaging Platform (UMP) SDK.
- Consent form is requested on first initialization for FREE users.
- `AdRequest` is automatically configured with `nonPersonalizedAds: true` if consent is not explicitly obtained.

---

## 14. Extending the System

### Adding a new tier (e.g., "ENTERPRISE")

1. Insert into `subscription_plans` in Supabase
2. Update `PLAN_RANK` in `supabase/functions/resolve-user-access/index.ts`
3. Re-deploy the Edge Function: `supabase functions deploy resolve-user-access`
4. No Flutter code changes needed

### Adding a new feature key

1. Insert a row into the `features` table:
   ```sql
   INSERT INTO public.features (feature_key, display_name, description, min_plan_name)
   VALUES ('my_new_feature', 'My New Feature', 'Description', 'PRO');
   ```
2. Add the constant to `lib/core/constants/feature_keys.dart`:
   ```dart
   static const String myNewFeature = 'my_new_feature';
   ```
3. Use `FeatureGate(featureKey: FeatureKeys.myNewFeature, ...)` in UI
4. No other file changes needed

### Adding a new add-on

1. Insert into `addons` with the desired `disables_ads` and `price`
2. Insert `addon_features` rows mapping which feature keys the add-on grants
3. Implement your payment webhook to insert into `user_addons` on purchase
4. No Flutter code changes needed

### Granting a user a specific plan (e.g., post-payment)

```sql
-- Update an existing subscription to PRO
UPDATE public.user_subscriptions
SET plan_name = 'PRO',
    status    = 'active',
    started_at = now(),
    expires_at = now() + INTERVAL '30 days'
WHERE user_id = '<user_uuid>';
```

Then call `ref.read(accessProvider.notifier).refresh()` in Flutter after confirming payment.

---

## 13. Android Build Configuration

**File:** `android/app/build.gradle.kts`

### Current State (Local Testing RC)

The build is currently configured for **local testing release candidates**:

```kotlin
getByName("release") {
  // LOCAL TESTING: uses debug signing key
  signingConfig = signingConfigs.getByName("debug") //signingConfigs.getByName("release") - for release build

  // LOCAL TESTING: minification disabled (avoids R8 compilation issues)
  isMinifyEnabled = false //true - for release build
  isShrinkResources = false //true - for release build
  proguardFiles(
    getDefaultProguardFile("proguard-android-optimize.txt"),
    "proguard-rules.pro"
  )
}
```

### To Build Local Testing APKs (ARM)

```powershell
flutter build apk --release --target-platform android-arm,android-arm64 --split-per-abi
```

**Output location:**
- `build/app/outputs/flutter-apk/app-armeabi-v7a-release.apk`
- `build/app/outputs/flutter-apk/app-arm64-v8a-release.apk`

### To Prepare for Production Release

1. Configure `android/key.properties` with real keystore credentials
2. Update `build.gradle.kts`:
   - Switch `signingConfig` to `signingConfigs.getByName("release")`
   - Set `isMinifyEnabled = true`
   - Set `isShrinkResources = true`
3. Run: `flutter build appbundle --release` (for Play Store) or the APK command above

### Known Gradle Requirement

- `ndk { abiFilters }` must **not** be present in `defaultConfig` when using `--split-per-abi`
  — they conflict. The flag-based approach via CLI is used instead.
- `compileSdk = 36`, `targetSdk = 36`, `minSdk` from Flutter defaults
- Core library desugaring enabled (`desugar_jdk_libs:2.0.3`)
