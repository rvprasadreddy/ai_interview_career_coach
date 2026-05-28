# 🎯 Practice Hub - Implementation Roadmap Summary

**Date**: 2026-02-14  
**Document**: Quick Reference for Development Team

---

## 🔴 **PHASE 1: Foundation** (Weeks 1-2) - **START HERE**

### **Critical Blockers to Fix First**

#### **1. YouTube Video Playback on Mobile** ⚠️ **HIGHEST PRIORITY**
**Problem**: Videos don't play in Android/iOS emulator  
**Solution**: Platform-specific player implementation

**Files to Modify**:
- `lib/features/practice/screens/video_player_screen.dart` (create new)
- `pubspec.yaml` (add `youtube_player_flutter` package)

**Implementation**:
```dart
// Add to pubspec.yaml
dependencies:
  youtube_player_flutter: ^9.1.3  # For mobile
  youtube_player_iframe: ^5.2.2   # For web (already exists)

// Create video_player_screen.dart
import 'dart:io' show Platform;
import 'package:flutter/foundation.dart' show kIsWeb;
import 'package:youtube_player_flutter/youtube_player_flutter.dart';

class VideoPlayerScreen extends StatefulWidget {
  final LearningContent content;
  
  @override
  Widget build(BuildContext context) {
    if (kIsWeb) {
      return _buildWebPlayer();  // Use iframe
    } else {
      return _buildMobilePlayer();  // Use native player
    }
  }
}
```

**Testing Checklist**:
- [ ] Test on Android emulator
- [ ] Test on iOS simulator
- [ ] Test on web browser
- [ ] Test on real Android device
- [ ] Test on real iOS device

---

#### **2. Backend Edge Function Implementation**
**Problem**: `get_learning_recommendations` action doesn't exist  
**Solution**: Add action to `ai-interview-coach` Edge Function

**File**: `supabase/functions/ai-interview-coach/index.ts`

**Add this action**:
```typescript
case 'get_learning_recommendations': {
  const { userId } = payload;
  
  // Step 1: Get user's last 10 interviews
  const { data: interviews } = await supabaseAdmin
    .from('interviews')
    .select('id, competency_scores, overall_score, created_at')
    .eq('user_id', userId)
    .order('created_at', { ascending: false })
    .limit(10);
  
  if (!interviews || interviews.length === 0) {
    return new Response(JSON.stringify({
      overallReadiness: 0,
      stats: { interviewCount: 0, strengths: [], weaknesses: [] },
      recommendations: [],
      journey: { currentLevel: 'Beginner', nextMilestone: 'Complete your first interview', progress: 0 }
    }), { headers: { 'Content-Type': 'application/json' } });
  }
  
  // Step 2: Analyze category performance
  const categoryScores = analyzeCategoryPerformance(interviews);
  
  // Step 3: Identify weak areas (score < 70)
  const weakAreas = categoryScores.filter(c => c.score < 70);
  
  // Step 4: Get relevant content from database
  const { data: allContent } = await supabaseAdmin
    .from('learning_content')
    .select('*')
    .eq('is_active', true);
  
  // Step 5: Match content to weak areas
  const recommendations = matchContentToWeakAreas(allContent, weakAreas, categoryScores);
  
  // Step 6: Calculate overall readiness
  const overallReadiness = Math.round(
    categoryScores.reduce((sum, c) => sum + c.score, 0) / categoryScores.length
  );
  
  return new Response(JSON.stringify({
    overallReadiness,
    stats: {
      interviewCount: interviews.length,
      strengths: categoryScores.filter(c => c.score >= 80).slice(0, 3),
      weaknesses: categoryScores.filter(c => c.score < 70).slice(0, 3)
    },
    recommendations: recommendations.slice(0, 15),
    journey: calculateUserJourney(overallReadiness, interviews.length)
  }), { headers: { 'Content-Type': 'application/json' } });
}

// Helper functions
function analyzeCategoryPerformance(interviews) {
  const categoryMap = new Map();
  
  interviews.forEach(interview => {
    const scores = interview.competency_scores || {};
    Object.entries(scores).forEach(([category, score]) => {
      if (!categoryMap.has(category)) {
        categoryMap.set(category, []);
      }
      categoryMap.get(category).push(score);
    });
  });
  
  return Array.from(categoryMap.entries()).map(([name, scores]) => ({
    name,
    score: Math.round(scores.reduce((a, b) => a + b, 0) / scores.length)
  }));
}

function matchContentToWeakAreas(allContent, weakAreas, categoryScores) {
  const recommendations = [];
  
  weakAreas.forEach(area => {
    // Determine difficulty based on score
    let difficulty;
    if (area.score < 50) difficulty = 'foundation';
    else if (area.score < 65) difficulty = 'intermediate';
    else difficulty = 'advanced';
    
    // Find matching content
    const matching = allContent.filter(c => 
      c.category.toLowerCase() === area.name.toLowerCase() &&
      c.level === difficulty
    );
    
    recommendations.push(...matching.slice(0, 3));
  });
  
  return recommendations;
}

function calculateUserJourney(readiness, interviewCount) {
  if (readiness < 40) {
    return {
      currentLevel: 'Foundation Builder',
      nextMilestone: 'Reach 50% readiness',
      progress: readiness / 50
    };
  } else if (readiness < 70) {
    return {
      currentLevel: 'Intermediate Learner',
      nextMilestone: 'Reach 70% readiness',
      progress: (readiness - 40) / 30
    };
  } else {
    return {
      currentLevel: 'Advanced Candidate',
      nextMilestone: 'Maintain 85%+ readiness',
      progress: (readiness - 70) / 30
    };
  }
}
```

---

#### **3. Database Schema Updates**
**Problem**: Missing tables and enum values  
**Solution**: Create migration file

**File**: `supabase/migrations/20260214_practice_hub_enhancements.sql`

```sql
-- Add 'faq' to content_type enum
ALTER TYPE content_type ADD VALUE IF NOT EXISTS 'faq';

-- Create user_learning_progress table
CREATE TABLE IF NOT EXISTS public.user_learning_progress (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  content_id UUID NOT NULL REFERENCES public.learning_content(id) ON DELETE CASCADE,
  status TEXT NOT NULL DEFAULT 'not_started' CHECK (status IN ('not_started', 'in_progress', 'completed')),
  progress_percentage INT DEFAULT 0 CHECK (progress_percentage >= 0 AND progress_percentage <= 100),
  time_spent_minutes INT DEFAULT 0,
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  last_accessed_at TIMESTAMPTZ DEFAULT now(),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, content_id)
);

-- Enable RLS
ALTER TABLE public.user_learning_progress ENABLE ROW LEVEL SECURITY;

-- RLS Policies
CREATE POLICY "Users can view their own progress" ON public.user_learning_progress
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can update their own progress" ON public.user_learning_progress
  FOR ALL USING (auth.uid() = user_id);

-- Create indexes
CREATE INDEX idx_user_learning_progress_user_id ON public.user_learning_progress(user_id);
CREATE INDEX idx_user_learning_progress_content_id ON public.user_learning_progress(content_id);
CREATE INDEX idx_user_learning_progress_status ON public.user_learning_progress(status);

-- Create content_recommendations table
CREATE TABLE IF NOT EXISTS public.content_recommendations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  content_id UUID NOT NULL REFERENCES public.learning_content(id) ON DELETE CASCADE,
  recommended_at TIMESTAMPTZ DEFAULT now(),
  reason TEXT,
  relevance_score DECIMAL(3,2) CHECK (relevance_score >= 0 AND relevance_score <= 1),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Enable RLS
ALTER TABLE public.content_recommendations ENABLE ROW LEVEL SECURITY;

-- RLS Policies
CREATE POLICY "Users can view their own recommendations" ON public.content_recommendations
  FOR SELECT USING (auth.uid() = user_id);

-- Create indexes
CREATE INDEX idx_content_recommendations_user_id ON public.content_recommendations(user_id);
CREATE INDEX idx_content_recommendations_active ON public.content_recommendations(is_active) WHERE is_active = true;
```

---

#### **4. Expand Content Library**
**Problem**: Only 7 videos in database  
**Solution**: Add 50+ high-quality learning resources

**File**: `supabase/migrations/20260214_expand_content_library.sql`

```sql
-- Add more Python videos
INSERT INTO public.learning_content (title, description, type, url, category, level, duration_minutes, instructor, priority, tags)
VALUES 
-- Foundation Python
('Python Basics for Beginners', 'Complete Python fundamentals course', 'video', 'kqtD5dpn9C8', 'Python', 'foundation', 240, 'Programming with Mosh', 'critical', ARRAY['python', 'basics', 'programming']),
('Python OOP Tutorial', 'Object-Oriented Programming in Python', 'video', 'Ej_02ICOIgs', 'Python', 'intermediate', 45, 'Tech With Tim', 'essential', ARRAY['python', 'oop', 'classes']),

-- SQL Content
('SQL for Data Analysis', 'Advanced SQL techniques', 'video', 'QSqw7boV-8M', 'SQL', 'intermediate', 180, 'Alex The Analyst', 'essential', ARRAY['sql', 'data-analysis', 'joins']),
('Database Design Fundamentals', 'Learn database normalization', 'video', 'ztHopE5Wnpc', 'SQL', 'foundation', 60, 'Lucid Software', 'critical', ARRAY['database', 'design', 'normalization']),

-- Machine Learning
('ML Crash Course', 'Google ML Crash Course', 'article', 'https://developers.google.com/machine-learning/crash-course', 'Machine Learning', 'foundation', 180, 'Google', 'critical', ARRAY['ml', 'tensorflow', 'basics']),
('Deep Learning Specialization', 'Neural networks explained', 'video', 'CS4cs9xVecY', 'Machine Learning', 'advanced', 480, 'Andrew Ng', 'essential', ARRAY['deep-learning', 'neural-networks', 'ai']),

-- System Design
('System Design Primer', 'Complete system design guide', 'article', 'https://github.com/donnemartin/system-design-primer', 'System Design', 'intermediate', 300, 'Donne Martin', 'critical', ARRAY['system-design', 'architecture', 'scalability']),
('Designing Data-Intensive Applications', 'Book summary and key concepts', 'article', 'https://dataintensive.net/', 'System Design', 'advanced', 600, 'Martin Kleppmann', 'essential', ARRAY['databases', 'distributed-systems', 'architecture']),

-- Behavioral
('STAR Method Examples', 'Top behavioral interview questions', 'faq', 'https://www.themuse.com/advice/star-interview-method', 'Behavioral', 'foundation', 15, 'The Muse', 'critical', ARRAY['behavioral', 'star-method', 'examples']),
('Leadership Principles', 'Amazon leadership principles explained', 'article', 'https://www.amazon.jobs/en/principles', 'Behavioral', 'intermediate', 30, 'Amazon', 'essential', ARRAY['leadership', 'behavioral', 'amazon']);
```

---

## 🟡 **PHASE 2: Intelligence** (Weeks 3-4)

### **Key Features**
1. **AI-Powered Relevance Scoring**
2. **User Progress Tracking**
3. **Free vs. Paid User Logic**

### **Implementation Priority**
1. Progress tracking (enables engagement metrics)
2. Relevance scoring (improves recommendations)
3. Tier-based logic (monetization)

---

## 🟢 **PHASE 3: Automation** (Weeks 5-6)

### **Key Features**
1. **Auto-refresh triggers**
2. **Cost optimization**
3. **Analytics dashboard**

### **Implementation Priority**
1. Caching (reduces costs immediately)
2. Auto-refresh (improves user experience)
3. Analytics (enables data-driven decisions)

---

## ✅ **Immediate Action Items**

### **This Week**
1. ✅ Fix YouTube video playback on mobile
2. ✅ Implement basic Edge Function logic
3. ✅ Apply database migrations
4. ✅ Expand content library to 50+ items

### **Next Week**
1. Add progress tracking UI
2. Implement relevance scoring
3. Test on real devices
4. Deploy to staging

---

## 📊 **Success Criteria**

### **Phase 1 Complete When**:
- [ ] Videos play on Android emulator
- [ ] Videos play on iOS simulator
- [ ] Recommendations show for users with 10+ interviews
- [ ] Content library has 50+ items
- [ ] No errors in Flutter analyze
- [ ] Response time < 2 seconds

---

## 🚨 **Known Issues to Address**

1. **Video Playback**: Currently broken on mobile
2. **Static Content**: No dynamic recommendations
3. **No Progress Tracking**: Can't track user engagement
4. **Missing FAQs**: FAQ type exists in UI but not in database
5. **Hardcoded Metrics**: Relevance and ratings are fake (95%, 4.8)

---

## 📝 **Notes**

- All code should follow AntiGravity theme
- Test on both emulator and real devices
- Document all changes
- Keep costs in mind (use caching, batch processing)

---

**Ready to Start**: Begin with video playback fix, then Edge Function implementation.
