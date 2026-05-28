# 🚀 Notification System - Complete Deployment Guide

**Version**: 1.0  
**Date**: 2026-02-14  
**Estimated Time**: 30-45 minutes

---

## 📋 Prerequisites

Before starting, ensure you have:

- [x] Supabase project created
- [x] Supabase CLI installed (`npm install -g supabase`)
- [x] Project linked to Supabase (`supabase link`)
- [x] Database access (SQL Editor or CLI)
- [x] Flutter project set up with Supabase

---

## 🗂️ Deployment Overview

```
Step 1: Database Setup (10 min)
   ├── Apply migration
   ├── Verify tables
   └── Test functions

Step 2: Edge Function Deployment (10 min)
   ├── Create function files
   ├── Deploy to Supabase
   └── Test function

Step 3: Cron Jobs Setup (5 min)
   ├── Schedule notifications
   └── Verify schedules

Step 4: Flutter Integration (10 min)
   ├── Update code
   ├── Test locally
   └── Build & deploy

Step 5: Verification (5 min)
   └── End-to-end testing
```

---

## 📊 Step 1: Database Setup

### 1.1 Apply Database Migration

**Option A: Using Supabase Dashboard**

1. Go to your Supabase project dashboard
2. Navigate to **SQL Editor**
3. Click **New Query**
4. Copy the contents of `20260214_daily_drill_notifications.sql`
5. Paste into the editor
6. Click **Run**

**Option B: Using Supabase CLI**

```bash
# Navigate to project root
cd c:\flutter_apps\intervi_prep

# Apply migration
supabase db push
```

### 1.2 Verify Tables Created

Run this query in SQL Editor:

```sql
-- Check if tables exist
SELECT table_name, table_type 
FROM information_schema.tables 
WHERE table_schema = 'public' 
AND table_name IN ('notifications', 'web_push_subscriptions', 'notification_analytics')
ORDER BY table_name;
```

**Expected Output**:
```
table_name                  | table_type
----------------------------|------------
notification_analytics      | BASE TABLE
notifications               | BASE TABLE
web_push_subscriptions      | BASE TABLE
```

### 1.3 Verify RLS Policies

```sql
-- Check RLS is enabled
SELECT 
    tablename, 
    rowsecurity as rls_enabled
FROM pg_tables 
WHERE schemaname = 'public' 
AND tablename IN ('notifications', 'web_push_subscriptions');
```

**Expected Output**:
```
tablename                  | rls_enabled
---------------------------|-------------
notifications              | true
web_push_subscriptions     | true
```

### 1.4 Test Helper Functions

```sql
-- Test get_notification_eligible_users
SELECT * FROM get_notification_eligible_users('morning');

-- Test get_unread_notification_count (replace with your user_id)
SELECT get_unread_notification_count('YOUR_USER_ID');
```

### 1.5 Enable Realtime for Notifications Table

```sql
-- Enable Realtime broadcasting
ALTER PUBLICATION supabase_realtime ADD TABLE notifications;
```

**Verify in Dashboard**:
1. Go to **Database** → **Replication**
2. Check that `notifications` table is listed under **Realtime**

---

## 🔧 Step 2: Edge Function Deployment

### 2.1 Create Edge Function Directory

```bash
# Create function directory
mkdir -p supabase/functions/daily-drill-notifier

# Create index.ts file
# (Content provided in next section)
```

### 2.2 Create Edge Function Code

Create `supabase/functions/daily-drill-notifier/index.ts`:

```typescript
// See EDGE_FUNCTION_CODE.md for complete code
// (Will be created in next step)
```

### 2.3 Create deno.json Configuration

Create `supabase/functions/daily-drill-notifier/deno.json`:

```json
{
  "imports": {
    "supabase": "https://esm.sh/@supabase/supabase-js@2.39.0"
  }
}
```

### 2.4 Deploy Edge Function

```bash
# Deploy the function
supabase functions deploy daily-drill-notifier

# Expected output:
# Deploying daily-drill-notifier (project ref: YOUR_REF)
# Deployed Function daily-drill-notifier
```

### 2.5 Test Edge Function

**Test Morning Notification**:

```bash
curl -X POST https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-notifier \
  -H "Authorization: Bearer YOUR_ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "action": "send_morning_notification"
  }'
```

**Expected Response**:
```json
{
  "success": true,
  "message": "Morning notifications sent",
  "count": 5,
  "timestamp": "2026-02-14T02:30:00.000Z"
}
```

**Test Reminder Notification**:

```bash
curl -X POST https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-notifier \
  -H "Authorization: Bearer YOUR_ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "action": "send_reminder_notification"
  }'
```

### 2.6 Check Function Logs

```bash
# View function logs
supabase functions logs daily-drill-notifier

# Or in dashboard:
# Edge Functions → daily-drill-notifier → Logs
```

---

## ⏰ Step 3: Cron Jobs Setup

### 3.1 Schedule Morning Notification (8:00 AM IST)

```sql
-- Morning notification at 8:00 AM IST (2:30 AM UTC)
SELECT cron.schedule(
    'daily-drill-morning-notification',
    '30 2 * * *',  -- 2:30 AM UTC = 8:00 AM IST
    $$
    SELECT net.http_post(
        url := 'https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-notifier',
        headers := '{"Content-Type": "application/json", "Authorization": "Bearer YOUR_SERVICE_ROLE_KEY"}'::jsonb,
        body := '{"action": "send_morning_notification"}'::jsonb
    ) as request_id;
    $$
);
```

### 3.2 Schedule Reminder Notification (11:00 AM IST)

```sql
-- Reminder notification at 11:00 AM IST (5:30 AM UTC)
SELECT cron.schedule(
    'daily-drill-reminder-notification',
    '30 5 * * *',  -- 5:30 AM UTC = 11:00 AM IST
    $$
    SELECT net.http_post(
        url := 'https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-notifier',
        headers := '{"Content-Type": "application/json", "Authorization": "Bearer YOUR_SERVICE_ROLE_KEY"}'::jsonb,
        body := '{"action": "send_reminder_notification"}'::jsonb
    ) as request_id;
    $$
);
```

### 3.3 Schedule Cleanup Job (Midnight UTC)

```sql
-- Cleanup old notifications at midnight UTC
SELECT cron.schedule(
    'daily-drill-cleanup-notifications',
    '0 0 * * *',  -- Midnight UTC
    $$
    SELECT cleanup_old_notifications();
    $$
);
```

### 3.4 Verify Cron Jobs

```sql
-- List all cron jobs
SELECT 
    jobid,
    jobname,
    schedule,
    active,
    command
FROM cron.job 
WHERE jobname LIKE 'daily-drill%'
ORDER BY jobname;
```

**Expected Output**:
```
jobid | jobname                              | schedule    | active | command
------|--------------------------------------|-------------|--------|----------
1     | daily-drill-morning-notification     | 30 2 * * *  | true   | SELECT...
2     | daily-drill-reminder-notification    | 30 5 * * *  | true   | SELECT...
3     | daily-drill-cleanup-notifications    | 0 0 * * *   | true   | SELECT...
```

### 3.5 Check Cron Job Execution History

```sql
-- View recent cron job executions
SELECT 
    j.jobname,
    r.runid,
    r.status,
    r.start_time,
    r.end_time,
    r.return_message
FROM cron.job_run_details r
JOIN cron.job j ON j.jobid = r.jobid
WHERE j.jobname LIKE 'daily-drill%'
ORDER BY r.start_time DESC
LIMIT 10;
```

---

## 📱 Step 4: Flutter Integration

### 4.1 Update Dependencies

Ensure `pubspec.yaml` has:

```yaml
dependencies:
  flutter:
    sdk: flutter
  supabase_flutter: ^2.0.0
  # NO Firebase dependencies needed!
```

Run:
```bash
flutter pub get
```

### 4.2 Initialize Notification Service

**In `lib/main.dart`**:

```dart
import 'package:supabase_flutter/supabase_flutter.dart';
import 'core/services/notification_service.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize Supabase
  await Supabase.initialize(
    url: 'YOUR_SUPABASE_URL',
    anonKey: 'YOUR_ANON_KEY',
  );
  
  runApp(MyApp());
}
```

**After User Login** (in your auth flow):

```dart
Future<void> _handleSuccessfulLogin() async {
  // ... existing login logic
  
  // Initialize notification service
  await NotificationService().initialize();
  
  // Set up notification handlers
  NotificationService().onNotificationReceived = (notification) {
    _showNotificationBanner(notification);
  };
  
  NotificationService().onNotificationTapped = (notification) {
    _handleNotificationTap(notification);
  };
  
  // Navigate to home
  Navigator.pushReplacementNamed(context, '/home');
}
```

### 4.3 Display Notifications

**Example notification banner**:

```dart
void _showNotificationBanner(NotificationModel notification) {
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(
      content: Row(
        children: [
          Icon(Icons.notifications, color: Colors.white),
          SizedBox(width: 12),
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              mainAxisSize: MainAxisSize.min,
              children: [
                Text(
                  notification.title,
                  style: TextStyle(fontWeight: FontWeight.bold),
                ),
                Text(notification.body),
              ],
            ),
          ),
        ],
      ),
      action: SnackBarAction(
        label: 'View',
        onPressed: () {
          NotificationService().markAsClicked(notification.id);
          _navigateToScreen(notification.data['screen']);
        },
      ),
      duration: Duration(seconds: 5),
      behavior: SnackBarBehavior.floating,
    ),
  );
}
```

### 4.4 Test Locally

```bash
# Run the app
flutter run

# Test notification reception:
# 1. Login to the app
# 2. Trigger a test notification from Supabase SQL Editor:
```

```sql
-- Insert test notification
INSERT INTO notifications (
    user_id,
    type,
    title,
    body,
    data,
    status
) VALUES (
    'YOUR_USER_ID',
    'morning',
    '🎯 Test Notification',
    'This is a test notification from Supabase!',
    '{"screen": "DailyDrillScreen"}'::jsonb,
    'pending'
);
```

**Expected Result**: Notification should appear in the app immediately!

### 4.5 Build Release Version

```bash
# Android
flutter build apk --release

# iOS
flutter build ios --release

# Web
flutter build web --release
```

---

## ✅ Step 5: Verification & Testing

### 5.1 End-to-End Test Checklist

- [ ] **Database**
  - [ ] Tables created
  - [ ] RLS policies active
  - [ ] Helper functions working
  - [ ] Realtime enabled

- [ ] **Edge Function**
  - [ ] Deployed successfully
  - [ ] Manual test passes
  - [ ] Logs show no errors

- [ ] **Cron Jobs**
  - [ ] All jobs scheduled
  - [ ] Jobs are active
  - [ ] Execution history shows success

- [ ] **Flutter App**
  - [ ] NotificationService initializes
  - [ ] Receives notifications
  - [ ] Displays notifications
  - [ ] Marks as read/clicked
  - [ ] Unread count updates

### 5.2 Manual Testing Scenarios

**Test 1: Receive Notification**
1. Login to app
2. Insert test notification (SQL above)
3. Verify notification appears
4. Check unread count increments

**Test 2: Mark as Read**
1. Receive notification
2. Tap "Dismiss" or mark as read
3. Verify unread count decrements
4. Check database status updated

**Test 3: Deep Linking**
1. Receive notification with screen data
2. Tap notification
3. Verify navigation to correct screen
4. Check status marked as "clicked"

**Test 4: Notification History**
1. Navigate to notification history screen
2. Verify all notifications displayed
3. Check read/unread status
4. Test pagination

**Test 5: Preferences**
1. Navigate to settings
2. Toggle notification preferences
3. Verify preferences saved
4. Check database updated

### 5.3 Monitor Performance

**Query: Delivery Rate**
```sql
SELECT 
    DATE(created_at) as date,
    COUNT(*) as total_sent,
    COUNT(*) FILTER (WHERE status = 'delivered') as delivered,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'delivered') / COUNT(*), 2) as delivery_rate
FROM notifications
WHERE created_at >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY DATE(created_at)
ORDER BY date DESC;
```

**Query: Click-Through Rate**
```sql
SELECT 
    type,
    COUNT(*) FILTER (WHERE status = 'delivered') as delivered,
    COUNT(*) FILTER (WHERE status = 'clicked') as clicked,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'clicked') / 
          NULLIF(COUNT(*) FILTER (WHERE status = 'delivered'), 0), 2) as ctr
FROM notifications
WHERE created_at >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY type;
```

---

## 🐛 Troubleshooting

### Issue 1: Notifications Not Received

**Symptoms**: App doesn't receive notifications

**Solutions**:
```dart
// Check initialization
if (!NotificationService().isInitialized) {
  print('NotificationService not initialized!');
  await NotificationService().initialize();
}

// Check user authentication
final user = Supabase.instance.client.auth.currentUser;
if (user == null) {
  print('User not authenticated');
}

// Check Realtime connection
print('Realtime status: ${Supabase.instance.client.realtime.channels}');
```

### Issue 2: Cron Jobs Not Running

**Check job status**:
```sql
SELECT * FROM cron.job WHERE jobname LIKE 'daily-drill%';
```

**Check execution errors**:
```sql
SELECT * FROM cron.job_run_details 
WHERE status = 'failed' 
ORDER BY start_time DESC 
LIMIT 10;
```

**Fix**: Ensure `pg_cron` extension is enabled:
```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
```

### Issue 3: Edge Function Errors

**View logs**:
```bash
supabase functions logs daily-drill-notifier --tail
```

**Common fixes**:
- Check environment variables
- Verify service role key
- Check function permissions
- Review error messages in logs

### Issue 4: RLS Policy Blocking Access

**Test RLS**:
```sql
-- Test as authenticated user
SET LOCAL role authenticated;
SET LOCAL request.jwt.claims.sub TO 'YOUR_USER_ID';

SELECT * FROM notifications WHERE user_id = 'YOUR_USER_ID';
```

**Fix**: Review and update RLS policies if needed

---

## 📊 Post-Deployment Monitoring

### Daily Checks (First Week)

- [ ] Check cron job execution history
- [ ] Monitor notification delivery rate
- [ ] Review error logs
- [ ] Check user engagement metrics
- [ ] Verify database performance

### Weekly Checks

- [ ] Analyze notification CTR
- [ ] Review user feedback
- [ ] Optimize notification timing
- [ ] Clean up old data
- [ ] Update documentation

### Monitoring Queries

Save these for regular monitoring:

```sql
-- Today's stats
SELECT 
    COUNT(*) as total,
    COUNT(*) FILTER (WHERE status = 'delivered') as delivered,
    COUNT(*) FILTER (WHERE status = 'clicked') as clicked,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'delivered') / COUNT(*), 2) as delivery_rate,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'clicked') / 
          NULLIF(COUNT(*) FILTER (WHERE status = 'delivered'), 0), 2) as ctr
FROM notifications
WHERE created_at >= CURRENT_DATE;
```

---

## 🎉 Deployment Complete!

Congratulations! Your notification system is now live.

### Next Steps:

1. **Monitor**: Watch metrics for first 48 hours
2. **Optimize**: Adjust timing based on engagement
3. **Scale**: Handle increased user base
4. **Iterate**: Add new notification types

### Resources:

- **Documentation**: See `docs/project/` folder
- **Code**: `lib/core/services/notification_service.dart`
- **Database**: `supabase/migrations/20260214_daily_drill_notifications.sql`
- **Edge Function**: `supabase/functions/daily-drill-notifier/`

---

**Deployment Status**: ✅ Complete  
**System Status**: 🟢 Operational  
**Next Review**: 24 hours

---

**Need Help?** Check the troubleshooting section or review the documentation.
