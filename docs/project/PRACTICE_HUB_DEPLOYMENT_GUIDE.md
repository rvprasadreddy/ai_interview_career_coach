# 🚀 Practice Hub - Deployment Guide

**Date**: 2026-02-14  
**Status**: Ready for Deployment  
**Estimated Time**: 30-45 minutes

---

## ✅ **Pre-Deployment Checklist**

- [x] Database migrations created
- [x] Content expansion migration created
- [x] Error handling implemented
- [x] Caching service implemented
- [x] Progress tracking service created
- [ ] Dependencies added to pubspec.yaml
- [ ] SharedPreferences initialized
- [ ] Migrations deployed to Supabase
- [ ] Code tested locally

---

## 📋 **Step-by-Step Deployment**

### **Step 1: Add Dependencies** (2 minutes)

Add to `pubspec.yaml`:

```yaml
dependencies:
  shared_preferences: ^2.2.2  # For caching
```

Run:
```bash
flutter pub get
```

---

### **Step 2: Initialize SharedPreferences** (3 minutes)

Update `lib/main.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'core/services/cache_service.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize Supabase (existing code)
  await Supabase.initialize(
    url: SupabaseConfig.supabaseUrl,
    anonKey: SupabaseConfig.supabaseAnonKey,
  );
  
  // Initialize SharedPreferences for caching
  final prefs = await SharedPreferences.getInstance();
  
  runApp(
    ProviderScope(
      overrides: [
        sharedPreferencesProvider.overrideWithValue(prefs),
      ],
      child: const MyApp(),
    ),
  );
}
```

---

### **Step 3: Deploy Database Migrations** (10-15 minutes)

#### **Option A: Via Supabase Dashboard** (Recommended)

1. Go to https://supabase.com/dashboard
2. Select your project
3. Navigate to **SQL Editor**
4. Click **New Query**
5. Copy contents of `supabase/migrations/20260214_practice_hub_enhancements.sql`
6. Paste and click **Run**
7. Verify success message
8. Repeat for `supabase/migrations/20260214_expand_content_library.sql`

#### **Option B: Via Supabase CLI**

```bash
# Navigate to project directory
cd c:\flutter_apps\intervi_prep

# Apply migrations
supabase db push

# Or apply specific migration
supabase migration up --file supabase/migrations/20260214_practice_hub_enhancements.sql
supabase migration up --file supabase/migrations/20260214_expand_content_library.sql
```

---

### **Step 4: Verify Database Changes** (5 minutes)

Run these queries in Supabase SQL Editor:

```sql
-- Check if tables exist
SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'public' 
AND table_name IN ('user_learning_progress', 'content_recommendations');

-- Check content count
SELECT 
  type,
  COUNT(*) as count
FROM public.learning_content
GROUP BY type
ORDER BY type;

-- Should show:
-- article: ~50
-- faq: ~60
-- video: ~90
-- Total: ~100+

-- Check if helper functions exist
SELECT routine_name 
FROM information_schema.routines 
WHERE routine_schema = 'public' 
AND routine_name IN (
  'get_user_learning_stats',
  'start_learning_content',
  'complete_learning_content'
);

-- Test helper function
SELECT * FROM get_user_learning_stats('00000000-0000-0000-0000-000000000000');
```

---

### **Step 5: Test Error Handling** (5-10 minutes)

#### **Test Network Error**
1. Turn off WiFi/internet
2. Open Practice Hub
3. Should see "No Internet Connection" error with WiFi icon
4. Turn on internet
5. Click "Try Again"
6. Should load successfully

#### **Test Timeout Error**
1. Simulate slow network (Chrome DevTools → Network → Slow 3G)
2. Open Practice Hub
3. Should see timeout error after delay
4. Should auto-retry with exponential backoff
5. Should show retry count

#### **Test Auth Error**
1. Sign out
2. Try to access Practice Hub
3. Should see "Authentication Required" error
4. Should redirect to login

---

### **Step 6: Test Caching** (5 minutes)

#### **Test Cache Hit**
1. Open Practice Hub (first load - API call)
2. Note load time
3. Close and reopen Practice Hub (cache hit)
4. Should load much faster
5. Check console for cache logs

#### **Test Cache Expiration**
1. Open Practice Hub
2. Wait 24 hours (or manually delete cache)
3. Reopen Practice Hub
4. Should fetch fresh data

#### **Test Manual Refresh**
1. Open Practice Hub
2. Pull to refresh or click refresh button
3. Should invalidate cache and fetch fresh data

---

### **Step 7: Test Progress Tracking** (5 minutes)

#### **Test Start Content**
1. Click on a video/article
2. Check database:
```sql
SELECT * FROM user_learning_progress 
WHERE user_id = 'YOUR_USER_ID' 
ORDER BY started_at DESC 
LIMIT 5;
```
3. Should see new entry with status = 'in_progress'

#### **Test Complete Content**
1. Watch video to end or scroll article to bottom
2. Check database:
```sql
SELECT * FROM user_learning_progress 
WHERE user_id = 'YOUR_USER_ID' 
AND status = 'completed'
ORDER BY completed_at DESC 
LIMIT 5;
```
3. Should see entry with status = 'completed'

---

### **Step 8: Verify Content Library** (3 minutes)

```sql
-- Check content distribution
SELECT 
  category,
  type,
  level,
  COUNT(*) as count
FROM public.learning_content
GROUP BY category, type, level
ORDER BY category, type, level;

-- Check for FAQs
SELECT COUNT(*) FROM public.learning_content WHERE type = 'faq';
-- Should return ~60

-- Check for articles
SELECT COUNT(*) FROM public.learning_content WHERE type = 'article';
-- Should return ~50

-- Check for videos
SELECT COUNT(*) FROM public.learning_content WHERE type = 'video';
-- Should return ~90
```

---

## 🧪 **Testing Checklist**

### **Functional Tests**
- [ ] Practice Hub loads without errors
- [ ] Recommendations display correctly
- [ ] Content filtering works (difficulty, type)
- [ ] Content cards clickable
- [ ] Video player opens
- [ ] Article viewer opens
- [ ] FAQ viewer opens

### **Error Handling Tests**
- [ ] Network error shows correct message
- [ ] Timeout error shows retry logic
- [ ] Auth error redirects to login
- [ ] Server error shows status code
- [ ] Generic error shows fallback message
- [ ] Retry button works
- [ ] Exponential backoff works

### **Caching Tests**
- [ ] First load fetches from API
- [ ] Second load uses cache (faster)
- [ ] Cache expires after 24 hours
- [ ] Manual refresh invalidates cache
- [ ] Corrupted cache handled gracefully

### **Progress Tracking Tests**
- [ ] Opening content creates progress entry
- [ ] Completing content updates status
- [ ] Progress percentage updates
- [ ] Time spent tracked
- [ ] In-progress content shows badge
- [ ] Completed content shows checkmark

---

## 🐛 **Troubleshooting**

### **Issue: SharedPreferences not initialized**
**Error**: `UnimplementedError: SharedPreferences not initialized`

**Solution**:
```dart
// Ensure this is in main.dart before runApp()
final prefs = await SharedPreferences.getInstance();
runApp(
  ProviderScope(
    overrides: [
      sharedPreferencesProvider.overrideWithValue(prefs),
    ],
    child: const MyApp(),
  ),
);
```

---

### **Issue: Migration fails**
**Error**: `relation "user_learning_progress" already exists`

**Solution**:
```sql
-- Check if table exists
SELECT * FROM information_schema.tables 
WHERE table_name = 'user_learning_progress';

-- If exists, migration already applied
-- If not, check for syntax errors in migration file
```

---

### **Issue: Cache not working**
**Error**: Cache always misses

**Solution**:
```dart
// Check if provider is overridden
final prefs = await SharedPreferences.getInstance();
print('SharedPreferences initialized: ${prefs != null}');

// Check cache service
final cacheService = CacheService(prefs);
await cacheService.set('test', {'data': 'test'}, Duration(hours: 1));
final cached = await cacheService.get('test');
print('Cache test: $cached');
```

---

### **Issue: Progress tracking not working**
**Error**: Progress not saved to database

**Solution**:
```sql
-- Check RLS policies
SELECT * FROM pg_policies 
WHERE tablename = 'user_learning_progress';

-- Test RPC function
SELECT * FROM start_learning_content(
  'YOUR_USER_ID'::uuid,
  'CONTENT_ID'::uuid
);
```

---

## 📊 **Post-Deployment Verification**

### **Database Verification**
```sql
-- Total content count
SELECT COUNT(*) FROM public.learning_content;
-- Expected: 100+

-- Content by type
SELECT type, COUNT(*) FROM public.learning_content GROUP BY type;
-- Expected: video: ~90, article: ~50, faq: ~60

-- Progress tracking tables
SELECT COUNT(*) FROM public.user_learning_progress;
SELECT COUNT(*) FROM public.content_recommendations;
```

### **App Verification**
- [ ] App builds without errors
- [ ] App runs without crashes
- [ ] Practice Hub loads
- [ ] Content displays
- [ ] Filtering works
- [ ] Error handling works
- [ ] Caching works
- [ ] Progress tracking works

---

## 🎉 **Success Criteria**

✅ **Deployment Successful If**:
1. All migrations applied without errors
2. 100+ content items in database
3. Error handling shows specific messages
4. Caching reduces load time by ~80%
5. Progress tracking saves to database
6. No crashes or breaking changes
7. All tests pass

---

## 📝 **Rollback Plan**

If deployment fails:

```sql
-- Rollback migrations
DROP TABLE IF EXISTS public.user_learning_progress CASCADE;
DROP TABLE IF EXISTS public.content_recommendations CASCADE;
DROP FUNCTION IF EXISTS get_user_learning_stats CASCADE;
DROP FUNCTION IF EXISTS start_learning_content CASCADE;
DROP FUNCTION IF EXISTS complete_learning_content CASCADE;

-- Rollback content
DELETE FROM public.learning_content 
WHERE created_at > '2026-02-14';
```

---

## 🚀 **Next Steps After Deployment**

1. Monitor error logs for 24 hours
2. Check cache hit rate
3. Verify progress tracking working
4. Gather user feedback
5. Plan next phase of fixes:
   - Analytics tracking
   - Free vs. paid logic
   - Loading skeletons
   - Remove hardcoded values

---

**Estimated Total Time**: 30-45 minutes  
**Risk Level**: Low (backward compatible)  
**Rollback Time**: 5 minutes

✅ **Ready to deploy!**
