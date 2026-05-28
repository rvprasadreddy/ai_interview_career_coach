# Migration Guide: Push Notification Service → Notification Service

**Version**: 1.0  
**Date**: 2026-02-14  
**Type**: Breaking Change - Firebase Removal

---

## 🎯 Overview

This guide helps you migrate from the Firebase-based `PushNotificationService` to the Supabase-native `NotificationService`.

**Why migrate?**
- ✅ No Firebase dependency
- ✅ Simpler architecture
- ✅ Better integration with Supabase
- ✅ Production-grade code quality
- ✅ Type-safe models
- ✅ Better error handling

---

## 📋 Pre-Migration Checklist

- [ ] Backup current code
- [ ] Review current notification usage
- [ ] Identify all files importing `push_notification_service.dart`
- [ ] Test current notification flow
- [ ] Read this guide completely

---

## 🔄 Step-by-Step Migration

### Step 1: Update Imports

Find and replace all imports:

```dart
// ❌ OLD
import 'package:intervi_prep/core/services/push_notification_service.dart';

// ✅ NEW
import 'package:intervi_prep/core/services/notification_service.dart';
```

**Files to check**:
- `lib/main.dart`
- `lib/features/daily_drill/screens/*.dart`
- `lib/features/home/screens/*.dart`
- Any screen that displays notifications

---

### Step 2: Update Initialization

**In `main.dart`**:

```dart
// ❌ OLD
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Firebase initialization
  await Firebase.initializeApp();
  
  // Supabase initialization
  await Supabase.initialize(...);
  
  // Push notification service
  await PushNotificationService().initialize();
  
  runApp(MyApp());
}

// ✅ NEW
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Supabase initialization
  await Supabase.initialize(...);
  
  // Notification service (after user login)
  // Will be called in app after authentication
  
  runApp(MyApp());
}
```

**After User Login**:

```dart
// In your authentication flow
Future<void> _handleSuccessfulLogin() async {
  // ... existing login logic
  
  // Initialize notification service
  await NotificationService().initialize();
  
  // Navigate to home
}
```

---

### Step 3: Update Notification Listeners

**OLD Approach**:
```dart
class HomeScreen extends StatefulWidget {
  @override
  void initState() {
    super.initState();
    
    // OLD: Firebase message handling
    PushNotificationService().onMessageReceived = (RemoteMessage message) {
      final title = message.notification?.title ?? '';
      final body = message.notification?.body ?? '';
      _showNotification(title, body);
    };
  }
}
```

**NEW Approach**:
```dart
class HomeScreen extends StatefulWidget {
  @override
  void initState() {
    super.initState();
    
    // NEW: Supabase Realtime handling
    NotificationService().onNotificationReceived = (NotificationModel notification) {
      _showNotification(notification);
    };
    
    NotificationService().onNotificationTapped = (NotificationModel notification) {
      _handleNotificationTap(notification);
    };
  }
  
  void _showNotification(NotificationModel notification) {
    // notification is now a typed model!
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text('${notification.title}: ${notification.body}'),
        action: SnackBarAction(
          label: 'View',
          onPressed: () => _handleNotificationTap(notification),
        ),
      ),
    );
  }
  
  void _handleNotificationTap(NotificationModel notification) {
    // Mark as clicked
    NotificationService().markAsClicked(notification.id);
    
    // Navigate based on notification data
    final screen = notification.data['screen'] as String?;
    if (screen == 'DailyDrillScreen') {
      Navigator.pushNamed(context, '/daily-drill');
    }
  }
}
```

---

### Step 4: Update Notification Display

**OLD Approach**:
```dart
// Manual notification display
void _showForegroundNotification(RemoteMessage message) {
  showDialog(
    context: context,
    builder: (context) => AlertDialog(
      title: Text(message.notification?.title ?? ''),
      content: Text(message.notification?.body ?? ''),
    ),
  );
}
```

**NEW Approach**:
```dart
// Type-safe notification display
void _showNotification(NotificationModel notification) {
  showDialog(
    context: context,
    builder: (context) => AlertDialog(
      title: Text(notification.title),
      content: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(notification.body),
          SizedBox(height: 8),
          Text(
            'Type: ${notification.type}',
            style: TextStyle(fontSize: 12, color: Colors.grey),
          ),
        ],
      ),
      actions: [
        TextButton(
          onPressed: () {
            NotificationService().markAsRead(notification.id);
            Navigator.pop(context);
          },
          child: Text('Dismiss'),
        ),
        ElevatedButton(
          onPressed: () {
            NotificationService().markAsClicked(notification.id);
            Navigator.pop(context);
            // Handle navigation
          },
          child: Text('View'),
        ),
      ],
    ),
  );
}
```

---

### Step 5: Update Unread Count Display

**OLD Approach**:
```dart
// Manual count tracking
int _unreadCount = 0;

void _loadUnreadCount() async {
  final response = await supabase
      .from('notification_logs')
      .select()
      .eq('user_id', userId)
      .eq('status', 'pending');
  
  setState(() {
    _unreadCount = response.length;
  });
}
```

**NEW Approach**:
```dart
// Built-in count tracking
Widget build(BuildContext context) {
  return Badge(
    label: Text('${NotificationService().unreadCount}'),
    isLabelVisible: NotificationService().unreadCount > 0,
    child: IconButton(
      icon: Icon(Icons.notifications),
      onPressed: _showNotifications,
    ),
  );
}

// Refresh count after marking as read
void _markAsRead(String id) async {
  await NotificationService().markAsRead(id);
  await NotificationService().refreshUnreadCount();
  setState(() {}); // Rebuild to show new count
}
```

---

### Step 6: Update Notification Preferences

**OLD Approach**:
```dart
Future<void> _updatePreferences() async {
  final success = await PushNotificationService().updateNotificationPreferences(
    enableMorningNotification: true,
    enableReminderNotification: false,
  );
  
  if (success) {
    // Show success message
  }
}
```

**NEW Approach**:
```dart
Future<void> _updatePreferences() async {
  try {
    await NotificationService().updateNotificationPreferences(
      enableMorningNotification: true,
      enableReminderNotification: false,
      morningTime: '08:00:00',
      reminderTime: '11:00:00',
    );
    
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Preferences updated')),
    );
  } catch (e) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Error: $e')),
    );
  }
}

// Load preferences
Future<void> _loadPreferences() async {
  final prefs = await NotificationService().getNotificationPreferences();
  if (prefs != null) {
    setState(() {
      _morningEnabled = prefs.enableMorningNotification;
      _reminderEnabled = prefs.enableReminderNotification;
      _morningTime = prefs.morningNotificationTime;
      _reminderTime = prefs.reminderNotificationTime;
    });
  }
}
```

---

### Step 7: Update Notification History Screen

**NEW Implementation**:
```dart
class NotificationHistoryScreen extends StatefulWidget {
  @override
  _NotificationHistoryScreenState createState() => _NotificationHistoryScreenState();
}

class _NotificationHistoryScreenState extends State<NotificationHistoryScreen> {
  List<NotificationModel> _notifications = [];
  bool _loading = true;

  @override
  void initState() {
    super.initState();
    _loadNotifications();
  }

  Future<void> _loadNotifications() async {
    setState(() => _loading = true);
    
    final notifications = await NotificationService().getAllNotifications(
      limit: 50,
      offset: 0,
    );
    
    setState(() {
      _notifications = notifications;
      _loading = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    if (_loading) {
      return Center(child: CircularProgressIndicator());
    }

    return ListView.builder(
      itemCount: _notifications.length,
      itemBuilder: (context, index) {
        final notification = _notifications[index];
        
        return ListTile(
          leading: Icon(
            _getIconForType(notification.type),
            color: notification.isUnread ? Colors.blue : Colors.grey,
          ),
          title: Text(
            notification.title,
            style: TextStyle(
              fontWeight: notification.isUnread ? FontWeight.bold : FontWeight.normal,
            ),
          ),
          subtitle: Text(notification.body),
          trailing: Text(
            _formatDate(notification.createdAt),
            style: TextStyle(fontSize: 12, color: Colors.grey),
          ),
          onTap: () => _handleNotificationTap(notification),
        );
      },
    );
  }

  IconData _getIconForType(String type) {
    switch (type) {
      case 'morning':
        return Icons.wb_sunny;
      case 'reminder':
        return Icons.alarm;
      case 'streak':
        return Icons.local_fire_department;
      case 'achievement':
        return Icons.emoji_events;
      default:
        return Icons.notifications;
    }
  }

  String _formatDate(DateTime date) {
    final now = DateTime.now();
    final diff = now.difference(date);
    
    if (diff.inMinutes < 60) {
      return '${diff.inMinutes}m ago';
    } else if (diff.inHours < 24) {
      return '${diff.inHours}h ago';
    } else {
      return '${diff.inDays}d ago';
    }
  }

  void _handleNotificationTap(NotificationModel notification) async {
    await NotificationService().markAsClicked(notification.id);
    await NotificationService().refreshUnreadCount();
    
    setState(() {
      // Update UI to reflect read status
    });
    
    // Navigate based on notification data
    final screen = notification.data['screen'] as String?;
    if (screen != null) {
      Navigator.pushNamed(context, '/$screen');
    }
  }
}
```

---

### Step 8: Remove Firebase Dependencies

**In `pubspec.yaml`**:

```yaml
# ❌ REMOVE these if present
dependencies:
  # firebase_core: ^2.x.x
  # firebase_messaging: ^14.x.x
  # flutter_local_notifications: ^16.x.x
```

Run:
```bash
flutter pub get
flutter clean
flutter pub get
```

---

### Step 9: Delete Old Files

After verifying everything works:

```bash
# Delete old service file
rm lib/core/services/push_notification_service.dart

# Delete Firebase config files (if any)
rm android/app/google-services.json
rm ios/Runner/GoogleService-Info.plist
```

---

## 🧪 Testing Checklist

### Functional Testing:
- [ ] User can receive notifications when app is open
- [ ] Notifications display correctly
- [ ] Unread count updates correctly
- [ ] Mark as read works
- [ ] Mark as clicked works
- [ ] Notification history loads
- [ ] Preferences can be updated
- [ ] Preferences persist correctly

### Integration Testing:
- [ ] Cron job triggers notification
- [ ] Edge Function creates notification
- [ ] Realtime broadcasts to client
- [ ] Client receives and displays
- [ ] Deep linking works
- [ ] Analytics tracked correctly

### Error Handling:
- [ ] Graceful handling when user not authenticated
- [ ] Proper error messages on network failure
- [ ] Recovery from Realtime disconnection
- [ ] Handling of expired notifications

---

## 🐛 Common Issues & Solutions

### Issue 1: Notifications not received

**Problem**: NotificationService initialized but no notifications received

**Solution**:
```dart
// Check initialization status
if (!NotificationService().isInitialized) {
  await NotificationService().initialize();
}

// Check user authentication
final user = Supabase.instance.client.auth.currentUser;
if (user == null) {
  print('User not authenticated - cannot receive notifications');
}
```

### Issue 2: Unread count not updating

**Problem**: Count doesn't update after marking as read

**Solution**:
```dart
// Always refresh count after marking as read
await NotificationService().markAsRead(id);
await NotificationService().refreshUnreadCount();
setState(() {}); // Trigger rebuild
```

### Issue 3: Callback not triggered

**Problem**: onNotificationReceived not called

**Solution**:
```dart
// Set callback BEFORE initialize
NotificationService().onNotificationReceived = (notification) {
  // Handle notification
};

await NotificationService().initialize();
```

---

## 📊 Comparison

| Feature | Old (Firebase) | New (Supabase) |
|---------|----------------|----------------|
| Setup Complexity | High | Low |
| Dependencies | 3+ packages | 1 package |
| Type Safety | Poor | Excellent |
| Error Handling | Basic | Comprehensive |
| Documentation | Minimal | Extensive |
| Maintenance | Medium | Low |
| Performance | Good | Excellent |
| Cost | Free tier | Free tier |

---

## ✅ Post-Migration Checklist

- [ ] All imports updated
- [ ] Initialization moved to correct location
- [ ] Notification listeners updated
- [ ] Unread count display working
- [ ] Preferences screen working
- [ ] History screen working
- [ ] Deep linking working
- [ ] Firebase dependencies removed
- [ ] Old files deleted
- [ ] All tests passing
- [ ] Production deployment successful

---

## 🚀 Next Steps

After successful migration:

1. **Monitor**: Watch logs for any errors
2. **Optimize**: Fine-tune notification timing
3. **Enhance**: Add more notification types
4. **Analyze**: Track engagement metrics
5. **Scale**: Handle increased user base

---

**Migration Complete!** 🎉

You now have a production-grade, Supabase-native notification system with no Firebase dependency!

---

**Need Help?**
- Check `CODE_ANALYSIS_REPORT.md` for technical details
- Review `NOTIFICATION_SYSTEM_SUMMARY.md` for architecture
- See `notification_service.dart` for API documentation
