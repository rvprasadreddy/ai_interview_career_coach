# LinkedIn Sync - Log Analysis & Diagnostic Script

## 🔍 Step-by-Step Diagnostic Process

### Step 1: Capture Flutter App Logs

**Run the app and perform LinkedIn login, then copy ALL console output here:**

```
[Paste Flutter console output here]

Look for these specific sections:
1. === LinkedIn Metadata Received ===
2. LinkedIn provider access token available (or not)
3. Prepared LinkedIn profile payload
4. LinkedIn sync completed
```

---

### Step 2: Capture Supabase Edge Function Logs

**Go to Supabase Dashboard → Edge Functions → linkedin-profile-sync → Logs**

**Copy the most recent logs here:**

```
[Paste Supabase Edge Function logs here]

Look for:
1. LinkedIn Sync initiated for user: xxx
2. Access token available (or not)
3. LinkedIn profile data received
4. Normalized data
5. Checking normalized data values
6. ✅ Updating... or ❌ No ... from LinkedIn
```

---

### Step 3: Check Database State

**Run these queries BEFORE and AFTER login:**

**BEFORE Login:**
```sql
SELECT 
  user_id,
  name,
  email,
  profile_image_url,
  headline,
  current_position,
  skills,
  last_linkedin_sync_at,
  updated_at,
  auth_provider
FROM users
WHERE email = 'your-email@example.com';
```

**Result BEFORE:**
```
[Paste query result here]
```

**AFTER Login (immediately after LinkedIn sync):**
```sql
SELECT 
  user_id,
  name,
  email,
  profile_image_url,
  headline,
  current_position,
  skills,
  last_linkedin_sync_at,
  updated_at,
  auth_provider,
  profile_meta
FROM users
WHERE email = 'your-email@example.com';
```

**Result AFTER:**
```
[Paste query result here]
```

---

## 🔧 Common Issues & Quick Fixes

### Issue Pattern 1: No Provider Token

**Flutter Logs Show:**
```
No provider access token available, will use OIDC data only
```

**Edge Function Logs Show:**
```
No access token available, using OIDC data only
```

**Root Cause**: Supabase not providing provider token

**Quick Fix**:
The issue is that `session.providerToken` might not be available immediately after OAuth.
We need to get the token from the user's identity instead.

---

### Issue Pattern 2: Empty Normalized Data

**Edge Function Logs Show:**
```
Checking normalized data values: {
  profile_image_url: EMPTY
  headline: EMPTY
  skills_count: 0
}
```

**Root Cause**: Data extraction failing

**Quick Fix**:
Check the `enrichedProfile` object structure - LinkedIn API might have changed response format.

---

### Issue Pattern 3: Update Not Persisting

**Edge Function Logs Show:**
```
✅ Updating headline: Senior Engineer
✅ Updating skills: ...
```

**But Database Shows**: Old values

**Root Cause**: Database update failing silently or RLS blocking update

**Quick Fix**:
Check for update errors in Edge Function logs and verify RLS policies.

---

## 🛠️ Immediate Fixes to Try

### Fix 1: Get Provider Token from Identity

The provider token might be in `user.identities` instead of `session.providerToken`.

Let me update the Flutter code:

```dart
// In auth_service.dart, syncLinkedInProfile method

// Get the LinkedIn provider access token
final session = _client.auth.currentSession;
String? providerToken;

// Try to get from session first
if (session != null && session.providerToken != null) {
  providerToken = session.providerToken;
  debugPrint('LinkedIn provider access token from session');
} else if (user.identities != null && user.identities!.isNotEmpty) {
  // Try to get from user identities
  final linkedInIdentity = user.identities!.firstWhere(
    (identity) => identity.provider == 'linkedin_oidc',
    orElse: () => user.identities!.first,
  );
  
  // Access token might be in identity data
  if (linkedInIdentity.identityData != null) {
    providerToken = linkedInIdentity.identityData!['provider_token'] as String?;
    debugPrint('LinkedIn provider access token from identity');
  }
}

if (providerToken != null) {
  debugPrint('✅ Provider token available (length: ${providerToken.length})');
} else {
  debugPrint('❌ No provider access token available');
}
```

### Fix 2: Force Database Update with Explicit Values

If the conditional updates aren't working, let's try forcing the update:

```typescript
// In Edge Function, for existing users
const updatePayload: any = {
  // Force update these fields regardless
  name: normalizedData.name || existingUser.name,
  profile_image_url: normalizedData.profile_image_url || existingUser.profile_image_url,
  headline: normalizedData.headline || existingUser.headline,
  current_position: normalizedData.current_position || existingUser.current_position,
  location: normalizedData.location || existingUser.location,
  city: normalizedData.city || existingUser.city,
  
  // Always update timestamps
  last_linkedin_sync_at: new Date().toISOString(),
  updated_at: new Date().toISOString(),
}
```

### Fix 3: Add Explicit Error Logging

```typescript
const { data: updatedUser, error: updateError } = await supabaseClient
  .from('users')
  .update(updatePayload)
  .eq('user_id', user_id)
  .select()
  .single()

if (updateError) {
  console.error('❌ DATABASE UPDATE ERROR:', updateError)
  console.error('Error code:', updateError.code)
  console.error('Error message:', updateError.message)
  console.error('Error details:', updateError.details)
  console.error('Error hint:', updateError.hint)
  throw updateError
}

console.log('✅ Database update successful')
console.log('Updated user data:', updatedUser)
```

---

## 📊 Diagnostic Checklist

Please check and provide information for each:

### Flutter App
- [ ] App version/build number: ___________
- [ ] Flutter version: ___________
- [ ] Supabase Flutter SDK version: ___________
- [ ] Device/Emulator: ___________
- [ ] Console shows "LinkedIn provider access token available": YES / NO
- [ ] Console shows "LinkedIn sync completed": YES / NO
- [ ] Any error messages in console: ___________

### Supabase
- [ ] Project URL: ___________
- [ ] Edge Function deployed: YES / NO
- [ ] Edge Function version/timestamp: ___________
- [ ] LinkedIn OIDC enabled in Auth settings: YES / NO
- [ ] LinkedIn app credentials configured: YES / NO

### Database
- [ ] User exists in database: YES / NO
- [ ] `auth_provider` value: ___________
- [ ] `last_linkedin_sync_at` updates after login: YES / NO
- [ ] `updated_at` updates after login: YES / NO
- [ ] RLS policies enabled: YES / NO

### LinkedIn
- [ ] LinkedIn app is active: YES / NO
- [ ] Redirect URL configured: ___________
- [ ] Scopes requested: ___________
- [ ] User has complete LinkedIn profile: YES / NO

---

## 🎯 Next Steps Based on Logs

**After you provide the logs above, I can:**

1. Identify the exact point of failure
2. Determine if it's a token issue, data extraction issue, or database issue
3. Provide a targeted fix
4. Create a patch if needed

**Please copy and paste:**
1. ✅ Complete Flutter console output (from login start to finish)
2. ✅ Complete Supabase Edge Function logs (most recent execution)
3. ✅ Database query results (before and after)
4. ✅ Any error messages you see

This will help me pinpoint the exact issue and provide a precise fix! 🔍
