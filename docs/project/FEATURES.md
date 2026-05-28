# Codebase Feature Documentation

This document serves as the single source of truth for all implemented features in the InterviPrep application. It details the logic, scenarios covered, and missed scenarios for future development.

---

## 1. AI Interview Simulator (Live Interview)
The core engine of the application, providing a realistic, hands-free interview experience.

### Logic & Implementation
- **State Machine**: Managed by `InterviewController` via 11 distinct phases (initialization, intro, preparation, speaking, listening, processing, reporting, etc.).
- **Synchronized Backend**: All critical logic (question generation, answer submission, follow-ups) is handled by Supabase Edge Functions to ensure data integrity and prevent technical leakage.
- **Hands-Free Automation**: `autoProgressEnabled` (default `true`) allows the interview to flow automatically from AI speaking to user listening and back.
- **Silence Detection**: Uses a persistent timer to auto-submit answers after a configurable period of silence (default 6s).
- **STT/TTS Orchestration**: Native Speech-to-Text (STT) for transcription and Text-to-Speech (TTS) for interviewer voice delivery.
- **Micro-Animations**: Real-time voice visualizers (Sine wave) and AI status indicators (Thinking, Speaking, Listening).

### Covered Scenarios
- **Preparation Phase**: Integrated "Thinking Time" (default: 15s) where users view the question and plan their response before the microphone activates.
- **Selective Scrolling**: Optimized layout where only the active message area/transcript scrolls, while core interaction elements (Avatar, Mic) remain fixed.
- **Focused Response Mode**: Hidden transcript view during the answering phase to prioritize voice interaction and reduce cognitive load.
- **Warm-up Questions**: Non-scored introductory questions to gather candidate background for better follow-up context.
- **Adaptive Difficulty**: Difficulty level adjusts dynamically mid-interview based on the scores of the last 3 answers.
- **Dynamic Follow-ups**: AI generates targeted follow-up questions based on specific keywords or vague statements in the candidate's answer.
- **Robust Completion Flow**: Atomic transition logic ensuring that manual "Finish & Report" actions immediately halt all background AI processing and TTS.

### Missed Scenarios / Future Work
- **Group/Panel Interviews**: Multiple AI personalities with different behaviors in one session.
- **Interruption Handling**: Allowing the user to interrupt the AI while it's speaking.
- **Video Recording/Analysis**: Real-time facial expression and eye contact analysis.

---

## 2. AI-Powered Analytics & Feedback
Provides deep insights into interview performance beyond simple scoring.

### Logic & Implementation
- **Competency Mapping**: Answers are analyzed across 5-6 dimensions (Relevance, Structure, Technical Accuracy, Depth, Communication).
- **Detailed Feedback**: GPT-4 generated summaries, specific strengths, and actionable improvements for every answer.
- **Comprehensive Final Report**: Aggregates all answers into a hiring recommendation (Strong Hire, Hire, Maybe, No Hire) with a detailed rationale and blowout performance chart.

### Covered Scenarios
- **Skipped Questions**: Graceful handling of skips with 0 points but encouraging feedback.
- **ATS Background Sync**: Performance trends are calculated over the last 10 interviews to show growth.

### Missed Scenarios / Future Work
- **Peer Comparison**: Benchmarking performance against other users for the same role.
- **Voice Sentiment Analysis**: Detecting confidence or anxiety levels via audio frequency analysis.

---

## 3. Practice Hub (Personalized Roadmap)
A learning dashboard that evolves with the user's interview history.

### Logic & Implementation
- **Readiness Score**: A proprietary 0-100 score calculating "Interview Readiness" based on consistency, depth, and technical accuracy.
- **Automated Roadmap**: Edge Function analyzes historical gaps and recommends specific content (Videos, FAQs, Blogs).
- **Difficulty Filtering**: Users can filter learning resources by foundation, intermediate, advanced, and expert levels.

### Covered Scenarios
- **Empty State**: Encourages users to take their first interview if no data is available.
- **Format-Specific Tabs**: Organized content types for structured learning.

### Missed Scenarios / Future Work
- **Guided Practice Modules**: Interactive Q&A for specific technical topics (e.g., "React Hooks Deep Dive").
- **Flashcard System**: Spaced repetition for common behavioral questions.

---

## 4. Smart Onboarding & Profile Setup
Streamlined professional identity creation using external integrations.

### Logic & Implementation
- **LinkedIn OAuth 2.0 Integration**: Syncs full name, headline, avatar, skills, and current position automatically.
- **Smart Location Detection**: IP-based location auto-fill with manual override.
- **Resume Analysis (ATS)**: Uploads PDF/Docx to Supabase Storage, calculates an ATS match score, and extracts core skills using AI.

### Covered Scenarios
- **Multi-step Form**: 3-step intuitive process (Personal -> Experience -> Resume).
- **Edit Mode**: Unified UI for both first-time onboarding and profile updates.

### Missed Scenarios / Future Work
- **GitHub/Portfolio Sync**: Automatically extracting technical projects from repositories.
- **Job Description Scraper**: Direct URL pasting to extract interview requirements instantly.

---

## 5. Security & Production Hardening
Engineered for reliability and data safety.

### Logic & Implementation
- **Error Sanitization**: Backend errors (DB schema, API keys) are masked with user-friendly messages before reaching the client.
- **Input Validation**: All Edge Function payloads are validated for required fields, lengths, and ranges (e.g., `transcript` min 10 chars).
- **Atomic Registration**: Database triggers ensure user profiles are created immediately upon auth signup without race conditions.

### Covered Scenarios
- **STT Engine Desync**: Handlers to reset silence timers when the STT engine restarts mid-listening.
- **TTS "$1" Cleanup**: Regex sanitization to prevent TTS engines from speaking regex capture group references.

---

## 6. Audio Infrastructure & Cloud Storage
Reliable audio handling for archival and playback.

### Logic & Implementation
- **High-Fidelity Recording**: `InterviewAudioRecorder` captures user answers in `.m4a` format (adaptive bitrate).
- **Multipart Upload**: `StorageService` handles uploading audio files to Supabase buckets with user-isolated paths (`interviews/{user_id}/{interview_id}/...`).
- **Memory Management**: Automatic cleanup of local temporary audio files after successful cloud sync.

### Covered Scenarios
- **Background Uploads**: Uploads are initiated immediately after answer submission to prevent perceived latency.
- **WAV/M4A Support**: Compatible with both iOS and Android native formats.

### Missed Scenarios / Future Work
- **Real-time Streaming**: Moving from post-recording upload to real-time Opus streaming for lower latency analysis.
- **Audio Post-processing**: Noise cancellation and volume normalization before upload.
