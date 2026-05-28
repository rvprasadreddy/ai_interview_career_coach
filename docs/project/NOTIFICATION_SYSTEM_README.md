# 🔔 Daily Drill Notification System

> Production-grade push notification system for the InterviPrep Daily Drill feature

## 📋 Overview

This notification system delivers **two daily notifications** to users:

1. **Morning Notification (8:00 AM IST)** - Announces the new daily question to all users
2. **Reminder Notification (11:00 AM IST)** - Reminds users who haven't completed today's drill

## ✨ Features

### Core Capabilities
- ✅ **Scheduled Notifications**: Automated daily notifications via Supabase Cron
- ✅ **Batch Processing**: Handles 500+ notifications per batch efficiently
- ✅ **Multi-Platform**: Supports Android, iOS, and Web
- ✅ **Deep Linking**: Direct navigation to Daily Drill screen
- ✅ **Personalization**: Includes user streak count and personalized messages
- ✅ **Analytics**: Comprehensive tracking of delivery, clicks, and engagement

### Production Features
- ✅ **Error Handling**: Automatic retry logic and graceful degradation
- ✅ **Token Management**: Automatic expiration and cleanup of invalid tokens
- ✅ **Rate Limiting**: Prevents FCM quota exhaustion
- ✅ **GDPR Compliance**: 90-day log retention and user data controls
- ✅ **Security**: Row Level Security (RLS) and input sanitization
- ✅ **Monitoring**: Real-time metrics and performance tracking

## 🏗️ Architecture

```
Supabase Cron → Edge Function → Firebase FCM → User Devices
                      ↓
              Database Logging & Metrics
```

**Key Components**:
1. **Database Schema** - Stores tokens, logs, and metrics
2. **Edge Function** - Handles notification logic and FCM integration
3. **Cron Jobs** - Schedules automated notifications
4. **Flutter Service** - Manages token registration and notification display

## 📂 Project Structure

```
supabase/
├── migrations/
│   ├── 20260214_daily_drill_notifications.sql    # Database schema
│   └── 20260214_notification_cron_jobs.sql       # Cron configuration
└── functions/
    └── daily-drill-notifier/
        └── index.ts                               # Notification publisher

lib/
└── core/
    └── services/
        └── push_notification_service.dart         # Flutter integration

docs/
└── project/
    ├── DAILY_DRILL_NOTIFICATION_ARCHITECTURE.md   # System architecture
    ├── NOTIFICATION_DEPLOYMENT_GUIDE.md           # Deployment steps
    ├── NOTIFICATION_IMPLEMENTATION_SUMMARY.md     # Implementation details
    └── NOTIFICATION_QUICK_REFERENCE.md            # Quick commands
```

## 🚀 Quick Start

### 1. Deploy Database Schema

```bash
# Apply migration
supabase db push --db-url "your-db-url"
```

### 2. Deploy Edge Function

```bash
# Set FCM Server Key
supabase secrets set FCM_SERVER_KEY="your-fcm-key"

# Deploy function
supabase functions deploy daily-drill-notifier
```

### 3. Schedule Cron Jobs

```sql
-- Run in Supabase SQL Editor
-- See: supabase/migrations/20260214_notification_cron_jobs.sql
```

### 4. Integrate Flutter App

```dart
// In main.dart
await Firebase.initializeApp();
await PushNotificationService().initialize();
```

## 📖 Documentation

| Document | Description |
|----------|-------------|
| [Architecture](./DAILY_DRILL_NOTIFICATION_ARCHITECTURE.md) | System design and data flow |
| [Deployment Guide](./NOTIFICATION_DEPLOYMENT_GUIDE.md) | Step-by-step setup instructions |
| [Implementation Summary](./NOTIFICATION_IMPLEMENTATION_SUMMARY.md) | Technical details and features |
| [Quick Reference](./NOTIFICATION_QUICK_REFERENCE.md) | Common commands and queries |

## 🧪 Testing

### Manual Notification Test

```sql
-- Send test morning notification
SELECT net.http_post(
    url:='https://YOUR_REF.supabase.co/functions/v1/daily-drill-notifier',
    headers:='{"Content-Type": "application/json", "Authorization": "Bearer YOUR_KEY"}'::jsonb,
    body:='{"action": "send_morning_notification"}'::jsonb
);
```

### Verify Delivery

```sql
-- Check notification logs
SELECT * FROM notification_logs 
WHERE created_at >= CURRENT_DATE 
ORDER BY created_at DESC;
```

## 📊 Monitoring

### Key Metrics

```sql
-- Delivery rate
SELECT 
    notification_type,
    COUNT(*) as total,
    COUNT(*) FILTER (WHERE status = 'sent') as sent,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'sent') / COUNT(*), 2) as delivery_rate
FROM notification_logs
WHERE created_at >= CURRENT_DATE
GROUP BY notification_type;
```

### Active Tokens

```sql
-- Tokens by device type
SELECT device_type, COUNT(*) as count
FROM user_fcm_tokens
WHERE is_active = true
GROUP BY device_type;
```

## 🔧 Troubleshooting

### Common Issues

**Notifications not received?**
```sql
-- Check user setup
SELECT 
    uft.fcm_token,
    uft.is_active,
    udp.notification_enabled,
    udp.enable_morning_notification
FROM user_fcm_tokens uft
JOIN user_drill_progress udp ON uft.user_id = udp.user_id
WHERE uft.user_id = 'USER_ID';
```

**High failure rate?**
```sql
-- Check error patterns
SELECT error_message, COUNT(*) 
FROM notification_logs 
WHERE status = 'failed' AND created_at >= CURRENT_DATE 
GROUP BY error_message;
```

**Cron not running?**
```sql
-- Check job status
SELECT * FROM cron.job_run_details 
WHERE jobid IN (SELECT jobid FROM cron.job WHERE jobname LIKE 'daily-drill%')
ORDER BY start_time DESC LIMIT 10;
```

## 🎯 Success Metrics

### Target KPIs

- **Delivery Rate**: > 95%
- **Click-Through Rate**: > 15%
- **Completion Rate Lift**: > 10%
- **Token Expiration**: < 5% monthly

### Current Performance

```sql
-- Today's performance
SELECT 
    notification_type,
    COUNT(*) as sent,
    COUNT(*) FILTER (WHERE status = 'delivered') as delivered,
    COUNT(*) FILTER (WHERE status = 'clicked') as clicked,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'clicked') / 
          NULLIF(COUNT(*) FILTER (WHERE status = 'delivered'), 0), 2) as ctr
FROM notification_logs
WHERE created_at >= CURRENT_DATE
GROUP BY notification_type;
```

## 🔐 Security

### Implemented Measures

- ✅ Row Level Security (RLS) on all tables
- ✅ Input validation and sanitization
- ✅ Rate limiting (500 per batch, 1s delay)
- ✅ Automatic token expiration (60 days)
- ✅ GDPR-compliant log retention (90 days)
- ✅ Secure FCM key storage (Supabase Secrets)

## 📅 Notification Schedule

| Time | Type | Target Audience | Cron Expression |
|------|------|-----------------|-----------------|
| 8:00 AM IST | Morning | All users | `30 2 * * *` (2:30 AM UTC) |
| 11:00 AM IST | Reminder | Incomplete users | `30 5 * * *` (5:30 AM UTC) |
| 12:00 AM UTC | Cleanup | System | `0 0 * * *` |

## 🛠️ Maintenance

### Daily Tasks
- Monitor delivery rates
- Check error logs
- Review user feedback

### Weekly Tasks
- Analyze engagement metrics
- Optimize notification timing
- A/B test notification copy

### Monthly Tasks
- Review token expiration rates
- Update notification content
- Optimize database indexes

## 🚧 Roadmap

### Phase 1: MVP ✅
- [x] Morning notifications
- [x] Reminder notifications
- [x] FCM token management
- [x] Basic analytics

### Phase 2: Optimization 🔄
- [ ] Multi-timezone support
- [ ] Personalized timing
- [ ] A/B testing framework
- [ ] Rich notifications with actions

### Phase 3: Advanced Features 📋
- [ ] ML-based optimal timing
- [ ] Streak milestone notifications
- [ ] Achievement unlock notifications
- [ ] Weekly progress summaries

## 🤝 Contributing

### Adding New Notification Types

1. Update `notification_type` enum in `notification_logs` table
2. Add new action handler in Edge Function
3. Create notification payload generator
4. Update Flutter service for deep linking
5. Add analytics queries

### Modifying Notification Content

Edit the payload generators in:
- `supabase/functions/daily-drill-notifier/index.ts`

### Changing Schedule

Update cron expressions in:
- `supabase/migrations/20260214_notification_cron_jobs.sql`

## 📞 Support

For issues or questions:
1. Check [Quick Reference](./NOTIFICATION_QUICK_REFERENCE.md)
2. Review [Deployment Guide](./NOTIFICATION_DEPLOYMENT_GUIDE.md)
3. See [Troubleshooting](#troubleshooting) section above

## 📄 License

Part of the InterviPrep project.

---

**Version**: 1.0  
**Last Updated**: 2026-02-14  
**Status**: Production Ready ✅
