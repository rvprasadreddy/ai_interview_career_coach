# User Access Resolution System (User Resolve Access)

**Version:** 2.1
**Updated:** 2026-03-18
**Status:** ✅ Production

## 1. Overview
InterviPrep uses a **server-driven access resolution system**. All logic for determining what a user can access (Features, Ads, and Usage Quotas) is computed on the Supabase backend in an Edge Function and delivered to the Flutter app as a resolved state object.

The Flutter app contains **zero business logic** about plans, tier names, or trial periods. It only consumes the computed `UserAccessState`.

---

## 2. The Decision Engine (Basis of Logic)
The `resolve-user-access` Edge Function merges data from four independent pillars to derive its final response.

### Pillar 1: Subscription Tier
Stored in `public.user_subscriptions`.
- **Logic**: IF `status == 'active'` AND (`expires_at` is NULL OR `expires_at > now()`) THEN use `plan_name`. ELSE fallback to `FREE`.
- **Tiers**: `FREE` (Rank 0), `PRO` (Rank 1), `ELITE` (Rank 2).

### Pillar 2: Trial State
Stored in `public.user_trials`.
- **Logic**: Effective Rank is boosted to `ELITE` (Rank 2) for feature permissions only.
- **Rule**: Trial does **not** lift usage quotas or remove ads (unlike a paid Elite subscription).

### Pillar 3: Add-on Entitlements
Stored in `public.user_addons`.
- **Logic**: Add-ons merge features "sideways" into the user's enabled list.
- **Rule**: If any active add-on has `disables_ads = true`, ads are hidden regardless of the user's tier.

### Pillar 4: Usage Limits & Quotas
Stored in `public.usage_limit_rules` and `public.user_usage_tracking`.
- **Logic**: `Remaining = (Base Limit + Bonus Credits) - Used Count`.
- **Bonus Credits**: One-time purchases (e.g., JD Power Pack) increment the `bonus_count` for a specific `rule_key`.

---

## 3. Core Database Columns
To manually manage or debug a user's access, use these columns:

| Entity | Table | Critical Columns |
| :--- | :--- | :--- |
| **Subscription** | `user_subscriptions` | `tier`, `status`, `expires_at`, `plan_name` |
| **Free Trial** | `user_trials` | `expires_at`, `started_at` |
| **Add-ons** | `user_addons` | `addon_id`, `status`, `expires_at` |
| **Usage Tracking** | `user_usage_tracking` | `rule_key`, `usage_count`, `bonus_count` |

---

## 4. Feature Gating Resolution
For every feature defined in the `public.features` table, the system checks:
1. Does the user's **Effective Rank** (Tier/Trial) satisfy the `min_plan_name`?
2. OR, is this `feature_key` explicitly granted by an active `addon_id`?

If either is true, the feature is added to `features_enabled`.

---

## 5. Ad Resolution Logic
Ads are displayed strictly based on this rule:
- **Ads Enabled IF**:
  - `effectiveTierRank < PRO_RANK` (meaning user is basic/free)
  - AND `!addonDisablesAds` (no ad-free add-ons active)

*Note: The 3-day ELITE trial specifically leaves ads enabled.*

---

## 6. Client-Side Implementation (Flutter)
The app receives a JSON payload and maps it to `UserAccessState`.

### Key Components:
- **`FeatureGate`**: Wraps any widget. Hides/overlays content if `featureKey` is not in the enabled list.
- **`AdGate`**: Wraps ad slots. Automatically hides the slot if `adsEnabled` is false.
- **`accessStateProvider`**: The global Riverpod provider holding the current resolved state.

### Purchase Refresh Flow:
1. User completes a purchase via `PurchaseService`.
2. Backend updates the database and returns success.
3. Flutter calls `accessProvider.notifier.refresh()`.
4. The app re-calls the Edge Function, ensuring the new Pro features are unlocked instantly.
