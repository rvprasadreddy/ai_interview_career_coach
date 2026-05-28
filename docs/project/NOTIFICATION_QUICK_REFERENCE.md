# Daily Drill Notifications - Quick Reference

## 🚀 Quick Commands

### Deploy Edge Function
```bash
supabase secrets set FCM_SERVER_KEY="your-key"
supabase functions deploy daily-drill-notifier
```

### Test Notifications Manually
```sql
-- Morning Notification
SELECT net.http_post(
    url:='https://YOUR_REF.supabase.co/functions/v1/daily-drill-notifier',
    headers:='{"Content-Type": "application/json", "Authorization": "Bearer YOUR_KEY"}'::jsonb,
    body:='{"action": "send_morning_notification"}'::jsonb
);

-- Reminder Notification
SELECT net.http_post(
    url:='https://YOUR_REF.supabase.co/functions/v1/daily-drill-notifier',
    headers:='{"Content-Type": "application/json", "Authorization": "Bearer YOUR_KEY"}'::jsonb,
    body:='{"action": "send_reminder_notification"}'::jsonb
);
```

### Check System Status
```sql
-- Active tokens
SELECT device_type, COUNT(*) FROM user_fcm_tokens WHERE is_active = true GROUP BY device_type;

-- Today's notifications
SELECT notification_type, status, COUNT(*) FROM notification_logs 
WHERE created_at >= CURRENT_DATE GROUP BY notification_type, status;

-- Cron job status
SELECT * FROM cron.job_run_details ORDER BY start_time DESC LIMIT 5;
```

## 📊 Key Metrics

### Delivery Rate
```sql
SELECT 
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'sent') / COUNT(*), 2) as delivery_rate
FROM notification_logs WHERE created_at >= CURRENT_DATE;
```

### Click-Through Rate
```sql
SELECT 
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'clicked') / 
          NULLIF(COUNT(*) FILTER (WHERE status = 'delivered'), 0), 2) as ctr
FROM notification_logs WHERE created_at >= CURRENT_DATE;
```

### Completion Rate Impact
```sql
SELECT 
    COUNT(DISTINCT udd.user_id) FILTER (WHERE udd.status = 'completed') as completed,
    COUNT(DISTINCT nl.user_id) as notified,
    ROUND(100.0 * COUNT(DISTINCT udd.user_id) FILTER (WHERE udd.status = 'completed') / 
          COUNT(DISTINCT nl.user_id), 2) as completion_rate
FROM notification_logs nl
LEFT JOIN user_daily_drills udd ON nl.user_id = udd.user_id AND udd.assigned_date = CURRENT_DATE
WHERE nl.created_at >= CURRENT_DATE;
```

## 🔧 Troubleshooting

### Issue: No notifications received
```sql
-- 1. Check user has FCM token
SELECT * FROM user_fcm_tokens WHERE user_id = 'USER_ID' AND is_active = true;

-- 2. Check user preferences
SELECT notification_enabled, enable_morning_notification, enable_reminder_notification 
FROM user_drill_progress WHERE user_id = 'USER_ID';

-- 3. Check notification logs
SELECT * FROM notification_logs WHERE user_id = 'USER_ID' ORDER BY created_at DESC LIMIT 5;
```

### Issue: High failure rate
```sql
-- Check error patterns
SELECT error_message, COUNT(*) 
FROM notification_logs 
WHERE status = 'failed' AND created_at >= CURRENT_DATE 
GROUP BY error_message;
```

### Issue: Cron not running
```sql
-- Check job schedule
SELECT * FROM cron.job WHERE jobname LIKE 'daily-drill%';

-- Check recent runs
SELECT * FROM cron.job_run_details WHERE jobid IN (
    SELECT jobid FROM cron.job WHERE jobname LIKE 'daily-drill%'
) ORDER BY start_time DESC LIMIT 10;
```

## 📱 Flutter Integration

### Initialize Service
```dart
// In main.dart
await Firebase.initializeApp();
await PushNotificationService().initialize();
```

### Update Preferences
```dart
await PushNotificationService().updateNotificationPreferences(
  notificationEnabled: true,
  enableMorningNotification: true,
  enableReminderNotification: true,
);
```

## 🎯 Notification Schedule

| Time | Type | Target | Action |
|------|------|--------|--------|
| 8:00 AM IST | Morning | All users | `send_morning_notification` |
| 11:00 AM IST | Reminder | Incomplete users | `send_reminder_notification` |
| 12:00 AM UTC | Cleanup | System | `cleanup` |

## 📁 File Locations

```
supabase/
├── migrations/
│   ├── 20260214_daily_drill_notifications.sql    # Schema
│   └── 20260214_notification_cron_jobs.sql       # Cron jobs
└── functions/
    └── daily-drill-notifier/
        └── index.ts                               # Edge function

lib/
└── core/
    └── services/
        └── push_notification_service.dart         # Flutter service

docs/
└── project/
    ├── DAILY_DRILL_NOTIFICATION_ARCHITECTURE.md   # Architecture
    ├── NOTIFICATION_DEPLOYMENT_GUIDE.md           # Deployment
    └── NOTIFICATION_IMPLEMENTATION_SUMMARY.md     # Summary
```

## 🔑 Environment Variables

```bash
# Supabase
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_ANON_KEY=eyJxxx...
SUPABASE_SERVICE_ROLE_KEY=eyJxxx...

# Firebase
FCM_SERVER_KEY=AAAAxxx...
```

## 📞 Support

- **Architecture**: See `DAILY_DRILL_NOTIFICATION_ARCHITECTURE.md`
- **Deployment**: See `NOTIFICATION_DEPLOYMENT_GUIDE.md`
- **Summary**: See `NOTIFICATION_IMPLEMENTATION_SUMMARY.md`

---

**Last Updated**: 2026-02-14  
**Version**: 1.0
