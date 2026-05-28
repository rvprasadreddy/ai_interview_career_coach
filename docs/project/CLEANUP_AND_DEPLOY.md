# 🗑️ CLEANUP & FRESH DEPLOYMENT GUIDE

**Date**: 2026-02-14  
**Purpose**: Clean slate deployment of Daily Drill & Notification System

---

## ⚠️ WARNING

**This will DELETE ALL DATA related to:**
- Daily Drill questions
- User drill progress
- User drill history
- Notifications
- Notification analytics
- All related functions, triggers, and views

**Make sure you have backups if needed!**

---

## 📋 Step-by-Step Process

### **Step 1: Cleanup Existing Objects** (2 minutes)

1. Open **Supabase Dashboard** → **SQL Editor**
2. Click **New Query**
3. Copy entire contents of:
   ```
   supabase/migrations/cleanup_daily_drill_notifications.sql
   ```
4. Paste into SQL Editor
5. Click **Run** (or press Ctrl+Enter)
6. Wait for completion

**Expected Output**:
```
✅ Removed notifications from Realtime publication
✅ Removed morning notification cron job
✅ Removed reminder notification cron job
✅ Removed cleanup cron job
========================================
CLEANUP VERIFICATION REPORT
========================================
Remaining Tables: 0
Remaining Functions: 0
Remaining Triggers: 0
Remaining Views: 0
========================================
✅ SUCCESS: All Daily Drill & Notification objects removed!
========================================
```

---

### **Step 2: Apply Fresh Migration** (5 minutes)

1. Stay in **SQL Editor**
2. Click **New Query**
3. Copy entire contents of:
   ```
   supabase/migrations/20260214_complete_daily_drill_notifications.sql
   ```
4. Paste into SQL Editor
5. Click **Run**
6. Wait for completion (may take 30-60 seconds)

**Verify Success**:
```sql
-- Should return 7
SELECT COUNT(*) FROM information_schema.tables 
WHERE table_schema = 'public' 
  AND (table_name LIKE '%drill%' OR table_name LIKE '%notification%');

-- Should return 5 columns: user_id, current_streak, total_completed, readiness_score, timezone
SELECT * FROM get_notification_eligible_users('morning') LIMIT 1;

-- Should return 12
SELECT COUNT(*) FROM daily_drill_questions WHERE is_active = true;
```

---

### **Step 3: Deploy Edge Function** (3 minutes)

```bash
cd c:\flutter_apps\intervi_prep
supabase functions deploy daily-drill-notifier
```

**Verify**:
```bash
# List functions
supabase functions list

# Test
curl -X POST https://YOUR_REF.supabase.co/functions/v1/daily-drill-notifier ^
  -H "Authorization: Bearer YOUR_ANON_KEY" ^
  -H "Content-Type: application/json" ^
  -d "{\"action\": \"test\"}"
```

---

### **Step 4: Schedule Cron Jobs** (5 minutes)

Run in **SQL Editor**:

```sql
-- Enable pg_cron extension
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Morning notification (8 AM IST = 2:30 AM UTC)
SELECT cron.schedule(
  'daily-drill-morning-notification',
  '30 2 * * *',
  $$
  SELECT net.http_post(
    url := 'https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-notifier',
    headers := '{"Content-Type": "application/json", "Authorization": "Bearer YOUR_SERVICE_ROLE_KEY"}'::jsonb,
    body := '{"action": "send_morning_notification"}'::jsonb
  );
  $$
);

-- Reminder notification (11 AM IST = 5:30 AM UTC)
SELECT cron.schedule(
  'daily-drill-reminder-notification',
  '30 5 * * *',
  $$
  SELECT net.http_post(
    url := 'https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-notifier',
    headers := '{"Content-Type": "application/json", "Authorization": "Bearer YOUR_SERVICE_ROLE_KEY"}'::jsonb,
    body := '{"action": "send_reminder_notification"}'::jsonb
  );
  $$
);

-- Cleanup job (Midnight UTC)
SELECT cron.schedule(
  'daily-drill-cleanup-notifications',
  '0 0 * * *',
  $$
  SELECT cleanup_old_notifications();
  SELECT cleanup_inactive_web_push_subscriptions();
  $$
);
```

**Replace**:
- `YOUR_PROJECT_REF` - Your Supabase project reference
- `YOUR_SERVICE_ROLE_KEY` - Your service role key (Settings > API)

**Verify**:
```sql
-- Should return 3 jobs
SELECT jobname, schedule, active 
FROM cron.job 
WHERE jobname LIKE 'daily-drill%';
```

---

## ✅ Final Verification

### Database Check
```sql
-- Tables (should return 7)
SELECT table_name FROM information_schema.tables 
WHERE table_schema = 'public' 
  AND (table_name LIKE '%drill%' OR table_name LIKE '%notification%')
ORDER BY table_name;

-- Functions (should return 11)
SELECT p.proname FROM pg_proc p
JOIN pg_namespace n ON p.pronamespace = n.oid
WHERE n.nspname = 'public'
  AND (p.proname LIKE '%drill%' OR p.proname LIKE '%notification%')
ORDER BY p.proname;

-- Sample questions (should return 12)
SELECT COUNT(*) FROM daily_drill_questions WHERE is_active = true;

-- Realtime enabled (should return 1)
SELECT tablename FROM pg_publication_tables 
WHERE pubname = 'supabase_realtime' AND tablename = 'notifications';
```

### Edge Function Check
```bash
# Test daily drill
curl -X POST https://YOUR_REF.supabase.co/functions/v1/daily-drill-notifier ^
  -H "Authorization: Bearer YOUR_USER_TOKEN" ^
  -H "Content-Type: application/json" ^
  -d "{\"action\": \"get_today_question\"}"

# Test notification
curl -X POST https://YOUR_REF.supabase.co/functions/v1/daily-drill-notifier ^
  -H "Authorization: Bearer YOUR_SERVICE_KEY" ^
  -H "Content-Type: application/json" ^
  -d "{\"action\": \"test\"}"
```

### Cron Jobs Check
```sql
-- Check scheduled jobs
SELECT jobname, schedule, active FROM cron.job 
WHERE jobname LIKE 'daily-drill%';

-- Manually trigger for testing
SELECT net.http_post(
  url := 'https://YOUR_REF.supabase.co/functions/v1/daily-drill-notifier',
  headers := '{"Content-Type": "application/json", "Authorization": "Bearer YOUR_SERVICE_KEY"}'::jsonb,
  body := '{"action": "send_morning_notification"}'::jsonb
);

-- Check notification created
SELECT * FROM notifications 
WHERE created_at > now() - INTERVAL '5 minutes'
ORDER BY created_at DESC;
```

---

## 📊 What Gets Created

### Tables (7)
1. `notifications` - Notification delivery tracking (Realtime ✅)
2. `web_push_subscriptions` - Web push subscriptions
3. `notification_analytics` - Aggregate metrics
4. `notification_conversion_tracking` - Conversion tracking
5. `daily_drill_questions` - Question repository (12 seeded)
6. `user_daily_drills` - User assignments
7. `user_drill_progress` - User progress & stats

### Functions (11)
1. `initialize_user_drill_progress()` - Auto-create progress
2. `update_weak_categories(UUID)` - Update weak areas
3. `calculate_readiness_score(UUID)` - Calculate score
4. `update_user_streak(UUID, DATE)` - Update streaks
5. `update_drill_progress_on_completion()` - Auto-update
6. `get_or_assign_daily_question(UUID, DATE)` - Assign question
7. `get_notification_eligible_users(TEXT)` - Get eligible users
8. `cleanup_old_notifications()` - Cleanup old notifications
9. `cleanup_inactive_web_push_subscriptions()` - Cleanup subscriptions
10. `get_unread_notification_count(UUID)` - Get unread count
11. `track_notification_conversion(UUID, UUID, UUID)` - Track conversions

### Triggers (4)
1. `on_auth_user_created_drill_progress` - Auto-initialize users
2. `on_drill_status_changed` - Auto-update progress
3. `track_drill_completion_trigger` - Auto-track conversions
4. `update_web_push_subscriptions_updated_at` - Update timestamps

### Views (1)
1. `notification_conversion_metrics` - Analytics view

### Cron Jobs (3)
1. Morning notification (8 AM IST)
2. Reminder notification (11 AM IST)
3. Cleanup job (Midnight UTC)

---

## 🎯 Success Criteria

- ✅ All old objects removed
- ✅ 7 tables created
- ✅ 11 functions created
- ✅ 4 triggers created
- ✅ 1 view created
- ✅ 12 sample questions loaded
- ✅ Realtime enabled on notifications
- ✅ Edge Function deployed
- ✅ 3 cron jobs scheduled
- ✅ Test notification sent successfully

---

## 🐛 Troubleshooting

### Issue: Cleanup fails with "cannot drop table because other objects depend on it"
**Solution**: The cleanup script uses `CASCADE` which should handle this. If it still fails:
```sql
-- Force drop with CASCADE
DROP TABLE IF EXISTS public.notifications CASCADE;
DROP TABLE IF EXISTS public.user_daily_drills CASCADE;
-- etc.
```

### Issue: Migration fails with "relation already exists"
**Solution**: Run cleanup script again to ensure all objects are removed.

### Issue: Function returns wrong columns
**Solution**: Drop and recreate the function:
```sql
DROP FUNCTION IF EXISTS get_notification_eligible_users(TEXT) CASCADE;
-- Then run the CREATE FUNCTION from the migration
```

---

## 📁 Files Used

**Cleanup**:
- `supabase/migrations/cleanup_daily_drill_notifications.sql`

**Fresh Deployment**:
- `supabase/migrations/20260214_complete_daily_drill_notifications.sql`
- `supabase/functions/daily-drill-notifier/index.ts`

**Documentation**:
- `docs/project/CLEANUP_AND_DEPLOY.md` (this file)
- `docs/project/COMPLETE_DEPLOYMENT_GUIDE.md`

---

## ✅ You're All Set!

**Total Time**: ~15 minutes  
**Difficulty**: Easy  
**Risk**: Low (fresh start)

Follow the steps above and you'll have a clean, production-ready Daily Drill & Notification System! 🚀
