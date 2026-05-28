# iOS Deep Link Configuration for LinkedIn OAuth

## Overview
Since the iOS directory doesn't exist yet in this project, here's the configuration you'll need to add when you initialize iOS support.

## Info.plist Configuration

Add the following to your `ios/Runner/Info.plist` file (inside the `<dict>` tag):

```xml
<!-- LinkedIn OAuth Deep Link Configuration -->
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>CFBundleURLName</key>
        <string>com.antigravity.ai_interview_coach</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>com.antigravity.aiinterviewcoach</string>
            <string>interviewcoach</string>
        </array>
    </dict>
</array>

<!-- Allow LinkedIn OAuth to open in external browser if needed -->
<key>LSApplicationQueriesSchemes</key>
<array>
    <string>https</string>
    <string>http</string>
</array>
```

## Complete Info.plist Example

Here's what your complete `Info.plist` should look like:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleDevelopmentRegion</key>
    <string>$(DEVELOPMENT_LANGUAGE)</string>
    
    <key>CFBundleDisplayName</key>
    <string>AI Interview Coach</string>
    
    <key>CFBundleExecutable</key>
    <string>$(EXECUTABLE_NAME)</string>
    
    <key>CFBundleIdentifier</key>
    <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
    
    <key>CFBundleInfoDictionaryVersion</key>
    <string>6.0</string>
    
    <key>CFBundleName</key>
    <string>ai_interview_coach</string>
    
    <key>CFBundlePackageType</key>
    <string>APPL</string>
    
    <key>CFBundleShortVersionString</key>
    <string>$(FLUTTER_BUILD_NAME)</string>
    
    <key>CFBundleSignature</key>
    <string>????</string>
    
    <key>CFBundleVersion</key>
    <string>$(FLUTTER_BUILD_NUMBER)</string>
    
    <key>LSRequiresIPhoneOS</key>
    <true/>
    
    <key>UILaunchStoryboardName</key>
    <string>LaunchScreen</string>
    
    <key>UIMainStoryboardFile</key>
    <string>Main</string>
    
    <key>UISupportedInterfaceOrientations</key>
    <array>
        <string>UIInterfaceOrientationPortrait</string>
        <string>UIInterfaceOrientationLandscapeLeft</string>
        <string>UIInterfaceOrientationLandscapeRight</string>
    </array>
    
    <key>UISupportedInterfaceOrientations~ipad</key>
    <array>
        <string>UIInterfaceOrientationPortrait</string>
        <string>UIInterfaceOrientationPortraitUpsideDown</string>
        <string>UIInterfaceOrientationLandscapeLeft</string>
        <string>UIInterfaceOrientationLandscapeRight</string>
    </array>
    
    <key>UIViewControllerBasedStatusBarAppearance</key>
    <false/>
    
    <key>CADisableMinimumFrameDurationOnPhone</key>
    <true/>
    
    <key>UIApplicationSupportsIndirectInputEvents</key>
    <true/>
    
    <!-- LinkedIn OAuth Deep Link Configuration -->
    <key>CFBundleURLTypes</key>
    <array>
        <dict>
            <key>CFBundleTypeRole</key>
            <string>Editor</string>
            <key>CFBundleURLName</key>
            <string>com.antigravity.aiinterviewcoach</string>
            <key>CFBundleURLSchemes</key>
            <array>
                <string>com.antigravity.aiinterviewcoach</string>
                <string>interviewcoach</string>
            </array>
        </dict>
    </array>
    
    <!-- Allow LinkedIn OAuth to open in external browser if needed -->
    <key>LSApplicationQueriesSchemes</key>
    <array>
        <string>https</string>
        <string>http</string>
    </array>
    
    <!-- Microphone permission for speech-to-text -->
    <key>NSMicrophoneUsageDescription</key>
    <string>This app needs access to your microphone to record your interview responses.</string>
    
    <!-- Speech recognition permission -->
    <key>NSSpeechRecognitionUsageDescription</key>
    <string>This app uses speech recognition to transcribe your interview responses.</string>
</dict>
</plist>
```

## How to Initialize iOS Support

If you want to add iOS support to this project, run:

```bash
flutter create --platforms=ios .
```

This will generate the `ios` directory with all necessary files.

## Testing Deep Links on iOS

After adding the configuration, test the deep link:

```bash
# Using xcrun simctl (iOS Simulator)
xcrun simctl openurl booted "com.antigravity.ai_interview_coach://login-callback"

# Or using adb for real device
# (You'll need to use Xcode for real device testing)
```

## Verification Checklist

- [ ] `ios/Runner/Info.plist` exists
- [ ] `CFBundleURLTypes` added with correct schemes
- [ ] `LSApplicationQueriesSchemes` added for external browser
- [ ] Microphone permissions added (for speech-to-text)
- [ ] Deep link tested in iOS Simulator
- [ ] Deep link tested on real device

## Notes

1. **Bundle Identifier**: Make sure your bundle identifier in Xcode matches `com.antigravity.aiinterviewcoach`
2. **URL Schemes**: Both `com.antigravity.aiinterviewcoach` and `interviewcoach` are configured for backward compatibility
3. **Permissions**: Microphone and speech recognition permissions are included for the interview recording feature
4. **Testing**: Always test deep links on both simulator and real devices

## Troubleshooting

### Deep Link Not Working

1. Clean and rebuild the iOS app
2. Verify bundle identifier matches
3. Check Xcode logs for URL scheme conflicts
4. Ensure app is installed and running

### OAuth Redirect Fails

1. Verify redirect URL in LinkedIn app settings
2. Check Supabase OAuth configuration
3. Test with a simple deep link first
4. Review iOS system logs

## Related Files

- Android configuration: `android/app/src/main/AndroidManifest.xml` ✅ (Already configured)
- Auth service: `lib/features/auth/services/auth_service.dart`
- Callback handler: `lib/features/auth/screens/linkedin_callback_screen.dart`
