# Error Log & Incremental Mistakes

This file documents technical mistakes, compilation errors, and logic bugs encountered during development, along with their resolutions.

## 1. Missing Method Name in `AuthNotifier`
- **Error**: Compilation error due to a missing method name `signInWithLinkedIn` in `auth_provider.dart`.
- **Cause**: Accidental deletion or incomplete code generation during a file update.
- **Resolution**: Restored the method name and its implementation.
- **Lesson**: Double-check file diffs during large replacements to ensure method signatures remain intact.

## 2. Missing `OtpType` in `OtpScreen`
- **Error**: `lib/features/auth/screens/otp_screen.dart:79:33: Error: The getter 'OtpType' isn't defined.`
- **Cause**: Used `OtpType.signup` and `OtpType.email` (from `supabase_flutter`) without importing the package in `otp_screen.dart`.
- **Resolution**: Added `import 'package:supabase_flutter/supabase_flutter.dart';`.
- **Lesson**: Always ensure external types from packages are backed by appropriate imports, especially after refactoring logic into a new screen.

## 3. Router Instance Reset to Initial Location
- **Error**: App redirected to Login screen instead of OTP screen after signup.
- **Cause**: Even when the router provider was stable, certain lifecycle events (like Android activity rebuilds or accidental parent widget refreshes) caused the `GoRouter` instance to be re-instantiated. Since `initialLocation` was hardcoded to `/login`, any new instance defaulted to the Login screen, regardless of the `redirect` logic's intentions.
- **Resolution**: 
  1. Implemented a private singleton pattern (`_AppRouter.getInstance`) in `routes.dart` to ensure it's impossible to create more than one router instance during the app's life.
  2. Changed `initialLocation` to be dynamic: it now evaluates `verifyingEmail` and `isAuthenticated` state at the moment of creation to start the app in the correct flow.
- **Lesson**: `initialLocation` is a powerful "fallback" but can be a trap if not aligned with current state. For critical flows like signup, ensure the router's birth state matches the app's global state.

## 5. Unmounted Widget during SignUp Navigation
- **Error**: `LoginScreen: signUp completed but widget is NOT mounted.`
- **Cause**: A race condition where global state changes (triggered by Supabase or Riverpod) caused a full context rebuild *during* the transition between signup and navigation.
- **Resolution**: (Resolved) Transitioned to a 100% state-driven navigation model. The `LoginScreen` no longer attempts to "push" a route. Instead, it updates the `AuthState`, and the `GoRouter` (now persistent and stable) handles the redirection globally. Added a 100ms artificial delay to allow the Flutter frame to settle.
- **Lesson**: If an operation triggers a global state change that impacts the router, avoid using `context` or local navigation. Rely on the router's global `redirect` logic instead.

## 6. Supabase Premature User Population
- **Error**: App thinking user is authenticated before OTP verification.
- **Cause**: `Supabase.auth.currentUser` is sometimes populated immediately after `signUp`, even if email confirmation is required.
- **Resolution**: Redefined `isAuthenticated` to check for `currentSession != null`.
- **Lesson**: Always tie "Authenticated" status to a valid session in Supabase, especially when using OTP or email confirmation.

## 7. Router Redirect Loop on Password Reset
- **Error**: Reset Password screen unmounting immediately after clicking "Update Password".
- **Cause**: The `updatePassword` method in `AuthNotifier` set `state.isLoading = true`. The `GoRouter` redirect logic was configured to show the Splash Screen whenever `isLoading` is true. This caused the Reset Screen to be replaced by the Splash Screen mid-operation, cancelling the subsequent navigation logic.
- **Resolution**: Removed the global `isLoading` update from `updatePassword` and `verifyOtp`. Used local widget state (`_isLoading`) to show spinners instead.
- **Lesson**: Avoid setting global routing-impacting states for local operations that require the current screen to remain active.

## 8. State Loss on Router Refresh (Reset Password)
- **Error**: After verifying OTP, the Reset Password screen would sometimes reset to the "Enter Email" step.
- **Cause**: Verifying OTP changes the Auth State (logs the user in). This triggers `GoRouter` to re-evaluate routes. Even if it stays on the same route, the widget might be rebuilt, losing local `_isOtpVerified` state.
- **Resolution**: Utilized `ref.watch(authStateProvider).isRecoveringPassword` as a persistent source of truth. Since this state is held in the global provider, it survives widget rebuilds.
- **Lesson**: For multi-step flows that span across auth state changes, rely on Global Provider state rather than Ephemeral Widget state.

## 9. Large Refactoring Code Loss
- **Error**: Methods (`_handleSubmit`, etc.) disappeared from `LoginScreen.dart`.
- **Cause**: Using the LLM `replace_file_content` tool with a large range (e.g., matching `// ... existing code`) caused the tool to literally replace the code with the comment, deleting the implementation.
- **Resolution**: Immediately restored the full file content using `write_to_file`.
- **Lesson**: When modifying large files, use specific, small `search/replace` chunks or rewrite the entire file if the changes are scattered. Avoid using "lazy" matching placeholders in `replace_file_content`.

## 10. Router Redirect Preventing Error UI
- **Error**: Login failed dialogs were not appearing, despite code being logically correct.
- **Cause**: The `AuthNotifier.signIn` method set `state.isLoading = true`. The `GoRouter` global redirect logic was listening to this state and immediately redirecting the user to the Splash Screen (`/initial`). This unmounted the `LoginScreen` *before* the `catch` block could execute `showDialog`, causing the error to be swallowed silently.
- **Resolution**: Removed `isLoading = true` from `signIn` and `signUp`. Relied on local widget state (`_isLoading`) to show spinners, keeping the screen mounted during the attempt.
- **Lesson**: Never toggle a global state that triggers a Router Redirect for an operation that requires current-page error handling (like Login/Signup). Use local state for "button spinners".

## 11. Raw Exception Objects in UI
- **Error**: Dialogs showing `AuthApiException(message: ..., code: ...)` instead of clean text.
- **Cause**: Error handling blocks were falling back to `e.toString()` which prints the full object structure. Additionally, assuming strictly `e is AuthException` sometimes failed if wrappers or type mismatches occurred.
- **Resolution**: Implemented a "Dynamic Message Extractor" that attempts to read `(e as dynamic).message` before falling back to string, and also specifically parses `AuthException` fields to show clean messages like "Invalid credentials" or "Rate limit exceeded".
- **Lesson**: Always prioritize extracting the `message` property via dynamic check or specific type check before dumping `e.toString()` to the user.
## 12. OTP Redirect Loop (Conflict with Onboarding)
- **Error**: App redirected users back to OTP screen even after successful validation during signup.
- **Cause**: The `verifyingEmail` state was not being cleared after successful validation. The `GoRouter` redirect logic was hard-coded to force `/otp` if `verifyingEmail` was true, creating an infinite loop between the Onboarding and OTP screens.
- **Resolution**: Added a `clearVerifyingEmail()` call in `AuthNotifier` after successful OTP verification and ensure it's cleared when the profile is successfully loaded.
- **Lesson**: State flags used for routing must be strictly lifecycled; once a transition is "done", the flag must be explicitly reset to prevent state-based capture.

## 13. Disruptive Background Profile Refresh
- **Error**: App would flicker to the Splash/Loading screen whenever the profile was refreshed in the background (e.g., after resume upload).
- **Cause**: `_loadUserProfile` was setting `state.isLoading = true` globally. Since the Router listens to `isLoading`, it triggered a redirect to `/initial` every time data was fetched.
- **Resolution**: Modified `_loadUserProfile` to accept an optional `setGlobalLoading` parameter. Set it to `false` for background refreshes, keeping the current screen mounted.
- **Lesson**: Not all data fetches require a global loading state. Background updates should never trigger navigation-level state changes.

## 14. Blank Avatar Initials
- **Error**: The profile avatar showed "?" even when a name was entered.
- **Cause**: The `_initials` getter was only evaluating the initial state and didn't have a listener to the `name` controller's updates.
- **Resolution**: Added an `onChanged` listener to `_nameController` that calls `setState(() {})` to re-trigger the `_initials` calculation and update the header UI in real-time.
- **Lesson**: Static getters that depend on dynamic input controllers must be paired with re-rendering triggers to stay in sync.

## 15. GoRouter Race Condition after Profile Update
- **Error**: User redirected back to Onboarding/Step 1 after saving profile changes in Edit mode.
- **Cause**: The `GoRouter` redirect logic was checking `authState.isOnboarded` before the background `refreshProfile()` call finished. Since the state wasn't updated yet, it assumed the user was still un-onboarded.
- **Resolution**: Implemented immediate state synchronization. The `upsertUserProfile` method now returns the updated `UserModel`, which is passed directly to `authNotifier.updateProfile()` synchronously, triggering a valid state change *before* the router re-evaluates.
- **Lesson**: Don't rely solely on background refreshes for critical navigation-impacting state changes. Sync the state immediately from the server response.
## 16. Type Mismatch in UI Shape Constants
- **Error**: `The getter 'RoundedRectangle' isn't defined for the type '_HomeScreenState'.`
- **Cause**: Attempted to use `RoundedRectangle.circular()` which isn't a valid Material 3 constant.
- **Resolution**: Corrected to `RoundedRectangleBorder(borderRadius: BorderRadius.circular(16))`.
- **Lesson**: Flutter's `ShapeBorder` classes have specific naming conventions; verify constants before implementation.

## 17. Broken Provider Declaration during Cleanup
- **Error**: `Error: Variables must be declared using the keywords 'const', 'final', 'var' or a type name.`
- **Cause**: During linting cleanup, the `final` keyword and the `(ref)` parameter context were accidentally removed from the `interviewControllerProvider` declaration.
- **Resolution**: Restored the proper `StateNotifierProvider` syntax.
- **Lesson**: Even small cleanup tasks should be verified with a quick compile check if high-fidelity tools aren't used.

## 18. AI Provider Pivot (Gemini to OpenAI)
- **Transition**: Migrated from local `google_generative_ai` calls to Supabase Edge Functions.
- **Resolution**: 
  - Created `ai-interview-coach` Deno edge function.
  - Moved API key to server-side Secrets.
  - Refactored `AiService` to use `functions.invoke()`.
- **Benefit**: Improved security (API keys never leave the server) and lower client-side app size.

## 19. Missing Transitive Dependency (`uuid`)
- **Error**: UI error when trying to generate unique recording filenames.
- **Resolution**: Formally added `uuid: ^4.5.1` to `pubspec.yaml` instead of relying on transitive availability.

## 20. Reference Code Folder Causing Build Failures
- **Error**: `Target of URI doesn't exist: '../../core/app_export.dart'` and multiple undefined class errors.
- **Cause**: A `lib/Code References/` folder containing sample/prototype code with incomplete imports was included in the project. Flutter's analyzer tried to compile these files, causing cascading errors.
- **Resolution**: Removed the entire `lib/Code References/` folder using `Remove-Item -Recurse -Force`.
- **Lesson**: Exclude reference/prototype code from the `lib/` folder. Use a separate `docs/` or `_reference/` folder outside the compilation scope.

## 21. Unused Imports in New Interview Widgets
- **Warning**: Multiple `unused_import` warnings in newly created widget files (`ai_avatar_widget.dart`, `recording_visualizer.dart`, etc.).
- **Cause**: Imports were added proactively during scaffolding but some were not used in the final implementation.
- **Resolution**: These are non-blocking warnings. Can be cleaned up using `dart fix --apply` or manual removal.
- **Lesson**: Run `flutter analyze` after creating new files to catch unused imports early.

## 22. Deno/TypeScript Lints in Edge Function
- **Warning**: IDE reports `Cannot find module 'https://esm.sh/@supabase/supabase-js@2'` and `Cannot find name 'Deno'`.
- **Cause**: Local TypeScript analyzer doesn't recognize Deno runtime types and ESM imports.
- **Resolution**: These are **false positives** in the local IDE. The Edge Function compiles and runs correctly in the Supabase Deno runtime. Optionally, add a `// @deno-types` directive or configure a `deno.json` for local development.
- **Lesson**: Supabase Edge Functions use Deno, not Node.js. IDE linting for Deno requires separate configuration.

## 23. Android STT + Audio Recording Conflict
- **Issue**: Cannot record audio files while using native speech-to-text on Android.
- **Cause**: Android's speech recognition system takes exclusive control of the microphone, preventing the `record` package from accessing it simultaneously.
- **Resolution**: Implemented STT-only mode for interview transcription. Audio file recording is disabled during STT sessions. This is a platform limitation, not a code issue.
- **Lesson**: On Android, choose between: (a) STT for real-time transcription, OR (b) audio recording for file storage. Cannot have both simultaneously. iOS may support both but requires testing.

## 24. STT Not Working on Android Emulator
- **Issue**: Speech-to-text may fail or return no results on Android emulator.
- **Cause**: Many Android emulators don't have speech recognition services properly configured or lack the necessary Google speech services.
- **Resolution**: Test STT functionality on a **physical Android device**. The emulator may show STT as "not available" or "error".
- **Lesson**: Always test speech-related features on physical devices. Emulators often lack proper audio/speech service support.

## 25. TypeScript Variable Casing in Edge Function
- **Error**: `Property 'ai_personality' does not exist on type 'InterviewConfig'.`
- **Cause**: Backend DB uses snake_case (`ai_personality`) while the TypeScript interface for the payload was using camelCase (`aiPersonality`).
- **Resolution**: Aligned the Edge Function logic to use camelCase internally for variables and map them to snake_case when inserting into Postgres.
- **Lesson**: Maintain consistent mapping layers when bridging between snake_case databases and camelCase application code.

## 26. JavaScript Date vs. Flutter DateTime
- **Error**: `ReferenceError: DateTime is not defined` in Edge Function.
- **Cause**: Attempted to use `new DateTime().toISOString()` instead of `new Date().toISOString()` in the Deno environment.
- **Resolution**: Corrected to `new Date()`.
- **Lesson**: Don't confuse Dart's `DateTime` class with JavaScript's `Date` object when working in Edge Functions.

## 27. Dynamic Follow-up State Desync
- **Error**: Interview history showing follow-up questions but the "Total Questions" count remaining frozen.
- **Cause**: Initially, questions were fetched once and the state wasn't updated when the backend injected follow-ups.
- **Resolution**: Implemented a "Queue Injection" strategy in `InterviewController`. When `submit_answer` returns a `followUp` object, it is immediately inserted into the local `questions` list at the next index, allowing the UI to naturally progress through it.
- **Lesson**: State machines must be flexible enough to handle "just-in-time" data additions from the backend, especially for AI-driven conversational flows.

## 28. Speech-to-Text (STT) 404/CORS in Web
- **Error**: `ERR_FAILED 404 (Not Found)` when trying to play neural audio from `translate.google.com/translate_tts`.
- **Cause**: Browser security (CORS) prevents direct streaming from unofficial Google endpoints. Regional language packs were also missing locally on Windows.
- **Resolution**: 
  1. Implemented a "Native Priority" strategy that searches for Chrome's built-in `Google తెలుగు` or `Google Hindi` voices.
  2. Optimized voice initialization to wait for the browser's asynchronous voice loading.
  3. Switched to `translate.googleapis.com` with `client=gtx` for stable, CORS-friendly neural fallbacks.
- **Lesson**: Never rely on a single speech engine for web-based regional languages. Always use a hybrid of native discovery and stable API fallbacks.

## 29. UI Freeze during TTS-to-STT Transition
- **Error**: The interview screen would "freeze" or become unresponsive for several seconds after the AI finished speaking.
- **Cause**: Awaiting the `speak()` call on Web blocked the main UI thread because the native browser driver was waiting for the audio buffer to flush.
- **Resolution**: Refactored `speakAndWait` to use a non-blocking `trigger-and-listen` model on Web. Added concurrency guards to prevent multiple speech requests from clashing.
- **Lesson**: Native platform calls (especially audio) can be blocking on Web. Use completers and timeouts rather than direct awaits for smoother UI transitions. 

## 30. Premature STT Stop (Technical Patience)
- **Error**: The microphone would shut off after only 5 seconds of silence, cutting off users in the middle of thinking about technical answers.
- **Cause**: Default STT timeout settings were too aggressive for the "thinking" gaps inherent in technical interviews.
- **Resolution**: 
  1. Increased default silence timeout to 15 seconds.
  2. Implemented an automatic engine resume that keeps the session "alive" across browser engine pauses.
- **Lesson**: Interviews require higher-than-average "Technical Patience." Timeouts must be tuned for conversation, not just command-and-control.

## 31. STT Infinite Resume Loop
- **Error**: The app became stuck in a permanent "Listening" state and would never progress to processing, even after the user stopped talking.
- **Cause**: The "Auto-Resume" logic was resetting the silence timer every time the engine restarted. Since the engine restarts frequently in Chrome, the timer could never finish.
- **Resolution**: Decoupled the **Inactivity Timer** from the **Engine State**. The timer now only resets when *new text* is actually detected, ensuring that 15 seconds of true silence always triggers submission.
- **Lesson**: State persistence should distinguish between "hardware restarts" and "user activity." Restarting a low-level driver shouldn't lose high-level session progress.

## 32. Greeting Interruption Desync
- **Error**: The AI continued speaking its greeting message even after the user clicked "Start", causing overlapping audio with the first question.
- **Cause**: A-guards blocked new speech requests from interrupting active ones to avoid crashes.
- **Resolution**: Rebuilt the voice engine to be **proactively interruptible**. The controller now explicitly kills the greeting audio before starting the first question.
- **Lesson**: UI transitions and Audio transitions must be perfectly synchronized. If a user moves to a new phase, all old audio must be purged immediately.

## 34. Recording Visualizer Artifacts (Box Model Clipping)
- **Error**: Sharp/angular shadows appeared at the corners of the recording visualizer's ripple circles.
- **Cause**: Using standard `Container` scaling within a `Stack` caused "box-model clipping" when the ripples expanded. Standard scaling can sometimes struggle with anti-aliasing at higher speeds.
- **Resolution**: Re-implemented the `RecordingVisualizer` using a `CustomPainter`. Drawing directly on the canvas allowed for smooth, artifact-free anti-aliased ripples that are rendered with high precision.
- **Lesson**: For high-fidelity, premium animations, native `CustomPainter` is superior to standard widget scaling for achieving professional visual quality.

## 35. Scoring Logic Inconsistency (Pity Scores)
- **Error**: Overall interview score was 10% even when all individual categories were scored at 0%.
- **Cause**: The scoring logic in the Edge Function was filtering out 0-value scores when calculating averages and had a default fallback of 70%. Additionally, the AI wasn't strictly instructed to keep the summary score aligned with the category breakdown.
- **Resolution**: Updated the average calculation in `index.ts` to include 0s and set the fallback to 0. Enforced "Scoring Integrity" via the AI prompt to ensure the `overall_score` represents the true mathematical average.
- **Lesson**: Numerical processing layers (like averages) must explicitly handle valid zero inputs to prevent misleading analysis.

## 36. Static Audio Spectrum (Sensitivity Bug)
- **Error**: The audio spectrum bars in the recording visualizer appeared frozen or non-responsive while the user was speaking.
- **Cause**: The height calculation used a strict binary threshold that was tuned too high for most device microphones, causing most bars to remain at their base height.
- **Resolution**: Rebuilt the formula to use an "active-participation" model. All bars now react with different sensitivities based on a Gaussian (bell-curve) distribution, ensuring the visualizer "dances" dynamically even at lower volumes.
- **Lesson**: UX feedback models should prioritize "activity induction" over "binary thresholding" for a more responsive feel.

## 37. Linting & Deprecation Cleanup
- **Error**: Multiple technical warnings regarding unnecessary imports (`dart:ui`) and deprecated methods (`withOpacity`).
- **Cause**: Proactive scaffolding and evolving Flutter standards.
- **Resolution**: Cleaned up the `ActiveInterviewScreen` imports and migrated all color logic to use Flutter's modern `withValues` approach.
- **Lesson**: Regular maintenance of build-time warnings ensures the project remains compatible with future Flutter releases and maintains a clean architecture.

## 38. Bottom Navigation Selection Desync
- **Error**: Bottom navigation icons remained selected even when navigating to sub-pages (like Resume Analysis) that weren't part of the primary nav items.
- **Cause**: The `_updateCurrentIndex` logic explicitly ignored index `-1` (not found), causing the last selected primary index to persist indefinitely.
- **Resolution**: Refactored `MainShell` to allow `_currentIndex = -1`. This properly deselects all primary icons when the user is on a "utility" page (e.g., Resume Analysis, Theme Settings, or Interview Setup), providing accurate visual feedback of the actual navigation state.
- **Lesson**: Navigation UI must reflect the *actual* state of the router; if a page isn't in the nav bar, no icon should be "lit" up.

## 39. Radar Chart Artifacts & Precision
- **Error**: Jagged edges and lack of comparison data in the interview readiness chart.
- **Cause**: Initial implementation used a simple polygon-based rings and lacked a baseline/target for user comparison.
- **Resolution**: 
  1. Rebuilt the background grid using smooth concentric circles.
  2. Implemented a dual-polygon model: **Actual vs Target (85%)**.
  3. Added percentage labels and a premium glow effect to data points using `MaskFilter.blur`.
- **Lesson**: Visual data should always be contextual. A score of 70% is only meaningful when compared to a 100% boundary or an 85% "target" line.

## 40. FAB Overlap with Navigation Bar
- **Error**: The "Start Interview" button was either hidden behind the bottom navigation bar or overlapped it awkwardly on smaller devices.
- **Cause**: Hardcoded padding in the `Scaffold` wasn't accounting for the custom floating navigation panel height.
- **Resolution**: 
  1. Switched to `FloatingActionButtonLocation.endFloat`.
  2. Implemented a 80px bottom-padding within the `_AnimatedStartButton` widget itself. This ensures the button always floats exactly above the glassmorphism panel regardless of screen size.
- **Lesson**: Use standard FAB locations for platform consistency, but use internal widget-level padding to manage custom UI offsets.
## 35. Invalid Timestamp Syntax in Date Filtering
- **Error**: `Failed to fetch interviews: invalid input syntax for type timestamp with time zone: "2026-01-27T23:59:59.999.000"`
- **Cause**: The normalization logic was using a string `replace` approach that accidentally appended `.999` to an already existing `.000` fractional second part in the timestamp.
- **Resolution**: Updated the logic to split the timestamp at the `T` character and append the correct time + UTC suffix: `split('T')[0] + 'T23:59:59.999Z'`.
- **Lesson**: String replacement for timestamp normalization is risky if the source format is inconsistent. Always split and reconstruct or use a proper date-time library.

## 36. Undefined Method `warningImpact` in `HapticFeedback`
- **Error**: `The method 'warningImpact' isn't defined for the type 'HapticFeedback'.`
- **Cause**: Attempted to use a non-existent method `warningImpact()` which was likely a hallucinated or platform-specific extension.
- **Resolution**: Changed to `HapticFeedback.vibrate()` which is the standard cross-platform method for informational tactile feedback.
- **Lesson**: Verify API surface area for platform-specific services like `HapticFeedback` before implementation to avoid compilation errors.
## 37. Connectivity Plus API Breaking Changes
- **Error**: `unrelated_type_equality_checks` and `return_of_invalid_type` after updating to `connectivity_plus: ^6.1.0`.
- **Cause**: The latest version of the package now returns `List<ConnectivityResult>` instead of a single `ConnectivityResult` to support devices with multiple network interfaces (e.g., WiFi + Cellular).
- **Resolution**: Updated the `OfflineService` and `PracticeHubNotifier` to check `results.contains(ConnectivityResult.none)` or `results.isNotEmpty`.
- **Lesson**: Check breaking changes for utility packages frequently, especially those dealing with hardware/system services.

## 38. Event Queue Desync (Level Up Dialogs)
- **Error**: Gamification overlays (Level Up) appearing multiple times or after the event was already acknowledged.
- **Cause**: The `pendingEvents` list in the Riverpod state was being updated but not cleared immediately after the UI reacted.
- **Resolution**: Implemented a `clearEvents()` method in the notifier and called it using `Future.microtask` inside the `ref.listen` block on the screens.
- **Lesson**: When using state lists to trigger one-shot UI actions (like dialogs), ensure a robust "consume-and-clear" pattern is applied to prevent ghost notifications.

## 41. Database Error during Signup ("unexpected_failure")
- **Error**: `{"code":"unexpected_failure", "message":"Database error saving new user"}`
- **Cause**: The `handle_new_user_signup` Postgres trigger was strictly expecting a `name` key in the user metadata. However, the app was sending the name as `full_name`. Because the `name` column in the `public.users` table was marked as `NOT NULL`, the insert failed silently on the backend, throwing a generic "Database error" back to the Flutter client.
- **Resolution**: 
  1. Applied a SQL migration to update the trigger. It now uses `COALESCE` to check for `full_name`, `name`, or `displayName` sequentially. 
  2. Added a fallback to `split_part(email, '@', 1)` to ensure the insert always succeeds even if metadata is missing.
- **Lesson**: Backend triggers involved in auth flows must be extremely defensive with metadata keys, as different providers (Email, LinkedIn, Google) use different naming conventions.

## 42. Technical JSON Leaking to UI
- **Error**: User saw raw JSON strings like `{"code":...}` in error dialogs.
- **Cause**: Supabase sometimes returns error messages as JSON strings rather than plain text. The `_getErrorMessage` logic was only performing simple string matches on lowercase text, which failed to cleanly extract the "message" part of a JSON structure.
- **Resolution**: 
  1. Implemented a "JSON Safety-Net" in `AuthNotifier`. It now attempts to detect and parse JSON strings to extract just the human-readable `message` block.
  2. Added a secondary UI-layer parser in `LoginScreen` to catch any technical strings that slip through and transform them into polite, branded messages (e.g., "System busy" instead of "Database error").
- **Lesson**: Never assume backend error messages are human-ready. Always implement a "Sanitization Filter" that converts technical jargon or data structures into user-friendly instructions.

## 43. TTS "$1" Voice Issue
- **Error**: The TTS engine was speaking "$1" (sounds like "one dollar") instead of the intended punctuation pause.
- **Cause**: `String.replaceAll()` in Dart does not support regex capture group references like `r'$1'`. It treated it as a literal string.
- **Resolution**: Switched to `replaceAllMapped` using `(match) => '${match.group(1)} '`.
- **Lesson**: Use `replaceAllMapped` when you need to reference capture groups in Dart regex replacements.

## 44. STT 6-Second Silence Timer Desync
- **Error**: Silence detection stopped working after the STT engine auto-restarted (approx. every 90 seconds on Web).
- **Cause**: The engine restart call was passing `resetTimer: false`, meaning the inactivity timer was never re-initialized for the new session.
- **Resolution**: Updated `_onSttStatusChanged` to pass `resetTimer: true` and decoupled the inactivity timer from the hardware driver state to ensure true silence is tracked across engine cycles.
- **Lesson**: State flags for timing must be explicitly re-initialized when their underlying hardware/service dependencies restart.

## 45. "Done" Button Empty Transcript Fallback
- **Error**: Clicking "Done" occasionally showed "No speech detected" even after the user had finished speaking.
- **Cause**: `stopListening()` sometimes returns an empty string before the final buffer is flushed, especially on Web.
- **Resolution**: Added a fallback to `state.currentTranscript` (which stores the real-time stream). The error is now only thrown if *both* sources are empty.
- **Lesson**: Implement multi-source redundancy when dealing with asynchronous hardware buffers to prevent data loss.

## 46. Edge Function Information Leakage
- **Error**: Backend error messages were exposing internal database schema details like `"relation 'interview_answers' not found"`.
- **Cause**: The `catch` block was returning `error.message` directly from the Supabase/Postgres driver to the client.
- **Resolution**: Implemented a `sanitizeErrorMessage` utility in the Edge Function that maps technical database errors to generic, user-friendly messages while logging the details server-side.
- **Lesson**: Always sanitize backend errors before reaching the client; technical details are for logs, not users.

## 48. Audio File Persistence Bug
- **Error**: Audio files were being recorded locally but not appearing in the Supabase bucket or the `interview_answers` table.
- **Cause**: The `uploadBinary` call was using an incorrect bucket name, and the `audio_url` wasn't being correctly passed to the `submit_answer` Edge Function action in some cases due to an asynchronous race condition.
- **Resolution**: 
  1. Synchronized the bucket name to `audio-recordings` across both the Flutter client and Supabase Storage.
  2. Implemented a robust `await` for the audio upload before triggering the backend analysis, ensuring the `audioUrl` is always included.
  3. Added backend support in `index.ts` to explicitly handle the `audioUrl` field and save it to the database record.
- **Lesson**: Data persistence across Storage and Database requires a strictly sequential sequence of operations (Capture → Upload → Reference).

## 49. Static STT Silence Detection on Android
- **Error**: The interview would "stall" and never auto-submit on Android, even after the user stopped talking.
- **Cause**: The `soundLevelStream` wasn't being correctly listened to on Android physical devices, meaning the silence timer never reset, but also the "listening" status from the native engine didn't properly trigger the "stop" event.
- **Resolution**: 
  1. Implemented a more sensitive `soundLevel` threshold (1.5) and applied it to the global `InterviewController` listeners.
  2. Decoupled the inactivity timer from the engine state to ensure that hardware cycles don't disrupt high-level phase transitions.
- **Lesson**: Platform-specific STT sensitivity varies wildly. Always use a combination of text updates and raw sound levels to track user activity.


## 51. Notification Function Column Mismatch
- **Error**: Edge Function failing with "undefined" when accessing `total_completed` or `readiness_score`.
- **Cause**: The Postgres RPC `get_notification_eligible_users` was updated in the backend migration but the return table signature was missing the new columns required by the TypeScript interface.
- **Resolution**: Updated the `.sql` migration file to explicitly include `total_drills_completed` and `readiness_score` in the `RETURNS TABLE` definition.
- **Lesson**: Interface contracts between Edge Functions (TypeScript) and Database Functions (PL/pgSQL) must be manually synchronized. Always verify the `RETURNS` clause matches the expected JSON structure.

## 52. Notification Schema Preference Conflict
- **Error**: Ambiguity in which column controls the notification timing (`preferred_notification_time` vs `morning_notification_time`).
- **Cause**: The original schema used a generic `preferred_time` column, but the new Phase 2 requirements introduced specific `morning` and `reminder` times. The migration added new columns without deprecating the old one, leading to potential data skew.
- **Resolution**: Standardized on the specific `morning_notification_time` and planned a data migration to move existing preferences to the new column before dropping the old one.
- **Lesson**: When refactoring data models, explicit migration strategies (rename or copy-and-drop) are required to prevent "zombie columns" that confuse future developers.

## 53. Legacy Firebase Code in "Native" Edge Function
- **Error**: `ReferenceError: Deno is not defined` (or similar build errors) when deploying the daily drill notifier.
- **Cause**: The file `index.ts` contained legacy Firebase Admin SDK imports and logic, even though the architecture was decided to be Supabase Native.
- **Resolution**: Replaced the entire `index.ts` content with the clean, Deno-compatible `index_supabase_native.ts` implementation, removing all NPM dependencies related to Firebase.
- **Lesson**: When pivoting architectures, ensure the "default" entry point is updated immediately. Leaving legacy code in the main file ("just in case") leads to deployment confusion.

## 54. Negative Blur Radius Crash in Navigation Bar
- **Error**: App crashed during navigation transition with the error: `sigmaX and sigmaY must be non-negative`.
- **Cause**: Using an animation curve (like `Curves.elasticOut` or `Curves.bounceOut`) that caused the animated value for `blurRadius` to dip below zero when starting from a small or zero value.
- **Resolution**: Changed the animation curve to `Curves.easeOut` and ensured the `sigmaX` and `sigmaY` values passed to `ImageFilter.blur` are always clamped to a minimum of 0.0.
- **Lesson**: Animation curves that "overshoot" or "undershoot" their targets can produce invalid values for properties like blur radius, opacity, or scale. Always use safe curves or clamp the animated values.

## 55. Practice Hub Data Frequency & Legacy Mismatch
- **Error**: Readiness scores and interview counts were inconsistent or missing for users with older interview records.
- **Cause**: The recommendation engine was expecting a specific JSON structure for `category_scores` that didn't exist in older "legacy" interview records. These records were being skipped, leading to inaccurate readiness calculations for long-time users (e.g., users with 77+ historical records).
- **Resolution**: Implemented a "Robust Parsing Engine" in the `learning-recommendations` Edge Function. It now detects multiple potential data keys (`score`, `value`, `pt`, `rating`) and handles nested objects in both the `category_scores` and `overall_feedback` columns. Added heuristic normalization for different score ranges (0-1, 1-10, 0-100).
- **Lesson**: As data schemas evolve, backend logic must remain backwards-compatible. Use "flexible parsing" to extract value from all available historical data to maintain user continuity.
