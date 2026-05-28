# LinkedIn Authentication - Quick Setup Guide

## Prerequisites

- Supabase project configured
- LinkedIn Developer Account
- Flutter development environment

## Step 1: LinkedIn App Configuration

1. Go to [LinkedIn Developers](https://www.linkedin.com/developers/apps)
2. Create a new app or select existing app
3. Navigate to "Auth" tab
4. Add redirect URL:
   ```
   https://[YOUR-PROJECT-REF].supabase.co/auth/v1/callback
   ```
5. Add mobile redirect URL:
   ```
   com.antigravity.aiinterviewcoach://login-callback
   ```
6. Request OAuth 2.0 scopes:
   - `openid`
   - `profile`
   - `email`
7. Copy your **Client ID** and **Client Secret**

## Step 2: Supabase Configuration

1. Open Supabase Dashboard
2. Navigate to **Authentication** → **Providers**
3. Enable **LinkedIn** provider
4. Enter your LinkedIn **Client ID** and **Client Secret**
5. Save configuration

## Step 3: Deploy Database Migration

```bash
cd supabase
supabase db push
```

This will apply the migration: `20260201000000_linkedin_auth_enhancement.sql`

## Step 4: Deploy Edge Function

```bash
supabase functions deploy linkedin-profile-sync
```

Verify deployment:
```bash
supabase functions list
```

## Step 5: Test Edge Function (Optional)

```bash
curl -X POST \
  https://[YOUR-PROJECT-REF].supabase.co/functions/v1/linkedin-profile-sync \
  -H "Authorization: Bearer [YOUR-ANON-KEY]" \
  -H "Content-Type: application/json" \
  -d '{
    "linkedin_profile": {
      "id": "test-id",
      "email": "test@example.com",
      "name": "Test User",
      "headline": "Software Engineer"
    },
    "user_id": "[TEST-USER-UUID]"
  }'
```

## Step 6: Update Android Configuration

Edit `android/app/src/main/AndroidManifest.xml`:

```xml
<activity
    android:name=".MainActivity"
    ...>
    
    <!-- Existing intent filters -->
    
    <!-- Add LinkedIn OAuth callback -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="com.antigravity.aiinterviewcoach"
            android:host="login-callback" />
    </intent-filter>
</activity>
```

## Step 7: Update iOS Configuration

Edit `ios/Runner/Info.plist`:

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>com.antigravity.aiinterviewcoach</string>
        </array>
    </dict>
</array>
```

## Step 8: Test Authentication Flow

### Test New User Signup

1. Run the app
2. Navigate to Login screen
3. Click "Sign Up with LinkedIn"
4. Authorize the application
5. Verify redirect to onboarding screen
6. Check that profile fields are pre-populated

### Test Existing User Login

1. Create a user account (via email)
2. Complete onboarding
3. Logout
4. Click "Continue with LinkedIn" on login screen
5. Authorize with same email
6. Verify redirect to home screen
7. Check that profile data is updated

## Verification Checklist

- [ ] LinkedIn OAuth configured in Supabase
- [ ] Database migration applied
- [ ] Edge Function deployed
- [ ] Android deep link configured
- [ ] iOS URL scheme configured
- [ ] New user signup works
- [ ] Existing user login works
- [ ] Profile data syncs correctly
- [ ] Skills are merged (not replaced)
- [ ] Profile image displays
- [ ] Location auto-populates

## Troubleshooting

### OAuth Redirect Fails

**Problem**: App doesn't open after LinkedIn authorization

**Solution**:
1. Verify redirect URL in LinkedIn app settings
2. Check Android/iOS deep link configuration
3. Test deep link manually:
   ```bash
   adb shell am start -W -a android.intent.action.VIEW \
     -d "com.antigravity.aiinterviewcoach://login-callback"
   ```

### Profile Sync Fails

**Problem**: User authenticated but profile not created

**Solution**:
1. Check Edge Function logs in Supabase dashboard
2. Verify user has required LinkedIn permissions
3. Test Edge Function directly (see Step 5)
4. Check database for user record

### Skills Not Merging

**Problem**: Existing skills are replaced instead of merged

**Solution**:
1. Check Edge Function logic in `extractLinkedInData()`
2. Verify `skills` array is being properly merged
3. Review database update query

### Profile Image Not Displaying

**Problem**: LinkedIn profile image not showing

**Solution**:
1. Verify `profile_image_url` is being extracted
2. Check image URL is publicly accessible
3. Test URL directly in browser
4. Verify `CachedNetworkImage` is configured correctly

## Environment Variables

Ensure these are set in your `.env` file:

```env
SUPABASE_URL=https://[YOUR-PROJECT-REF].supabase.co
SUPABASE_ANON_KEY=[YOUR-ANON-KEY]
```

## Security Notes

1. **Never commit** LinkedIn Client Secret to version control
2. **Always use** Supabase environment variables for secrets
3. **Rotate tokens** if compromised
4. **Monitor** Edge Function logs for suspicious activity
5. **Implement rate limiting** for production

## Next Steps

After successful setup:

1. Monitor authentication metrics in Supabase dashboard
2. Set up error tracking (e.g., Sentry)
3. Configure analytics for LinkedIn conversion
4. Test with multiple LinkedIn accounts
5. Prepare for production deployment

## Support Resources

- [Supabase Auth Documentation](https://supabase.com/docs/guides/auth)
- [LinkedIn OAuth Documentation](https://docs.microsoft.com/en-us/linkedin/shared/authentication/authentication)
- [Flutter Deep Linking](https://docs.flutter.dev/development/ui/navigation/deep-linking)
- Project Documentation: `LINKEDIN_AUTH_IMPLEMENTATION.md`
