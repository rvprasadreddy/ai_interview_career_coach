# Interview History Data Contract

## Overview
This document defines the data contract for fetching interview history with computed category scores based on question types.

---

## Database Schema Analysis

### Current Schema Structure

#### 1. **interviews** table
```sql
- id (UUID, PK)
- user_id (UUID, FK -> users.user_id)
- job_title (TEXT) -- target_role
- company (TEXT, nullable) -- target_company
- status (TEXT) -- 'in_progress', 'completed', 'abandoned'
- overall_score (FLOAT8, nullable)
- overall_feedback (JSONB, nullable)
- started_at (TIMESTAMPTZ) -- interview_date
- completed_at (TIMESTAMPTZ, nullable)
- difficulty_level (TEXT, default 'midLevel')
- ai_personality (TEXT, default 'professional')
- config (JSONB, default '{}')
- turns (JSONB, default '[]')
- created_at (TIMESTAMPTZ)
- updated_at (TIMESTAMPTZ)
```

**Note**: The schema does NOT currently have:
- `category_scores` column (needs to be added or computed dynamically)
- `audio_recording_url` column (needs to be added)
- `report_download_url` column (needs to be added)

#### 2. **interview_questions** table
```sql
- id (UUID, PK)
- interview_id (UUID, FK -> interviews.id)
- question_text (TEXT)
- question_type (TEXT) -- 'technical', 'behavioral', 'situational', 'leadership', 'problemSolving'
- order_index (INT, default 0)
- is_follow_up (BOOLEAN, default false)
- parent_question_id (UUID, nullable, FK -> interview_questions.id)
- ai_personality (TEXT, nullable)
- expected_topics (TEXT[], default '{}')
- estimated_duration_minutes (INT, default 5)
- metadata (JSONB, default '{}')
- created_at (TIMESTAMPTZ)
```

#### 3. **interview_answers** table
```sql
- id (UUID, PK)
- interview_id (UUID, FK -> interviews.id)
- question_id (UUID, FK -> interview_questions.id)
- transcript (TEXT)
- audio_url (TEXT, nullable)
- duration_seconds (INT, nullable)
- feedback (JSONB, nullable)
- score (INT, nullable) -- 0-100
- metadata (JSONB, default '{}') -- Contains: competencies, covered_topics, missed_topics
- answered_at (TIMESTAMPTZ)
- created_at (TIMESTAMPTZ)
```

---

## Data Contract

### Response Schema

```typescript
interface InterviewHistoryResponse {
  interviews: InterviewRecord[];
  pagination: {
    total: number;
    page: number;
    pageSize: number;
    hasMore: boolean;
  };
  statistics: {
    totalInterviews: number;
    averageScore: number;
    bestScore: number;
    completionRate: number;
  };
}

interface InterviewRecord {
  interview_id: string;           // UUID
  user_id: string;                // UUID
  target_role: string;            // job_title
  target_company: string | null;  // company (nullable)
  overall_score: number | null;   // 0-100 (nullable)
  category_scores: CategoryScores; // Computed from answers
  audio_recording_url: string | null; // Optional
  interview_date: string;         // ISO 8601 timestamp (started_at)
  completed_at: string | null;    // ISO 8601 timestamp (nullable)
  report_download_url: string | null; // Optional
  status: 'in_progress' | 'completed' | 'abandoned';
  difficulty_level: string;
  ai_personality: string;
  duration_minutes: number | null; // Computed: (completed_at - started_at) / 60
  total_questions: number;        // Count of questions
  answered_questions: number;     // Count of answers
  config: Record<string, any>;    // Interview configuration
}

interface CategoryScores {
  technical?: CategoryScore;
  behavioral?: CategoryScore;
  situational?: CategoryScore;
  leadership?: CategoryScore;
  problemSolving?: CategoryScore;
  [key: string]: CategoryScore | undefined; // Allow dynamic question types
}

interface CategoryScore {
  score: number;        // Average score for this category (0-100)
  count: number;        // Number of questions answered in this category
  percentage: number;   // score as percentage (for UI display)
}
```

---

## Category Score Computation Logic

### Algorithm

1. **Group answers by question type**:
   - Join `interview_answers` with `interview_questions` on `question_id`
   - Group by `question_type`

2. **Calculate average score per category**:
   - For each question type, compute `AVG(score)` where score IS NOT NULL
   - Round to nearest integer
   - Include count of answered questions

3. **Handle edge cases**:
   - If no answers exist for a category, omit it from the result
   - If a question has no answer, exclude it from calculations
   - If score is NULL, exclude from average calculation

### SQL Implementation

```sql
-- Function to compute category scores for a single interview
CREATE OR REPLACE FUNCTION compute_category_scores(p_interview_id UUID)
RETURNS JSONB AS $$
DECLARE
    v_category_scores JSONB := '{}'::jsonb;
    v_category RECORD;
BEGIN
    FOR v_category IN
        SELECT 
            q.question_type,
            ROUND(AVG(a.score))::INT as avg_score,
            COUNT(a.id) as answer_count
        FROM interview_questions q
        INNER JOIN interview_answers a ON a.question_id = q.id
        WHERE q.interview_id = p_interview_id
          AND a.score IS NOT NULL
        GROUP BY q.question_type
    LOOP
        v_category_scores := jsonb_set(
            v_category_scores,
            ARRAY[v_category.question_type],
            jsonb_build_object(
                'score', v_category.avg_score,
                'count', v_category.answer_count,
                'percentage', v_category.avg_score
            )
        );
    END LOOP;
    
    RETURN v_category_scores;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## Query Logic

### Primary Query: Fetch Interview History

```sql
-- Optimized query to fetch interview history with computed category scores
SELECT 
    i.id as interview_id,
    i.user_id,
    i.job_title as target_role,
    i.company as target_company,
    i.overall_score,
    compute_category_scores(i.id) as category_scores,
    NULL as audio_recording_url, -- To be implemented
    i.started_at as interview_date,
    i.completed_at,
    NULL as report_download_url, -- To be implemented
    i.status,
    i.difficulty_level,
    i.ai_personality,
    i.config,
    -- Computed fields
    CASE 
        WHEN i.completed_at IS NOT NULL 
        THEN EXTRACT(EPOCH FROM (i.completed_at - i.started_at)) / 60 
        ELSE NULL 
    END as duration_minutes,
    (SELECT COUNT(*) FROM interview_questions WHERE interview_id = i.id) as total_questions,
    (SELECT COUNT(*) FROM interview_answers WHERE interview_id = i.id) as answered_questions
FROM interviews i
WHERE i.user_id = $1 -- Parameter: user_id
  AND i.status IN ('completed', 'in_progress') -- Filter out abandoned
ORDER BY i.started_at DESC
LIMIT $2 OFFSET $3; -- Parameters: limit, offset
```

### Statistics Query

```sql
-- Compute user statistics
SELECT 
    COUNT(*) as total_interviews,
    ROUND(AVG(overall_score), 2) as average_score,
    MAX(overall_score) as best_score,
    ROUND(
        (COUNT(*) FILTER (WHERE status = 'completed')::FLOAT / 
         NULLIF(COUNT(*), 0)) * 100, 
        2
    ) as completion_rate
FROM interviews
WHERE user_id = $1
  AND status IN ('completed', 'in_progress');
```

---

## Edge Cases & Error Handling

### 1. Missing Optional Fields
```typescript
// Always provide default values for nullable fields
const safeInterview: InterviewRecord = {
  ...interview,
  target_company: interview.target_company ?? 'Not Specified',
  overall_score: interview.overall_score ?? 0,
  audio_recording_url: interview.audio_recording_url ?? null,
  report_download_url: interview.report_download_url ?? null,
  completed_at: interview.completed_at ?? null,
  duration_minutes: interview.duration_minutes ?? null,
};
```

### 2. Empty Category Scores
```typescript
// If no answers exist, return empty object
const safeCategoryScores: CategoryScores = 
  Object.keys(interview.category_scores || {}).length > 0 
    ? interview.category_scores 
    : {};
```

### 3. In-Progress Interviews
```typescript
// Handle interviews that are not yet completed
if (interview.status === 'in_progress') {
  // overall_score may be null
  // completed_at will be null
  // duration_minutes will be null
  // Some questions may not have answers yet
}
```

### 4. Zero Scores
```typescript
// Distinguish between "no score" (null) and "zero score" (0)
const displayScore = interview.overall_score !== null 
  ? interview.overall_score 
  : 'Not Scored';
```

---

## Performance Optimization

### Indexes Required
```sql
-- Already exist in migration
CREATE INDEX idx_interviews_user_id ON interviews(user_id);
CREATE INDEX idx_interviews_started_at ON interviews(started_at DESC);
CREATE INDEX idx_interview_answers_interview_id ON interview_answers(interview_id);
CREATE INDEX idx_interview_questions_interview_id ON interview_questions(interview_id);
```

### Caching Strategy
1. **Cache category_scores in interviews table**:
   - Add `category_scores JSONB` column to `interviews` table
   - Update via trigger when answers are inserted/updated
   - Eliminates need for real-time computation

2. **Materialized View** (Alternative):
   - Create materialized view for interview history
   - Refresh on-demand or periodically

---

## API Endpoint Design

### GET /api/interviews/history

**Query Parameters**:
- `user_id` (required): UUID of the user
- `page` (optional, default: 1): Page number
- `page_size` (optional, default: 20): Items per page
- `status` (optional): Filter by status ('completed', 'in_progress', 'all')

**Response Example**:
```json
{
  "interviews": [
    {
      "interview_id": "25378e7e-4e49-4201-8991-f68a76ba65d0",
      "user_id": "user-uuid-here",
      "target_role": "Director",
      "target_company": "Google",
      "overall_score": 10,
      "category_scores": {
        "technical": {
          "score": 10,
          "count": 1,
          "percentage": 10
        },
        "behavioral": {
          "score": 0,
          "count": 0,
          "percentage": 0
        }
      },
      "audio_recording_url": null,
      "interview_date": "2026-01-25T03:57:57.125Z",
      "completed_at": "2026-01-25T03:59:24.388Z",
      "report_download_url": null,
      "status": "completed",
      "difficulty_level": "midLevel",
      "ai_personality": "professional",
      "duration_minutes": 1.45,
      "total_questions": 5,
      "answered_questions": 1
    }
  ],
  "pagination": {
    "total": 15,
    "page": 1,
    "pageSize": 20,
    "hasMore": false
  },
  "statistics": {
    "totalInterviews": 15,
    "averageScore": 45.5,
    "bestScore": 85,
    "completionRate": 80.0
  }
}
```

---

## Flutter/Dart Model

```dart
class InterviewHistoryResponse {
  final List<InterviewRecord> interviews;
  final Pagination pagination;
  final Statistics statistics;

  InterviewHistoryResponse({
    required this.interviews,
    required this.pagination,
    required this.statistics,
  });

  factory InterviewHistoryResponse.fromJson(Map<String, dynamic> json) {
    return InterviewHistoryResponse(
      interviews: (json['interviews'] as List)
          .map((e) => InterviewRecord.fromJson(e))
          .toList(),
      pagination: Pagination.fromJson(json['pagination']),
      statistics: Statistics.fromJson(json['statistics']),
    );
  }
}

class InterviewRecord {
  final String interviewId;
  final String userId;
  final String targetRole;
  final String? targetCompany;
  final double? overallScore;
  final Map<String, CategoryScore> categoryScores;
  final String? audioRecordingUrl;
  final DateTime interviewDate;
  final DateTime? completedAt;
  final String? reportDownloadUrl;
  final String status;
  final String difficultyLevel;
  final String aiPersonality;
  final double? durationMinutes;
  final int totalQuestions;
  final int answeredQuestions;

  InterviewRecord({
    required this.interviewId,
    required this.userId,
    required this.targetRole,
    this.targetCompany,
    this.overallScore,
    required this.categoryScores,
    this.audioRecordingUrl,
    required this.interviewDate,
    this.completedAt,
    this.reportDownloadUrl,
    required this.status,
    required this.difficultyLevel,
    required this.aiPersonality,
    this.durationMinutes,
    required this.totalQuestions,
    required this.answeredQuestions,
  });

  factory InterviewRecord.fromJson(Map<String, dynamic> json) {
    return InterviewRecord(
      interviewId: json['interview_id'],
      userId: json['user_id'],
      targetRole: json['target_role'],
      targetCompany: json['target_company'],
      overallScore: json['overall_score']?.toDouble(),
      categoryScores: (json['category_scores'] as Map<String, dynamic>?)
              ?.map((key, value) => MapEntry(
                    key,
                    CategoryScore.fromJson(value),
                  )) ??
          {},
      audioRecordingUrl: json['audio_recording_url'],
      interviewDate: DateTime.parse(json['interview_date']),
      completedAt: json['completed_at'] != null
          ? DateTime.parse(json['completed_at'])
          : null,
      reportDownloadUrl: json['report_download_url'],
      status: json['status'],
      difficultyLevel: json['difficulty_level'],
      aiPersonality: json['ai_personality'],
      durationMinutes: json['duration_minutes']?.toDouble(),
      totalQuestions: json['total_questions'],
      answeredQuestions: json['answered_questions'],
    );
  }
}

class CategoryScore {
  final int score;
  final int count;
  final int percentage;

  CategoryScore({
    required this.score,
    required this.count,
    required this.percentage,
  });

  factory CategoryScore.fromJson(Map<String, dynamic> json) {
    return CategoryScore(
      score: json['score'],
      count: json['count'],
      percentage: json['percentage'],
    );
  }
}
```

---

## Implementation Checklist

- [x] Analyze existing database schema
- [x] Design category score computation logic
- [x] Create SQL functions for score computation
- [ ] Add missing columns to interviews table:
  - [ ] `category_scores JSONB`
  - [ ] `audio_recording_url TEXT`
  - [ ] `report_download_url TEXT`
- [ ] Create database trigger to auto-update category_scores
- [ ] Create Supabase Edge Function for interview history API
- [ ] Implement Flutter service to fetch interview history
- [ ] Add error handling and retry logic
- [ ] Write unit tests for score computation
- [ ] Performance testing with large datasets
- [ ] Documentation update

---

## Next Steps

1. **Apply Migration**: Run the migration script to add the `interview_answers` table and supporting functions
2. **Update Edge Function**: Modify the `ai-interview-coach` Edge Function to use the new schema
3. **Create History Endpoint**: Add a new action `get_interview_history` to the Edge Function
4. **Update Flutter App**: Implement the interview history service and UI
5. **Testing**: Verify category score computation with real data
