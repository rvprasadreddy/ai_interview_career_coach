# Daily Drill - Quick Reference

## 🚀 Deployment Commands

```bash
# 1. Install dependencies
flutter pub get

# 2. Generate Freezed models
flutter pub run build_runner build --delete-conflicting-outputs

# 3. Apply database migration
supabase db push

# 4. Deploy Edge Function
supabase functions deploy daily-drill-engine

# 5. Run the app
flutter run
```

## 📍 Routes

- **Main Screen**: `/daily-drill`
- **History Screen**: `/daily-drill/history`

## 🔑 Key Components

### Models
- `DailyDrillQuestion` - Question data
- `UserDailyDrill` - User assignment
- `UserDrillProgress` - Progress tracking
- `DrillStatus` - pending, viewed, completed, skipped

### Providers
- `todayDrillProvider` - Today's drill
- `drillStatsProvider` - User statistics
- `streakInfoProvider` - Streak information
- `dailyDrillNotifierProvider` - Main state
- `drillHistoryNotifierProvider` - History with pagination

### Screens
- `DailyDrillScreen` - Main drill interface
- `DailyDrillHistoryScreen` - Drill history

### Widgets
- `DrillQuestionCard` - Question display
- `DrillAnswerCard` - Answer display
- `DrillConfidenceSelector` - 1-5 star rating
- `DrillStatsHeader` - Streak header
- `DrillCompletionDialog` - Celebration dialog

## 🗄️ Database Tables

### `daily_drill_questions`
Global question repository (shared across users)

### `user_daily_drills`
User assignment tracking (one per user per day)

### `user_drill_progress`
Progress analytics and streaks

## 🔧 Edge Function Actions

### `get_today_question`
Fetch or assign today's question

### `submit_drill_response`
Update drill status (viewed, completed, skipped)

### `get_drill_stats`
Fetch user progress and statistics

### `get_drill_history`
Get paginated drill history

## 📊 Readiness Score Formula

```
Readiness Score = (Streak × 30%) + (Completion Rate × 40%) + (Avg Confidence × 30%)
```

## 🎯 Question Selection Logic

1. **70% probability** - Select from weak categories
2. **30% probability** - Random selection
3. **Avoid repeats** - Never assign same question twice to same user
4. **Difficulty matching** - Adapt based on readiness score
5. **Role matching** - Match user's experience level

## 🔐 Security

- ✅ Row-Level Security (RLS) on all tables
- ✅ Users can only access their own data
- ✅ Server-side question assignment
- ✅ Input validation in Edge Function

## 📈 Metrics to Monitor

- Daily Active Users (DAU)
- Completion Rate
- Average Streak Length
- User Retention (day-over-day)
- Readiness Score trends
- Category performance

## 🐛 Common Issues

### Freezed models not generating
```bash
flutter clean
flutter pub get
flutter pub run build_runner build --delete-conflicting-outputs
```

### No questions showing
```sql
-- Check seed data
SELECT COUNT(*) FROM daily_drill_questions;
```

### Streak not updating
```sql
-- Verify trigger exists
SELECT * FROM pg_trigger WHERE tgname = 'update_drill_progress_trigger';
```

## 📝 Adding Questions

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

## 🎨 UI Theme

- **Primary Color**: Orange (for streaks)
- **Success Color**: Green (for completion)
- **Card Style**: Glassmorphism
- **Animations**: Confetti, fade-in, slide-up
- **Icons**: Fire (streak), Trophy (best), Star (confidence)

## 📚 Documentation

- `DAILY_DRILL_ARCHITECTURE.md` - System design
- `DAILY_DRILL_IMPLEMENTATION.md` - Complete feature list
- `DAILY_DRILL_SETUP.md` - Detailed setup guide
- `DAILY_DRILL_COMPLETE.md` - Completion summary

---

**Status**: ✅ Production Ready  
**Version**: 1.0.0  
**Last Updated**: February 10, 2026
