# Identification of Issues & Planned Enhancements

This log records identified technical issues, debt, and planned feature enhancements discovered during the codebase analysis.

---

## 1. Technical Issues & Bugs

### [HIGH] Path Consistency: Practice vs Learning
- **Symptom**: Practice Hub implementation is split between `lib/features/learning` (screens) and `lib/features/practice` (doesn't exist but referenced in docs).
- **Location**: `lib/features/learning/screens/practice_hub_screen.dart`
- **Impact**: Confusing for developers; documentation references `practice` feature but folder is named `learning`.
- **Recommendation**: Standardize on one name and rename directory/references accordingly.

### [MEDIUM] Empty State in Statistics
- **Symptom**: `userStatisticsProvider` might return empty/null if no interviews are present, but UI components might not all handle the "Zero Data" state gracefully with shimmers or placeholders.
- **Location**: `lib/features/analytics/providers/statistics_provider.dart` (Check implementations).

### [LOW] TTS Character Limits
- **Symptom**: Highly detailed AI transition messages might exceed the standard 4000-character limit of some TTS engines if not truncated.
- **Location**: `EnhancedTtsService.speak()`

---

## 2. Technical Debt

### Hardcoded Constants
- **Observation**: Several timeouts and thresholds (6s silence, 30s prep) are hardcoded in `InterviewController` instead of being injected via an `AppConfig` or `InterviewSettings` class.
- **Impact**: Harder to A/B test or change global defaults.

### Inline Logic in Edge Function
- **Observation**: `ai-interview-coach/index.ts` is getting very large (2500+ lines).
- **Impact**: Maintenance nightmare.
- **Recommendation**: Refactor into smaller modules (e.g., `handlers/`, `utils/`, `prompts/`).

---

## 3. Top Planned Enhancements (Post-Analysis)

### 1. Interruption Support
- **Description**: Stop AI speech immediately if user starts talking.
- **Complexity**: High (Requires real-time audio energy detection alongside TTS playback).

### 2. Multi-Candidate Benchmarking
- **Description**: Add a "Percentile" ranking to the final report (e.g., "You scored better than 85% of candidates for this role").
- **Complexity**: Medium (Requires a background aggregation function in Supabase).

### 3. Interview Video Archiving
- **Description**: Allow users to save and download their interview recordings for later review.
- **Complexity**: High (Storage costs and video processing latency).

### 4. Interactive Code Editor
- **Description**: For Technical/Coding questions, provide an embedded Monaco editor instead of just speech.
- **Complexity**: High (Requires web-embedding support for mobile).
