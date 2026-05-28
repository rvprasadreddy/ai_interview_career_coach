# InterviPrep — In-App Purchase System
## Validation Report · Architecture · Production Readiness

> Audited: 2026-03-09 · IAP Plugin: `in_app_purchase ^3.2.0` · Backend: Supabase Edge Functions

---

## Executive Summary

| Layer | Status |
|---|---|
| Flutter IAP Plugin setup | ✅ Complete |
| Product definitions (Dart) | ✅ Fixed (4 → 6 products) |
| Purchase flow & stream handler | ✅ Complete |
| Server-side verification (Android) | ✅ Complete (Play API v3) |
| Server-side verification (iOS) | ✅ Fixed (App Store Server API v2 — replaces deprecated `/verifyReceipt`) |
| DB subscription activation RPC | ✅ Complete |
| DB addon activation | ✅ Complete (upsert/insert based on consumability) |
| DB schema — `iap_purchases` | ✅ Exists |
| DB schema — `subscription_plans` product IDs | ✅ Fixed — added columns via migration |
| DB schema — `addons` product IDs | ✅ Fixed — added columns via migration |
| Paywall UI (PaywallScreen) | ✅ Complete with real IAP |
| SubscriptionScreen purchase calls | ✅ Fixed — was "Coming Soon" stub |
| IAP eager initialization | ✅ Fixed — now warms up on HomeScreen |
| Restore Purchases (iOS) | ✅ Complete |
| Feature gating post-purchase | ✅ Complete (accessProvider.invalidate) |
| **Production credentials** | ❌ Still using dev bypass |

---

## Gaps Found & Fixed

### GAP-1 🔴 SubscriptionScreen Used Placeholder Instead of Real IAP
**Severity:** Critical — users could not buy anything  
**File:** `lib/features/subscription/screens/subscription_screen.dart`

Both `_handleUpgrade()` and `_handleAddonPurchase()` called `_showComingSoonSheet()` which only showed a "contact support" message. The fully-built `PaywallScreen` (with real `purchaseProvider` calls) was never opened.

**Fix:** Replaced both stubs with:
- `_handleUpgrade()` → opens `PaywallScreen.show()` (full IAP UI)
- `_handleAddonPurchase()` → calls `purchaseProvider.notifier.purchase(product)` directly
- Added `ref.listen<IapState>` for success/error SnackBars
- Added in-flight loading overlay via `Stack` + `iapState.isPurchasing`

---

### GAP-2 🔴 iOS Used Deprecated `/verifyReceipt` API (Removed by Apple in 2024)
**Severity:** Critical — iOS purchases would fail in production  
**File:** `supabase/functions/verify-purchase/index.ts`

The original `verifyIos()` function called `buy.itunes.apple.com/verifyReceipt` which Apple deprecated in 2023 and removed in 2024.

**Fix:** Upgraded to **App Store Server API v2** (`api.storekit.itunes.apple.com/inApps/v1/transactions/:id`):
- Uses `ECDSA P-256` JWT authentication (App Store Connect credentials)
- Falls back to sandbox endpoint when production returns error `4040010`
- Decodes signed `JWS` transaction payload to extract expiry
- Updated Flutter payload to include `transaction_id` alongside receipt data

**New env vars required (iOS):**
```
APPLE_PRIVATE_KEY   = <contents of .p8 key file from App Store Connect>
APPLE_KEY_ID        = <Key ID, e.g. ABC1234DEF>
APPLE_ISSUER_ID     = <Issuer ID from App Store Connect>
APPLE_BUNDLE_ID     = <e.g. com.interviprep.app>
```

---

### GAP-3 🟡 Product IDs Missing from `subscription_plans` and `addons` Tables
**Severity:** Medium — EF had hardcoded product ID mapping  
**Migration:** `20260309_iap_product_ids_on_plans_and_addons`

Added `product_id_android`, `product_id_ios`, `billing_period` to `subscription_plans`.  
Added `product_id_android`, `product_id_ios` to `addons`.

Now seedeed:
| Table | Name | Android ID | iOS ID |
|---|---|---|---|
| subscription_plans | PRO (monthly) | `interviprep_pro_monthly` | `interviprep_pro_monthly` |
| subscription_plans | PRO_QUARTERLY | `interviprep_pro_quarterly` | `interviprep_pro_quarterly` |
| addons | jd_company_mock | `interviprep_addon_jd_mock` | `interviprep_addon_jd_mock` |
| addons | resume_rewrite_pro | `interviprep_addon_resume_rewrite` | `interviprep_addon_resume_rewrite` |
| addons | ads_free | `interviprep_addon_ads_free` | `interviprep_addon_ads_free` |
| addons | elite_tools | `interviprep_addon_elite_tools` | `interviprep_addon_elite_tools` |

---

### GAP-4 🟡 Two Add-on Product IDs Missing from Dart Constants
**Severity:** Medium — ads_free and elite_tools never queried from store  
**File:** `lib/core/services/iap/iap_product_ids.dart`

Added `addonAdsFree` and `addonEliteTools` to the Dart constants, and updated `all` and `addons` sets.

---

### GAP-5 🟡 `purchaseProvider` Never Eagerly Initialized
**Severity:** Medium — users saw a spinner when opening paywall  
**File:** `lib/features/home/screens/home_screen.dart`

`PurchaseService.init()` queries product details from the Play/App Store — takes 1-3 seconds. Without eager init, users waited on the paywall. Now `ref.read(purchaseProvider)` in `HomeScreen.initState` warms it up immediately after login. Added `import purchase_provider.dart`.

---

## Complete Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    FLUTTER CLIENT                               │
│                                                                 │
│  HomeScreen.initState()                                         │
│    └─ ref.read(purchaseProvider)  ← eager IAP warm-up          │
│                                                                 │
│  SubscriptionScreen / PaywallScreen                             │
│    └─ ref.read(purchaseProvider.notifier).purchase(product)     │
│                                                                 │
│  PurchaseNotifier (StateNotifier)                               │
│    └─ PurchaseService                                           │
│         ├─ _iap.queryProductDetails([6 product IDs])            │
│         ├─ _iap.purchaseStream.listen(_onPurchaseUpdate)        │
│         └─ _verifyAndComplete(purchase)                         │
│              └─ supabase.functions.invoke('verify-purchase')    │
│                   ├─ payload: { platform, product_id,           │
│                   │            purchase_token / receipt_data,   │
│                   │            transaction_id (iOS), ... }      │
│                   └─ on HTTP 200: completePurchase()            │
│                          └─ accessProvider.invalidate()         │
└─────────────────────────────────────────────────────────────────┘
                               │
                    SUPABASE EDGE FUNCTION
                    verify-purchase (v2)
                               │
        ┌──────────────────────┼──────────────────────┐
        │                                             │
   Android path                               iOS path
        │                                             │
   Google Play API v3                   App Store Server API v2
   subscriptionsv2/tokens/:token        /inApps/v1/transactions/:id
        │                                             │
        └──────────────────────┬──────────────────────┘
                               │
               ┌───────────────▼───────────────┐
               │  DATABASE                      │
               │                               │
               │  iap_purchases.upsert()       │
               │  (audit trail)                │
               │                               │
               │  IF subscription:             │
               │    activate_subscription_     │
               │    from_purchase() RPC        │
               │    → user_subscriptions       │
               │    → users (legacy columns)   │
               │                               │
               │  IF addon:                    │
               │    user_addons.insert()       │
               │    (consumable = multi-row)   │
               │    user_addons.upsert()       │
               │    (non-consumable = 1 row)   │
               └───────────────────────────────┘
```

---

## Database Schema Reference

### `iap_purchases` (audit log)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `user_id` | uuid | FK → auth.users |
| `platform` | text | `'android'` \| `'ios'` |
| `product_id` | text | e.g. `interviprep_pro_monthly` |
| `is_subscription` | boolean | |
| `purchase_token` | text | Android only |
| `transaction_id` | text | iOS only |
| `order_id` | text | Platform order reference |
| `verification_status` | text | `'verified'` \| `'invalid'` |
| `expires_at` | timestamptz | Subscription expiry |
| `raw_response` | jsonb | Full store API response |

### `user_subscriptions` (entitlements)
| Column | Type | Notes |
|---|---|---|
| `user_id` | uuid | unique, FK → auth.users |
| `tier` | text | `'FREE'` \| `'PRO'` \| `'ELITE'` |
| `plan_name` | text | `'PRO'` \| `'PRO_QUARTERLY'` |
| `valid_until` | timestamptz | Subscription expiry |
| `is_active` | boolean | Set to `false` on expiry |
| `status` | text | `'active'` \| `'expired'` \| `'cancelled'` |

### `user_addons`
| Column | Type | Notes |
|---|---|---|
| `user_id` | uuid | |
| `addon_id` | uuid | FK → addons.id |
| `status` | text | `'active'` \| `'expired'` \| `'refunded'` |
| `purchased_at` | timestamptz | |
| UNIQUE | `(user_id, addon_id)` | Prevents duplicate non-consumables |

---

## Product ID Reference

> These IDs must be created **exactly as written** in Google Play Console and App Store Connect.

| Product | Type | Android/iOS ID | Price |
|---|---|---|---|
| Pro Monthly | Subscription | `interviprep_pro_monthly` | ₹249/mo |
| Pro Quarterly | Subscription | `interviprep_pro_quarterly` | ₹599/3mo |
| JD Mock | Consumable | `interviprep_addon_jd_mock` | ₹99 |
| Resume Rewrite | Non-consumable | `interviprep_addon_resume_rewrite` | ₹999 |
| Ad-Free Experience | Non-consumable | `interviprep_addon_ads_free` | ₹499 |
| Elite Tools Pack | Non-consumable | `interviprep_addon_elite_tools` | ₹1499 |

---

## Edge Function: `verify-purchase` (v2)

### Request
```http
POST /functions/v1/verify-purchase
Authorization: Bearer <user-jwt>
Content-Type: application/json
```

**Android body:**
```json
{
  "platform": "android",
  "product_id": "interviprep_pro_monthly",
  "purchase_token": "<play-purchase-token>",
  "package_name": "com.interviprep.app",
  "is_subscription": true
}
```

**iOS body:**
```json
{
  "platform": "ios",
  "product_id": "interviprep_pro_monthly",
  "receipt_data": "<base64-app-receipt>",
  "transaction_id": "<original-transaction-id>",
  "is_subscription": true
}
```

### Response
```json
{ "success": true, "product_id": "interviprep_pro_monthly" }
```

### Error Responses
| Code | Meaning |
|---|---|
| 401 | Missing/invalid JWT |
| 402 | Store verification failed (receipt invalid) |
| 500 | Internal error |

---

## Dart Constants Reference

```dart
// Product IDs
IapProductIds.proMonthly        // 'interviprep_pro_monthly'
IapProductIds.proQuarterly      // 'interviprep_pro_quarterly'
IapProductIds.addonJdMock       // 'interviprep_addon_jd_mock'
IapProductIds.addonResumeRewrite // 'interviprep_addon_resume_rewrite'
IapProductIds.addonAdsFree      // 'interviprep_addon_ads_free'
IapProductIds.addonEliteTools   // 'interviprep_addon_elite_tools'
IapProductIds.all               // Set of all 6
IapProductIds.isSubscription()  // bool check
IapProductIds.isConsumable()    // bool check (only addonJdMock)
IapProductIds.displayName()     // Human-readable name

// Provider
ref.watch(purchaseProvider)     // IapState
ref.read(purchaseProvider.notifier).purchase(product)
ref.read(purchaseProvider.notifier).restore()
ref.read(purchaseProvider.notifier).productById(id)
ref.read(purchaseProvider.notifier).clearError()

// Convenience providers
ref.watch(iapReadyProvider)     // bool — products loaded
ref.watch(iapProductsProvider)  // List<ProductDetails>
```

---

## Validation Scenarios

### Scenario V-1: New User Buys Pro Monthly (Android)

1. User opens `SubscriptionScreen` → `_handleUpgrade()` fires
2. `PaywallScreen.show()` opens (fullscreen dialog)
3. `state.products` shows real prices from Play (loaded since HomeScreen)
4. User taps "Subscribe for ₹249"
5. `purchaseProvider.notifier.purchase(product)` → `_iap.buyNonConsumable()`
6. Google Play sheet appears
7. User confirms → `purchaseStream` fires `PurchaseStatus.purchased`
8. `_verifyAndComplete()` called → POST to `verify-purchase` EF
9. EF calls Play API: `GET subscriptionsv2/tokens/:token`
10. If `SUBSCRIPTION_STATE_ACTIVE` → `iap_purchases.upsert()` + `activate_subscription_from_purchase()`
11. EF returns HTTP 200
12. Flutter: `_iap.completePurchase()` + `accessProvider.invalidate()`
13. `resolve-user-access` EF called → returns tier=PRO
14. UI refreshes → ads disappear, features unlock, `SubscriptionScreen` shows "PRO Active"

### Scenario V-2: iOS User Restores Purchase

1. User taps "Restore Purchases" on `PaywallScreen`
2. `purchaseProvider.notifier.restore()` → `_iap.restorePurchases()`
3. For each restored purchase → `purchaseStream` fires `PurchaseStatus.restored`
4. Same `_verifyAndComplete()` flow as V-1
5. EF activates tier → UI refreshes

### Scenario V-3: Network Error During Verification

1. Purchase verified by Play/App Store
2. Supabase EF call fails (network timeout)
3. `_verifyAndComplete()` catches error
4. **Does NOT call `completePurchase()`** — purchase stays in OS queue
5. On next app launch, `purchaseStream.listen()` re-delivers the pending purchase
6. Verification retried → user's subscription activated without repurchase

### Scenario V-4: User Buys JD Mock Add-on (Consumable)

1. User taps "JD Precision Mock" on `SubscriptionScreen`
2. `productById('interviprep_addon_jd_mock')` finds product
3. `purchase(product)` → `_iap.buyConsumable()`
4. Verification → EF `activateAddon()` → `user_addons.insert()` (new row each time, not upsert)
5. `accessProvider.invalidate()` → `addonsActive` now contains `'jd_company_mock'`
6. Interview Setup screen enables JD-tailored interview mode

### Scenario V-5: Addon Not Yet in Store

1. User taps "Company Power Pack"
2. `productId` resolves to `null` (not in `IapProductIds`)
3. `_showComingSoonSheet()` gracefully shown with contact info

### Scenario V-6: Dev/Sandbox Bypass

If `GOOGLE_SERVICE_ACCOUNT_JSON` / `APPLE_PRIVATE_KEY` are not set in Supabase secrets:
- EF logs a warning and returns `{ valid: true, order_id: "dev_bypass" }`
- Purchase is recorded in `iap_purchases` with `order_id = 'dev_bypass'`
- User tier is activated
- ⚠️ Remove this bypass by setting credentials before production launch

---

## Store Configuration Checklist

### Google Play Console

```
Apps → InterviPrep → Monetize → Subscriptions
  ├─ Create: interviprep_pro_monthly
  │    Base plan: 1 month / ₹249
  │    Free trial: 3 days
  ├─ Create: interviprep_pro_quarterly
  │    Base plan: 3 months / ₹599
  │    Tag: Best Value
  └─ One-time products (Managed Products):
       ├─ interviprep_addon_jd_mock       (₹99, consumable)
       ├─ interviprep_addon_resume_rewrite (₹999, non-consumable)
       ├─ interviprep_addon_ads_free       (₹499, non-consumable)
       └─ interviprep_addon_elite_tools    (₹1499, non-consumable)

Setup:
  • Enable Google Play Billing API in Google Cloud Console
  • Create Service Account → give "Financial data" permission
  • Download JSON key → set as GOOGLE_SERVICE_ACCOUNT_JSON secret in Supabase
  • Set up License Testing accounts for internal QA
```

### App Store Connect

```
Apps → InterviPrep → In-App Purchases
  ├─ Subscription Group: "InterviPrep Pro"
  │    ├─ interviprep_pro_monthly     Auto-renewable, ₹249/month
  │    └─ interviprep_pro_quarterly   Auto-renewable, ₹599/3months
  │         (same group = upgrade/downgrade handled by Apple)
  └─ One-Time Purchases:
       ├─ interviprep_addon_jd_mock       Consumable, ₹99
       ├─ interviprep_addon_resume_rewrite Non-consumable, ₹999
       ├─ interviprep_addon_ads_free       Non-consumable, ₹499
       └─ interviprep_addon_elite_tools    Non-consumable, ₹1499

App Store Server Notifications (required for renewals):
  • URL: https://qogderhgxrhyuptimxsi.supabase.co/functions/v1/iap-webhook
  • (Create a new edge function or extend verify-purchase)

App Store Connect API Keys:
  • Create API key with "App Manager" role
  • Download .p8 file
  • Set secrets in Supabase:
      APPLE_PRIVATE_KEY  = (contents of .p8 file)
      APPLE_KEY_ID       = (key ID)
      APPLE_ISSUER_ID    = (issuer ID)
      APPLE_BUNDLE_ID    = com.interviprep.app
```

---

## Production Go-Live Sequence

```
BACKEND SETUP
─────────────
1. Create Google Cloud Service Account + download JSON
   Set in Supabase: GOOGLE_SERVICE_ACCOUNT_JSON = <JSON>

2. Create App Store Connect API Key (.p8)
   Set in Supabase:
     APPLE_PRIVATE_KEY, APPLE_KEY_ID, APPLE_ISSUER_ID, APPLE_BUNDLE_ID

3. Create all 6 products in Play Console + App Store Connect
   (exact IDs from product table above)

4. Set up App Store Server Notifications endpoint (for renewal webhooks)

ANDROID
────────
5. Add BILLING permission to AndroidManifest.xml:
   <uses-permission android:name="com.android.vending.BILLING" />
   (already present via in_app_purchase plugin)

6. Sign APK with production keystore
7. Upload to internal testing track
8. Add license testing accounts

iOS
────
9. Add SKAdNetwork entries (full Google list)
10. Build with Xcode distribution certificate
11. Upload to TestFlight
12. Add sandbox tester Apple IDs

VALIDATION
───────────
13. Complete Scenario V-1 through V-5 on physical devices
14. Verify iap_purchases table rows created in Supabase
15. Verify user_subscriptions.tier = 'PRO' after purchase
16. Verify accessProvider refresh unlocks Pro features
17. Verify restore purchases flow (iOS)
18. Switch ENVIRONMENT=production in .env

LAUNCH
───────
19. Submit app to Play Store / App Store review
20. Enable subscription products in both stores after approval
```

---

## Pending Work (Production-Only)

| Priority | Item | Effort |
|---|---|---|
| 🔴 P0 | Set `GOOGLE_SERVICE_ACCOUNT_JSON` in Supabase secrets | 30 min |
| 🔴 P0 | Set `APPLE_PRIVATE_KEY/KEY_ID/ISSUER_ID/BUNDLE_ID` | 30 min |
| 🔴 P0 | Create all 6 products in Play Console + App Store Connect | 2-3 hrs |
| 🟡 P1 | Create `iap-webhook` EF for App Store Server Notifications (renewals/cancellations) | 3-4 hrs |
| 🟡 P1 | Handle subscription expiry in `resolve-user-access` (check `valid_until > now()`) | 1 hr |
| 🟡 P1 | Call `adService.logUpgradeAfterAd()` after IAP success | 30 min |
| 🟢 P2 | Add "Company Power Pack" as a real product (currently shows "Coming Soon") | 1 hr |
| 🟢 P2 | Deep-link from interview usage limit hit → `PaywallScreen` with `triggerReason` | 1 hr |
