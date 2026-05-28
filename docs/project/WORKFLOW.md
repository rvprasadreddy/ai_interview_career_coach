# Development Workflow & Features

## 1. Secure Password Reset Flow
- **Objective**: Implement a secure, user-friendly password reset mechanism without leaving the app.
- **Implementation**:
  - **Identify Phase**: User enters email. System sends OTP via Supabase.
  - **Verify Phase**: User enters 6-digit OTP. App automatically verifies it.
  - **Reset Phase**: Upon verification, UI unlocks "New Password" fields. Using `authState.isRecoveringPassword` to persist this state.
  - **Completion**: User updates password, receives success feedback, and is routed to Login.

## 2. Inline Password Validation
- **Objective**: Provide immediate feedback on password complexity rules.
- **Rules**: Minimum 6 chars, 1 Uppercase, 1 Lowercase, 1 Digit.
- **Implementation**:
  - Added strict logic to `Validators.validatePassword`.
  - Implemented real-time `onChanged` listeners in `SignUpScreen` and `ResetPasswordScreen`.
  - Errors update dynamically as the user types, blocking submission until valid.

## 3. Navigation Stability & State Management
- **Enhancement**: Decoupled UI loading states from Global Auth loading states.
- **Reason**: Global loading states were triggering the Router to redirect to the Splash screen, killing active forms (like the Reset Password screen).
- **Fix**: Localized `isLoading` state in screens for operations that should not trigger a full app route refresh.

## 4. Unified Profile Onboarding Workflow
- **Objective**: Create a low-friction, high-engagement first impression.
- **Workflow**:
  - **Categorized Sections**: Instead of tabs, all fields are on one scrollable screen with visual headers (Basic, Career, Goals).
  - **Real-time Identity**: The user's avatar updates instantly with initials or uploaded image, creating an immediate sense of ownership.
  - **Dynamic Progression**: Validation occurs locally; the "Continue" button remains the primary anchor until completion.
  - **Supabase Sync**: All 15+ user attributes are synced in a single `upsert` operation before moving to the resume step.

## 5. Real-time AI Resume Analysis (ATS)
- **Objective**: Provide immediate professional value after resume upload.
- **Workflow**:
  - **Upload Verification**: Checks for file presence and applies a 2-hour rate limit to prevent API abuse.
  - **Background Computation**: Triggers Supabase Edge Function immediately while showing the user a progress indicator.
  - **Actionable Insights**: Once complete, the UI instantly flips to show a numerical ATS score, an AI-generated summary, and specific "Strengths/Fixes" chips.
  - **Persistent Feedback**: This analysis is cached in the `users` table and remains visible on the Dashboard and Profile for long-term reference.

## 6. Premium Profile Experience & Data Consistency
- **Objective**: Elevate the user's secondary interaction point (Profile) to a premium dashboard.
- **Workflow**:
  - **Thematic Ionization**: Every data field is paired with a relevant icon and sub-label, improving scannability.
  - **Resume Integrity**: The system captures and displays the original filename of uploaded resumes, using dynamic icon logic to distinguish between PDF and Word documents.
  - **Synchronous Navigation**: Profile updates trigger immediate local state refreshes (ahead of background syncs) to ensure the Router transitions the user seamlessly between Edit and Read-only views without redirection loops.
## 7. AI Mock Interview Engine
- **Objective**: Provide a safe, realistic environment for users to practice their speaking and communication skills with an AI-powered interviewer.
- **Workflow**:
  ### 7.1 Setup Phase
  - User selects target role, company, question types (Behavioral, Technical, Situational, Leadership, Problem Solving).
  - User configures question count (3-10) and optionally pastes job description.
  - User skills are auto-populated from profile.
  
  ### 7.2 Synchronized Question Generation
  - AI generates role-specific initial questions via the `start_interview` Edge Function action.
  - The backend creates the interview session and persists questions to the `interview_questions` table in a single atomic operation.
  - Questions include AI personality tags for curated conversation flow and are returned to the Flutter UI immediately.
  
  ### 7.3 Native STT & Full Audio Recording
  - User speaks their answer while the device's native Speech-to-Text engine provides real-time transcription.
  - A live transcript preview and a reactive waveform (based on STT sound levels) provide immediate visual feedback.
  - **New: Integrated Audio Recording**: The system now simultaneously captures high-quality audio (`.m4a`) using the `InterviewAudioRecorder` service. This audio is synced to Supabase Storage and linked to the interview answer record.
  
  ### 7.4 Synchronized Answer Submission & Follow-up
  - The transcript is sent to the `submit_answer` Edge Function action.
  - The backend persists the answer to `interview_answers`, performs AI analysis (score, feedback), and **dynamically determines if a follow-up is required**.
  - If a follow-up is needed, it is generated, saved to the database, and returned in the same response, enabling immediate injection into the interview flow.
  
  ### 7.5 Final Report & Session Closure
  - Upon completing all questions, the `complete_interview` action is triggered.
  - The AI generates a comprehensive evaluation across multiple competencies and provides a final hiring recommendation.
  - The interview status is marked as `completed` and the final report is persisted to the `interviews` table.

## 8. Synchronized Backend Architecture (Supabase Edge Functions)
- **Objective**: Centralize interview logic, protect API keys, and ensure data consistency between frontend and backend.
- **Production Edge Function Actions**:
  | Action | Purpose |
  |--------|---------|
  | `start_interview` | Atomic session creation and initial question generation |
  | `submit_answer` | Answer persistence, AI analysis, and dynamic follow-up generation |
  | `complete_interview` | Final report generation and session lifecycle closure |
  | `get_interview_details` | Retrieves full Q&A transcript and feedback for a specific session |
| `practice-hub` (New) | Manages practice sessions, AI feedback, and gamification logic independently with support for legacy and current data formats. |

- **Workflow**:
  - **Trigger**: Flutter invokes `supabase.functions.invoke('ai-interview-coach', body: {action, payload})`.
  - **Single Source of Truth**: The backend manages the state of the interview (questions, answers, status), ensuring that the interview session can be resumed or reviewed from any device.
  - **AI Personality System**: Three interviewer personalities (professional, friendly, challenging) drive the tone and follow-up style of the AI.
  - **Real-time Synchronization**: The Flutter UI updates its local `InterviewState` based on structured JSON responses from the backend, maintaining perfect sync.
  - **Error Sanitization**: Backend prevents technical leakage by mapping internal exceptions to user-friendly messages during the synchronization process.

## 9. Interview State Machine
- **Objective**: Manage complex interview flow with predictable state transitions.
- **Phases**: `setup` → `generating` → `ready` → `listening` → `recording` → `processing` → `feedback` → `transitioning` → `completed`
- **Hands-Free Progression**: 
  - The `InterviewController` orchestrates transitions automatically when `autoProgressEnabled` is true.
  - Phase 4 (`listening`) waits for TTS completion before triggering Phase 5 (`recording`).
  - Phase 7 (`feedback`) automatically triggers Phase 8 (`transitioning`) after a 3-second display delay.
  - This creates a continuous, conversational loop until the interview is completed.
- **Implementation**: `InterviewController` (StateNotifier) manages the lifecycle, handling data persistence and real-time backend synchronization.


## 10. Practice Lab Workflow
- **Objective**: Provide categorized practice questions with AI-powered hints for targeted skill development.
- **Categories**:
  - **Behavioral**: STAR method practice with common behavioral questions
  - **Technical**: Role-specific technical questions with concept explanations
  - **Leadership**: Management and team leadership scenarios
  - **Situational**: Real-world problem-solving scenarios
  - **Problem Solving**: Analytical and critical thinking challenges
- **Workflow**:
  - **Browse**: User selects category from Practice Lab home
  - **Expand**: Taps question card to reveal full question and hint button
  - **AI Hints**: Requests AI-generated hints via Edge Function
  - **Practice**: User formulates answer (no recording in practice mode)
  - **Sample Answers**: AI provides example STAR-formatted responses
- **Implementation**: Questions stored in `practice_questions` table with category tags.
- .

## 11. Theme Customization Workflow
- **Objective**: Provide users with personalized visual experience.
- **Options**:
  - **Theme Mode**: Light, Dark, or System (follows device settings)
  - **Accent Colors**: 6 vibrant options (Blue, Purple, Green, Orange, Pink, Teal)
- **Workflow**:
  - User navigates to Profile → Theme Settings
  - Selects theme mode and accent color
  - Changes apply instantly with smooth transitions
  - Preferences saved to `users.theme_preference` JSONB column
- **Implementation**: `ThemeProvider` (StateNotifier) manages theme state with persistence

## 12. Interview History & Analytics Workflow
- **Objective**: Provide a data-rich environment for tracking progress and identifying skill gaps over time.
- **Synchronized Architecture**:
  - **Fetch**: On entry, the screen invokes the `get_interview_history` Edge Function action.
  - **Compute**: The backend dynamically aggregates scores into category buckets (Technical, Behavioral, etc.) based on question types.
  - **Stats**: Real-time user statistics (Total, Average, Best) are computed server-side to ensure accuracy.
  - **Persistence**: Results are paginated and cached in the `InterviewHistoryState` for a responsive scrolling experience.
- **Workflow**:
  - User navigates to Home → Interview History card.
  - Screen fetches enriched records from Edge Function.
  - Cards display in collapsed state by default (job title, company, date, overall score).
  - User taps card to expand and view category scores or delete the record.
- **UX States**:
  - **Loading**: Shimmer skeleton cards while fetching data.
  - **Empty**: Encouraging message with CTA to start first interview.
  - **Error**: Error message with retry button.
- **Implementation**: `InterviewHistoryNotifier` maps Edge Function JSON responses directly to `InterviewHistoryItem` models. Enhanced with a robust parsing engine in the backend to handle legacy and current data formats across 77+ historical interview types.
 
- .
 
## 13. Legal Document Awareness & Transparency
- **Objective**: Ensure regulatory compliance and user trust by providing easy access to legal documents.
- **Workflow**:
  - **Discovery**: Links to "Terms of Service" and "Privacy Policy" are embedded directly in the Login screen using `Text.rich` and `TapGestureRecognizer`.
  - **Dynamic Viewing**: Tapping a link opens the `LegalViewerScreen`, which dynamically loads and renders Markdown content from local assets (`assets/legal/`).
  - **Styling**: The viewer uses high-fidelity typography and follows the Anti-Gravity 2.0 theme.
- **Implementation**:
  - `LegalViewerScreen`: A reusable Markdown renderer widget.
  - `assets/legal/`: Storage for `privacy_policy.md` and `terms_of_service.md`.
- **State Management**:
  - Use Riverpod `StateNotifierProvider` for complex state
  - Keep global state minimal (auth, theme, interview)
  - Use local widget state for UI-only concerns (loading spinners, form validation)
- **Navigation**:
  - Never set global `isLoading` for local operations
  - Use router redirect logic for auth-based navigation
  - Maintain persistent router instance (singleton pattern)
- **Error Handling**:
  - Extract user-friendly messages from exceptions
  - Use `AlertDialog` for critical errors, `SnackBar` for info
  - Never show raw exception objects to users
- **Data Persistence**:
  - Sync critical state changes immediately (don't rely on background refresh)
  - Use optimistic updates for better UX
  - Maintain data integrity with proper RLS policies

## 14. Testing Workflow
- **Pre-commit Checks**:
  1. Run `flutter analyze` to catch warnings and errors
  2. Run `dart fix --dry-run` to preview auto-fixes
  3. Test critical user flows (signup, login, interview setup)
- **Build Verification**:
  1. Debug build: `flutter build apk --debug`
  2. Release build: `flutter build apk --release`
  3. Verify APK size and performance
- **Device Testing**:
  1. Test on physical device for audio recording
  2. Verify OAuth redirects work correctly
  3. Test offline behavior and error states


## 15. Backend Hardening & Data Validation Workflow
- **Objective**: Ensure high reliability and security of Edge Function operations.
- **Workflow**:
  - **Validation First**: Every request is sanitized using custom `ValidationError` and `NotFoundError` classes to ensure data integrity.
  - **Sanitized Errors**: Backend filters technical database/API exceptions into user-friendly strings before reaching the UI.
  - **Robust State Tracking**: Behavioral metrics like preparation time (`preparation_seconds`) are explicitly validated (0-300s range) and persisted.
- **Implementation**: Centralized validation and sanitization utility functions in the `ai-interview-coach` Edge Function.

## 16. Daily Drill Notification Workflow (Supabase Native)
- **Objective**: Re-engage users daily with personalized drill notifications without external dependencies (Firebase).
- **Architecture**:
  - **Cron Job**: Scheduled pg_cron task triggers Edge Function at 8:00 AM IST.
  - **Edge Function**: `daily-drill-notifier` calculates eligible users and creates notification records.
  - **Realtime**: Supabase Realtime broadcasts new inserts on the `notifications` table to the Flutter app.
- **Workflow**:
  - **Trigger**: Cron job invokes `daily-drill-notifier` with `action: 'send_morning_notification'`.
  - **Eligibility**: Function queries `get_notification_eligible_users` to find users with `notification_enabled = true`.
  - **Creation**: Inserts a record into `notifications` table with personalization (streak, user name).
  - **Delivery**: 
    - **Foreground**: App listening to `postgres_changes` displays in-app banner.
    - **Background/Terminated**: (Future Phase) Relies on system-level background fetch or OS-level push if integrated later.
  - **Interaction**: User taps notification -> Deep link navigates to `DailyDrillScreen`.
- **Implementation**: 
  - **Database**: `notifications` table with RLS and `get_notification_eligible_users` RPC.
  - **Edge Function**: Pure Supabase types (no Firebase Admin SDK).
  - **Flutter**: `NotificationService` singleton listening to Supabase Realtime stream.

