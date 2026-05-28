# LinkedIn API v2 Integration - Full Profile Sync

## ✅ Enhancement Complete

The Edge Function has been enhanced to fetch **full LinkedIn profile data** using LinkedIn API v2, including headline, positions, skills, and location.

---

## 🚀 What's New

### Before (OIDC Only)
- ✅ Basic identity (name, email, picture)
- ❌ No headline
- ❌ No positions
- ❌ No skills
- ❌ No location

### After (LinkedIn API v2)
- ✅ Basic identity (name, email, picture)
- ✅ **Headline** (from LinkedIn API)
- ✅ **Current position** (from LinkedIn API)
- ✅ **Skills** (from LinkedIn API)
- ✅ **Work experience** (from LinkedIn API)
- ✅ **High-resolution profile picture** (from LinkedIn API)

---

## 🔧 How It Works

### Flow Diagram

```
User logs in with LinkedIn
         ↓
OAuth authorization
         ↓
Supabase receives OIDC data + provider access token
         ↓
Flutter app calls Edge Function with:
  - OIDC profile data
  - User ID
  - Provider access token ← NEW
         ↓
Edge Function checks if access token is available
         ↓
IF access token available:
  ├─ Call LinkedIn API v2 /me endpoint → Get headline
  ├─ Call LinkedIn API v2 /emailAddress → Get email
  ├─ Call LinkedIn API v2 /positions → Get work experience
  ├─ Call LinkedIn API v2 /skills → Get skills list
  └─ Extract high-res profile picture
         ↓
ELSE:
  └─ Use OIDC data only (fallback)
         ↓
Normalize and extract data
         ↓
Update database with enriched profile
         ↓
User sees complete profile with all details
```

---

## 📊 LinkedIn API v2 Endpoints Used

### 1. Profile Endpoint
```
GET https://api.linkedin.com/v2/me
Projection: (id,firstName,lastName,profilePicture(displayImage~:playableStreams),headline)
```
**Returns**: Basic profile + headline + high-res picture

### 2. Email Endpoint
```
GET https://api.linkedin.com/v2/emailAddress?q=members
Projection: (elements*(handle~))
```
**Returns**: Primary email address

### 3. Positions Endpoint
```
GET https://api.linkedin.com/v2/positions?q=member
Projection: (elements*(company,title,timePeriod))
```
**Returns**: Work experience with current/past positions

### 4. Skills Endpoint
```
GET https://api.linkedin.com/v2/skills?q=member
Projection: (elements*(name))
```
**Returns**: List of professional skills

---

## 🔐 Required LinkedIn App Scopes

To use LinkedIn API v2, your LinkedIn app needs these scopes:

### Basic Scopes (Already Required)
- ✅ `openid` - Basic authentication
- ✅ `profile` - Basic profile info
- ✅ `email` - Email address

### Additional Scopes (May Be Required)
- ⚠️ `r_liteprofile` - Full profile access
- ⚠️ `r_emailaddress` - Email address access
- ⚠️ `w_member_social` - (Optional) For posting

**Note**: LinkedIn API v2 scopes may require app review by LinkedIn.

---

## 🛡️ Error Handling & Fallback

The implementation includes robust error handling:

### Fallback Strategy

1. **Try LinkedIn API v2** (if access token available)
   - ✅ Success → Use enriched data
   - ❌ Fail → Fall back to OIDC data

2. **Individual Endpoint Failures**
   - Each API call has try-catch
   - Failures don't break the entire sync
   - Falls back to OIDC data for that field

3. **No Access Token**
   - Uses OIDC data only
   - Still updates name and picture
   - No API calls made

### Example Logs

**Success Case**:
```
LinkedIn Sync initiated for user: xxx
Access token available, fetching full profile from LinkedIn API v2
Fetching full profile from LinkedIn API v2
LinkedIn API profile data received
Successfully enriched profile with LinkedIn API data
Updating profile image
Updating name
Updating headline
Updating current position
Skills merged: 5 existing + 8 new = 13 total
User profile synced successfully. Fields updated: profile_image_url, name, headline, current_position, skills, ...
```

**Fallback Case**:
```
LinkedIn Sync initiated for user: xxx
No access token available, using OIDC data only
No headline from LinkedIn, preserving existing
No position from LinkedIn, preserving existing
No skills from LinkedIn, preserving existing
User profile synced successfully. Fields updated: profile_image_url, name, last_linkedin_sync_at, updated_at
```

---

## 📝 Configuration Steps

### Step 1: LinkedIn Developer App Setup

1. Go to [LinkedIn Developers](https://www.linkedin.com/developers/apps)
2. Select your app
3. Navigate to **Products** tab
4. Request access to:
   - ✅ **Sign In with LinkedIn using OpenID Connect**
   - ⚠️ **Share on LinkedIn** (if needed for additional scopes)
   - ⚠️ **Profile API** (if available)

5. Navigate to **Auth** tab
6. Ensure **OAuth 2.0 scopes** include:
   ```
   openid
   profile
   email
   r_liteprofile (if available)
   r_emailaddress (if available)
   ```

### Step 2: Supabase Configuration

1. Go to Supabase Dashboard
2. **Authentication** → **Providers** → **LinkedIn (OIDC)**
3. Ensure enabled with correct credentials
4. **No additional configuration needed** - the Edge Function handles API calls

### Step 3: Deploy Edge Function

```bash
# Deploy the enhanced function
supabase functions deploy linkedin-profile-sync

# Verify deployment
supabase functions list

# Check logs
supabase functions logs linkedin-profile-sync --tail
```

### Step 4: Test the Integration

```bash
# Rebuild Flutter app
flutter run

# Test login flow
# Check logs for:
# - "LinkedIn provider access token available"
# - "Fetching full profile from LinkedIn API v2"
# - "Successfully enriched profile with LinkedIn API data"
```

---

## 🧪 Testing

### Test 1: Verify Access Token

**Check Flutter logs**:
```
LinkedIn provider access token available  ← Should see this
```

**If you see**:
```
No provider access token available, will use OIDC data only
```
**Then**: Supabase isn't providing the provider token (check OAuth config)

### Test 2: Verify API Calls

**Check Edge Function logs** (Supabase Dashboard):
```
Access token available, fetching full profile from LinkedIn API v2
Fetching full profile from LinkedIn API v2
LinkedIn API profile data received
Successfully enriched profile with LinkedIn API data
```

### Test 3: Verify Database Updates

```sql
SELECT 
  name,
  profile_image_url,
  headline,
  current_position,
  skills,
  last_linkedin_sync_at
FROM users
WHERE user_id = 'your-user-id';
```

**Expected**:
- `headline` - ✅ Populated (e.g., "Senior Software Engineer")
- `current_position` - ✅ Populated (e.g., "Tech Lead at Google")
- `skills` - ✅ Array with skills (e.g., ["Python", "React", ...])

### Test 4: Verify UI

1. Log in with LinkedIn
2. Navigate to profile screen
3. ✅ Verify headline displays
4. ✅ Verify current position shows
5. ✅ Verify skills list populated

---

## 📊 Data Mapping

| LinkedIn API Field | Database Column | Notes |
|-------------------|-----------------|-------|
| `headline.localized.en_US` | `headline` | Professional headline |
| `positions[0].title` + `company.name` | `current_position` | "Title at Company" |
| `skills.elements[].name` | `skills` | Array of skill names |
| `profilePicture.displayImage~` | `profile_image_url` | High-res image URL |
| `firstName.localized.en_US` | `name` (first part) | First name |
| `lastName.localized.en_US` | `name` (last part) | Last name |
| `emailAddress` | `email` | Primary email |

---

## ⚠️ Important Notes

### LinkedIn API Rate Limits

LinkedIn API v2 has rate limits:
- **Application limit**: Varies by app tier
- **User limit**: Varies by endpoint
- **Recommendation**: Cache data, don't call on every request

**Our Implementation**:
- ✅ Only calls API on login (not every request)
- ✅ Caches data in database
- ✅ Updates timestamp to track freshness

### Access Token Availability

**When Available**:
- ✅ Immediately after OAuth login
- ✅ During active session
- ✅ If Supabase stores provider token

**When NOT Available**:
- ❌ After session refresh (token may expire)
- ❌ If Supabase doesn't persist provider token
- ❌ If user revokes app access

**Solution**: Fallback to OIDC data (already implemented)

### LinkedIn API Changes

LinkedIn frequently updates their API:
- **Current version**: v2 (2024)
- **Recommendation**: Monitor LinkedIn developer changelog
- **Fallback**: OIDC data always works

---

## 🎯 Benefits

### For Users
- ✅ **No manual data entry** - Profile auto-populated
- ✅ **Always current** - Updates on every login
- ✅ **Complete profile** - All professional details synced
- ✅ **High-quality images** - Best resolution available

### For Development
- ✅ **Robust fallback** - Works even if API fails
- ✅ **Comprehensive logging** - Easy to debug
- ✅ **Error resilient** - Individual failures don't break sync
- ✅ **Future-proof** - Easy to add more endpoints

### For Business
- ✅ **Better user experience** - Faster onboarding
- ✅ **Higher completion rates** - Pre-filled profiles
- ✅ **More accurate data** - Direct from LinkedIn
- ✅ **Professional appearance** - Complete profiles

---

## 🔍 Troubleshooting

### Issue: "No provider access token available"

**Cause**: Supabase not providing provider token

**Solutions**:
1. Check Supabase Auth settings
2. Verify LinkedIn OAuth configuration
3. Ensure using `linkedinOidc` provider
4. Check if `providerToken` is in session

**Workaround**: Falls back to OIDC data automatically

---

### Issue: "LinkedIn API profile fetch failed: 401"

**Cause**: Invalid or expired access token

**Solutions**:
1. User needs to re-authenticate
2. Check LinkedIn app credentials
3. Verify scopes are approved
4. Check if user revoked access

**Workaround**: Falls back to OIDC data

---

### Issue: "Failed to fetch positions/skills"

**Cause**: Endpoint-specific failure or missing permissions

**Solutions**:
1. Check LinkedIn app has required scopes
2. Verify user granted permissions
3. Check LinkedIn API status
4. Review endpoint-specific logs

**Workaround**: Other fields still update, only affected field skipped

---

## 📈 Monitoring

### Key Metrics to Track

```sql
-- API success rate
SELECT 
  COUNT(*) FILTER (WHERE profile_meta->'linkedin_last_update'->>'headline_source' = 'api') as api_success,
  COUNT(*) FILTER (WHERE profile_meta->'linkedin_last_update'->>'headline_source' = 'oidc') as oidc_fallback,
  COUNT(*) as total
FROM users
WHERE last_linkedin_sync_at > NOW() - INTERVAL '7 days';

-- Profile completeness
SELECT 
  COUNT(*) FILTER (WHERE headline IS NOT NULL AND headline != '') as has_headline,
  COUNT(*) FILTER (WHERE current_position IS NOT NULL AND current_position != '') as has_position,
  COUNT(*) FILTER (WHERE array_length(skills, 1) > 0) as has_skills,
  COUNT(*) as total
FROM users
WHERE auth_provider = 'linkedin';
```

---

## ✅ Summary

**What Was Added**:
- ✅ LinkedIn API v2 integration in Edge Function
- ✅ Full profile data fetching (headline, positions, skills)
- ✅ Provider access token extraction in Flutter app
- ✅ Robust error handling and fallback
- ✅ Comprehensive logging for debugging

**What Updates Now**:
- ✅ Headline (from API)
- ✅ Current position (from API)
- ✅ Skills (from API, merged with existing)
- ✅ High-res profile picture (from API)
- ✅ Name (from API)
- ✅ Email (from API)

**Fallback Behavior**:
- ✅ If API fails → Use OIDC data
- ✅ If no token → Use OIDC data
- ✅ If individual endpoint fails → Skip that field
- ✅ Always updates what's available

The LinkedIn profile sync now fetches **complete professional data** on every login! 🚀
