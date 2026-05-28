# 🚀 Learning Recommendations Edge Function - Deployment Guide

**Created**: 2026-02-14  
**Function Name**: `learning-recommendations`  
**Status**: Ready for Deployment

---

## 📋 Overview

This Edge Function provides AI-powered learning content recommendations based on a user's interview performance. It analyzes the last 10 interviews, identifies weak areas, and matches appropriate learning content.

---

## 🔧 Deployment Steps

### **1. Deploy the Edge Function**

```bash
# Navigate to project root
cd c:\flutter_apps\intervi_prep

# Deploy the function to Supabase
supabase functions deploy learning-recommendations

# Verify deployment
supabase functions list
```

### **2. Set Environment Variables**

The function requires these environment variables (automatically available in Supabase):
- `SUPABASE_URL` - Your Supabase project URL
- `SUPABASE_SERVICE_ROLE_KEY` - Service role key for admin operations

These are automatically injected by Supabase, no manual configuration needed.

### **3. Test the Function**

#### **Test with curl**:
```bash
curl -i --location --request POST 'https://YOUR_PROJECT_REF.supabase.co/functions/v1/learning-recommendations' \
  --header 'Authorization: Bearer YOUR_ANON_KEY' \
  --header 'Content-Type: application/json' \
  --data '{"userId":"USER_UUID_HERE"}'
```

#### **Expected Response**:
```json
{
  "overallReadiness": 65,
  "stats": {
    "interviewCount": 10,
    "strengths": [
      { "name": "Python", "score": 85 },
      { "name": "SQL", "score": 82 }
    ],
    "weaknesses": [
      { "name": "System Design", "score": 45 },
      { "name": "Machine Learning", "score": 62 }
    ]
  },
  "recommendations": [
    {
      "id": "uuid",
      "title": "System Design Fundamentals",
      "type": "video",
      "category": "System Design",
      "level": "foundation",
      "priority": "critical",
      ...
    }
  ],
  "journey": {
    "currentLevel": "Intermediate Learner",
    "nextMilestone": "Reach 70% readiness",
    "progress": 0.83
  }
}
```

---

## 🔍 Function Logic

### **Input**
```typescript
{
  "userId": "uuid-string"
}
```

### **Processing Steps**

1. **Fetch Interviews** - Get last 10 interviews for the user
2. **Analyze Performance** - Calculate average score per category
3. **Identify Weak Areas** - Find categories with score < 70
4. **Match Content** - Find appropriate learning content based on:
   - Category match
   - Difficulty level (based on score)
   - Content priority (critical > essential > recommended)
5. **Calculate Readiness** - Overall readiness score (0-100)
6. **Build Journey** - User's current level and next milestone

### **Content Matching Logic**

| Score Range | Difficulty Level |
|-------------|------------------|
| 0-49 | Foundation |
| 50-64 | Intermediate |
| 65-79 | Advanced |
| 80+ | Expert |

### **Recommendation Priority**

1. **Weak Areas** (score < 70) - 3 items per weak area
2. **Growth Areas** (score >= 85) - 1 expert-level item per strong area
3. **Critical Content** - Fill remaining slots with critical priority content

---

## 📊 Database Dependencies

### **Required Tables**

#### **`interviews`**
```sql
- id: uuid
- user_id: uuid
- competency_scores: jsonb  -- { "Python": 85, "SQL": 70, ... }
- overall_score: integer
- created_at: timestamptz
```

#### **`learning_content`**
```sql
- id: uuid
- title: text
- description: text
- type: content_type ('video', 'article', 'faq')
- url: text
- thumbnail_url: text
- category: text
- level: difficulty_level ('foundation', 'intermediate', 'advanced', 'expert')
- duration_minutes: integer
- instructor: text
- priority: content_priority ('critical', 'essential', 'recommended')
- tags: text[]
- is_active: boolean
- created_at: timestamptz
```

---

## 🧪 Testing Scenarios

### **Scenario 1: New User (No Interviews)**
**Input**: User with 0 interviews  
**Expected**: 
- `overallReadiness: 0`
- `interviewCount: 0`
- `recommendations`: 5 foundation-level critical content items
- `journey.currentLevel`: "Beginner"

### **Scenario 2: Intermediate User**
**Input**: User with 10 interviews, mixed scores  
**Expected**:
- `overallReadiness`: 50-70
- Recommendations focused on weak areas
- Mix of foundation and intermediate content

### **Scenario 3: Advanced User**
**Input**: User with 10 interviews, high scores (80+)  
**Expected**:
- `overallReadiness`: 80+
- Expert-level content for growth
- `journey.currentLevel`: "Advanced Candidate" or "Interview Expert"

---

## 🔐 Security

### **Row Level Security (RLS)**
- Function uses `SUPABASE_SERVICE_ROLE_KEY` to bypass RLS
- Validates `userId` from request
- Only returns data for the specified user

### **CORS**
- Allows all origins (`*`) for development
- **Production**: Update `corsHeaders` to restrict origins

---

## 📈 Performance Considerations

### **Current Implementation**
- **Database Queries**: 2-3 per request
  1. Fetch interviews
  2. Fetch learning content
  3. (Optional) Fetch starter content for new users

### **Optimization Opportunities** (Phase 2)
1. **Caching**: Cache recommendations for 24 hours
2. **Indexing**: Add indexes on frequently queried columns
3. **Pagination**: Limit content fetch to relevant categories only
4. **Materialized Views**: Pre-compute category scores

---

## 🐛 Troubleshooting

### **Issue: "No interviews found"**
**Cause**: User hasn't completed any interviews  
**Solution**: Returns starter content automatically

### **Issue: "No recommendations returned"**
**Cause**: No matching content in database  
**Solution**: Ensure `learning_content` table has data

### **Issue: "500 Internal Server Error"**
**Cause**: Database connection or query error  
**Solution**: Check Supabase logs:
```bash
supabase functions logs learning-recommendations
```

---

## 📝 Monitoring

### **Key Metrics to Track**
1. **Response Time** - Target: < 2 seconds
2. **Error Rate** - Target: < 1%
3. **Recommendation Accuracy** - Track user engagement
4. **Cache Hit Rate** (Phase 2)

### **Logging**
The function logs:
- User ID for each request
- Number of interviews found
- Category scores calculated
- Number of recommendations generated
- Overall readiness score

**View logs**:
```bash
supabase functions logs learning-recommendations --tail
```

---

## 🔄 Integration with Flutter App

### **Service Call** (`ai_service.dart`)
```dart
Future<LearningRecommendationsResponse> getLearningRecommendations({
  required String userId,
}) async {
  final response = await _client.functions.invoke(
    'learning-recommendations',
    body: {
      'userId': userId,
    },
  );
  
  return LearningRecommendationsResponse.fromJson(response.data);
}
```

### **Provider** (`practice_hub_provider.dart`)
```dart
final practiceRecommendationsProvider = FutureProvider<LearningRecommendationsResponse>((ref) async {
  final aiService = ref.watch(aiServiceProvider);
  final user = ref.watch(authStateProvider).user;
  
  return aiService.getLearningRecommendations(userId: user.id);
});
```

---

## 🚀 Future Enhancements (Phase 2 & 3)

### **Phase 2: Intelligence**
- [ ] User progress tracking (viewed/completed content)
- [ ] Free vs. paid user logic
- [ ] Content retention strategy
- [ ] AI-powered relevance scoring

### **Phase 3: Automation**
- [ ] Automatic refresh on new interview
- [ ] Scheduled weekly refresh (paid users)
- [ ] Caching layer (24-hour TTL)
- [ ] Analytics dashboard

---

## ✅ Deployment Checklist

- [ ] Function deployed to Supabase
- [ ] Environment variables verified
- [ ] Test with curl (success response)
- [ ] Test with Flutter app
- [ ] Verify logs in Supabase dashboard
- [ ] Monitor initial performance metrics
- [ ] Document any issues or edge cases

---

## 📞 Support

**Function Location**: `supabase/functions/learning-recommendations/index.ts`  
**Documentation**: This file  
**Logs**: `supabase functions logs learning-recommendations`

---

**Status**: ✅ **READY FOR DEPLOYMENT**

**Next Steps**: 
1. Deploy function to Supabase
2. Test with real user data
3. Monitor performance
4. Proceed to Phase 1, Task 3 (Database Migrations)
