Features Documented with Validation Scenarios
🟦 Daily Drill — Quota + Ad Bonus (4 scenarios)
# Google Auth Sign 
# Drill 
#	Scenario	Key Assertion
D-1	First reveal (FREE)	record-drill-reveal fires; usage_count = 1; ad CTA appears
D-2	Watch ad for bonus	bonus_count += 1; effective limit = 2; next Q loads
D-3	PRO/ELITE — no gate	limit = -1; no ad CTA ever shown
D-4	Daily reset	New window_start = today; usage_count = 0 again
🟦 Interview Setup — Quota Chip (3 scenarios)
#	Scenario	Key Assertion
I-1	FREE with remaining	🟢 green pill showing 1 of 1 remaining
I-2	FREE at limit	🟡 amber banner + Upgrade → link + modal blocks
I-3	PRO — chip hidden	SizedBox.shrink() rendered, zero visual clutter
🟦 Free Trial Activation (4 scenarios)
#	Scenario	Key Assertion
T-1	Eligible FREE user	EF → RPC → trial_active = true; purple SnackBar; features unlock
T-2	Already used trial	Card not shown; only comparison table visible
T-3	PRO user	Card not shown (isFree = false)
T-4	Network error	Red SnackBar with reason; DB unchanged (atomic rollback)
🗄️ Also included:
Live DB schema for usage_limit_rules, user_usage_tracking, user_subscriptions
All 4 Edge Function contracts (req/res shapes)
6 SQL validation queries ready to paste into Supabase SQL Editor (including a dev-only trial reset query)
Dart constants quick reference for feature keys and access state API
