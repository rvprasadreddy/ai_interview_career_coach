# Daily Drill Notification System - Supabase-Native Architecture

## 🎯 Overview

This notification system is **100% Supabase-native** with NO Firebase dependency. It uses:

1. **Supabase Realtime** - For in-app notifications
2. **Web Push API** - For browser push notifications (Android, iOS, Desktop)
3. **OneSignal/Expo Push** (Optional) - For native mobile apps
4. **Supabase Edge Functions** - For notification orchestration

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
│  └────────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Job 2: Daily Reminder Notification (11:00 AM IST)             │ │
│  │  - Triggers: daily-drill-notifier Edge Function                │ │
│  │  - Action: send_reminder_notification                          │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  SUPABASE EDGE FUNCTION: daily-drill-notifier                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  1. Query eligible users                                       │ │
│  │  2. Generate notification payloads                             │ │
│  │  3. Insert into notifications table                            │ │
│  │  4. Broadcast via Supabase Realtime                            │ │
│  │  5. Send Web Push (if subscribed)                              │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                               │
                ┌──────────────┴──────────────┐
                ▼                             ▼
┌───────────────────────────┐   ┌───────────────────────────┐
│  SUPABASE REALTIME        │   │  WEB PUSH API             │
│  (In-App Notifications)   │   │  (Browser Push)           │
│  - Instant delivery       │   │  - Works when app closed  │
│  - No setup required      │   │  - Android, iOS, Desktop  │
│  - Works when app open    │   │  - VAPID keys required    │
└───────────────────────────┘   └───────────────────────────┘
                │                             │
                └──────────────┬──────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  FLUTTER APP (Client)                                               │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  - Subscribes to Realtime notifications channel               │ │
│  │  - Registers for Web Push (browser)                            │ │
│  │  - Displays notifications in-app                               │ │
│  │  - Handles deep linking to Daily Drill screen                  │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Database Schema (Simplified)

### 1. Notifications Table

```sql
CREATE TABLE public.notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    
    -- Notification content
    type TEXT NOT NULL CHECK (type IN ('morning', 'reminder', 'streak', 'achievement')),
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    data JSONB DEFAULT '{}'::jsonb,
    
    -- Delivery tracking
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'delivered', 'read', 'clicked')),
    delivered_at TIMESTAMPTZ,
    read_at TIMESTAMPTZ,
    clicked_at TIMESTAMPTZ,
    
    -- Metadata
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ DEFAULT now() + INTERVAL '7 days'
);

-- Indexes
CREATE INDEX idx_notifications_user_status ON notifications(user_id, status);
CREATE INDEX idx_notifications_created ON notifications(created_at DESC);
CREATE INDEX idx_notifications_expires ON notifications(expires_at) WHERE status = 'pending';

-- RLS Policies
ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view their own notifications"
    ON notifications FOR SELECT
    USING (auth.uid() = user_id);

CREATE POLICY "Users can update their own notifications"
    ON notifications FOR UPDATE
    USING (auth.uid() = user_id);
```

### 2. Web Push Subscriptions Table

```sql
CREATE TABLE public.web_push_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    
    -- Web Push subscription data
    endpoint TEXT NOT NULL,
    p256dh TEXT NOT NULL,
    auth TEXT NOT NULL,
    
    -- Device info
    user_agent TEXT,
    device_type TEXT CHECK (device_type IN ('android', 'ios', 'desktop', 'unknown')),
    
    -- Status
    is_active BOOLEAN DEFAULT true,
    last_used_at TIMESTAMPTZ DEFAULT now(),
    
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(user_id, endpoint)
);

-- Indexes
CREATE INDEX idx_web_push_user ON web_push_subscriptions(user_id);
CREATE INDEX idx_web_push_active ON web_push_subscriptions(is_active) WHERE is_active = true;

-- RLS Policies
ALTER TABLE web_push_subscriptions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can manage their own subscriptions"
    ON web_push_subscriptions FOR ALL
    USING (auth.uid() = user_id);
```

---

## 🔔 Notification Delivery Methods

### Method 1: Supabase Realtime (Primary - In-App)

**Advantages**:
- ✅ No setup required
- ✅ Instant delivery
- ✅ Works perfectly with Flutter
- ✅ 100% Supabase-native
- ✅ Free tier friendly

**How it works**:
1. Edge Function inserts notification into `notifications` table
2. Supabase Realtime broadcasts to subscribed clients
3. Flutter app receives notification instantly
4. Display in-app notification banner

**Flutter Implementation**:
```dart
// Subscribe to notifications channel
final subscription = supabase
    .from('notifications')
    .stream(primaryKey: ['id'])
    .eq('user_id', userId)
    .eq('status', 'pending')
    .listen((List<Map<String, dynamic>> data) {
      // New notification received
      for (var notification in data) {
        _showInAppNotification(notification);
      }
    });
```

### Method 2: Web Push API (Secondary - Background)

**Advantages**:
- ✅ Works when app is closed
- ✅ Native browser notifications
- ✅ Supports Android, iOS (PWA), Desktop
- ✅ No third-party service needed

**How it works**:
1. User grants notification permission
2. Browser generates push subscription
3. App sends subscription to Supabase
4. Edge Function sends Web Push via VAPID

**Requirements**:
- VAPID keys (generated once)
- HTTPS (Supabase provides this)
- Service Worker (for Flutter Web)

---

## 🚀 Implementation

### Step 1: Database Migration

**File**: `supabase/migrations/20260214_supabase_native_notifications.sql`

```sql
-- Notifications table
CREATE TABLE public.notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    type TEXT NOT NULL CHECK (type IN ('morning', 'reminder', 'streak', 'achievement')),
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    data JSONB DEFAULT '{}'::jsonb,
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'delivered', 'read', 'clicked')),
    delivered_at TIMESTAMPTZ,
    read_at TIMESTAMPTZ,
    clicked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ DEFAULT now() + INTERVAL '7 days'
);

CREATE INDEX idx_notifications_user_status ON notifications(user_id, status);
CREATE INDEX idx_notifications_created ON notifications(created_at DESC);

ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view their own notifications"
    ON notifications FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can update their own notifications"
    ON notifications FOR UPDATE USING (auth.uid() = user_id);

-- Web Push subscriptions table
CREATE TABLE public.web_push_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    endpoint TEXT NOT NULL,
    p256dh TEXT NOT NULL,
    auth TEXT NOT NULL,
    user_agent TEXT,
    device_type TEXT CHECK (device_type IN ('android', 'ios', 'desktop', 'unknown')),
    is_active BOOLEAN DEFAULT true,
    last_used_at TIMESTAMPTZ DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, endpoint)
);

CREATE INDEX idx_web_push_user ON web_push_subscriptions(user_id);
CREATE INDEX idx_web_push_active ON web_push_subscriptions(is_active) WHERE is_active = true;

ALTER TABLE web_push_subscriptions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can manage their own subscriptions"
    ON web_push_subscriptions FOR ALL USING (auth.uid() = user_id);

-- Helper function to get eligible users
CREATE OR REPLACE FUNCTION get_notification_eligible_users(notification_type TEXT)
RETURNS TABLE (
    user_id UUID,
    current_streak INT,
    timezone TEXT
) AS $$
BEGIN
    IF notification_type = 'morning' THEN
        RETURN QUERY
        SELECT u.id, udp.current_streak, udp.timezone
        FROM auth.users u
        INNER JOIN user_drill_progress udp ON u.id = udp.user_id
        WHERE udp.notification_enabled = true
          AND udp.enable_morning_notification = true;
    ELSIF notification_type = 'reminder' THEN
        RETURN QUERY
        SELECT u.id, udp.current_streak, udp.timezone
        FROM auth.users u
        INNER JOIN user_drill_progress udp ON u.id = udp.user_id
        LEFT JOIN user_daily_drills udd ON u.id = udd.user_id 
            AND udd.assigned_date = CURRENT_DATE
        WHERE udp.notification_enabled = true
          AND udp.enable_reminder_notification = true
          AND (udd.status IS NULL OR udd.status != 'completed');
    END IF;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Cleanup old notifications
CREATE OR REPLACE FUNCTION cleanup_old_notifications()
RETURNS INT AS $$
DECLARE
    affected_count INT;
BEGIN
    DELETE FROM notifications WHERE expires_at < now();
    GET DIAGNOSTICS affected_count = ROW_COUNT;
    RETURN affected_count;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### Step 2: Edge Function (Supabase-Native)

**File**: `supabase/functions/daily-drill-notifier/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const SUPABASE_URL = Deno.env.get('SUPABASE_URL')!;
const SUPABASE_SERVICE_KEY = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!;

// Web Push VAPID keys (generate using: npx web-push generate-vapid-keys)
const VAPID_PUBLIC_KEY = Deno.env.get('VAPID_PUBLIC_KEY')!;
const VAPID_PRIVATE_KEY = Deno.env.get('VAPID_PRIVATE_KEY')!;
const VAPID_SUBJECT = 'mailto:support@intervi-prep.com';

serve(async (req) => {
    try {
        const { action } = await req.json();
        const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_KEY);

        if (action === 'send_morning_notification') {
            return await sendMorningNotifications(supabase);
        } else if (action === 'send_reminder_notification') {
            return await sendReminderNotifications(supabase);
        }

        throw new Error('Unknown action');
    } catch (error) {
        return new Response(JSON.stringify({ error: error.message }), {
            status: 400,
            headers: { 'Content-Type': 'application/json' },
        });
    }
});

async function sendMorningNotifications(supabase: any) {
    // Get eligible users
    const { data: users, error } = await supabase.rpc(
        'get_notification_eligible_users',
        { notification_type: 'morning' }
    );

    if (error || !users) {
        throw error;
    }

    // Create notifications for each user
    const notifications = users.map((user: any) => ({
        user_id: user.user_id,
        type: 'morning',
        title: '🎯 Your Daily Drill is Ready!',
        body: 'A new question awaits. Build your interview confidence today!',
        data: {
            screen: 'DailyDrillScreen',
            action: 'open_daily_drill',
            drill_date: new Date().toISOString().split('T')[0],
            current_streak: user.current_streak,
        },
        status: 'pending',
    }));

    // Insert notifications (Realtime will broadcast automatically)
    const { data: inserted, error: insertError } = await supabase
        .from('notifications')
        .insert(notifications)
        .select();

    if (insertError) throw insertError;

    // Send Web Push to subscribed users
    await sendWebPushNotifications(supabase, inserted);

    return new Response(
        JSON.stringify({
            success: true,
            notifications_sent: inserted.length,
        }),
        { headers: { 'Content-Type': 'application/json' } }
    );
}

async function sendReminderNotifications(supabase: any) {
    const { data: users, error } = await supabase.rpc(
        'get_notification_eligible_users',
        { notification_type: 'reminder' }
    );

    if (error || !users) throw error;

    const notifications = users.map((user: any) => ({
        user_id: user.user_id,
        type: 'reminder',
        title: '⏰ Don\'t Break Your Streak!',
        body: `Complete today's drill and keep growing. Current streak: ${user.current_streak} days 🔥`,
        data: {
            screen: 'DailyDrillScreen',
            action: 'open_daily_drill',
            drill_date: new Date().toISOString().split('T')[0],
            current_streak: user.current_streak,
        },
        status: 'pending',
    }));

    const { data: inserted, error: insertError } = await supabase
        .from('notifications')
        .insert(notifications)
        .select();

    if (insertError) throw insertError;

    await sendWebPushNotifications(supabase, inserted);

    return new Response(
        JSON.stringify({
            success: true,
            notifications_sent: inserted.length,
        }),
        { headers: { 'Content-Type': 'application/json' } }
    );
}

async function sendWebPushNotifications(supabase: any, notifications: any[]) {
    // Get Web Push subscriptions for these users
    const userIds = notifications.map(n => n.user_id);
    
    const { data: subscriptions } = await supabase
        .from('web_push_subscriptions')
        .select('*')
        .in('user_id', userIds)
        .eq('is_active', true);

    if (!subscriptions || subscriptions.length === 0) return;

    // Send Web Push to each subscription
    for (const subscription of subscriptions) {
        const notification = notifications.find(n => n.user_id === subscription.user_id);
        if (!notification) continue;

        try {
            await sendWebPush(subscription, notification);
        } catch (error) {
            console.error('Web Push failed:', error);
            // Mark subscription as inactive if it failed
            await supabase
                .from('web_push_subscriptions')
                .update({ is_active: false })
                .eq('id', subscription.id);
        }
    }
}

async function sendWebPush(subscription: any, notification: any) {
    // Use web-push library or native fetch
    // This is a simplified example - you'd use the web-push npm package
    const payload = JSON.stringify({
        title: notification.title,
        body: notification.body,
        data: notification.data,
    });

    // Implementation would use web-push library
    // For now, this is a placeholder
    console.log('Sending Web Push:', payload);
}
```

### Step 3: Flutter Integration (Supabase Realtime)

**File**: `lib/core/services/notification_service.dart`

```dart
import 'package:supabase_flutter/supabase_flutter.dart';
import 'package:flutter/material.dart';

class NotificationService {
  static final NotificationService _instance = NotificationService._internal();
  factory NotificationService() => _instance;
  NotificationService._internal();

  RealtimeChannel? _notificationChannel;

  /// Initialize notification service
  Future<void> initialize() async {
    final supabase = Supabase.instance.client;
    final userId = supabase.auth.currentUser?.id;

    if (userId == null) {
      debugPrint('Cannot initialize notifications: User not authenticated');
      return;
    }

    // Subscribe to notifications via Realtime
    _notificationChannel = supabase
        .channel('notifications:$userId')
        .onPostgresChanges(
          event: PostgresChangeEvent.insert,
          schema: 'public',
          table: 'notifications',
          filter: PostgresChangeFilter(
            type: PostgresChangeFilterType.eq,
            column: 'user_id',
            value: userId,
          ),
          callback: (payload) {
            _handleNewNotification(payload.newRecord);
          },
        )
        .subscribe();

    debugPrint('Notification service initialized');
  }

  /// Handle new notification
  void _handleNewNotification(Map<String, dynamic> notification) {
    debugPrint('New notification received: ${notification['title']}');

    // Show in-app notification
    _showInAppNotification(
      title: notification['title'],
      body: notification['body'],
      data: notification['data'],
    );

    // Mark as delivered
    _markAsDelivered(notification['id']);
  }

  /// Show in-app notification
  void _showInAppNotification({
    required String title,
    required String body,
    required Map<String, dynamic> data,
  }) {
    // Use your preferred notification display method
    // e.g., SnackBar, overlay, or notification banner
    debugPrint('Showing notification: $title - $body');
  }

  /// Mark notification as delivered
  Future<void> _markAsDelivered(String notificationId) async {
    final supabase = Supabase.instance.client;
    await supabase
        .from('notifications')
        .update({
          'status': 'delivered',
          'delivered_at': DateTime.now().toIso8601String(),
        })
        .eq('id', notificationId);
  }

  /// Mark notification as read
  Future<void> markAsRead(String notificationId) async {
    final supabase = Supabase.instance.client;
    await supabase
        .from('notifications')
        .update({
          'status': 'read',
          'read_at': DateTime.now().toIso8601String(),
        })
        .eq('id', notificationId);
  }

  /// Mark notification as clicked
  Future<void> markAsClicked(String notificationId) async {
    final supabase = Supabase.instance.client;
    await supabase
        .from('notifications')
        .update({
          'status': 'clicked',
          'clicked_at': DateTime.now().toIso8601String(),
        })
        .eq('id', notificationId);
  }

  /// Get unread notifications
  Future<List<Map<String, dynamic>>> getUnreadNotifications() async {
    final supabase = Supabase.instance.client;
    final userId = supabase.auth.currentUser?.id;

    if (userId == null) return [];

    final response = await supabase
        .from('notifications')
        .select()
        .eq('user_id', userId)
        .in_('status', ['pending', 'delivered'])
        .order('created_at', ascending: false)
        .limit(50);

    return List<Map<String, dynamic>>.from(response);
  }

  /// Dispose
  void dispose() {
    _notificationChannel?.unsubscribe();
  }
}
```

---

## ✅ Advantages of Supabase-Native Approach

1. **No Firebase Dependency**
   - ✅ No Firebase project needed
   - ✅ No FCM server key
   - ✅ No google-services.json
   - ✅ Simpler setup

2. **100% Supabase Ecosystem**
   - ✅ Single platform for everything
   - ✅ Unified authentication
   - ✅ Consistent API
   - ✅ Better integration

3. **Cost Effective**
   - ✅ Realtime included in Supabase free tier
   - ✅ No additional service costs
   - ✅ Predictable pricing

4. **Simpler Architecture**
   - ✅ Fewer moving parts
   - ✅ Easier to maintain
   - ✅ Less configuration
   - ✅ Faster development

5. **Better for Web/PWA**
   - ✅ Web Push API works great
   - ✅ No native app required
   - ✅ Progressive Web App friendly

---

## 📊 Comparison

| Feature | Firebase FCM | Supabase Native |
|---------|--------------|-----------------|
| Setup Complexity | High | Low |
| Dependencies |Supabase only |
| In-App Notifications | ✅ | ✅ (Realtime) |
| Background Notifications | ✅ | ✅ (Web Push) |
| Native Mobile | ✅ Excellent | ⚠️ Web Push (PWA) |
| Web/Desktop | ✅ | ✅ Excellent |
| Cost | Free tier | Free tier |
| Maintenance | Medium | Low |

**Recommendation**: Use Supabase-native for web/PWA apps. For native mobile apps with background notifications, consider OneSignal (free tier) or Expo Push as lightweight alternatives.

---

**Created**: 2026-02-14  
**Version**: 3.0 - Supabase Native  
**Status**: Ready for Implementation
