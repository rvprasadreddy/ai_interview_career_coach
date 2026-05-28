# LinkedIn Profile Sync - Testing & Validation Guide

## 🧪 Complete Testing Checklist

### Pre-Deployment Steps

1. **Deploy Enhanced Edge Function**
   ```bash
   supabase functions deploy linkedin-profile-sync
   ```

2. **Rebuild Flutter App**
   ```bash
   flutter run
   ```

---

## Test 1: Existing User Login

### Objective
Verify that profile_image_url, headline, and skills update for existing users.

### Steps

1. **Prepare Test User**
   - Ensure you have an existing user in the database
   - Note current values:
   ```sql
   SELECT 
     user_id,
     name,
     profile_image_url,
     headline,
     current_position,
     skills,
     last_linkedin_sync_at
   FROM users
   WHERE email = 'your-test-email@example.com';
   ```

2. **Update LinkedIn Profile** (Optional)
   - Change your headline on LinkedIn
   - Add new skills on LinkedIn
   - Update profile picture on LinkedIn

3. **Log Out and Log In**
   - Log out of the app
   - Click "Continue with LinkedIn"
   - Complete OAuth flow

4. **Check Flutter Console Logs**
   Look for these specific logs:

   ```
   === LinkedIn Metadata Received ===
   All metadata keys: [...]
   LinkedIn provider access token available  ← MUST SEE THIS

   Checking normalized data values: {
     profile_image_url: https://...  ← Should have URL
     headline: Your Headline  ← Should have text
     skills_count: 5  ← Should have number > 0
   }

   ✅ Updating profile image: https://...
   ✅ Updating name: Your Name
   ✅ Updating headline: Your Headline
   ✅ Skills merged: 3 existing + 5 new = 8 total
   ```

5. **Check Supabase Edge Function Logs**
   - Go to Supabase Dashboard
   - Edge Functions → linkedin-profile-sync → Logs
   
   Look for:
   ```
   Access token available, fetching full profile from LinkedIn API v2
   Fetching full profile from LinkedIn API v2
   LinkedIn API profile data received
   Successfully enriched profile with LinkedIn API data
   
   Checking normalized data values: {
     profile_image_url: https://...
     headline: Your Headline
     skills_count: 5
   }
   
   ✅ Updating profile image: https://...
   ✅ Updating headline: Your Headline
   ✅ Skills merged: ...
   ```

6. **Verify Database Update**
   ```sql
   SELECT 
     user_id,
     name,
     profile_image_url,
     headline,
     current_position,
     skills,
     last_linkedin_sync_at,
     updated_at
   FROM users
   WHERE email = 'your-test-email@example.com';
   ```

   **Expected**:
   - ✅ `profile_image_url` - Updated with new URL
   - ✅ `headline` - Updated with LinkedIn headline
   - ✅ `skills` - Array with merged skills
   - ✅ `last_linkedin_sync_at` - Recent timestamp
   - ✅ `updated_at` - Recent timestamp

7. **Verify UI Update**
   - Navigate to profile screen
   - ✅ Profile picture should show new image
   - ✅ Headline should display
   - ✅ Skills should be listed
   - ✅ Current position should show

---

## Test 2: New User Signup

### Objective
Verify that new users get complete profile data from LinkedIn.

### Steps

1. **Create Test LinkedIn Account** (or use a different one)

2. **Sign Up with LinkedIn**
   - Click "Sign Up with LinkedIn"
   - Complete OAuth flow
   - Should redirect to onboarding

3. **Check Flutter Console Logs**
   ```
   LinkedIn provider access token available
   
   LinkedIn profile data received: {
     has_headline: true
     has_positions: true
     has_skills: true
   }
   
   Normalized data: {
     profile_image_url: https://...
     headline: Your Headline
     skills: [...]
   }
   
   New LinkedIn user detected - creating profile
   ```

4. **Check Database**
   ```sql
   SELECT 
     user_id,
     name,
     email,
     profile_image_url,
     headline,
     current_position,
     skills,
     auth_provider,
     onboarding_completed,
     created_at
   FROM users
   WHERE email = 'new-user@example.com';
   ```

   **Expected**:
   - ✅ `name` - Populated
   - ✅ `email` - Populated
   - ✅ `profile_image_url` - Populated
   - ✅ `headline` - Populated
   - ✅ `current_position` - Populated
   - ✅ `skills` - Array with skills
   - ✅ `auth_provider` - 'linkedin'
   - ✅ `onboarding_completed` - false

5. **Verify Onboarding Screen**
   - Should show profile setup screen
   - ✅ Name field pre-filled
   - ✅ Email field pre-filled
   - ✅ Profile picture displayed
   - ✅ Headline pre-filled (if field exists)

---

## Test 3: UI Refresh After Sync

### Objective
Verify that UI updates immediately after profile sync.

### Steps

1. **Log in with LinkedIn**

2. **Immediately Check Profile Screen**
   - Navigate to profile/settings
   - ✅ Profile picture should be updated
   - ✅ Name should be updated
   - ✅ Headline should be displayed
   - ✅ Skills should be listed

3. **Check State Management**
   - Profile data should be in app state
   - No need to refresh manually
   - Changes should be immediate

---

## 🔍 Troubleshooting

### Issue 1: "No provider access token available"

**Symptoms**:
```
No provider access token available, will use OIDC data only
❌ No headline from LinkedIn, preserving existing
❌ No skills from LinkedIn, preserving existing
```

**Root Cause**: Supabase not providing provider token

**Debug Steps**:
1. Check `session.providerToken` in Flutter:
   ```dart
   final session = _client.auth.currentSession;
   debugPrint('Provider token: ${session?.providerToken}');
   ```

2. Check Supabase Auth settings:
   - Dashboard → Authentication → Providers → LinkedIn (OIDC)
   - Verify enabled and configured

3. Try re-authenticating

**Workaround**: Falls back to OIDC data (name and picture only)

---

### Issue 2: "LinkedIn API profile fetch failed"

**Symptoms**:
```
Access token available, fetching full profile from LinkedIn API v2
LinkedIn API profile fetch failed: 401
Failed to fetch from LinkedIn API, using OIDC data
```

**Root Cause**: Invalid or expired access token, or missing permissions

**Debug Steps**:
1. Check LinkedIn app scopes
2. Verify user granted all permissions
3. Check if access token is valid
4. Try re-authenticating

**Workaround**: Falls back to OIDC data

---

### Issue 3: Empty Values in Normalized Data

**Symptoms**:
```
Checking normalized data values: {
  profile_image_url: EMPTY
  headline: EMPTY
  skills_count: 0
}
```

**Root Cause**: Data extraction failing or LinkedIn not providing data

**Debug Steps**:
1. Check `enrichedProfile` object in logs
2. Verify LinkedIn API response structure
3. Check if user has data on LinkedIn profile
4. Review `extractLinkedInData()` function

**Solutions**:
1. Ensure user has complete LinkedIn profile
2. Verify API endpoints are correct
3. Check data mapping in extraction function

---

### Issue 4: Database Not Updating

**Symptoms**:
- Logs show "✅ Updating..." but database unchanged

**Debug Steps**:
1. Check Edge Function logs for update errors
2. Verify database permissions
3. Check RLS policies
4. Query database immediately after sync

**SQL Debug Query**:
```sql
-- Check recent updates
SELECT 
  user_id,
  headline,
  skills,
  last_linkedin_sync_at,
  updated_at,
  profile_meta->'linkedin_last_update' as last_update
FROM users
WHERE last_linkedin_sync_at > NOW() - INTERVAL '5 minutes'
ORDER BY last_linkedin_sync_at DESC;
```

---

### Issue 5: UI Not Refreshing

**Symptoms**:
- Database updated but UI shows old data

**Debug Steps**:
1. Check if `_loadUserProfile()` is called
2. Verify profile service fetches from database
3. Check state management updates
4. Look for caching issues

**Flutter Debug**:
```dart
// In AuthNotifier.handleLinkedInCallback
debugPrint('Before reload: ${state.profile?.headline}');
await _loadUserProfile(setGlobalLoading: false);
debugPrint('After reload: ${state.profile?.headline}');
```

---

## 📊 Success Metrics

### Existing User Login
- ✅ Profile image updates: **100%**
- ✅ Headline updates: **100%** (if API available)
- ✅ Skills merge: **100%** (if API available)
- ✅ UI refreshes: **100%**
- ✅ No errors: **100%**

### New User Signup
- ✅ Profile created: **100%**
- ✅ All fields populated: **100%** (if API available)
- ✅ Redirects to onboarding: **100%**
- ✅ No errors: **100%**

### Fallback Behavior
- ✅ Works without API token: **100%**
- ✅ Works with API failures: **100%**
- ✅ No data loss: **100%**

---

## 📝 Test Report Template

```markdown
## LinkedIn Profile Sync Test Report

**Date**: 2026-02-01
**Tester**: [Your Name]
**Environment**: [Development/Staging/Production]

### Test 1: Existing User Login
- [ ] Provider token available
- [ ] LinkedIn API called successfully
- [ ] Profile image updated in database
- [ ] Headline updated in database
- [ ] Skills merged in database
- [ ] UI refreshed with new data
- [ ] No errors in logs

**Notes**: 
_[Any observations or issues]_

### Test 2: New User Signup
- [ ] Profile created successfully
- [ ] All fields populated
- [ ] Redirected to onboarding
- [ ] Profile data visible in onboarding
- [ ] No errors in logs

**Notes**:
_[Any observations or issues]_

### Test 3: Fallback Scenarios
- [ ] Works without provider token
- [ ] Works with API failures
- [ ] Preserves existing data
- [ ] No crashes or errors

**Notes**:
_[Any observations or issues]_

### Overall Result
- [ ] ✅ All tests passed
- [ ] ⚠️ Some tests passed with warnings
- [ ] ❌ Tests failed

**Summary**:
_[Overall assessment and recommendations]_
```

---

## ✅ Final Validation

Before marking as complete, verify:

1. **Code Quality**
   - [ ] Edge Function deployed successfully
   - [ ] Flutter app builds without errors
   - [ ] No lint warnings

2. **Functionality**
   - [ ] Existing users: profile updates
   - [ ] New users: profile created
   - [ ] UI refreshes automatically
   - [ ] Fallback works correctly

3. **Data Integrity**
   - [ ] No data loss
   - [ ] Skills merge correctly
   - [ ] Timestamps update
   - [ ] Database constraints satisfied

4. **User Experience**
   - [ ] Fast sync (< 3 seconds)
   - [ ] Smooth UI updates
   - [ ] No visible errors
   - [ ] Professional appearance

5. **Documentation**
   - [ ] All docs updated
   - [ ] Deployment guide complete
   - [ ] Troubleshooting documented
   - [ ] Test cases documented

---

## 🚀 Ready for Production

Once all tests pass:
- ✅ Deploy to production
- ✅ Monitor logs for 24 hours
- ✅ Gather user feedback
- ✅ Document any issues
- ✅ Plan improvements

**The LinkedIn profile sync is production-ready!** 🎉
