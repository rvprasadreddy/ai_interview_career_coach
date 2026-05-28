# LinkedIn Profile Auto-Sync on Every Login

## ✅ Feature Overview

The LinkedIn authentication system now **automatically syncs profile data on every login**, ensuring users always have the latest information from their LinkedIn profile.

---

## 🔄 What Gets Synced on Every Login

### Always Updated Fields

When a user logs in with LinkedIn, the following fields are **automatically updated** with the latest data from LinkedIn:

1. **Profile Image** (`profile_image_url`)
   - Latest profile picture from LinkedIn
   - Ensures avatar is always current

2. **Headline** (`headline`)
   - Current professional headline
   - Reflects latest LinkedIn bio

3. **Current Position** (`current_position`)
   - Latest job title and company
   - Format: "Title at Company"

4. **Location** (`location` and `city`)
   - Current location from LinkedIn
   - City extracted automatically

5. **Name** (`name`)
   - Full name from LinkedIn
   - Updated if changed on LinkedIn

6. **Skills** (`skills`)
   - **Merge strategy**: Adds new skills from LinkedIn
   - **Preserves**: All existing skills (never removes)
   - **Result**: Growing list of unique skills

### Preserved Fields

These fields are **never overwritten** by LinkedIn sync:

- ✅ `onboarding_completed` - App-specific progress
- ✅ `interview_count` - User activity metrics
- ✅ `practice_count` - User activity metrics
- ✅ `total_score` - Performance data
- ✅ `average_score` - Performance data
- ✅ `strengths` - AI-generated insights
- ✅ `weaknesses` - AI-generated insights
- ✅ `improvement_areas` - AI-generated insights
- ✅ `created_at` - Original signup date
- ✅ Custom user preferences

---

## 🔍 How It Works

### Login Flow with Auto-Sync

```
1. User clicks "Continue with LinkedIn"
   ↓
2. LinkedIn OAuth authorization
   ↓
3. User approves app
   ↓
4. Redirect to app via deep link
   ↓
5. LinkedInCallbackScreen processes callback
   ↓
6. Edge Function (linkedin-profile-sync) called
   ↓
7. Check if user exists in database
   ↓
8. EXISTING USER PATH:
   - Fetch latest LinkedIn profile data
   - Update profile_image_url (if available)
   - Update headline (if available)
   - Update current_position (if available)
   - Update location and city (if available)
   - Update name (if changed)
   - Merge skills (add new, keep existing)
   - Update last_linkedin_sync_at timestamp
   - Log all updated fields
   ↓
9. User routed to Home screen
   ↓
10. UI automatically reflects updated profile
```

---

## 📊 Database Updates

### Update Query Structure

```sql
UPDATE users
SET
  profile_image_url = COALESCE(linkedin_data.image, existing.image),
  headline = COALESCE(linkedin_data.headline, existing.headline),
  current_position = COALESCE(linkedin_data.position, existing.position),
  location = COALESCE(linkedin_data.location, existing.location),
  city = COALESCE(linkedin_data.city, existing.city),
  name = COALESCE(linkedin_data.name, existing.name),
  skills = ARRAY_UNIQUE(existing.skills || linkedin_data.skills),
  last_linkedin_sync_at = NOW(),
  updated_at = NOW(),
  profile_meta = JSONB_SET(
    existing.profile_meta,
    '{last_linkedin_sync}',
    to_jsonb(NOW())
  )
WHERE user_id = $1
```

### Metadata Tracking

The `profile_meta` JSONB field tracks sync history:

```json
{
  "last_linkedin_sync": "2026-02-01T11:57:25Z",
  "sync_source": "login",
  "linkedin_last_update": {
    "timestamp": "2026-02-01T11:57:25Z",
    "fields_updated": [
      "profile_image_url",
      "headline",
      "current_position",
      "skills",
      "location",
      "city"
    ]
  }
}
```

---

## 🎯 Sync Strategy

### New User (Signup)
- ✅ Create profile with all LinkedIn data
- ✅ Set `auth_provider` to 'linkedin'
- ✅ Set `onboarding_completed` to false
- ✅ Route to onboarding screen

### Existing User (Login)
- ✅ **Always update** LinkedIn-sourced fields
- ✅ **Preserve** app-specific data
- ✅ **Merge** skills (additive only)
- ✅ **Track** sync timestamp
- ✅ **Log** what was updated
- ✅ Route to home screen

---

## 📝 Logging & Debugging

### Edge Function Logs

On every login, the Edge Function logs:

```typescript
console.log('Existing LinkedIn user detected - syncing latest profile data')

console.log('Updating user profile with LinkedIn data:', {
  user_id: 'uuid-here',
  fields: ['profile_image_url', 'headline', 'current_position', ...],
  has_new_image: true,
  has_new_headline: true,
  has_new_position: true,
  skills_count: 15
})

console.log('User profile synced successfully with latest LinkedIn data')
```

### Flutter App Logs

```dart
debugPrint('AuthService: Syncing LinkedIn profile for user ${user.id}')
debugPrint('AuthService: LinkedIn sync completed. Is new user: false')
debugPrint('AuthNotifier: LinkedIn sync complete. New user: false')
```

---

## 🧪 Testing Auto-Sync

### Test Scenario 1: Profile Image Update

1. Update profile picture on LinkedIn
2. Log out of app
3. Log in with LinkedIn
4. ✅ Verify new profile picture appears immediately

### Test Scenario 2: Headline Update

1. Change headline on LinkedIn
2. Log out of app
3. Log in with LinkedIn
4. ✅ Verify new headline appears in profile

### Test Scenario 3: Skills Addition

1. Add new skills on LinkedIn
2. Log in with LinkedIn
3. ✅ Verify new skills added to existing skills
4. ✅ Verify old skills still present

### Test Scenario 4: Position Change

1. Update current job on LinkedIn
2. Log in with LinkedIn
3. ✅ Verify new position appears

---

## 🔐 Privacy & Data Control

### What Users Should Know

1. **Automatic Updates**: Profile syncs on every LinkedIn login
2. **Additive Skills**: New skills are added, never removed
3. **Preserved Progress**: App data (scores, progress) never changes
4. **Timestamp Tracking**: Last sync time is recorded
5. **Manual Control**: Users can edit profile in-app after sync

### Data Sources

| Field | Source | Update Frequency |
|-------|--------|------------------|
| Profile Image | LinkedIn | Every login |
| Headline | LinkedIn | Every login |
| Current Position | LinkedIn | Every login |
| Skills | LinkedIn + App | Merged on login |
| Location | LinkedIn | Every login |
| Interview Scores | App Only | Never synced |
| Preferences | App Only | Never synced |

---

## 🚀 Deployment

### Edge Function Deployment

```bash
# Deploy updated function
supabase functions deploy linkedin-profile-sync

# Verify deployment
supabase functions list

# Test function
curl -X POST \
  https://[PROJECT-REF].supabase.co/functions/v1/linkedin-profile-sync \
  -H "Authorization: Bearer [ANON-KEY]" \
  -H "Content-Type: application/json" \
  -d '{
    "linkedin_profile": {...},
    "user_id": "test-uuid"
  }'
```

### Database Migration

The `last_linkedin_sync_at` column is already added via migration:
```sql
-- From: 20260201000000_linkedin_auth_enhancement.sql
ALTER TABLE public.users 
ADD COLUMN IF NOT EXISTS last_linkedin_sync_at TIMESTAMPTZ;
```

---

## 📊 Monitoring

### Key Metrics to Track

1. **Sync Success Rate**
   - % of successful profile syncs
   - Target: >99%

2. **Sync Duration**
   - Average time to sync profile
   - Target: <2 seconds

3. **Fields Updated**
   - Which fields change most often
   - Helps understand user behavior

4. **Error Rate**
   - Failed syncs
   - Target: <1%

### Supabase Dashboard Queries

```sql
-- Check recent syncs
SELECT 
  user_id,
  name,
  last_linkedin_sync_at,
  profile_meta->'last_linkedin_sync' as last_sync,
  profile_meta->'linkedin_last_update'->'fields_updated' as updated_fields
FROM users
WHERE auth_provider = 'linkedin'
  AND last_linkedin_sync_at > NOW() - INTERVAL '24 hours'
ORDER BY last_linkedin_sync_at DESC;

-- Count syncs per day
SELECT 
  DATE(last_linkedin_sync_at) as sync_date,
  COUNT(*) as sync_count
FROM users
WHERE auth_provider = 'linkedin'
  AND last_linkedin_sync_at IS NOT NULL
GROUP BY DATE(last_linkedin_sync_at)
ORDER BY sync_date DESC;
```

---

## ✅ Benefits

1. **Always Current**: Profile data stays up-to-date automatically
2. **No Manual Work**: Users don't need to update profile in-app
3. **Professional Image**: Latest LinkedIn photo always shown
4. **Accurate Info**: Current position and headline reflect reality
5. **Growing Skills**: Skills list expands over time
6. **Audit Trail**: Sync history tracked in metadata

---

## 🎯 Summary

**What Changed**:
- ✅ Enhanced Edge Function to always sync on login
- ✅ Added detailed logging for debugging
- ✅ Added metadata tracking for sync history
- ✅ Added name field to updates
- ✅ Improved skills merging with logging

**What Happens Now**:
- ✅ Every LinkedIn login syncs latest profile data
- ✅ Profile image, headline, position always current
- ✅ Skills grow over time (never removed)
- ✅ App-specific data always preserved
- ✅ Sync timestamp tracked for audit

**User Experience**:
- ✅ Seamless profile updates
- ✅ No manual data entry needed
- ✅ Always shows latest professional info
- ✅ Progress and scores never affected

The LinkedIn profile auto-sync is now **production-ready**! 🚀
