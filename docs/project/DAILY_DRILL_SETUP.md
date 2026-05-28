# Daily Drill Feature - Setup Guide

## 🚀 Quick Start Guide

Follow these steps to deploy and test the Daily Drill feature.

---

## Step 1: Install Dependencies

```bash
flutter pub get
```

This will install the `confetti` package and other dependencies.

---

## Step 2: Generate Freezed Models

The Daily Drill models use Freezed for immutability. Generate the required files:

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

This will generate:
- `daily_drill_models.freezed.dart`
- `daily_drill_models.g.dart`

---

## Step 3: Apply Database Migration

Navigate to your Supabase project and apply the migration:

```bash
# Option 1: Using Supabase CLI
supabase db push

# Option 2: Manual SQL execution
# Copy the contents of supabase/migrations/20260210_daily_drill_system.sql
# and execute it in the Supabase SQL Editor
```

This will create:
- `daily_drill_questions` table
- `user_daily_drills` table
- `user_drill_progress` table
- RLS policies
- Database triggers
- Helper functions
- 11 seed questions

---

## Step 4: Deploy Edge Function

Deploy the Daily Drill engine to Supabase:

```bash
supabase functions deploy daily-drill-engine
```

Or use your existing deployment command from `supabase/connect_with_supabase_from_project.md`.

---

## Step 5: Verify Database Setup

Run these SQL queries in Supabase SQL Editor to verify:

```sql
-- Check if tables exist
SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'public' 
AND table_name LIKE 'daily_drill%' OR table_name LIKE 'user_drill%';

-- Check seed data
SELECT COUNT(*) as total_questions FROM daily_drill_questions;

-- Should return 11 questions
```

---

## Step 6: Test the Feature

### 6.1 Run the App

```bash
flutter run
```

### 6.2 Navigate to Daily Drill

1. Login to the app
2. On the Home Screen, tap the "Daily Drill" card
3. You should see today's question

### 6.3 Complete a Drill

1. Read the question
2. Tap "Reveal Ideal Answer"
3. Select a confidence level (1-5 stars)
4. Optionally add notes
5. Tap "Mark as Complete"
6. Enjoy the confetti celebration! 🎉

### 6.4 Check Streak

1. Complete a drill today
2. Come back tomorrow
3. Complete another drill
4. Your streak should increment to 2 days

### 6.5 View History

1. From Daily Drill screen, tap the history icon (top right)
2. Filter by status (All, Completed, Viewed, Pending)
3. Tap on any drill to view details

---

## 📊 Database Verification Queries

### Check User Progress

```sql
SELECT * FROM user_drill_progress 
WHERE user_id = 'YOUR_USER_ID';
```

### Check Today's Assignment

```sql
SELECT 
  udd.*,
  ddq.question_text,
  ddq.category,
  ddq.difficulty
FROM user_daily_drills udd
JOIN daily_drill_questions ddq ON udd.question_id = ddq.id
WHERE udd.user_id = 'YOUR_USER_ID'
AND udd.assigned_date = CURRENT_DATE;
```

### Check Streak Calculation

```sql
SELECT 
  current_streak,
  longest_streak,
  last_completed_date,
  total_drills_completed,
  readiness_score
FROM user_drill_progress
WHERE user_id = 'YOUR_USER_ID';
```

---

## 🧪 Testing Checklist

### Database Tests
- [ ] Tables created successfully
- [ ] RLS policies working (users can only see their own data)
- [ ] Triggers firing on drill completion
- [ ] Streak calculation working
- [ ] Readiness score updating
- [ ] Seed data loaded (11 questions)

### Backend Tests
- [ ] Edge Function deployed
- [ ] `get_today_question` returns a question
- [ ] Question assignment works (no duplicates for same user)
- [ ] `submit_drill_response` updates database
- [ ] `get_drill_stats` returns correct data
- [ ] `get_drill_history` returns paginated results

### Frontend Tests
- [ ] Daily Drill card shows on Home Screen
- [ ] Streak displays correctly
- [ ] Question preview shows
- [ ] Navigation to Daily Drill screen works
- [ ] Question card renders properly
- [ ] Reveal answer works
- [ ] Confidence selector works
- [ ] Notes input works
- [ ] Completion flow works
- [ ] Confetti animation plays
- [ ] History screen loads
- [ ] Filters work
- [ ] Pagination works
- [ ] Detail modal works

---

## 🐛 Troubleshooting

### Issue: Freezed models not generating

**Solution**:
```bash
flutter clean
flutter pub get
flutter pub run build_runner build --delete-conflicting-outputs
```

### Issue: Edge Function not deploying

**Solution**:
1. Check Supabase CLI is installed: `supabase --version`
2. Login: `supabase login`
3. Link project: `supabase link --project-ref YOUR_PROJECT_REF`
4. Deploy: `supabase functions deploy daily-drill-engine`

### Issue: No questions showing

**Solution**:
1. Verify seed data: `SELECT COUNT(*) FROM daily_drill_questions;`
2. If count is 0, re-run the migration
3. Check RLS policies are not blocking access

### Issue: Streak not updating

**Solution**:
1. Check database triggers are created
2. Verify `update_drill_progress_on_completion` function exists
3. Check `user_drill_progress` table has data

### Issue: Confetti not showing

**Solution**:
1. Verify `confetti` package is in pubspec.yaml
2. Run `flutter pub get`
3. Restart the app

---

## 📝 Adding More Questions

### Option 1: Manual SQL Insert

```sql
INSERT INTO daily_drill_questions (
  question_text,
  ideal_answer,
  evaluation_points,
  skill_tags,
  target_role,
  category,
  difficulty,
  estimated_time_minutes
) VALUES (
  'Your question here',
  'Ideal answer here',
  ARRAY['Point 1', 'Point 2', 'Point 3'],
  ARRAY['skill1', 'skill2'],
  'junior',
  'behavioral',
  'medium',
  5
);
```

### Option 2: Batch Import (Future Enhancement)

Create a CSV file and use Supabase's import feature or create a script to bulk insert questions.

---

## 🔄 Daily Maintenance

### Monitoring

Check these metrics daily:
- Number of active users
- Average streak length
- Completion rate
- Popular question categories
- Weak areas across users

### Database Cleanup

Run this query monthly to archive old drills:

```sql
-- Archive drills older than 90 days (optional)
UPDATE user_daily_drills
SET metadata = jsonb_set(
  COALESCE(metadata, '{}'::jsonb),
  '{archived}',
  'true'::jsonb
)
WHERE assigned_date < CURRENT_DATE - INTERVAL '90 days';
```

---

## 🎯 Success Metrics

After 1 week of deployment, check:
- [ ] Daily Active Users (DAU) using Daily Drill
- [ ] Average streak length
- [ ] Completion rate (completed / assigned)
- [ ] User retention (users returning next day)
- [ ] Readiness score improvement

---

## 📞 Support

If you encounter issues:
1. Check the logs in Supabase Dashboard → Edge Functions
2. Check Flutter console for errors
3. Verify database state with SQL queries above
4. Review `DAILY_DRILL_IMPLEMENTATION.md` for architecture details

---

**Last Updated**: February 10, 2026  
**Version**: 1.0.0  
**Status**: Production Ready ✅
