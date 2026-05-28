# Daily Drill Notification System - Architecture Design

## 🎯 Overview

This document outlines the architecture for a **two-tier notification system** for the Daily Drill feature:

1. **Morning Notification (8:00 AM)**: Sent to ALL users to announce the new daily question
2. **Reminder Notification (11:00 AM)**: Sent ONLY to users who haven't completed today's drill

Complete Daily Drill & Notification System
│
├── Database Layer
│   ├── notifications (Realtime ✅)
│   ├── daily_drill_questions (12 seeded)
│   ├── user_daily_drills
│   ├── user_drill_progress
│   ├── notification_analytics
│   ├── notification_conversion_tracking
│   └── web_push_subscriptions
│
├── Edge Function (Single Unified Function)
│   ├── Daily Drill Engine
│   │   ├── Get today's question
│   │   ├── Submit response
│   │   ├── Get stats
│   │   └── Get history
│   │
│   └── Notification Publisher
│       ├── Morning notifications
│       ├── Reminder notifications
│       └── Test notifications
│
├── Cron Jobs (3 Scheduled)
│   ├── Morning (8 AM IST)
│   ├── Reminder (11 AM IST)
│   └── Cleanup (Midnight UTC)
│
└── Analytics & Monitoring
    ├── Conversion metrics
    ├── Performance tracking
    └── User engagement

---

## 📐 System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                   NOTIFICATION ORCHESTRATION                         │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  SUPABASE CRON JOBS (pg_cron)                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Job 1: Daily Morning Notification (8:00 AM IST)               │ │
│  │  - Triggers: daily-drill-notifier Edge Function                │ │
│  │  - Action: send_morning_notification                           │ │
│  │  - Target: ALL users with notifications enabled                │ │
│  └────────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Job 2: Daily Reminder Notification (11:00 AM IST)             │ │
│  │  - Triggers: daily-drill-notifier Edge Function                │ │
│  │  - Action: send_reminder_notification                          │ │
│  │  - Target: Users who haven't completed today's drill           │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  SUPABASE EDGE FUNCTION: daily-drill-notifier                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Actions:                                                       │ │
│  │  - send_morning_notification                                   │ │
│  │  - send_reminder_notification                                  │ │
│  │  - register_fcm_token (from Flutter app)                       │ │
│  │  - update_notification_preferences                             │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  FIREBASE CLOUD MESSAGING (FCM)                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  - Handles push notification delivery                          │ │
│  │  - Supports Android, iOS, and Web                              │ │
│  │  - Manages device tokens and topics                            │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  FLUTTER APP (Client)                                               │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  - Receives notifications via firebase_messaging package       │ │
│  │  - Handles deep linking to Daily Drill screen                  │ │
│  │  - Manages notification permissions                            │ │
│  │  - Sends FCM token to backend on app launch                    │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Database Schema Extensions

### 1. New Table: `user_fcm_tokens`

Stores Firebase Cloud Messaging tokens for each user's devices.

```sql
CREATE TABLE public.user_fcm_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    fcm_token TEXT NOT NULL,
    device_type TEXT NOT NULL CHECK (device_type IN ('android', 'ios', 'web')),
    device_info JSONB DEFAULT '{}'::jsonb, -- Device name, OS version, etc.
    
    -- Status tracking
    is_active BOOLEAN DEFAULT true,
    last_used_at TIMESTAMPTZ DEFAULT now(),
    
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Constraints
    UNIQUE(user_id, fcm_token) -- Prevent duplicate tokens
);

-- Indexes
CREATE INDEX idx_user_fcm_tokens_user_id ON public.user_fcm_tokens(user_id);
CREATE INDEX idx_user_fcm_tokens_active ON public.user_fcm_tokens(is_active) WHERE is_active = true;
```

### 2. New Table: `notification_logs`

Tracks all sent notifications for analytics and debugging.

```sql
CREATE TABLE public.notification_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    notification_type TEXT NOT NULL CHECK (notification_type IN ('morning', 'reminder', 'streak', 'achievement')),
    
    -- Notification content
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    data JSONB DEFAULT '{}'::jsonb,
    
    -- Delivery tracking
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'sent', 'failed', 'delivered', 'clicked')),
    fcm_token TEXT,
    error_message TEXT,
    
    -- Timestamps
    scheduled_at TIMESTAMPTZ,
    sent_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    clicked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Indexes
CREATE INDEX idx_notification_logs_user_id ON public.notification_logs(user_id);
CREATE INDEX idx_notification_logs_type_status ON public.notification_logs(notification_type, status);
CREATE INDEX idx_notification_logs_created_at ON public.notification_logs(created_at DESC);
```

### 3. Update Existing Table: `user_drill_progress`

Add timezone support for personalized notification timing.

```sql
-- Add new columns to user_drill_progress
ALTER TABLE public.user_drill_progress 
ADD COLUMN IF NOT EXISTS timezone TEXT DEFAULT 'Asia/Kolkata',
ADD COLUMN IF NOT EXISTS morning_notification_time TIME DEFAULT '08:00:00',
ADD COLUMN IF NOT EXISTS reminder_notification_time TIME DEFAULT '11:00:00',
ADD COLUMN IF NOT EXISTS enable_morning_notification BOOLEAN DEFAULT true,
ADD COLUMN IF NOT EXISTS enable_reminder_notification BOOLEAN DEFAULT true;

-- Update existing notification_enabled to control all notifications
COMMENT ON COLUMN public.user_drill_progress.notification_enabled IS 'Master switch for all notifications';
COMMENT ON COLUMN public.user_drill_progress.enable_morning_notification IS 'Enable 8 AM morning notification';
COMMENT ON COLUMN public.user_drill_progress.enable_reminder_notification IS 'Enable 11 AM reminder notification';
```

---

## 🔔 Notification Logic

### Notification Type 1: Morning Notification (8:00 AM)

**Purpose**: Announce the new daily question to all users

**Target Audience**: 
- All users with `notification_enabled = true`
- All users with `enable_morning_notification = true`

**Notification Content**:
```json
{
  "title": "🎯 Your Daily Drill is Ready!",
  "body": "A new question awaits. Build your interview confidence today!",
  "data": {
    "screen": "DailyDrillScreen",
    "action": "open_daily_drill",
    "notification_type": "morning",
    "drill_date": "2026-02-14"
  },
  "android": {
    "priority": "high",
    "notification": {
      "sound": "default",
      "color": "#0072B1",
      "icon": "ic_notification"
    }
  },
  "apns": {
    "payload": {
      "aps": {
        "sound": "default",
        "badge": 1
      }
    }
  }
}
```

**Selection Query**:
```sql
-- Get all users eligible for morning notification
SELECT DISTINCT 
    u.id as user_id,
    uft.fcm_token,
    uft.device_type,
    udp.current_streak,
    udp.timezone
FROM auth.users u
INNER JOIN user_drill_progress udp ON u.id = udp.user_id
INNER JOIN user_fcm_tokens uft ON u.id = uft.user_id
WHERE udp.notification_enabled = true
  AND udp.enable_morning_notification = true
  AND uft.is_active = true
ORDER BY u.id;
```

---

### Notification Type 2: Reminder Notification (11:00 AM)

**Purpose**: Remind users who haven't completed today's drill

**Target Audience**: 
- Users with `notification_enabled = true`
- Users with `enable_reminder_notification = true`
- Users who have NOT completed today's drill (status != 'completed')

**Notification Content**:
```json
{
  "title": "⏰ Don't Break Your Streak!",
  "body": "You have {hours} hours left to complete today's drill. Current streak: {streak} days 🔥",
  "data": {
    "screen": "DailyDrillScreen",
    "action": "open_daily_drill",
    "notification_type": "reminder",
    "drill_date": "2026-02-14",
    "current_streak": 5
  },
  "android": {
    "priority": "high",
    "notification": {
      "sound": "default",
      "color": "#FF6B35",
      "icon": "ic_notification"
    }
  },
  "apns": {
    "payload": {
      "aps": {
        "sound": "default",
        "badge": 1
      }
    }
  }
}
```

**Selection Query**:
```sql
-- Get users who haven't completed today's drill
SELECT DISTINCT 
    u.id as user_id,
    uft.fcm_token,
    uft.device_type,
    udp.current_streak,
    udp.timezone,
    udd.status as drill_status
FROM auth.users u
INNER JOIN user_drill_progress udp ON u.id = udp.user_id
INNER JOIN user_fcm_tokens uft ON u.id = uft.user_id
LEFT JOIN user_daily_drills udd ON u.id = udd.user_id 
    AND udd.assigned_date = CURRENT_DATE
WHERE udp.notification_enabled = true
  AND udp.enable_reminder_notification = true
  AND uft.is_active = true
  AND (udd.status IS NULL OR udd.status != 'completed')
ORDER BY u.id;
```

---

## 🚀 Implementation Components

### Component 1: Supabase Edge Function - `daily-drill-notifier`

**File**: `supabase/functions/daily-drill-notifier/index.ts`

**Actions**:
1. `send_morning_notification` - Sends 8 AM notification to all users
2. `send_reminder_notification` - Sends 11 AM reminder to incomplete users
3. `register_fcm_token` - Registers device FCM token from Flutter app
4. `update_notification_preferences` - Updates user notification settings

**Key Features**:
- Batch processing (send notifications in batches of 500)
- Error handling and retry logic
- Logging to `notification_logs` table
- Rate limiting to prevent FCM quota exhaustion
- Timezone-aware scheduling

---

### Component 2: Supabase Cron Jobs (pg_cron)

**Setup via Supabase Dashboard or SQL**:

```sql
-- Enable pg_cron extension (if not already enabled)
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Job 1: Morning Notification (8:00 AM IST = 2:30 AM UTC)
SELECT cron.schedule(
    'daily-drill-morning-notification',
    '30 2 * * *',  -- 2:30 AM UTC = 8:00 AM IST
    $$
    SELECT
      net.http_post(
          url:='https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-notifier',
          headers:='{"Content-Type": "application/json", "Authorization": "Bearer YOUR_SERVICE_ROLE_KEY"}'::jsonb,
          body:='{"action": "send_morning_notification"}'::jsonb
      ) as request_id;
    $$
);

-- Job 2: Reminder Notification (11:00 AM IST = 5:30 AM UTC)
SELECT cron.schedule(
    'daily-drill-reminder-notification',
    '30 5 * * *',  -- 5:30 AM UTC = 11:00 AM IST
    $$
    SELECT
      net.http_post(
          url:='https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-notifier',
          headers:='{"Content-Type": "application/json", "Authorization": "Bearer YOUR_SERVICE_ROLE_KEY"}'::jsonb,
          body:='{"action": "send_reminder_notification"}'::jsonb
      ) as request_id;
    $$
);
```

**Note**: Adjust cron times based on your target timezone. IST is UTC+5:30.

---

### Component 3: Flutter App Integration

**Package**: `firebase_messaging: ^14.7.0`

**Key Implementation Points**:

1. **Request Notification Permission**
```dart
// On app launch
final messaging = FirebaseMessaging.instance;
final settings = await messaging.requestPermission(
  alert: true,
  badge: true,
  sound: true,
);
```

2. **Get FCM Token and Register**
```dart
final fcmToken = await messaging.getToken();
if (fcmToken != null) {
  await _registerFCMToken(fcmToken);
}

// Listen for token refresh
messaging.onTokenRefresh.listen((newToken) {
  _registerFCMToken(newToken);
});
```

3. **Handle Foreground Notifications**
```dart
FirebaseMessaging.onMessage.listen((RemoteMessage message) {
  // Show in-app notification or update UI
  _showLocalNotification(message);
});
```

4. **Handle Background/Terminated Notifications**
```dart
FirebaseMessaging.onMessageOpenedApp.listen((RemoteMessage message) {
  _handleNotificationTap(message);
});

// Check for notification that opened the app
final initialMessage = await messaging.getInitialMessage();
if (initialMessage != null) {
  _handleNotificationTap(initialMessage);
}
```

5. **Deep Linking to Daily Drill Screen**
```dart
void _handleNotificationTap(RemoteMessage message) {
  final data = message.data;
  if (data['screen'] == 'DailyDrillScreen') {
    context.go('/daily-drill');
  }
}
```

---

## 📊 Analytics & Monitoring

### Key Metrics to Track

1. **Notification Delivery Rate**
   - Total sent vs. delivered
   - Failed notifications and reasons

2. **Click-Through Rate (CTR)**
   - Notifications clicked / notifications delivered
   - Separate CTR for morning vs. reminder

3. **Completion Rate Impact**
   - Drill completion rate with notifications ON vs. OFF
   - Time to completion after notification

4. **Optimal Timing Analysis**
   - Which notification time has highest CTR
   - User timezone distribution

### Analytics Queries

```sql
-- Daily notification performance
SELECT 
    notification_type,
    COUNT(*) as total_sent,
    COUNT(*) FILTER (WHERE status = 'delivered') as delivered,
    COUNT(*) FILTER (WHERE status = 'clicked') as clicked,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'clicked') / 
          NULLIF(COUNT(*) FILTER (WHERE status = 'delivered'), 0), 2) as ctr_percentage
FROM notification_logs
WHERE created_at >= CURRENT_DATE
GROUP BY notification_type;

-- User engagement after notification
SELECT 
    nl.notification_type,
    COUNT(DISTINCT nl.user_id) as users_notified,
    COUNT(DISTINCT udd.user_id) FILTER (WHERE udd.status = 'completed') as users_completed,
    ROUND(100.0 * COUNT(DISTINCT udd.user_id) FILTER (WHERE udd.status = 'completed') / 
          NULLIF(COUNT(DISTINCT nl.user_id), 0), 2) as completion_rate
FROM notification_logs nl
LEFT JOIN user_daily_drills udd ON nl.user_id = udd.user_id 
    AND udd.assigned_date = CURRENT_DATE
WHERE nl.created_at >= CURRENT_DATE
GROUP BY nl.notification_type;
```

---

## 🔐 Security & Privacy

### 1. **Token Security**
- FCM tokens stored encrypted in database
- Tokens automatically invalidated after 60 days of inactivity
- Regular cleanup of expired tokens

### 2. **User Privacy**
- Users can disable notifications at any time
- Granular control (morning vs. reminder)
- No notification content contains sensitive data

### 3. **Rate Limiting**
- Max 2 notifications per user per day
- Batch processing to prevent FCM quota exhaustion
- Exponential backoff for failed deliveries

### 4. **GDPR Compliance**
- Notification logs retained for 90 days only
- User can request deletion of all notification data
- Clear opt-in/opt-out mechanism

---

## 🎯 Success Criteria

### Phase 1: MVP (Week 1-2)
- ✅ Morning notifications sent to all users at 8 AM IST
- ✅ Reminder notifications sent to incomplete users at 11 AM IST
- ✅ FCM token registration working
- ✅ Deep linking to Daily Drill screen functional

### Phase 2: Optimization (Week 3-4)
- ✅ Personalized notification timing based on user timezone
- ✅ A/B testing different notification copy
- ✅ Analytics dashboard for notification performance
- ✅ Automatic token cleanup and refresh

### Phase 3: Advanced Features (Week 5+)
- ✅ Streak milestone notifications (7, 30, 100 days)
- ✅ Achievement unlock notifications
- ✅ Weekly progress summary notifications
- ✅ Smart notification timing based on user behavior

---

## 🚧 Potential Challenges & Solutions

### Challenge 1: Timezone Handling
**Problem**: Users in different timezones receive notifications at wrong times

**Solution**: 
- Store user timezone in `user_drill_progress`
- Use multiple cron jobs for different timezone buckets
- Alternative: Use Supabase Edge Functions with scheduled tasks

### Challenge 2: FCM Token Expiration
**Problem**: Tokens become invalid over time

**Solution**:
- Implement token refresh listener in Flutter
- Mark tokens as inactive after delivery failures
- Automatic cleanup job to remove expired tokens

### Challenge 3: Notification Fatigue
**Problem**: Too many notifications annoy users

**Solution**:
- Limit to 2 notifications per day maximum
- Provide granular notification controls
- Implement "Do Not Disturb" hours
- Smart frequency based on user engagement

### Challenge 4: Cost Management (FCM Quota)
**Problem**: High notification volume may exceed free tier

**Solution**:
- Batch notifications (500 per batch)
- Use FCM topics for broadcast messages
- Implement exponential backoff for retries
- Monitor quota usage via analytics

---

## 📝 Next Steps

1. **Setup Firebase Project**
   - Create Firebase project
   - Add Android/iOS apps
   - Download `google-services.json` and `GoogleService-Info.plist`

2. **Create Edge Function**
   - Implement `daily-drill-notifier` function
   - Add FCM Admin SDK integration
   - Deploy and test

3. **Setup Database Schema**
   - Run migration to create `user_fcm_tokens` table
   - Run migration to create `notification_logs` table
   - Update `user_drill_progress` table

4. **Configure Cron Jobs**
   - Setup morning notification cron (8 AM IST)
   - Setup reminder notification cron (11 AM IST)
   - Test with manual triggers

5. **Flutter Integration**
   - Add `firebase_messaging` package
   - Implement FCM token registration
   - Add notification handlers
   - Test deep linking

6. **Testing & Monitoring**
   - Test on Android, iOS, and Web
   - Monitor notification delivery rates
   - Track user engagement metrics
   - Iterate based on analytics

---

**Created**: 2026-02-14  
**Author**: Antigravity AI Assistant  
**Project**: InterviPrep - Daily Drill Notification System  
**Version**: 1.0
