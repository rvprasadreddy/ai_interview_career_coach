# Daily Drill Notification System - Deployment Guide

## 📋 Prerequisites

Before deploying the notification system, ensure you have:

1. ✅ Supabase project set up
2. ✅ Firebase project created
3. ✅ Supabase CLI installed (`npm install -g supabase`)
4. ✅ Flutter project configured with Firebase

---

## 🚀 Deployment Steps

### Step 1: Firebase Setup

#### 1.1 Create Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Click "Add Project"
3. Enter project name: `intervi-prep` (or your preferred name)
4. Disable Google Analytics (optional)
5. Click "Create Project"

#### 1.2 Add Android App

1. In Firebase Console, click "Add App" → Android
2. Enter package name: `com.antigravity.aiinterviewcoach`
3. Download `google-services.json`
4. Place it in `android/app/google-services.json`

#### 1.3 Add iOS App

1. In Firebase Console, click "Add App" → iOS
2. Enter bundle ID: `com.antigravity.aiinterviewcoach`
3. Download `GoogleService-Info.plist`
4. Place it in `ios/Runner/GoogleService-Info.plist`

#### 1.4 Get FCM Server Key

1. In Firebase Console, go to Project Settings → Cloud Messaging
2. Under "Cloud Messaging API (Legacy)", enable the API if disabled
3. Copy the "Server Key" - you'll need this for Supabase

---

### Step 2: Database Migration

#### 2.1 Run Migration

```bash
# Navigate to project directory
cd c:\flutter_apps\intervi_prep

# Run the notification schema migration
supabase db push --db-url "your-supabase-db-url"

# Or apply manually via Supabase Dashboard SQL Editor
# Copy contents of: supabase/migrations/20260214_daily_drill_notifications.sql
```

#### 2.2 Verify Tables Created

Run this query in Supabase SQL Editor to verify:

```sql
SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'public' 
  AND table_name IN ('user_fcm_tokens', 'notification_logs', 'notification_metrics');
```

You should see all 3 tables.

---

### Step 3: Deploy Edge Function

#### 3.1 Set FCM Server Key as Secret

```bash
# Set FCM Server Key in Supabase
supabase secrets set FCM_SERVER_KEY="your-fcm-server-key-here"
```

#### 3.2 Deploy the Function

```bash
# Deploy daily-drill-notifier function
supabase functions deploy daily-drill-notifier

# Verify deployment
supabase functions list
```

#### 3.3 Test the Function

```bash
# Test FCM token registration (replace with your JWT token)
curl -X POST \
  'https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-notifier' \
  -H 'Authorization: Bearer YOUR_JWT_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "action": "register_fcm_token",
    "payload": {
      "fcm_token": "test-token-123",
      "device_type": "android",
      "device_info": {"test": true}
    }
  }'
```

---

### Step 4: Setup Cron Jobs

#### 4.1 Update Cron Configuration

1. Open `supabase/migrations/20260214_notification_cron_jobs.sql`
2. Replace `YOUR_PROJECT_REF` with your Supabase project reference
3. Replace `YOUR_SERVICE_ROLE_KEY` with your service role key (from Supabase Dashboard → Settings → API)

#### 4.2 Apply Cron Jobs

```sql
-- Run this in Supabase SQL Editor
-- Copy contents of: supabase/migrations/20260214_notification_cron_jobs.sql
```

#### 4.3 Verify Cron Jobs

```sql
-- View scheduled jobs
SELECT * FROM cron.job;

-- View job run history
SELECT * FROM cron.job_run_details 
ORDER BY start_time DESC 
LIMIT 10;
```

---

### Step 5: Flutter Integration

#### 5.1 Add Dependencies

Add to `pubspec.yaml`:

```yaml
dependencies:
  firebase_core: ^2.24.2
  firebase_messaging: ^14.7.9
  flutter_local_notifications: ^16.3.0
```

Run:
```bash
flutter pub get
```

#### 5.2 Android Configuration

**File**: `android/app/build.gradle`

```gradle
dependencies {
    // ... existing dependencies
    implementation platform('com.google.firebase:firebase-bom:32.7.0')
    implementation 'com.google.firebase:firebase-messaging'
}
```

**File**: `android/build.gradle`

```gradle
buildscript {
    dependencies {
        // ... existing dependencies
        classpath 'com.google.gms:google-services:4.4.0'
    }
}
```

**File**: `android/app/build.gradle` (bottom of file)

```gradle
apply plugin: 'com.google.gms.google-services'
```

**File**: `android/app/src/main/AndroidManifest.xml`

Add inside `<application>` tag:

```xml
<meta-data
    android:name="com.google.firebase.messaging.default_notification_channel_id"
    android:value="daily_drill_channel" />

<meta-data
    android:name="com.google.firebase.messaging.default_notification_icon"
    android:resource="@mipmap/ic_launcher" />

<meta-data
    android:name="com.google.firebase.messaging.default_notification_color"
    android:resource="@color/notification_color" />
```

#### 5.3 iOS Configuration

**File**: `ios/Runner/AppDelegate.swift`

```swift
import UIKit
import Flutter
import FirebaseCore
import FirebaseMessaging

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    FirebaseApp.configure()
    
    if #available(iOS 10.0, *) {
      UNUserNotificationCenter.current().delegate = self
    }
    
    GeneratedPluginRegistrant.register(with: self)
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
  
  override func application(_ application: UIApplication,
                            didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    Messaging.messaging().apnsToken = deviceToken
  }
}
```

**File**: `ios/Runner/Info.plist`

Add:

```xml
<key>FirebaseAppDelegateProxyEnabled</key>
<false/>
```

#### 5.4 Initialize in Flutter App

**File**: `lib/main.dart`

```dart
import 'package:firebase_core/firebase_core.dart';
import 'core/services/push_notification_service.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize Firebase
  await Firebase.initializeApp();
  
  // Initialize Supabase
  await SupabaseConfig.initialize();
  
  // Initialize Push Notifications
  await PushNotificationService().initialize();
  
  runApp(const MyApp());
}
```

---

### Step 6: Testing

#### 6.1 Test FCM Token Registration

1. Run the app on a device/emulator
2. Check logs for: `FCM Token obtained: ...`
3. Verify in Supabase Dashboard → Table Editor → `user_fcm_tokens`

#### 6.2 Test Manual Notification

```sql
-- Send test morning notification (run in Supabase SQL Editor)
SELECT
  net.http_post(
      url:='https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-notifier',
      headers:='{"Content-Type": "application/json", "Authorization": "Bearer YOUR_SERVICE_ROLE_KEY"}'::jsonb,
      body:='{"action": "send_morning_notification"}'::jsonb
  ) as request_id;
```

#### 6.3 Verify Notification Logs

```sql
-- Check notification logs
SELECT * FROM notification_logs 
ORDER BY created_at DESC 
LIMIT 10;

-- Check metrics
SELECT * FROM notification_metrics 
ORDER BY metric_date DESC;
```

---

## 📊 Monitoring & Analytics

### View Notification Performance

```sql
-- Daily notification stats
SELECT 
    notification_type,
    COUNT(*) as total,
    COUNT(*) FILTER (WHERE status = 'sent') as sent,
    COUNT(*) FILTER (WHERE status = 'failed') as failed,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'sent') / COUNT(*), 2) as success_rate
FROM notification_logs
WHERE created_at >= CURRENT_DATE
GROUP BY notification_type;
```

### View Active Tokens

```sql
-- Active FCM tokens by device type
SELECT 
    device_type,
    COUNT(*) as active_tokens
FROM user_fcm_tokens
WHERE is_active = true
GROUP BY device_type;
```

### View User Engagement

```sql
-- Users who completed drill after notification
SELECT 
    nl.notification_type,
    COUNT(DISTINCT nl.user_id) as notified,
    COUNT(DISTINCT udd.user_id) FILTER (WHERE udd.status = 'completed') as completed,
    ROUND(100.0 * COUNT(DISTINCT udd.user_id) FILTER (WHERE udd.status = 'completed') / 
          COUNT(DISTINCT nl.user_id), 2) as completion_rate
FROM notification_logs nl
LEFT JOIN user_daily_drills udd ON nl.user_id = udd.user_id 
    AND udd.assigned_date = CURRENT_DATE
WHERE nl.created_at >= CURRENT_DATE
GROUP BY nl.notification_type;
```

---

## 🔧 Troubleshooting

### Issue: Notifications not received

**Check:**
1. FCM token registered? `SELECT * FROM user_fcm_tokens WHERE user_id = 'YOUR_USER_ID';`
2. Token marked as active? `is_active = true`
3. User preferences enabled? `SELECT * FROM user_drill_progress WHERE user_id = 'YOUR_USER_ID';`
4. Check notification logs for errors: `SELECT * FROM notification_logs WHERE status = 'failed';`

### Issue: Cron jobs not running

**Check:**
1. Jobs scheduled? `SELECT * FROM cron.job;`
2. Check job history: `SELECT * FROM cron.job_run_details ORDER BY start_time DESC;`
3. Verify Edge Function URL is correct
4. Verify Service Role Key is valid

### Issue: FCM Server Key invalid

**Fix:**
1. Regenerate key in Firebase Console
2. Update Supabase secret: `supabase secrets set FCM_SERVER_KEY="new-key"`
3. Redeploy function: `supabase functions deploy daily-drill-notifier`

---

## 🎯 Production Checklist

- [ ] Firebase project created and configured
- [ ] FCM Server Key added to Supabase secrets
- [ ] Database migration applied successfully
- [ ] Edge function deployed and tested
- [ ] Cron jobs scheduled (8 AM and 11 AM IST)
- [ ] Flutter app integrated with Firebase
- [ ] FCM token registration working
- [ ] Test notifications sent successfully
- [ ] Notification logs being created
- [ ] Metrics tracking working
- [ ] Deep linking to Daily Drill screen functional
- [ ] User preferences UI implemented
- [ ] Cleanup job scheduled (midnight UTC)

---

## 📝 Next Steps

1. **Monitor Performance**: Check notification delivery rates daily
2. **A/B Testing**: Test different notification copy for better engagement
3. **User Feedback**: Collect feedback on notification timing
4. **Optimize Timing**: Analyze user timezone distribution and adjust
5. **Add Features**: Implement streak milestone notifications, achievement unlocks

---

**Created**: 2026-02-14  
**Version**: 1.0  
**Maintainer**: Development Team
