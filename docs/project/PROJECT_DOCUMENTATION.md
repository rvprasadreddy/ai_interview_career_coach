# InterviPrep Documentation

## Project Overview
InterviPrep is a Flutter-based mobile application designed to help users prepare for job interviews through AI-driven practice sessions and resume analysis.

## Features
- **[User Operational Guide](USER_OPERATIONAL_GUIDE.md)**: A comprehensive guide for end-users on how to use the application.

- **Multi-Channel Authentication**: 
  - **LinkedIn OAuth 2.0**: One-tap login with intelligent profile sync (new user signup + existing user login).
  - Secure **Email-OTP** for signup and passwordless entry.
  - Traditional Email/Password with robust complexity validation.
- **Legal Compliance**: Integrated links to "Terms of Service" and "Privacy Policy" directly into the authentication flow with a high-fidelity Markdown viewer.
- **Premium Onboarding**:
  - Unified **Profile Setup**: Categorized sections for Basic Info (including synced email and phone), Career Details, and Career Goals.
  - **Smart Identity**: Consolidated contact details into 'Basic Identity' for a faster first impression.
  - **Dynamic Avatar System**: Initials or upload with real-time preview.
  - **Resume Analysis**: Intelligent ATS parsing with real-time scoring and actionable insights (Strengths/Fixes).
- **AI Career Headquarters**:
  - **Resume Analysis (ATS)**: Instant scoring with AI-generated breakdown of Strengths and Improvements.
  - **Premium Profile**: High-fidelity, icon-enriched dashboard for managing personal details and career goals.
- **AI Mock Interview Simulator**: 
  - **Customizable Setup**: Configure role, company, 5 question types (Behavioral, Technical, Situational, Leadership, Problem Solving), and question count (3-10).
  - **AI-Powered Questions**: Dynamic, role-specific questions with personality-driven AI interviewer (professional, friendly, challenging).
  - **Premium Recording Experience**:
    - **CustomPainter Visualizer**: High-fidelity, artifact-free reactive waveform with Gaussian pulse effects.
    - **Dynamic Interaction**: Synchronized pulsing between AI Avatar and Candidate Microphone.
    - **"Answering..." Status**: Professional active-feedback indicator.
  - **Enhanced Reporting**: 
    - **Executive Summary**: Personality-driven "Interviewer Closure Notes" card.
    - **AI Voice Closure**: Personalized audible sign-off at the end of every session.
    - **Scoring Integrity**: Mathematically consistent overall scoring across 6 performance dimensions.
  - **Hands-Free Optimization**: Optimized 6-second silence detection for a fast-paced, realistic interview tempo.
  - **AI Avatar**: Animated interviewer with emotional transitions and color-unified listening states.
- **Practice Lab**: Categorized mini-labs with AI hints for behavioral, technical, and leadership prep.
- **Premium AI Dashboard**:
  - **Dynamic Card Layout**: Intuitive reordering of AI Practice Hub and Interview History for optimized focus.
  - **Interview Readiness 2.0**: 
    - Dual-visualization toggle (Bar vs. Radar Chart).
    - Premium **Spider/Radar Chart** with "Actual vs. Target (85%)" comparison, gradient fills, and glassmorphism styling.
    - Contextual performance labels and percentage ring indicators.
  - **Practice Hub 2.0**:
  - **Adaptive Learning Journey**: A high-fidelity vertical timeline representing 5 preparation stages (Foundation, Deep Dive, Simulation, High Stakes, Negotiation).
  - **Dynamic Stage Focus**: Smart cards that highlight current progress, category strengths/weaknesses (Strongest: Behavioral, Weakest: Technical), and upcoming rewards.
  - **Categorized Skill Hub**: Dedicated pods for Behavioral, Technical, Leadership, Situational, and Problem Solving questions with real-time performance scoring.
  - **Gamification Suite**: High-impact overlays for Level Ups, XP toasts for every action, and unlockable achievement badges.
- **Offline & Resilience Engine**:
  - **Persistent Action Queue**: Progress is saved locally via `FlutterSecureStorage` when the network is unavailable.
  - **Zero-Latency Start**: User stats are cached locally, allowing the Practice Hub to load instantly without waiting for the server.
  - **Automatic Sync**: A background connectivity listener flushes pending progress (XP, Completions) as soon as the device comes back online.
- **Improved Interview History**:
  - **Inclusive Date Filtering**: Robust timestamp normalization ensures records from the selected end-day are always included.
  - **Omni-Status Badging**: Consistent visual status indicators (Completed, In Progress, Abandoned) across all history records.
  - **Intelligent Interaction**: Protective dialog windows prevent empty state confusion for in-progress or abandoned sessions.
- **Historical Data Resilience**: Robust processing of 77+ legacy interview formats ensures a complete preparation history and accurate readiness scores.
- **High-Fidelity UI/UX**:
  - **Global Glassmorphic Sidebar**: A premium, `BackdropFilter`-powered drawer accessible from any primary screen (Home, Profile) for unified settings management.
  - **Glassmorphism NavigationBar**: Transparent, blurred bottom panel with stabilized animation transitions.
  - **Premium Animated FAB**: "Start Interview" button with dual-glow effects and pulsing animation.
  - **Dark Mode & Material 3**: Fully themed with 6 vibrant accent color options.
- **In-App Purchases (IAP)**: 
  - Cross-platform billing with `in_app_purchase`.
  - Server-side verification via Supabase Edge Functions for maximum security.
  - Dedicated `PaywallScreen` with localized pricing and subscription management.
- **Google Ads Integration**:
  - Balanced monetization via Google Mobile Ads.
  - Rewarded ad flows for unlocking specific features (e.g., Daily Drill reveals).
  - Privacy-first UMP Consent integration.
  - Automatic removal for Pro/Elite users.


## Commercial Model

InterviPrep operates on a tiered subscription model to balance accessibility with premium AI-driven value.

### 💎 Subscription Tiers

1.  **Freemium ($0/mo)**: 
    - Entry-level access for casual users.
    - Supported by non-intrusive Google Ads.
    - 1 full mock interview/week, 3 questions/session, daily drills (limited), basic ATS score.
2.  **Pro (₹899/mo)**: 
    - Full-featured, ad-free access for active job seekers.
    - Unlimited interviews, all AI personalities, detailed resume suggestions, full Practice Hub access, and audio replays.
3.  **Elite (₹1799/mo)**: 
    - Premium coaching experience with zero limits.
    - Everything in Pro + Big Tech/Finance company-specific packs and high-stakes content.

---

## Application Workflows

### Authentication Flow
1. **Email/Password & OTP**: Standard authentication with session management.
2. **LinkedIn OAuth 2.0**:
   - User clicks "Continue with LinkedIn"
   - OAuth redirect to LinkedIn authorization
   - User approves application access
   - LinkedIn redirects to: `com.antigravity.aiinterviewcoach://login-callback`
   - `LinkedInCallbackScreen` processes callback
   - Edge Function (`linkedin-profile-sync`) intelligently handles:
     - **New User**: Creates profile with LinkedIn data → Routes to onboarding
     - **Existing User**: Updates profile with latest LinkedIn data → Routes to home
   - Profile data synced:
     - Profile image, headline, current position
     - Skills (top 10, merged with existing)
     - Location and city
3. **State Update**: `AuthNotifier` detects session change and updates `authStateProvider`.
4. **Navigation**: `GoRouter` navigates to appropriate screen based on user status.

### Onboarding Flow
1. **Unified Profile Setup**: New users complete a single-page form with 15+ attributes. 
2. **Identity Optimization**: Email and Phone are consolidated into the initial 'Basic Identity' block. Email is synced automatically from the auth session.
3. **Invisible Location Sync**: System automatically detects the user's city and country in the background, pre-filling the 'Preferred Location' field while maintaining a hidden 'Current City' record.
4. **Resume Upload**: Users upload a PDF/Doc. The app triggers an immediate background call to a Supabase Edge Function.
5. **AI ATS Scoring**: Users receive instant feedback including a score and qualitative analysis.
6. **Completion**: Once "Complete Setup" is clicked, onboarding is finalized and the user is routed to the Home dashboard.

### AI Interview Workflow
1. **Setup Phase**: User configures interview on the Premium Setup Screen:
   - Targets Role and Optional Company.
   - Sets **Difficulty Level** (Junior to Lead).
   - Chooses **AI Personality** (Professional, Friendly, Challenging).
   - Configures **Voice Settings** (Speech Rate, Accents).
   - Selects Question Count (3-10) and Duration (15-60 min).

2. **Synchronized Generation & Progress**: 
   - **Initialization**: Flutter triggers `start_interview` action. The backend creates the session and initial questions atomically in the database.
   - **Live Interview**: Native STT transcribes user speech in real-time while high-fidelity audio is captured for persistence.
   - **Synchronized Submission**: Flutter triggers `submit_answer`. The backend persists the answer, analyzes the response, and injects a dynamic follow-up into the question queue if necessary.
   - **Full Audio Persistence**: Answer recordings are uploaded to Supabase Storage and linked to the interview record for session replay.
   - **Report Generation**: `complete_interview` action finalizes the session and produces a comprehensive hiring evaluation.

3. **Interview State Machine (9 Phases)**:
   - **`setup`**: Configuration and setup parameters.
   - **`generating`**: Backend is creating session and initial questions.
   - **`ready`**: AI interviewer greetings; ready for first question.
   - **`listening`**: AI is asking the current question (TTS active).
   - **`recording`**: User is answering (Native STT transcribes in real-time).
   - **`processing`**: Backend is analyzing the answer and evaluating next steps (follow-ups).
   - **`feedback`**: AI displays turn-based evaluation.
   - **`transitioning`**: AI generates conversation-style transition to the next question.
   - **`completed`**: Session closed; final report generated and displayed.

4. **Active UI Components**:
   - **AI Avatar**: animated expressions (happy, curious, encouraging).
   - **Reactive Waveform**: CustomPainter-based visualization reflecting voice amplitude.
   - **Transcript History**: Persistent bottom sheet for full conversation review.
   - **Session Timer**: Real-time tracking of interview duration.

5. **Final Evaluation**: 
   - `generate_final_report` creates a comprehensive evaluative summary.
   - Includes overall score, competency breakdown, and personalized tips.

## Development Best Practices

### Tech Stack
- **Framework**: Flutter
- **State Management**: Riverpod (`StateNotifierProvider`, `Provider`)
- **Backend**: Supabase (Auth, PostgreSQL, Storage)
- **Navigation**: GoRouter
- **Styling**: Custom Theme (`lib/theme/app_theme.dart`)

### File Structure
- `lib/config/`: App configuration, routes, and Supabase initialization.
- `lib/core/`: Reusable services, models, and UI components.
- `lib/features/`: Feature-based modular structure.
  - [Interview Feature](file:///c:/flutter_apps/antigravity/ai_interview_coach/lib/features/interview/README.md): AI Interview Simulator documentation.
  - [Auth Feature](file:///c:/flutter_apps/antigravity/ai_interview_coach/lib/features/auth/README.md): Authentication and OAuth documentation.
  - [Onboarding Feature](file:///c:/flutter_apps/antigravity/ai_interview_coach/lib/features/onboarding/README.md): User setup and resume analysis documentation.
- `lib/theme/`: Design system tokens and theme data.

### Interview History & Statistics Workflow
1. **Fetch Initiation**: Flutter triggers `get_interview_history` via Edge Function.
2. **Aggregated Results**: The backend queries individual interview scores and computes category-based performance (Technical, Communication, etc.) group by question types.
3. **Advanced Statistics**: The system calculates "Interview Readiness" scores (total interviews, average score, best performance) to drive the Dashboard's Radar Chart.
4. **Data Contract**: Edge Function returns a paginated list of `InterviewRecord` objects, including `category_scores` JSONB for high-fidelity visualization.

## Synchronized Backend Architecture (Supabase Edge Functions)
The application follows a **Server-Side Intelligence** model where all AI decision-making and data synchronization happens within Supabase Edge Functions.

### Production Hardening & Error Sanitization
To ensure a secure and resilient user experience, the system implements the following:
- **Information Sanitization**: Backend technical errors (SQL/API) are intercepted and translated into human-readable messages to prevent internal schema leakage.
- **Strict Payload Validation**: All incoming requests are validated for type correctness, UUID formatting, and permissible ranges (e.g., preparation time: 0-300s).
- **STT Resilience Engine**: Synchronized state management ensures that silence detection and auto-submission persist even when the STT engine auto-cycles on Web browsers.
- **Data Robustness Engine**: Implemented flexible JSON parsing in the backend to handle legacy and modern interview data formats, ensuring accurate readiness scores for long-term users.
- **Redundant Transcript Retrieval**: Implemented fallback logic that merges real-time transcript streams with native results, eliminating the "No speech detected" bug during rapid interactions.

| Action | Purpose |
|--------|---------|
| `start_interview` | Atomic session creation and initial question generation. |
| `submit_answer` | Answer persistence, AI analysis, and dynamic follow-up generation. |
| `complete_interview` | Final report generation and session lifecycle closure. |
| `generate_interviewer_response` | Personality-driven transition utility between turns. |
| `get_interview_history` | Fetches historical sessions with computed category scores and statistics. |
| `get_interview_details` | Retrieves full Q&A transcript and feedback for a specific session. |
| `practice-hub` (New) | Manages practice sessions, AI feedback, and gamification logic independently. |

## Required Configurations

### Environment Variables
Create a `.env` file in the project root with your Supabase credentials:
```
SUPABASE_URL=your_supabase_project_url
SUPABASE_ANON_KEY=your_supabase_anon_key
```

### Android Settings
- **Cleartext Traffic**: Enabled `android:usesCleartextTraffic="true"` in `AndroidManifest.xml` to allow OAuth redirects to local/dev URLs.

### Supabase Dashboard (IMPORTANT)
To ensure the LinkedIn authentication flow returns to the app:
1. **Redirect URL**: Add `com.antigravity.aiinterviewcoach://login-callback` to the Redirect URLs in the Supabase Auth settings.
2. **Secrets Configuration**: Set `OPENAI_API_KEY` using the Supabase CLI: `supabase secrets set OPENAI_API_KEY=xxx`.
3. **Database Setup**: Run `supabase_migration.sql` to initialize `interviews`, `interview_questions`, and `practice_questions` tables.
4. **Storage Buckets**: Create buckets named `resumes`, `profile_images`, and `audio-recordings` in your Supabase Storage.
5. **Edge Functions**: Deploy using `supabase functions deploy ai-interview-coach`.

### Future Enhancements
- Mock interview video analysis.
- Company-specific interview question packs.
- Real-time feedback using LLM agents.
- Offline mode for practice questions.
