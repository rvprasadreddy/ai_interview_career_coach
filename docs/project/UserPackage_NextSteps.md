Phase 1 — Wire the gates into existing screens (~1–2 days)
This is purely Flutter work. No backend changes. You gate the features that already exist.

Screen-by-screen gating map
Based on the routes and screens I've read:

Screen / Feature	Feature Key to Gate	Gate Method
PerformanceTrendsScreen
FeatureKeys.performanceTrends	Route-level redirect in 
routes.dart
PracticeHubScreen	FeatureKeys.practiceHub	Route-level redirect
ResumeAnalysisScreen (ATS)	FeatureKeys.resumeAts	Route-level redirect
VoiceSettingsScreen	FeatureKeys.voiceSettings	Route-level redirect
Profile → "Performance Trends" card	FeatureKeys.performanceTrends	
FeatureGate
 + 
UpgradePromptCard
Profile → "Voice Settings" card	FeatureKeys.voiceSettings	
FeatureGate
 + 
UpgradePromptCard
Home → Interview Feedback section	FeatureKeys.interviewFeedback	
FeatureGate
 with overlay
Interview setup → AI personality	FeatureKeys.aiCustomPersonality	
FeatureGate
 with overlay
Interview setup → unlimited sessions	FeatureKeys.unlimitedInterviews	Programmatic 
canAccess()
 check
Route-level gating pattern (recommended for whole screens)
Add access checks in the GoRouter redirect function in 
routes.dart
:

dart
// In routes.dart redirect:
import '../core/providers/access_provider.dart';
import '../core/constants/feature_keys.dart';
// Add this inside the redirect callback, after auth checks:
final access = ref.read(accessStateProvider);
// Gate full screens at the router level
if (path == Routes.performanceTrends && !access.canAccess(FeatureKeys.performanceTrends)) {
  return Routes.upgrade; // redirect to upgrade page
}
if (path == Routes.dashboard && !access.canAccess(FeatureKeys.resumeAts)) {
  return Routes.upgrade;
}
if (path == Routes.voiceSettings && !access.canAccess(FeatureKeys.voiceSettings)) {
  return Routes.upgrade;
}
Widget-level gating pattern (within screens)
For individual UI sections inside a screen (not full-screen redirects):

dart
// e.g., Inside ProfileScreen, gate the "Performance Trends" list tile:
FeatureGate(
  featureKey: FeatureKeys.performanceTrends,
  lockedBuilder: (_) => const UpgradePromptCard(
    title: 'Performance Trends is a Pro Feature',
    subtitle: 'Track your improvement over time with Pro.',
  ),
  child: ListTile(
    title: const Text('Performance Trends'),
    onTap: () => context.push(Routes.performanceTrends),
  ),
)
Phase 2 — Build the Subscription/Upgrade UI (~2–3 days)
There is currently no screen for users to see their plan or upgrade. This is the most visible gap.

What to build:
A) SubscriptionScreen (/subscription) — Required

Show current tier + trial status ("You're on ELITE trial — 2 days left")
Display all 3 plan cards (FREE / PRO / ELITE) with features list and pricing
Highlight the current plan
"Upgrade" CTA buttons on PRO and ELITE cards
B) AddonsScreen component — Nice to have in v1

Show ads_free and elite_tools add-on cards with pricing
"Purchase" CTA
C) Add route /subscription in 
routes.dart

D) Link from profile screen — add a "Manage Subscription" or "Upgrade" menu item to ProfileScreen

Data for the UI: Read from Supabase's subscription_plans and addons tables (they're public-read, so no auth needed). Or use accessStateProvider to show current plan state.

Phase 3 — Payment Integration (~3–5 days)
This makes the upgrade actually work. You need two parts: in-app purchase on the client, and a server-side webhook to update the database.

A) Add in_app_purchase package
yaml
# pubspec.yaml
in_app_purchase: ^3.2.0
This handles Google Play Billing (Android) and StoreKit (iOS).

B) Create PurchaseService (lib/core/services/purchase_service.dart)
dart
class PurchaseService {
  // Initialize the purchase stream
  // Map product IDs to Supabase plan names
  // Listen for purchase updates
  // On success → call updateSubscription() or updateAddon()
  // On success → call ref.read(accessProvider.notifier).refresh()
}
Product ID mapping:

dart
// Google Play / App Store product IDs must match exactly
const _planProductIds = {
  'com.antigravity.ai_interview_coach.pro_monthly': 'PRO',
  'com.antigravity.ai_interview_coach.elite_monthly': 'ELITE',
};
const _addonProductIds = {
  'com.antigravity.ai_interview_coach.ads_free': AddonKeys.adsFree,
};
C) Create a Supabase Edge Function process-purchase (or use a webhook)
When a purchase is confirmed on the client, you call this function with the receipt token. The function verifies with Google Play / App Store and then writes to user_subscriptions or user_addons.

POST /functions/v1/process-purchase
{
  "product_id": "com.antigravity.ai_interview_coach.pro_monthly",
  "purchase_token": "<receipt from store>",
  "platform": "android" | "ios"
}
The function:

Verifies the receipt with Google Play API or Apple StoreKit
Updates user_subscriptions (sets plan_name, expires_at)
Returns { success: true }
D) After purchase — refresh in Flutter
dart
// In PurchaseService, after DB write confirmed:
await ref.read(accessProvider.notifier).refresh();
// Navigate to subscription confirmation screen
Immediate Priority Order
Here's what to do right now to make immediate visible progress:

1. [TODAY]      Add route-level guards to routes.dart for PRO-gated screens
2. [TODAY]      Gate the Profile screen tiles (Performance Trends, Voice Settings, Resume ATS)
3. [THIS WEEK]  Build SubscriptionScreen + add route /subscription + link from Profile
4. [NEXT WEEK]  Wire in_app_purchase → PurchaseService → process-purchase Edge Function
5. [NEXT WEEK]  Set up Google Play Console products matching your product IDs
Quick wins to ship Phase 1 today
Start with 
routes.dart
 — it gates entire screens without touching individual screen files. Then gate the 3 profile menu items. That's enough to enforce the access model across the app immediately, even before payment is wired. FREE users get properly blocked with a redirect to wherever you want (even a simple "Upgrade coming soon" placeholder screen works temporarily).

Want me to start implementing Phase 1 — the route guards and profile screen gates?