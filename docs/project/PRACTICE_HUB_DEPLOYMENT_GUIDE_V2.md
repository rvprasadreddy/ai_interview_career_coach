# 🚀 Practice Hub: Step-by-Step Deployment Guide (Fresh Install)

This guide ensures a 100% clean deployment of the Practice Hub feature (Phases 1-3). It clears all partial implementation artifacts and provisions a production-ready environment.

---

## 🛠️ Prerequisites

1.  **Supabase CLI**: Installed and logged in (`supabase login`).
2.  **Environment Variables**: Ensure `OPENAI_API_KEY` is set in your Supabase secrets.
    ```bash
    supabase secrets set OPENAI_API_KEY=sk-...
    ```
3.  **Project ID**: Have your Supabase reference ID ready.

---

## 📂 Key Deployment Files

- **`supabase/migrations/PRACTICE_HUB_FULL_RESET.sql`**: The master schema file.
- **`supabase/functions/learning-recommendations/index.ts`**: The AI engine.
- **`lib/features/practice/`**: The Flutter UI implementation.

---

## ⏱️ Deployment Steps

### Step 1: Database Clean Slate & Schema Provisioning
Execute the master reset script in the Supabase SQL Editor (Dashboard) or via CLI if managed. This script drops all previous Practice Hub objects and rebuilds them.

**Action**: Copy content from `supabase/migrations/PRACTICE_HUB_FULL_RESET.sql` into the Supabase SQL Editor and run it.

**What it does**:
- Drops all old tables (`learning_content`, `user_learning_progress`, etc.).
- Re-creates 5 optimized tables with RLS.
- Provisions 4 critical RPC functions (`get_user_tier`, `start_learning_content`, etc.).
- Sets up PG triggers for automated roadmap refreshes.

---

### Step 2: Provision Content Library
If you need the full 100+ items library beyond the basic seed in the reset script, run the expansion script.

**Action**: Run `supabase/migrations/20260214_expand_content_library.sql` in the SQL Editor.

---

### Step 3: Deploy AI Edge Function
Deploy the recommendation engine to Supabase. This function handles the logic for both single users and batch processing.

**Action**:
```bash
supabase functions deploy learning-recommendations
```

**Verification**:
Go to **Functions** in Supabase and ensure `learning-recommendations` is active and the URL is available.

---

### Step 4: Verify Auth & Tiers
The Practice Hub relies on user subscriptions. The reset script automatically provisions the first user as 'paid' for testing.

**Action**:
Run this query to check status:
```sql
SELECT u.email, s.tier, s.is_active 
FROM auth.users u
LEFT JOIN user_subscriptions s ON u.id = s.user_id;
```

---

### Step 5: Flutter App Sync
Ensure the Flutter app is configured to point to these new objects.

1.  **Run Code Gen**:
    ```bash
    flutter pub run build_runner build --delete-conflicting-outputs
    ```
2.  **Clean Cache**:
    ```bash
    flutter clean && flutter pub get
    ```

---

## 🧪 Post-Deployment Verification (Smoke Test)

1.  **Check Empty State**: Log in with a user who has 0 interviews. The Hub should show "Keep Interviewing!".
2.  **Generate Roadmaps**: Complete an interview.
    - Paid users: Should see an auto-refresh triggered.
    - Free users: Should see recommendations after first pull.
3.  **Video Playback**: Tap a video card. Ensure the native player (Mobile) or iframe (Web) loads.
4.  **Feedback Loop**: Tap Thumbs Up/Down on a card. Verify entry in `content_engagement` table.

---

## 🆘 Troubleshooting

- **"RPC Function not found"**: Ensure you ran `PRACTICE_HUB_FULL_RESET.sql`. RPCs like `start_learning_content` must exist.
- **"Edge Function Timeout"**: Batch processing is limited to 50 users. If processing many users, trigger in smaller batches.
- **"CORS Error"**: Ensure the Edge Function `index.ts` has the correct `corsHeaders` (implemented in current version).

**Status**: ✅ Provisioning Guide Updated 2026-02-14
