# LinkedIn Full Profile Sync - Deployment Guide

## 🚀 Quick Deployment Steps

### Step 1: Deploy Enhanced Edge Function

```bash
# Navigate to project root
cd c:\flutter_apps\antigravity\ai_interview_coach

# Deploy the updated Edge Function
supabase functions deploy linkedin-profile-sync

# Verify deployment
supabase functions list
```

**Expected Output**:
```
linkedin-profile-sync | deployed | 2026-02-01 12:30:00
```

---

### Step 2: Rebuild Flutter App

```bash
# Hot restart (if app is running)
# Press 'r' in terminal

# OR full rebuild
flutter clean
flutter pub get
flutter run
```

---

### Step 3: Test the Integration

1. **Log out** of the app (if logged in)
2. **Click "Continue with LinkedIn"**
3. **Authorize the app** on LinkedIn
4. **Check Flutter console** for logs:

```
=== LinkedIn Metadata Received ===
...
LinkedIn provider access token available  ← Should see this!
...
AuthService: LinkedIn sync completed
```

5. **Check Supabase Edge Function logs**:
   - Go to Supabase Dashboard
   - Navigate to **Edge Functions** → **linkedin-profile-sync** → **Logs**
   - Look for:

```
Access token available, fetching full profile from LinkedIn API v2
Fetching full profile from LinkedIn API v2
LinkedIn API profile data received
Successfully enriched profile with LinkedIn API data
Updating headline
Updating current position
Skills merged: ...
```

---

### Step 4: Verify Database

```sql
-- Check your profile data
SELECT 
  name,
  profile_image_url,
  headline,
  current_position,
  skills,
  location,
  last_linkedin_sync_at
FROM users
WHERE user_id = 'your-user-id';
```

**Expected Results**:
- ✅ `headline` - Should be populated (e.g., "Senior Software Engineer")
- ✅ `current_position` - Should be populated (e.g., "Tech Lead at Google")
- ✅ `skills` - Should be array with skills
- ✅ `last_linkedin_sync_at` - Recent timestamp

---

## 🔍 Verification Checklist

### Flutter App
- [ ] App rebuilt successfully
- [ ] No compilation errors
- [ ] LinkedIn login button works
- [ ] OAuth flow completes
- [ ] Console shows "LinkedIn provider access token available"
- [ ] Console shows "LinkedIn sync completed"

### Edge Function
- [ ] Function deployed successfully
- [ ] No deployment errors
- [ ] Logs show "Fetching full profile from LinkedIn API v2"
- [ ] Logs show "Successfully enriched profile"
- [ ] Logs show fields being updated

### Database
- [ ] `headline` populated
- [ ] `current_position` populated
- [ ] `skills` array has items
- [ ] `profile_image_url` updated
- [ ] `last_linkedin_sync_at` updated

### UI
- [ ] Profile screen shows headline
- [ ] Profile screen shows current position
- [ ] Profile screen shows skills
- [ ] Profile picture displays correctly

---

## ⚠️ Troubleshooting

### Issue 1: "No provider access token available"

**Symptoms**:
```
No provider access token available, will use OIDC data only
```

**Possible Causes**:
1. Supabase not configured to store provider tokens
2. OAuth flow not completing properly
3. Session not persisting

**Solutions**:
1. Check Supabase Auth settings
2. Verify LinkedIn OAuth configuration
3. Re-authenticate with LinkedIn
4. Check if `session.providerToken` exists

**Impact**: Falls back to OIDC data (name and picture only)

---

### Issue 2: "LinkedIn API profile fetch failed: 401"

**Symptoms**:
```
LinkedIn API profile fetch failed: 401
Failed to fetch from LinkedIn API, using OIDC data
```

**Possible Causes**:
1. Access token expired
2. Invalid access token
3. LinkedIn app credentials incorrect
4. User revoked access

**Solutions**:
1. Re-authenticate with LinkedIn
2. Verify LinkedIn app credentials
3. Check LinkedIn app is active
4. Verify scopes are approved

**Impact**: Falls back to OIDC data

---

### Issue 3: "Failed to fetch positions/skills"

**Symptoms**:
```
Failed to fetch positions: ...
Failed to fetch skills: ...
```

**Possible Causes**:
1. LinkedIn API endpoint changed
2. Missing permissions/scopes
3. User privacy settings
4. Rate limiting

**Solutions**:
1. Check LinkedIn API documentation
2. Verify app has required scopes
3. Check user granted all permissions
4. Wait and retry (if rate limited)

**Impact**: Other fields still update, only affected field skipped

---

## 📊 Expected Behavior

### Scenario 1: Full API Success

**User Experience**:
1. Logs in with LinkedIn
2. Sees loading screen briefly
3. Profile fully populated:
   - ✅ Profile picture
   - ✅ Name
   - ✅ Headline
   - ✅ Current position
   - ✅ Skills list

**Logs**:
```
LinkedIn provider access token available
Fetching full profile from LinkedIn API v2
Successfully enriched profile with LinkedIn API data
Updating profile image
Updating name
Updating headline
Updating current position
Skills merged: 5 existing + 8 new = 13 total
```

---

### Scenario 2: API Fallback (No Token)

**User Experience**:
1. Logs in with LinkedIn
2. Sees loading screen briefly
3. Profile partially populated:
   - ✅ Profile picture (from OIDC)
   - ✅ Name (from OIDC)
   - ⚠️ Headline (preserved from previous or empty)
   - ⚠️ Position (preserved from previous or empty)
   - ⚠️ Skills (preserved from previous or empty)

**Logs**:
```
No provider access token available, will use OIDC data only
No headline from LinkedIn, preserving existing
No position from LinkedIn, preserving existing
No skills from LinkedIn, preserving existing
```

---

### Scenario 3: Partial API Success

**User Experience**:
1. Logs in with LinkedIn
2. Sees loading screen briefly
3. Profile mostly populated:
   - ✅ Profile picture (from API)
   - ✅ Name (from API)
   - ✅ Headline (from API)
   - ⚠️ Position (failed, preserved)
   - ⚠️ Skills (failed, preserved)

**Logs**:
```
LinkedIn provider access token available
Fetching full profile from LinkedIn API v2
LinkedIn API profile data received
Failed to fetch positions: ...
Failed to fetch skills: ...
Updating profile image
Updating name
Updating headline
No position from LinkedIn, preserving existing
No skills from LinkedIn, preserving existing
```

---

## 🎯 Success Criteria

### Minimum (OIDC Fallback)
- ✅ User can log in
- ✅ Name updates
- ✅ Profile picture updates
- ✅ No errors or crashes

### Optimal (Full API)
- ✅ User can log in
- ✅ Name updates
- ✅ Profile picture updates (high-res)
- ✅ Headline populates
- ✅ Current position populates
- ✅ Skills list populates
- ✅ No errors or crashes

---

## 📝 Post-Deployment Tasks

### 1. Monitor Logs (First 24 Hours)

```bash
# Watch Edge Function logs
supabase functions logs linkedin-profile-sync --tail

# Look for:
# - Success rate of API calls
# - Common error patterns
# - Performance metrics
```

### 2. Check User Profiles

```sql
-- Profile completeness report
SELECT 
  COUNT(*) as total_linkedin_users,
  COUNT(*) FILTER (WHERE headline IS NOT NULL AND headline != '') as has_headline,
  COUNT(*) FILTER (WHERE current_position IS NOT NULL AND current_position != '') as has_position,
  COUNT(*) FILTER (WHERE array_length(skills, 1) > 0) as has_skills
FROM users
WHERE auth_provider = 'linkedin'
  AND last_linkedin_sync_at > NOW() - INTERVAL '24 hours';
```

### 3. Gather Feedback

- Ask test users about profile accuracy
- Check if all expected fields populate
- Verify profile pictures are high quality
- Confirm skills list is comprehensive

---

## 🔄 Rollback Plan (If Needed)

If issues arise, you can rollback:

### Option 1: Revert Edge Function

```bash
# Deploy previous version (without API integration)
# You'll need to restore from git history or backup
```

### Option 2: Disable API Calls

In `auth_service.dart`, change:
```dart
'access_token': providerToken ?? 'managed_by_supabase',
```

To:
```dart
'access_token': 'managed_by_supabase',  // Force OIDC fallback
```

This disables API calls without changing Edge Function.

---

## ✅ Deployment Complete!

Once all checks pass:

1. ✅ Edge Function deployed
2. ✅ Flutter app rebuilt
3. ✅ LinkedIn login works
4. ✅ Profile data syncs
5. ✅ Database updates correctly
6. ✅ UI displays all fields
7. ✅ Logs show successful API calls

**Your LinkedIn integration now fetches complete profile data!** 🎉

---

## 📚 Related Documentation

- **Implementation Details**: `LINKEDIN_API_V2_INTEGRATION.md`
- **Debugging Guide**: `LINKEDIN_SYNC_DEBUG.md`
- **OAuth Setup**: `LINKEDIN_OAUTH_TROUBLESHOOTING.md`
- **Auto-Sync Feature**: `LINKEDIN_AUTO_SYNC.md`
