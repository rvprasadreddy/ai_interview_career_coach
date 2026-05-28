# Daily Interview Prep / Daily Drill System Architecture

## 🎯 System Overview


The Daily Drill system delivers exactly ONE curated interview question per user per day to build consistent interview preparation habits while remaining cost-effective at scale.

### **How it Works (High-Level)**
1.  **Selection**: A Postgres RPC (`get_or_assign_daily_question`) intelligently picks a question based on your role, readiness score, and weak categories.
2.  **Engagement**: Users interact with the question in the Flutter app, rating their confidence and adding notes.
3.  **Analytics**: Database triggers (`on_drill_status_changed`) automatically update streaks, category-wise performance, and the overarching **Readiness Score**.
4.  **Feedback**: The system identifies "Weak Categories" when completion rates drop below 60% over 3+ drills, feeding back into the Selection step.

---

## 📐 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DAILY DRILL SYSTEM                           │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    GLOBAL QUESTION REPOSITORY                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  daily_drill_questions                                         │ │
│  │  - Pre-generated interview questions (shared across users)     │ │
│  │  - Categorized by role, category, difficulty                   │ │
│  │  - Includes ideal answers and evaluation criteria              │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    DAILY ASSIGNMENT ENGINE                           │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Supabase Edge Function: daily-drill-engine                    │ │
│  │  Actions:                                                       │ │
│  │  - assign_daily_question (runs via cron at midnight UTC)       │ │
│  │  - get_today_question (fetch user's daily question)            │ │
│  │  - submit_drill_response (record user interaction)             │ │
│  │  - get_drill_stats (fetch streaks, progress)                   │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    USER ASSIGNMENT TRACKING                          │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  user_daily_drills                                             │ │
│  │  - Maps user_id → date → question_id                           │ │
│  │  - Tracks completion status and confidence                     │ │
│  │  - Records timestamps for analytics                            │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    USER PROGRESS & ANALYTICS                         │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  user_drill_progress                                           │ │
│  │  - Current streak, longest streak                              │ │
│  │  - Total drills completed                                      │ │
│  │  - Category-wise performance                                   │ │
│  │  - Readiness score (0-100)                                     │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    NOTIFICATION SYSTEM                               │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Push Notification Service                                     │ │
│  │  - Triggered daily at user's preferred time                    │ │
│  │  - Sends "Your Daily Drill is Ready!" notification             │ │
│  │  - Deep links to Daily Drill screen                            │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Database Schema

### 1. `daily_drill_questions` (Global Question Repository)

```sql
CREATE TABLE public.daily_drill_questions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_text TEXT NOT NULL,
    ideal_answer TEXT NOT NULL,
    evaluation_points TEXT[] NOT NULL DEFAULT '{}',
    skill_tags TEXT[] NOT NULL DEFAULT '{}',
    
    -- Categorization
    target_role TEXT NOT NULL CHECK (target_role IN ('intern', 'junior', 'mid', 'senior', 'lead')),
    category TEXT NOT NULL CHECK (category IN ('behavioral', 'dsa', 'sql', 'system_design', 'technical', 'leadership')),
    difficulty TEXT NOT NULL CHECK (difficulty IN ('easy', 'medium', 'hard')),
    
    -- Metadata
    estimated_time_minutes INT NOT NULL DEFAULT 5,
    source TEXT, -- e.g., 'AI Generated', 'Curated', 'Community'
    metadata JSONB DEFAULT '{}'::jsonb,
    
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_active BOOLEAN DEFAULT true
);

-- Indexes for efficient querying
CREATE INDEX idx_daily_drill_questions_role_category ON public.daily_drill_questions(target_role, category);
CREATE INDEX idx_daily_drill_questions_difficulty ON public.daily_drill_questions(difficulty);
CREATE INDEX idx_daily_drill_questions_active ON public.daily_drill_questions(is_active) WHERE is_active = true;
```

### 2. `user_daily_drills` (User Assignment Tracking)

```sql
CREATE TABLE public.user_daily_drills (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    question_id UUID NOT NULL REFERENCES public.daily_drill_questions(id) ON DELETE CASCADE,
    
    -- Assignment tracking
    assigned_date DATE NOT NULL,
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Completion tracking
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'viewed', 'completed', 'skipped')),
    viewed_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    
    -- User response
    confidence_level INT CHECK (confidence_level >= 1 AND confidence_level <= 5),
    user_notes TEXT,
    time_spent_seconds INT,
    
    -- Metadata
    metadata JSONB DEFAULT '{}'::jsonb,
    
    -- Constraints
    UNIQUE(user_id, assigned_date),
    UNIQUE(user_id, question_id) -- Prevent same question twice for same user
);

-- Indexes
CREATE INDEX idx_user_daily_drills_user_date ON public.user_daily_drills(user_id, assigned_date DESC);
CREATE INDEX idx_user_daily_drills_status ON public.user_daily_drills(user_id, status);
CREATE INDEX idx_user_daily_drills_assigned_date ON public.user_daily_drills(assigned_date);
```

### 3. `user_drill_progress` (User Progress & Analytics)

```sql
CREATE TABLE public.user_drill_progress (
    user_id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    
    -- Streak tracking
    current_streak INT NOT NULL DEFAULT 0,
    longest_streak INT NOT NULL DEFAULT 0,
    last_completed_date DATE,
    
    -- Overall stats
    total_drills_completed INT NOT NULL DEFAULT 0,
    total_drills_viewed INT NOT NULL DEFAULT 0,
    total_drills_skipped INT NOT NULL DEFAULT 0,
    
    -- Readiness score (0-100)
    readiness_score FLOAT8 NOT NULL DEFAULT 0,
    
    -- Category-wise performance (JSONB for flexibility)
    category_stats JSONB DEFAULT '{}'::jsonb,
    -- Example: {"behavioral": {"completed": 10, "avg_confidence": 4.2}, "dsa": {...}}
    
    -- Weak areas (for intelligent question selection)
    weak_categories TEXT[] DEFAULT '{}',
    
    -- Preferences
    preferred_notification_time TIME DEFAULT '09:00:00',
    notification_enabled BOOLEAN DEFAULT true,
    
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Index
CREATE INDEX idx_user_drill_progress_streak ON public.user_drill_progress(current_streak DESC);
```

---

## 🔄 Daily Assignment Logic

### Question Selection Algorithm

```typescript
async function selectDailyQuestion(userId: string, userProfile: any): Promise<Question> {
  // 1. Get user's drill history
  const attemptedQuestionIds = await getAttemptedQuestions(userId);
  
  // 2. Get user's weak areas and preferences
  const weakCategories = await getWeakCategories(userId);
  const userRole = userProfile.experience_level || 'junior';
  
  // 3. Build selection criteria (prioritize weak areas)
  const selectionCriteria = {
    target_role: userRole,
    category: weakCategories.length > 0 ? weakCategories[0] : null,
    difficulty: calculateDifficulty(userProfile.readiness_score),
    exclude_ids: attemptedQuestionIds
  };
  
  // 4. Query questions with weighted random selection
  const candidateQuestions = await fetchCandidateQuestions(selectionCriteria);
  
  // 5. Apply intelligent selection
  // - 70% from weak areas
  // - 20% from medium-strength areas
  // - 10% from strong areas (for maintenance)
  const selectedQuestion = weightedRandomSelection(candidateQuestions);
  
  return selectedQuestion;
}

function calculateDifficulty(readinessScore: number): string {
  if (readinessScore < 40) return 'easy';
  if (readinessScore < 70) return 'medium';
  return 'hard';
}
```

---

## 📱 Notification Logic

### Daily Notification Trigger

```typescript
// Supabase Edge Function: daily-drill-notifier
// Triggered via Supabase Cron (runs daily at midnight UTC)

async function sendDailyNotifications() {
  // 1. Get all users with notifications enabled
  const users = await supabase
    .from('user_drill_progress')
    .select('user_id, preferred_notification_time')
    .eq('notification_enabled', true);
  
  // 2. Group users by notification time
  const usersByTime = groupByNotificationTime(users);
  
  // 3. For each time slot, schedule notifications
  for (const [time, userIds] of Object.entries(usersByTime)) {
    await scheduleNotifications(userIds, time);
  }
}

async function scheduleNotifications(userIds: string[], time: string) {
  // Use Firebase Cloud Messaging or similar
  const notification = {
    title: "🎯 Your Daily Drill is Ready!",
    body: "Master one question today. Build your interview confidence.",
    data: {
      screen: "DailyDrillScreen",
      action: "open_daily_drill"
    }
  };
  
  await sendPushNotifications(userIds, notification);
}
```

---

## 📊 Scoring & Readiness Logic

### Readiness Score Calculation

```typescript
function calculateReadinessScore(userProgress: UserDrillProgress): number {
  const weights = {
    streak: 0.3,          // 30% weight on consistency
    completion: 0.4,      // 40% weight on completion rate
    confidence: 0.3       // 30% weight on average confidence
  };
  
  // Streak score (0-100)
  const streakScore = Math.min(100, userProgress.current_streak * 10);
  
  // Completion rate (0-100)
  const totalAssigned = userProgress.total_drills_completed + 
                        userProgress.total_drills_viewed + 
                        userProgress.total_drills_skipped;
  const completionRate = totalAssigned > 0 
    ? (userProgress.total_drills_completed / totalAssigned) * 100 
    : 0;
  
  // Average confidence (0-100)
  const avgConfidence = calculateAverageConfidence(userProgress.category_stats);
  
  // Weighted score
  const readinessScore = 
    (streakScore * weights.streak) +
    (completionRate * weights.completion) +
    (avgConfidence * weights.confidence);
  
  return Math.round(readinessScore);
}
```

### Streak Logic

```typescript
async function updateStreak(userId: string, completedDate: Date) {
  const progress = await getUserProgress(userId);
  const lastCompleted = progress.last_completed_date;
  
  let newStreak = 1;
  
  if (lastCompleted) {
    const daysDiff = dateDiff(lastCompleted, completedDate);
    
    if (daysDiff === 1) {
      // Consecutive day - increment streak
      newStreak = progress.current_streak + 1;
    } else if (daysDiff === 0) {
      // Same day - maintain streak
      newStreak = progress.current_streak;
    } else {
      // Streak broken - reset to 1
      newStreak = 1;
    }
  }
  
  const longestStreak = Math.max(progress.longest_streak, newStreak);
  
  await updateUserProgress(userId, {
    current_streak: newStreak,
    longest_streak: longestStreak,
    last_completed_date: completedDate
  });
}
```

---

## 🎨 UX Flow

### Daily Drill Screen Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      DAILY DRILL SCREEN                              │
└─────────────────────────────────────────────────────────────────────┘

STATE 1: Loading
  ├─ Show shimmer skeleton
  └─ Fetch today's question

STATE 2: Question Ready (Pending)
  ├─ Display question card
  ├─ Show "Reveal Answer" button (disabled initially)
  ├─ Show "I'm Ready" button
  └─ Timer starts tracking time spent

STATE 3: Question Viewed
  ├─ "Reveal Answer" button enabled
  ├─ User clicks → Show ideal answer
  ├─ Show evaluation points
  └─ Show confidence slider (1-5 stars)

STATE 4: Completed
  ├─ User selects confidence level
  ├─ Optional: Add personal notes
  ├─ Click "Mark as Complete"
  ├─ Show celebration animation
  ├─ Update streak counter
  ├─ Show next drill countdown (24h timer)
  └─ Award points/XP

STATE 5: Already Completed Today
  ├─ Show completed question with answer
  ├─ Display "Come back tomorrow!" message
  ├─ Show countdown to next drill
  └─ Display current streak
```

---

## 🚀 Optimization Strategies

### 1. **Question Reusability**
- Pre-generate 1000+ questions across all categories
- Batch generation via AI (cost-effective)
- Community contributions and curation

### 2. **Efficient Assignment**
- Use database indexes for fast lookups
- Cache user preferences in memory
- Batch assign questions for multiple users

### 3. **Analytics & Benchmarking**
- Track question difficulty vs. user confidence
- Identify trending weak areas across users
- A/B test question formats

### 4. **Scalability**
- Horizontal scaling via Supabase Edge Functions
- CDN caching for static question content
- Pagination for user history

---

## 📈 Success Metrics

- **Daily Active Users (DAU)**: Users completing daily drills
- **Streak Retention**: % of users maintaining 7+ day streaks
- **Readiness Score Growth**: Average improvement over 30 days
- **Notification CTR**: Click-through rate on push notifications
- **Question Quality**: Average confidence scores per question

---

## 🔐 Security & Privacy

- **RLS Policies**: Users can only access their own drill data
- **Rate Limiting**: Prevent abuse of question fetching
- **Data Encryption**: Sensitive user notes encrypted at rest
- **GDPR Compliance**: User data export and deletion support

---

## 🎭 Real-World Workflow Example

To understand the system in practice, let's follow a user named **Alex** (Mid-level Developer):

| Step | Component | Action / Data |
| :--- | :--- | :--- |
| **1. Open App** | Flutter UI | Alex opens the Daily Drill screen. Provider calls `get_today_question`. |
| **2. Selection** | DB RPC | `get_or_assign_daily_question` sees Alex is weak in **"System Design"**. It selects a **"Medium"** difficulty question on URL Shorteners. |
| **3. UI Load** | Flutter UI | UI renders the question card with a "10 min" estimate and "System Design" tags. |
| **4. Practice** | User | Alex thinks about the answer, reads the question text. |
| **5. Response** | User | Alex selects **Confidence: 4** and writes a note: *"Need to review Redis caching"*. Hits **Complete**. |
| **6. Processing** | Edge Function | The `submit_drill_response` action is sent. Notes are sanitized. |
| **7. Analytics** | DB Trigger | Trigger detects `status = 'completed'`. |
| **8. Progress** | DB Logic | **Streak** updates (3 → 4 days). **Readiness Score** recalculates. **Weak Categories** list updates. |
| **9. Result** | Flutter UI | UI shows a "Success" animation. The "Ideal Answer" is revealed for Alex to compare. |

---

## 📊 Example Data Structures

### 1. Edge Function Request (Submit Response)
```json
{
  "action": "submit_drill_response",
  "payload": {
    "drill_id": "550e8400-e29b-41d4-a716-446655440000",
    "action": "complete",
    "confidence_level": 4,
    "user_notes": "Need to review Redis caching",
    "time_spent_seconds": 320
  }
}
```

### 2. Edge Function Response (Success)
```json
{
  "drill": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "completed",
    "assigned_date": "2026-02-10",
    "confidence_level": 4,
    "user_notes": "Need to review Redis caching"
  },
  "progress": {
    "current_streak": 4,
    "readiness_score": 72.5,
    "weak_categories": ["SQL"]
  },
  "message": "Great job! Keep the streak going!"
}
```

---

**Created**: 2026-02-10  
**Author**: Antigravity AI Assistant  
**Project**: AntiGravity - Daily Drill System
