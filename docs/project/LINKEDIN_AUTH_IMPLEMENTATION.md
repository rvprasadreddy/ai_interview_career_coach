# LinkedIn Authentication Implementation

## Overview

This document describes the production-grade LinkedIn OAuth 2.0 authentication implementation for the InterviPrep application. The system supports both **new user signup** and **existing user login** with intelligent profile synchronization.

## Architecture

### Components

1. **Edge Function** (`linkedin-profile-sync`)
   - Handles profile data extraction and normalization
   - Intelligently detects new vs. existing users
   - Merges LinkedIn data with existing profiles
   - Preserves app-specific data (scores, progress, preferences)

2. **AuthService** (`auth_service.dart`)
   - `signInWithLinkedIn()` - Initiates OAuth flow
   - `syncLinkedInProfile()` - Calls Edge Function to sync data
   - `handleLinkedInCallback()` - Processes OAuth redirect

3. **AuthNotifier** (`auth_provider.dart`)
   - `signInWithLinkedIn()` - State management for OAuth
   - `handleLinkedInCallback()` - Coordinates profile sync and navigation

4. **LinkedInCallbackScreen** (`linkedin_callback_screen.dart`)
   - Handles OAuth redirect
   - Shows sync progress
   - Routes users based on status (new/existing)

5. **Database Migration** (`20260201000000_linkedin_auth_enhancement.sql`)
   - Adds LinkedIn-specific tracking columns
   - Enhances user creation trigger
   - Creates indexes for performance

## Authentication Flow

### New User (Signup)

```
1. User clicks "Sign Up with LinkedIn" on LoginScreen
2. OAuth redirect to LinkedIn
3. User authorizes application
4. LinkedIn redirects to: com.antigravity.aiinterviewcoach://login-callback
5. LinkedInCallbackScreen processes callback
6. AuthService.handleLinkedInCallback() called
7. Edge Function creates new user record with LinkedIn data:
   - Profile image
   - Headline
   - Current position
   - Skills (top 10)
   - Location
   - City
8. User redirected to ProfileSetupScreen to complete onboarding
```

### Existing User (Login)

```
1. User clicks "Continue with LinkedIn" on LoginScreen
2. OAuth redirect to LinkedIn
3. User authorizes application
4. LinkedIn redirects to callback
5. LinkedInCallbackScreen processes callback
6. Edge Function AUTOMATICALLY syncs latest profile data:
   - ✅ Refreshes profile image (always updated)
   - ✅ Updates headline (always updated)
   - ✅ Updates current position (always updated)
   - ✅ Merges skills (adds new, keeps existing - never removes)
   - ✅ Updates location and city (always updated)
   - ✅ Updates name if changed on LinkedIn
   - ✅ Preserves app-specific data (scores, progress, preferences)
   - ✅ Records sync timestamp
7. User redirected to HomeScreen with fresh profile data
```

**Note**: Profile sync happens **automatically on every login** to ensure data is always current.

## Database Schema

### New Columns (users table)

```sql
last_linkedin_sync_at TIMESTAMPTZ    -- Timestamp of last sync
linkedin_profile_url TEXT             -- Public LinkedIn URL
linkedin_access_token_hash TEXT       -- Hashed token (security audit)
```

### Existing Columns Used

- `linkedin_id` - LinkedIn unique identifier
- `profile_image_url` - Profile photo
- `headline` - Professional headline
- `current_position` - Current job title + company
- `skills` - Array of skills
- `location` - Full location string
- `city` - Extracted city name
- `auth_provider` - Set to 'linkedin'
- `profile_meta` - JSONB with sync metadata

## Edge Function Details

### Endpoint
`/functions/v1/linkedin-profile-sync`

### Request Payload
```json
{
  "linkedin_profile": {
    "id": "string",
    "email": "string",
    "name": "string",
    "profilePicture": { "displayImage": "url" },
    "headline": "string",
    "positions": { "values": [...] },
    "skills": { "values": [...] },
    "location": { "name": "string" }
  },
  "user_id": "uuid",
  "access_token": "managed_by_supabase"
}
```

### Response
```json
{
  "success": true,
  "is_new_user": false,
  "user_profile": { ... },
  "message": "LinkedIn profile refreshed successfully. Welcome back!"
}
```

## Security Considerations

1. **Token Management**
   - Access tokens managed by Supabase Auth
   - Never stored in plaintext
   - Only hashed version stored for audit

2. **Data Privacy**
   - Only essential profile data extracted
   - Skills limited to top 10
   - Raw LinkedIn payload stored in `profile_meta` (optional)

3. **RLS Policies**
   - Existing user policies apply
   - Users can only access their own data

## UI/UX Flow

### Login Screen
- "Continue with LinkedIn" button (existing users)
- "Sign Up with LinkedIn" button (new users)
- LinkedIn blue color (#0A66C2)
- Business icon

### Callback Screen
- LinkedIn logo/icon
- "Syncing your LinkedIn profile..." message
- Loading indicator
- Error handling with redirect to login

### Profile Setup (New Users)
- Pre-populated fields:
  - Name
  - Email
  - Profile image
  - Headline
  - Skills
  - Current position
  - Location
- User can edit/confirm before proceeding

### Home Screen (Existing Users)
- Updated profile data immediately visible
- Avatar refreshed
- Headline updated
- Skills merged

## Error Handling

### OAuth Errors
- Invalid credentials → Show error, redirect to login
- User cancels → Return to login screen
- Network timeout → Show retry option

### Sync Errors
- Edge Function failure → Show error message
- Profile not found → Create new user
- Database error → Log and show generic error

## Testing Checklist

- [ ] New user signup via LinkedIn
- [ ] Existing user login via LinkedIn
- [ ] Profile data correctly synced
- [ ] Skills properly merged
- [ ] Location extracted correctly
- [ ] Error handling for failed OAuth
- [ ] Error handling for failed sync
- [ ] Redirect to correct screen (onboarding vs home)
- [ ] Profile image displayed
- [ ] Headline updated

## Deployment Steps

1. **Database Migration**
   ```bash
   supabase db push
   ```

2. **Deploy Edge Function**
   ```bash
   supabase functions deploy linkedin-profile-sync
   ```

3. **Configure LinkedIn OAuth in Supabase**
   - Add LinkedIn as OAuth provider
   - Set redirect URL: `com.antigravity.aiinterviewcoach://login-callback`
   - Configure scopes: `openid profile email`

4. **Test Authentication Flow**
   - Test new user signup
   - Test existing user login
   - Verify profile sync

## Monitoring

### Key Metrics
- LinkedIn signup conversion rate
- Profile sync success rate
- Average sync time
- Error rate by type

### Logs to Monitor
- `AuthService: Initiating LinkedIn OAuth...`
- `AuthService: LinkedIn sync completed. Is new user: [true/false]`
- `LinkedInCallbackScreen: Processing LinkedIn callback...`
- Edge Function logs in Supabase dashboard

## Future Enhancements

1. **Periodic Sync**
   - Auto-refresh profile data on login
   - Background sync every 30 days

2. **Enhanced Data Extraction**
   - Education history
   - Certifications
   - Recommendations

3. **LinkedIn Integration**
   - Share interview results
   - Post achievements
   - Network recommendations

## Support

For issues or questions:
1. Check Supabase logs for Edge Function errors
2. Review Flutter console for client-side errors
3. Verify OAuth configuration in Supabase dashboard
4. Test with different LinkedIn accounts
