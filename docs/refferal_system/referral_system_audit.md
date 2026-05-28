# Referral System Audit: "Refer a Friend"
**Last Updated:** 2026-04-06

## 1. Feature Status Summary
The Referral System is now **Fully Operational**. The core loop (Share → Sign-up → Reward) is implemented with anti-fraud measures and dynamic configurations.

| Component | Status | Description |
| :--- | :--- | :--- |
| **Database Schema** | ✅ Complete | Tables for `referrals` and `referral_code` columns are in place. |
| **Backend Logic** | ✅ Complete | `apply_referral_code` RPC handles atomic rewards and returns notification context. |
| **Edge Function** | ✅ Complete | `apply-referral` acts as secure gateway and triggers **FCM Referrer Notifications**. |
| **Onboarding Flow** | ✅ Complete | Dynamic pre-fill from deep links and **Celebratory Success Dialog**. |
| **Post-Onboarding** | ✅ Complete | "Gifts & Referrals" with **Celebratory Success Dialog**. |
| **Dynamic Config** | ✅ Complete | `RemoteConfigService` controls reward durations (7 days default). |
| **Share Messaging** | ✅ Complete | WhatsApp and generic system share integrated with dynamic messaging. |

---

## 2. Technical Architecture

### Database & Logic (Supabase)
- **RPC `apply_referral_code`**: 
    - **Tier-Aware Rewards**: Automatically detects if the referrer is a `PAID` or `FREE` user.
    - **Notification Context**: Now returns `referrer_id`, `friend_name`, and `bonus_days` for the Edge Function to use.
- **Anti-Fraud Layer**:
    - **Self-Referral**: Blocked via `v_referrer_id = p_referred_user_id` check.
    - **Account Limit**: 1 referral code redemption per account (`UNIQUE(referred_user_id)`).
    - **Device Limit**: 1 referral redemption per physical device ID (`device_id` check).

### Implementation Details
- **`apply-referral` Edge Function**: Located at `supabase/functions/apply-referral`. 
    - **FCM Notifications**: Integrated with Google Cloud FCM to notify the referrer immediately after their code is used.
- **`ReferralSuccessDialog`**: A celebratory Flutter widget with confetti and premium status feedback.

---

## 3. Verified Functionality

### Phase 1: Referrer Loop
- **Instant Notifications**: Referrers get a push notification ("🎁 Referral Bonus!") the moment their friend's code is accepted.
- **Referral Screen**: Displays dynamic reward text.
- **Stats**: Real-time "Friends Referred" count.

### Phase 2: Friend Loop
- **Celebratory Feedback**: Friends see a high-polish `ReferralSuccessDialog` with confetti after successfully redeeming a code.
- **Deep Links**: Tapping a link sets the app's referral state.

---

## 4. Remaining Gaps & Known Potential Issues

### 🔴 High Priority: Notification Gap
- **Referrer Notifications**: Currently, the referrer is **not notified** when a friend successfully uses their code. They only see the change if they manually check their referral stats or "Trial Remaining" count.
- **Recommendation**: Implement a Supabase Database Webhook on the `referrals` table to trigger a Push Notification via the `send-notification` Edge Function.

### 🟡 Medium Priority: User Experience
- **Success Feedback**: Redemption currently results in a simple SnackBar. For a "win" event, a more celebratory UI (e.g., Confetti effect or a "Premium Unlocked" dialog) would improve retention.
- **Trial Visibility**: The Home screen shows a referral nudge for free users, but there is no specific "Referral Bonus Active" badge in the main UI to remind users WHY they have premium.

### 🟢 Low Priority: Monitoring
- **Admin Dashboard**: While the logic is secure, there is no admin UI to view `referrals` stats globally (top referrers, total bonus days granted).
- **Audit Logging**: Successful and failed (fraud-blocked) attempts should be logged to a dedicated audit table for future analysis.

---

## 5. Bug Watchlist
- **Migration Drift**: Ensure the `user_subscriptions` table in production has the `is_trial_active` and `trial_expires_at` columns indexed for performance if the user base exceeds 10k+.
- **Case Sensitivity**: The RPC uses `UPPER(TRIM())`, which is robust, but the frontend should still normalize input to uppercase for visual consistency.
