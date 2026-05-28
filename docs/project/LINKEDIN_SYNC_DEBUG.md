# LinkedIn Profile Sync - Debugging Guide

## 🐛 Issue: Profile Data Not Updating on Login

### Problem Statement
When existing users log in with LinkedIn, their profile data (image, headline, position, etc.) is not being updated in the database.

### Root Cause Analysis

**LinkedIn OIDC Limitation**: Supabase's LinkedIn OIDC provider only returns **basic OIDC claims**, not the full LinkedIn profile API data.

**What Supabase LinkedIn OIDC Provides**:
- ✅ `sub` - LinkedIn user ID
- ✅ `email` - Email address
- ✅ `name` - Full name
- ✅ `given_name` - First name
- ✅ `family_name` - Last name
- ✅ `picture` - Profile picture URL
- ❌ `headline` - NOT provided by default
- ❌ `positions` - NOT provided by default
- ❌ `skills` - NOT provided by default
- ❌ `location` - NOT provided by default

---

## 🔍 Debugging Steps

### Step 1: Check What Data is Received

**Run the app and check Flutter console logs**:

```
=== LinkedIn Metadata Received ===
All metadata keys: [sub, email, name, picture, given_name, family_name]
sub: abc123
email: user@example.com
name: John Doe
full_name: null
given_name: John
family_name: Doe
picture: https://media.licdn.com/dms/image/...
avatar_url: null
headline: null  ← THIS IS THE PROBLEM
positions: null  ← THIS IS THE PROBLEM
skills: null  ← THIS IS THE PROBLEM
location: null  ← THIS IS THE PROBLEM
=================================
```

### Step 2: Check Edge Function Logs

**In Supabase Dashboard** → **Edge Functions** → **linkedin-profile-sync** → **Logs**:

```
LinkedIn profile data received: {
  has_id: true,
  has_email: true,
  has_name: true,
  has_picture: true,
  has_headline: false,  ← Confirms no headline
  has_positions: false,  ← Confirms no positions
  has_skills: false,  ← Confirms no skills
  has_location: false,  ← Confirms no location
  raw_keys: ['id', 'email', 'name', 'profilePicture']
}

Normalized data: {
  linkedin_id: 'abc123',
  email: 'user@example.com',
  name: 'John Doe',
  profile_image_url: 'https://...',
  headline: '',  ← Empty because not provided
  current_position: '',  ← Empty because not provided
  skills: [],  ← Empty because not provided
  location: '',  ← Empty because not provided
  city: ''
}
```

### Step 3: Check Database Update

**Query the database**:

```sql
SELECT 
  user_id,
  name,
  profile_image_url,
  headline,
  current_position,
  skills,
  location,
  last_linkedin_sync_at,
  updated_at
FROM users
WHERE user_id = 'your-user-id';
```

**Expected Result with Current OIDC**:
- `name` - ✅ Updated (John Doe)
- `profile_image_url` - ✅ Updated (https://...)
- `headline` - ❌ Empty or unchanged (not provided by OIDC)
- `current_position` - ❌ Empty or unchanged (not provided by OIDC)
- `skills` - ❌ Empty array or unchanged (not provided by OIDC)
- `location` - ❌ Empty or unchanged (not provided by OIDC)
- `last_linkedin_sync_at` - ✅ Updated timestamp

---

## 💡 Solutions

### Solution 1: Accept Limited Data (Quick Fix)

**What Updates**:
- ✅ Profile image
- ✅ Name
- ✅ Email

**What Doesn't Update**:
- ❌ Headline
- ❌ Current position
- ❌ Skills
- ❌ Location

**Implementation**: Already done - the current code handles this gracefully.

**User Impact**: Users will see updated profile pictures and names, but professional details won't sync.

---

### Solution 2: Use LinkedIn API v2 (Full Profile Data)

To get full profile data, we need to:

1. **Request Additional Scopes** in Supabase LinkedIn config:
   ```
   openid
   profile
   email
   r_liteprofile
   r_emailaddress
   ```

2. **Make Additional API Call** to LinkedIn:
   ```typescript
   // In Edge Function, after OAuth
   const linkedInApiUrl = 'https://api.linkedin.com/v2/me';
   const profileResponse = await fetch(linkedInApiUrl, {
     headers: {
       'Authorization': `Bearer ${access_token}`,
       'X-Restli-Protocol-Version': '2.0.0'
     }
   });
   
   const profileData = await profileResponse.json();
   ```

3. **Extract Full Profile Data**:
   ```typescript
   // Get headline
   const headlineUrl = 'https://api.linkedin.com/v2/me?projection=(headline)';
   
   // Get positions
   const positionsUrl = 'https://api.linkedin.com/v2/me?projection=(positions)';
   
   // Get skills
   const skillsUrl = 'https://api.linkedin.com/v2/me?projection=(skills)';
   ```

**Pros**:
- ✅ Full profile data available
- ✅ Headline, position, skills, location all sync

**Cons**:
- ❌ More complex implementation
- ❌ Additional API calls (rate limits)
- ❌ Requires LinkedIn API app review
- ❌ Need to handle access token properly

---

### Solution 3: Hybrid Approach (Recommended)

1. **On OAuth**: Update basic info (name, picture) from OIDC
2. **Manual Sync Button**: Let users manually trigger full profile sync
3. **Periodic Sync**: Background job to refresh full data weekly

**Benefits**:
- ✅ Fast initial login (OIDC only)
- ✅ Full data available when needed
- ✅ User control over data sync
- ✅ Respects API rate limits

---

## 🔧 Immediate Fix: Update Edge Function Logic

Let's modify the update logic to only update fields that have actual data:

```typescript
// EXISTING USER - Login Flow
const updatePayload: any = {
  updated_at: new Date().toISOString(),
  last_linkedin_sync_at: new Date().toISOString(),
}

// Only update if we have new data
if (normalizedData.profile_image_url) {
  updatePayload.profile_image_url = normalizedData.profile_image_url
}

if (normalizedData.name) {
  updatePayload.name = normalizedData.name
}

// Don't update these if empty (preserve existing data)
if (normalizedData.headline) {
  updatePayload.headline = normalizedData.headline
}

if (normalizedData.current_position) {
  updatePayload.current_position = normalizedData.current_position
}

if (normalizedData.location) {
  updatePayload.location = normalizedData.location
  updatePayload.city = normalizedData.city
}

// Only merge skills if we have new ones
if (normalizedData.skills && normalizedData.skills.length > 0) {
  const existingSkills = existingUser.skills || []
  const mergedSkills = Array.from(new Set([...existingSkills, ...normalizedData.skills]))
  updatePayload.skills = mergedSkills
}
```

This way:
- ✅ Profile image and name always update (from OIDC)
- ✅ Professional details preserved if not provided
- ✅ No data loss
- ✅ Timestamp still updated

---

## 📊 Testing the Fix

### Test 1: Check Logs

1. Log in with LinkedIn
2. Check Flutter console for metadata
3. Check Supabase Edge Function logs
4. Verify what data is actually received

### Test 2: Database Verification

```sql
-- Before login
SELECT name, profile_image_url, headline, last_linkedin_sync_at 
FROM users WHERE user_id = 'xxx';

-- After login
SELECT name, profile_image_url, headline, last_linkedin_sync_at 
FROM users WHERE user_id = 'xxx';

-- Compare: name and image should update, headline should be preserved
```

### Test 3: UI Verification

1. Change profile picture on LinkedIn
2. Log out and log back in
3. ✅ Verify new picture shows immediately
4. ✅ Verify name updates if changed
5. ✅ Verify headline/position unchanged (preserved)

---

## 📝 Recommendations

### Short Term (Immediate)
1. ✅ Deploy enhanced logging (already done)
2. ✅ Test and verify what data is received
3. ✅ Update Edge Function to preserve empty fields
4. ✅ Document limitations to users

### Medium Term (Next Sprint)
1. Add manual "Sync LinkedIn Profile" button in settings
2. Use LinkedIn API v2 for full profile sync
3. Implement proper token management
4. Add user consent for extended permissions

### Long Term (Future)
1. Periodic background sync (weekly)
2. LinkedIn API integration for posts/shares
3. Network recommendations
4. Achievement sharing

---

## 🎯 Current Status

**What Works**:
- ✅ OAuth authentication
- ✅ Basic profile data (name, picture)
- ✅ Database updates
- ✅ Timestamp tracking
- ✅ Error handling

**What's Limited**:
- ⚠️ Headline not synced (OIDC limitation)
- ⚠️ Position not synced (OIDC limitation)
- ⚠️ Skills not synced (OIDC limitation)
- ⚠️ Location not synced (OIDC limitation)

**Next Steps**:
1. Run app with enhanced logging
2. Check what data is actually received
3. Decide on solution approach
4. Implement chosen solution

---

## 📞 Support

If you need help:
1. Check Flutter console logs for metadata
2. Check Supabase Edge Function logs
3. Query database to verify updates
4. Review this debugging guide
5. Consider LinkedIn API v2 integration for full data
