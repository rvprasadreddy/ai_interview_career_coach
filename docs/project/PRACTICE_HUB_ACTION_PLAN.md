# 🎯 Practice Hub - Production Action Plan

**Date**: 2026-02-14  
**Objective**: Bring Practice Hub to production-grade quality  
**Timeline**: 4 weeks  
**Estimated Effort**: 60-80 hours

---

## 📊 Quick Summary

**Current Production Readiness**: 60/100  
**Target Production Readiness**: 95/100  
**Critical Issues**: 5  
**High-Priority Issues**: 5  
**Medium-Priority Issues**: 8

---

## 🔥 Week 1: Critical Fixes (Must-Have for MVP)

### **Day 1-2: Database & Content** (8-10 hours)

#### **Task 1.1: Deploy Database Migration** ⏱️ 1 hour
- [x] Migration file created: `20260214_practice_hub_enhancements.sql`
- [ ] Deploy to Supabase via Dashboard or CLI
- [ ] Verify tables created
- [ ] Test RLS policies
- [ ] Run verification queries

**Files**:
- `supabase/migrations/20260214_practice_hub_enhancements.sql`

**Success Criteria**:
- ✅ `user_learning_progress` table exists
- ✅ `content_recommendations` table exists
- ✅ RLS policies working
- ✅ Helper functions callable

---

#### **Task 1.2: Expand Content Library** ⏱️ 4-6 hours
- [ ] Create `20260214_expand_content_library.sql`
- [ ] Add 20+ videos per category (Python, ML, SQL, System Design, Behavioral)
- [ ] Add 15+ FAQs per category
- [ ] Add 10+ articles per category
- [ ] Ensure diversity in difficulty levels
- [ ] Deploy migration

**Target**: 100+ total content items

**Categories to Cover**:
- Python (20 videos, 15 FAQs, 10 articles)
- Machine Learning (20 videos, 15 FAQs, 10 articles)
- SQL & Databases (15 videos, 10 FAQs, 8 articles)
- System Design (15 videos, 10 FAQs, 8 articles)
- Behavioral (10 videos, 15 FAQs, 10 articles)
- Data Structures & Algorithms (15 videos, 12 FAQs, 8 articles)

**Success Criteria**:
- ✅ 100+ content items in database
- ✅ All categories covered
- ✅ All difficulty levels represented
- ✅ All content types (video, article, faq) present

---

#### **Task 1.3: Implement Error Boundaries** ⏱️ 3-4 hours

**File**: `lib/features/practice/screens/practice_hub_screen.dart`

**Changes**:
1. Create error categorization system
2. Add specific error states for:
   - Network errors
   - Authentication errors
   - Timeout errors
   - Server errors
3. Implement retry logic with exponential backoff
4. Add error logging (Firebase Crashlytics)

**Code**:
```dart
// Create: lib/core/errors/app_exceptions.dart
class NetworkException implements Exception {}
class AuthException implements Exception {}
class TimeoutException implements Exception {}
class ServerException implements Exception {}

// Update: practice_hub_screen.dart
error: (err, stack) {
  // Log error
  FirebaseCrashlytics.instance.recordError(err, stack);
  
  // Categorize and display appropriate error state
  if (err.toString().contains('network')) {
    return _buildNetworkErrorState(context, ref);
  } else if (err.toString().contains('auth')) {
    return _buildAuthErrorState(context, ref);
  }
  
  return _buildGenericErrorState(context, err, ref);
}
```

**Success Criteria**:
- ✅ Specific error messages for each error type
- ✅ Retry button with exponential backoff
- ✅ Errors logged to analytics
- ✅ User-friendly error messages

---

### **Day 3-4: Progress Tracking & Caching** (8-10 hours)

#### **Task 1.4: Integrate Progress Tracking** ⏱️ 4-5 hours

**Files to Modify**:
1. `lib/features/practice/screens/video_player_screen.dart`
2. `lib/features/practice/screens/article_viewer_screen.dart`
3. `lib/features/practice/widgets/content_card.dart`
4. `lib/core/services/supabase_service.dart` (create if not exists)

**Implementation**:
```dart
// When user opens content
await supabase.rpc('start_learning_content', params: {
  'p_user_id': userId,
  'p_content_id': contentId,
});

// When user completes content (video ends, article scrolled to bottom)
await supabase.rpc('complete_learning_content', params: {
  'p_user_id': userId,
  'p_content_id': contentId,
});

// Update progress periodically
await supabase
  .from('user_learning_progress')
  .update({
    'progress_percentage': percentage,
    'time_spent_minutes': timeSpent,
  })
  .eq('user_id', userId)
  .eq('content_id', contentId);
```

**UI Updates**:
- Add progress indicators to content cards
- Show "In Progress" badge
- Show "Completed" checkmark
- Display time spent

**Success Criteria**:
- ✅ Progress tracked when content opened
- ✅ Completion tracked when content finished
- ✅ Progress percentage updated
- ✅ UI shows progress indicators

---

#### **Task 1.5: Implement Caching Strategy** ⏱️ 4-5 hours

**Create**: `lib/core/services/cache_service.dart`

**Implementation**:
```dart
class CacheService {
  final _storage = GetStorage();
  
  Future<CachedData?> get(String key) async {
    final json = _storage.read(key);
    if (json == null) return null;
    
    final cached = CachedData.fromJson(json);
    if (cached.isExpired) {
      await _storage.remove(key);
      return null;
    }
    
    return cached;
  }
  
  Future<void> set(String key, dynamic data, Duration duration) async {
    final cached = CachedData(
      data: data,
      expiresAt: DateTime.now().add(duration),
    );
    await _storage.write(key, cached.toJson());
  }
}
```

**Update**: `lib/features/practice/providers/practice_hub_provider.dart`

```dart
final practiceRecommendationsProvider = FutureProvider.autoDispose<LearningRecommendationsResponse>((ref) async {
  final cacheService = ref.read(cacheServiceProvider);
  final user = ref.watch(authStateProvider).user!;
  final cacheKey = 'recommendations_${user.id}';
  
  // Check cache first
  final cached = await cacheService.get(cacheKey);
  if (cached != null) {
    return LearningRecommendationsResponse.fromJson(cached.data);
  }
  
  // Fetch fresh data
  final aiService = ref.read(aiServiceProvider);
  final data = await aiService.getLearningRecommendations(userId: user.id);
  
  // Cache for 24 hours
  await cacheService.set(cacheKey, data.toJson(), Duration(hours: 24));
  
  return data;
});
```

**Success Criteria**:
- ✅ Recommendations cached for 24 hours
- ✅ Cache invalidated on manual refresh
- ✅ Faster load times on subsequent visits
- ✅ Offline mode support (show cached data)

---

### **Day 5: Testing & Validation** (4-5 hours)

#### **Task 1.6: End-to-End Testing** ⏱️ 4-5 hours

**Test Scenarios**:
1. **Fresh User Flow**:
   - [ ] User with 0 interviews sees empty state
   - [ ] User with 1-9 interviews sees partial recommendations
   - [ ] User with 10+ interviews sees full recommendations

2. **Content Interaction**:
   - [ ] Click video → opens player → progress tracked
   - [ ] Complete video → marked as completed
   - [ ] Click article → opens viewer → progress tracked
   - [ ] Click FAQ → displays content

3. **Filtering**:
   - [ ] Filter by difficulty level works
   - [ ] Filter by content type (Videos/FAQs/Blogs) works
   - [ ] Empty filter state shows correct message

4. **Error Handling**:
   - [ ] Network error shows retry button
   - [ ] Timeout error shows appropriate message
   - [ ] Invalid user shows auth error

5. **Caching**:
   - [ ] First load fetches from backend
   - [ ] Second load uses cache (faster)
   - [ ] Manual refresh invalidates cache

**Success Criteria**:
- ✅ All test scenarios pass
- ✅ No crashes or errors
- ✅ Smooth user experience

---

## 🎯 Week 2: High-Priority Features

### **Day 6-7: Free vs. Paid Logic** (8-10 hours)

#### **Task 2.1: Subscription System** ⏱️ 4-5 hours

**Create Migration**: `20260214_user_subscriptions.sql`

```sql
CREATE TABLE IF NOT EXISTS public.user_subscriptions (
  user_id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  tier TEXT NOT NULL DEFAULT 'free' CHECK (tier IN ('free', 'paid')),
  features JSONB DEFAULT '{}',
  valid_until TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

**Update Edge Function**: `supabase/functions/learning-recommendations/index.ts`

```typescript
// Get user subscription tier
const { data: subscription } = await supabaseAdmin
  .from('user_subscriptions')
  .select('tier')
  .eq('user_id', userId)
  .single();

const userTier = subscription?.tier || 'free';

// Apply tier-based logic
if (userTier === 'free') {
  // One-time generation
  const { data: existingRecs } = await supabaseAdmin
    .from('content_recommendations')
    .select('*')
    .eq('user_id', userId)
    .eq('is_active', true);
  
  if (existingRecs && existingRecs.length > 0) {
    // Return existing recommendations
    return existingRecs;
  }
}

// Limit recommendations based on tier
const maxRecommendations = userTier === 'paid' ? 20 : 10;
recommendations = recommendations.slice(0, maxRecommendations);
```

**Success Criteria**:
- ✅ Free users get one-time recommendations
- ✅ Paid users get dynamic recommendations
- ✅ Content limits enforced

---

#### **Task 2.2: Analytics Tracking** ⏱️ 4-5 hours

**Install**: `firebase_analytics`

**Create**: `lib/core/services/analytics_service.dart`

```dart
class AnalyticsService {
  final FirebaseAnalytics _analytics = FirebaseAnalytics.instance;
  
  Future<void> logContentView(String contentId, String contentType) async {
    await _analytics.logEvent(
      name: 'content_view',
      parameters: {
        'content_id': contentId,
        'content_type': contentType,
      },
    );
  }
  
  Future<void> logContentComplete(String contentId, int timeSpent) async {
    await _analytics.logEvent(
      name: 'content_complete',
      parameters: {
        'content_id': contentId,
        'time_spent_seconds': timeSpent,
      },
    );
  }
  
  Future<void> logRecommendationClick(String contentId, int position) async {
    await _analytics.logEvent(
      name: 'recommendation_click',
      parameters: {
        'content_id': contentId,
        'position': position,
      },
    );
  }
}
```

**Integrate in UI**:
```dart
// When content card clicked
onTap: () {
  analytics.logRecommendationClick(content.id, index);
  context.push(Routes.videoPlayer, extra: content);
}
```

**Success Criteria**:
- ✅ Content views tracked
- ✅ Completions tracked
- ✅ Click-through rates measured
- ✅ Dashboard shows metrics

---

### **Day 8-9: UI Polish** (8-10 hours)

#### **Task 2.3: Loading Skeletons** ⏱️ 3-4 hours

**Create**: `lib/core/widgets/skeleton_loader.dart`

```dart
class ContentCardSkeleton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Shimmer.fromColors(
      baseColor: Colors.grey[300]!,
      highlightColor: Colors.grey[100]!,
      child: Container(
        // Match actual content card layout
        child: Column(
          children: [
            Container(height: 180, color: Colors.white), // Thumbnail
            SizedBox(height: 12),
            Container(height: 16, width: double.infinity, color: Colors.white), // Title
            SizedBox(height: 8),
            Container(height: 12, width: 200, color: Colors.white), // Subtitle
          ],
        ),
      ),
    );
  }
}
```

**Success Criteria**:
- ✅ Skeleton matches actual layout
- ✅ Smooth transition to real content
- ✅ Better perceived performance

---

#### **Task 2.4: Remove Hardcoded Values** ⏱️ 2-3 hours

**Update**: `lib/features/practice/widgets/content_card.dart`

**Remove**:
```dart
// ❌ Delete these
Text('95% Match')
Text('4.8 ⭐')
```

**Add to Backend**:
```typescript
// Calculate relevance score
const relevanceScore = calculateRelevanceScore(content, weakArea);

// Add to response
return {
  ...content,
  relevanceScore: relevanceScore,
  userRating: await getUserRating(content.id),
};
```

**Update UI**:
```dart
// ✅ Use real data
if (content.relevanceScore != null) {
  Text('${(content.relevanceScore * 100).toInt()}% Match')
}

if (content.userRating != null) {
  Text('${content.userRating} ⭐')
}
```

**Success Criteria**:
- ✅ No hardcoded metrics
- ✅ Real relevance scores displayed
- ✅ Real user ratings shown

---

#### **Task 2.5: Input Validation** ⏱️ 3-4 hours

**Update**: `supabase/functions/learning-recommendations/index.ts`

```typescript
// UUID validation
const UUID_REGEX = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

if (!UUID_REGEX.test(userId)) {
  return new Response(
    JSON.stringify({ error: 'Invalid userId format' }),
    { status: 400, headers: corsHeaders }
  );
}

// User existence check
const { data: user, error: userError } = await supabaseAdmin
  .from('auth.users')
  .select('id')
  .eq('id', userId)
  .single();

if (userError || !user) {
  return new Response(
    JSON.stringify({ error: 'User not found' }),
    { status: 404, headers: corsHeaders }
  );
}

// Rate limiting (using Upstash Redis or similar)
const rateLimitKey = `rate_limit:${userId}`;
const requestCount = await redis.incr(rateLimitKey);

if (requestCount === 1) {
  await redis.expire(rateLimitKey, 60); // 1 minute window
}

if (requestCount > 10) {
  return new Response(
    JSON.stringify({ error: 'Rate limit exceeded' }),
    { status: 429, headers: corsHeaders }
  );
}
```

**Success Criteria**:
- ✅ Invalid UUIDs rejected
- ✅ Non-existent users handled
- ✅ Rate limiting prevents abuse
- ✅ Clear error messages

---

## 🔧 Week 3: Quality & Polish

### **Day 10-12: Testing & Accessibility** (12-15 hours)

#### **Task 3.1: Unit Tests** ⏱️ 6-8 hours

**Create**: `test/features/practice/providers/practice_hub_provider_test.dart`

```dart
void main() {
  group('PracticeRecommendationsProvider', () {
    test('returns recommendations for authenticated user', () async {
      // Arrange
      final container = ProviderContainer(overrides: [
        authStateProvider.overrideWithValue(mockAuthState),
        aiServiceProvider.overrideWithValue(mockAiService),
      ]);
      
      // Act
      final recommendations = await container.read(practiceRecommendationsProvider.future);
      
      // Assert
      expect(recommendations.recommendations.length, greaterThan(0));
    });
    
    test('throws exception for unauthenticated user', () async {
      // Test implementation
    });
  });
}
```

**Create**: `test/supabase/functions/learning-recommendations/index_test.ts`

```typescript
Deno.test('analyzeCategoryPerformance calculates correct averages', () => {
  const interviews = [
    { competency_scores: { Python: 80, SQL: 70 } },
    { competency_scores: { Python: 90, SQL: 60 } },
  ];
  
  const result = analyzeCategoryPerformance(interviews);
  
  assertEquals(result.find(c => c.name === 'Python')?.score, 85);
  assertEquals(result.find(c => c.name === 'SQL')?.score, 65);
});
```

**Target**: 80% code coverage

**Success Criteria**:
- ✅ 80%+ test coverage
- ✅ All critical paths tested
- ✅ Edge cases handled

---

#### **Task 3.2: Accessibility** ⏱️ 4-5 hours

**Update**: All widgets with Semantics

```dart
Semantics(
  label: 'Video: ${content.title}',
  hint: 'Double tap to play video',
  button: true,
  child: ContentCard(content: content),
)
```

**Test with**:
- TalkBack (Android)
- VoiceOver (iOS)
- Screen reader

**Success Criteria**:
- ✅ All interactive elements have labels
- ✅ Screen reader navigation works
- ✅ Keyboard navigation supported

---

#### **Task 3.3: Documentation** ⏱️ 2-3 hours

**Create**:
1. `docs/api/LEARNING_RECOMMENDATIONS_API.md` - Edge Function API docs
2. `docs/architecture/PRACTICE_HUB_ARCHITECTURE.md` - Architecture diagram
3. `docs/deployment/PRACTICE_HUB_DEPLOYMENT.md` - Deployment guide

**Update**:
- Add JSDoc comments to Edge Function
- Add DartDoc comments to Dart code
- Update README.md

**Success Criteria**:
- ✅ API fully documented
- ✅ Architecture clearly explained
- ✅ Deployment process documented

---

### **Day 13-14: Performance & Security** (8-10 hours)

#### **Task 3.4: Performance Optimization** ⏱️ 4-5 hours

**Implement**:
1. Lazy loading for content list
2. Image optimization (compress thumbnails)
3. Code splitting
4. Database query optimization

**Add**: Firebase Performance Monitoring

```dart
final trace = FirebasePerformance.instance.newTrace('load_recommendations');
await trace.start();

final recommendations = await aiService.getLearningRecommendations(userId: userId);

await trace.stop();
```

**Success Criteria**:
- ✅ Load time < 2 seconds
- ✅ Smooth scrolling (60 FPS)
- ✅ Memory usage optimized

---

#### **Task 3.5: Security Hardening** ⏱️ 4-5 hours

**Implement**:
1. CORS configuration
2. Input sanitization
3. SQL injection prevention
4. XSS prevention

**Update Edge Function**:
```typescript
const corsHeaders = {
  'Access-Control-Allow-Origin': 'https://yourapp.com', // Specific origin
  'Access-Control-Allow-Methods': 'POST',
  'Access-Control-Allow-Headers': 'authorization, content-type',
};
```

**Success Criteria**:
- ✅ CORS properly configured
- ✅ All inputs sanitized
- ✅ Security audit passed

---

## 🚀 Week 4: Launch Preparation

### **Day 15-16: Integration Testing** (8-10 hours)

**Test Scenarios**:
1. Complete user journey (signup → interview → recommendations → content view)
2. Error scenarios (network failure, timeout, invalid data)
3. Performance under load
4. Cross-platform testing (Android, iOS, Web)

**Success Criteria**:
- ✅ All user journeys work end-to-end
- ✅ No critical bugs
- ✅ Performance meets targets

---

### **Day 17-18: Beta Testing** (8-10 hours)

**Tasks**:
1. Deploy to staging environment
2. Invite 10-20 beta users
3. Collect feedback
4. Fix critical issues
5. Iterate based on feedback

**Success Criteria**:
- ✅ Beta users satisfied
- ✅ Critical bugs fixed
- ✅ Feedback incorporated

---

### **Day 19-20: Production Deployment** (8-10 hours)

**Checklist**:
- [ ] All tests passing
- [ ] Documentation complete
- [ ] Performance targets met
- [ ] Security audit passed
- [ ] Backup plan ready
- [ ] Monitoring configured
- [ ] Deploy to production
- [ ] Monitor for issues
- [ ] Celebrate! 🎉

---

## 📊 Success Metrics

**Week 1**:
- ✅ Database migration deployed
- ✅ 100+ content items added
- ✅ Error handling implemented
- ✅ Progress tracking working
- ✅ Caching implemented

**Week 2**:
- ✅ Free vs. paid logic working
- ✅ Analytics tracking active
- ✅ UI polished
- ✅ No hardcoded values

**Week 3**:
- ✅ 80%+ test coverage
- ✅ Accessibility compliant
- ✅ Documentation complete
- ✅ Performance optimized

**Week 4**:
- ✅ Beta testing complete
- ✅ Production deployed
- ✅ Monitoring active
- ✅ Users happy

---

**Next Action**: Start with Week 1, Day 1 - Deploy database migration!
