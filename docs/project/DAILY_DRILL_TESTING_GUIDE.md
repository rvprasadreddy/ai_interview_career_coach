# Daily Drill - Testing & Deployment Guide

**Version**: 2.0  
**Date**: February 10, 2026  
**Status**: Ready for Testing

---

## 🎯 Quick Start Testing

### Prerequisites
- Supabase project set up
- Flutter environment configured
- `.env` file with Supabase credentials

### 1. Deploy Database Migration

```bash
# Navigate to project root
cd c:\flutter_apps\antigravity\ai_interview_coach

# Apply migration
supabase db push

# Verify function exists
supabase db execute "SELECT proname, pronargs FROM pg_proc WHERE proname = 'get_or_assign_daily_question';"
```

**Expected Output**: Should show the function with 2 arguments

### 2. Deploy Edge Function

```bash
# Deploy the function
supabase functions deploy daily-drill-engine

# Test the function
supabase functions invoke daily-drill-engine \
  --body '{"action":"get_today_question","payload":{}}' \
  --header "Authorization: Bearer YOUR_USER_JWT_TOKEN"
```

**Expected Output**: JSON with drill and question data

### 3. Run Flutter App

```bash
# Get dependencies
flutter pub get

# Generate Freezed models (if needed)
flutter pub run build_runner build --delete-conflicting-outputs

# Run the app
flutter run
```

---

## 🧪 Comprehensive Testing Checklist

### A. Database Layer Tests

#### Test 1: Atomic Question Assignment
**Objective**: Verify no race conditions

```sql
-- Run this in two separate SQL editor tabs simultaneously
SELECT * FROM public.get_or_assign_daily_question(
    'YOUR_USER_ID'::uuid,
    CURRENT_DATE
);
```

**Expected**: Both should return the SAME question

**Pass Criteria**: ✅ Same drill_id in both results

#### Test 2: Weak Category Prioritization
**Objective**: Verify intelligent question selection

```sql
-- Set up weak categories
UPDATE user_drill_progress
SET weak_categories = ARRAY['dsa', 'system_design']
WHERE user_id = 'YOUR_USER_ID';

-- Assign 10 questions and check distribution
DO $$
DECLARE
    i INT;
    result RECORD;
    weak_count INT := 0;
BEGIN
    FOR i IN 1..10 LOOP
        -- Delete today's drill to test assignment
        DELETE FROM user_daily_drills 
        WHERE user_id = 'YOUR_USER_ID' AND assigned_date = CURRENT_DATE;
        
        -- Get new question
        SELECT * INTO result FROM public.get_or_assign_daily_question(
            'YOUR_USER_ID'::uuid,
            CURRENT_DATE
        );
        
        -- Check if it's from weak category
        IF (result.question_data->>'category')::text = ANY(ARRAY['dsa', 'system_design']) THEN
            weak_count := weak_count + 1;
        END IF;
    END LOOP;
    
    RAISE NOTICE 'Weak category questions: % out of 10', weak_count;
END $$;
```

**Expected**: ~7 out of 10 should be from weak categories

**Pass Criteria**: ✅ 5-9 questions from weak categories

#### Test 3: Streak Calculation
**Objective**: Verify streak logic

```sql
-- Complete a drill
UPDATE user_daily_drills
SET status = 'completed',
    completed_at = NOW(),
    confidence_level = 4
WHERE user_id = 'YOUR_USER_ID' 
  AND assigned_date = CURRENT_DATE;

-- Check streak
SELECT current_streak, longest_streak, last_completed_date
FROM user_drill_progress
WHERE user_id = 'YOUR_USER_ID';
```

**Expected**: Streak should increment

**Pass Criteria**: ✅ current_streak increases by 1

#### Test 4: Readiness Score Calculation
**Objective**: Verify score calculation

```sql
-- Trigger score calculation
SELECT public.calculate_readiness_score('YOUR_USER_ID'::uuid);

-- Check score
SELECT readiness_score FROM user_drill_progress
WHERE user_id = 'YOUR_USER_ID';
```

**Expected**: Score between 0-100

**Pass Criteria**: ✅ Score is calculated and reasonable

---

### B. Edge Function Tests

#### Test 1: Get Today's Question

```bash
# Using curl
curl -X POST https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-engine \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"action":"get_today_question","payload":{}}'
```

**Expected Response**:
```json
{
  "drill": {
    "id": "uuid",
    "status": "pending",
    "assigned_date": "2026-02-10"
  },
  "question": {
    "id": "uuid",
    "question_text": "...",
    "ideal_answer": "..."
  },
  "is_new": true
}
```

**Pass Criteria**: ✅ Returns valid drill and question

#### Test 2: Submit Drill Response

```bash
curl -X POST https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-engine \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "action":"submit_drill_response",
    "payload":{
      "drill_id":"YOUR_DRILL_ID",
      "action":"complete",
      "confidence_level":4,
      "user_notes":"Great question!",
      "time_spent_seconds":300
    }
  }'
```

**Expected Response**:
```json
{
  "drill": { "status": "completed" },
  "progress": { "current_streak": 1 },
  "message": "Great job! Keep the streak going!"
}
```

**Pass Criteria**: ✅ Drill marked as completed, streak updated

#### Test 3: Rate Limiting

```bash
# Run this 35 times in quick succession
for i in {1..35}; do
  curl -X POST https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-engine \
    -H "Authorization: Bearer YOUR_JWT_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"action":"get_drill_stats","payload":{}}' &
done
wait
```

**Expected**: Requests 31-35 should return 429 (Rate Limit Exceeded)

**Pass Criteria**: ✅ Rate limiting works

#### Test 4: Input Validation

```bash
# Test with invalid notes (XSS attempt)
curl -X POST https://YOUR_PROJECT_REF.supabase.co/functions/v1/daily-drill-engine \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "action":"submit_drill_response",
    "payload":{
      "drill_id":"YOUR_DRILL_ID",
      "action":"complete",
      "confidence_level":4,
      "user_notes":"<script>alert(\"XSS\")</script>Test",
      "time_spent_seconds":300
    }
  }'
```

**Expected**: Script tags should be stripped

**Pass Criteria**: ✅ Notes saved without script tags

---

### C. Frontend Tests

#### Test 1: Error Boundary

**Steps**:
1. Open Daily Drill screen
2. Force an error (e.g., invalid state)
3. Observe error boundary

**Expected**: 
- Error screen with "Critical Error" message
- Reload button available
- No app crash

**Pass Criteria**: ✅ Error caught and displayed gracefully

#### Test 2: Memory Leak Fix

**Steps**:
1. Open Daily Drill screen
2. Add notes: "Test notes 1"
3. Complete the drill
4. Wait for tomorrow's drill (or manually assign new drill)
5. Check if notes field is empty

**Expected**: Notes field should be cleared

**Pass Criteria**: ✅ Notes cleared when drill changes

#### Test 3: Loading State

**Steps**:
1. Open Daily Drill screen
2. Reveal answer
3. Select confidence level
4. Click "Mark as Complete"
5. Observe loading dialog

**Expected**:
- Loading dialog appears
- "Saving your progress..." message
- Cannot dismiss dialog
- Haptic feedback on success

**Pass Criteria**: ✅ Loading state shown, haptic feedback works

#### Test 4: Retry Logic

**Steps**:
1. Disconnect internet
2. Try to complete drill
3. Observe error snackbar
4. Click "Retry" button

**Expected**:
- Error snackbar with retry button
- Retry button triggers new attempt

**Pass Criteria**: ✅ Retry logic works

#### Test 5: Offline Support

**Steps**:
1. Load Daily Drill screen (with internet)
2. Disconnect internet
3. Pull to refresh

**Expected**: Cached drill should load

**Pass Criteria**: ✅ Offline mode works

---

### D. Integration Tests

#### Test 1: Complete User Flow

**Steps**:
1. Open app
2. Navigate to Daily Drill
3. View question
4. Click "Reveal Ideal Answer"
5. Select confidence level (4)
6. Add notes: "Learned about STAR method"
7. Click "Mark as Complete"
8. Observe confetti animation
9. Check streak updated
10. Navigate to history
11. Verify drill appears in history

**Expected**: Full flow works smoothly

**Pass Criteria**: ✅ All steps complete without errors

#### Test 2: Concurrent Users

**Setup**: 
- Create 5 test users
- Use 5 different devices/browsers

**Steps**:
1. All 5 users open Daily Drill at same time
2. All click to get today's question
3. Verify each gets a question
4. Check database for race conditions

**Expected**: No duplicate drills, no errors

**Pass Criteria**: ✅ No race conditions

#### Test 3: Streak Maintenance

**Steps**:
1. Complete drill on Day 1
2. Check streak = 1
3. Complete drill on Day 2
4. Check streak = 2
5. Skip Day 3
6. Complete drill on Day 4
7. Check streak = 1 (reset)

**Expected**: Streak logic works correctly

**Pass Criteria**: ✅ Streak resets after gap

---

## 🚀 Deployment Procedure

### Staging Deployment

```bash
# 1. Deploy database migration
supabase db push --project-ref YOUR_STAGING_PROJECT

# 2. Deploy Edge Function
supabase functions deploy daily-drill-engine --project-ref YOUR_STAGING_PROJECT

# 3. Build Flutter app
flutter build apk --release

# 4. Install on test device
adb install build/app/outputs/flutter-apk/app-release.apk
```

### Production Deployment

```bash
# 1. Backup database
supabase db dump --file backup_$(date +%Y%m%d).sql

# 2. Deploy database migration
supabase db push --project-ref YOUR_PROD_PROJECT

# 3. Verify migration
supabase db execute "SELECT * FROM pg_proc WHERE proname = 'get_or_assign_daily_question';" --project-ref YOUR_PROD_PROJECT

# 4. Deploy Edge Function
supabase functions deploy daily-drill-engine --project-ref YOUR_PROD_PROJECT

# 5. Test Edge Function
supabase functions invoke daily-drill-engine \
  --body '{"action":"get_today_question","payload":{}}' \
  --project-ref YOUR_PROD_PROJECT

# 6. Build and deploy Flutter app
flutter build apk --release
# Upload to Play Store / App Store
```

---

## 📊 Monitoring

### Key Metrics to Track

1. **Daily Active Users**: Users completing drills
2. **Completion Rate**: % of users completing daily drill
3. **Streak Retention**: % of users with 7+ day streaks
4. **Error Rate**: Edge Function errors
5. **API Latency**: p95 response time
6. **App Crash Rate**: Frontend crashes

### Supabase Dashboard

Monitor:
- Edge Function invocations
- Database query performance
- Error logs
- User activity

### Alerts to Set Up

1. **Error Rate > 1%**: Immediate alert
2. **API Latency > 1s**: Warning alert
3. **No drills assigned today**: Critical alert
4. **Database connection issues**: Critical alert

---

## 🐛 Troubleshooting

### Issue: Function not found

**Symptom**: Edge Function returns "function does not exist"

**Solution**:
```sql
-- Verify function exists
SELECT * FROM pg_proc WHERE proname = 'get_or_assign_daily_question';

-- If not, re-run migration
supabase db push
```

### Issue: Race condition detected

**Symptom**: Two drills assigned for same day

**Solution**:
```sql
-- Check for duplicates
SELECT user_id, assigned_date, COUNT(*)
FROM user_daily_drills
GROUP BY user_id, assigned_date
HAVING COUNT(*) > 1;

-- Fix: Delete duplicates, keep oldest
DELETE FROM user_daily_drills
WHERE id NOT IN (
    SELECT MIN(id)
    FROM user_daily_drills
    GROUP BY user_id, assigned_date
);
```

### Issue: Streak not updating

**Symptom**: Streak stays at 0 after completion

**Solution**:
```sql
-- Manually trigger streak update
SELECT public.update_user_streak('USER_ID'::uuid, CURRENT_DATE);

-- Check trigger is enabled
SELECT * FROM pg_trigger WHERE tgname = 'on_drill_status_changed';
```

### Issue: Confetti not showing

**Symptom**: No animation after completion

**Solution**:
```bash
# Verify confetti package installed
flutter pub get

# Check import in completion dialog
grep -r "confetti" lib/features/daily_drill/widgets/
```

---

## ✅ Sign-Off Checklist

Before marking as production-ready:

- [ ] All database tests passing
- [ ] All Edge Function tests passing
- [ ] All frontend tests passing
- [ ] All integration tests passing
- [ ] Staging deployment successful
- [ ] 48-hour monitoring complete
- [ ] No critical errors
- [ ] Performance metrics met
- [ ] Security review passed
- [ ] Documentation complete

---

**Testing Guide Version**: 1.0  
**Last Updated**: February 10, 2026  
**Next Review**: After first production deployment
