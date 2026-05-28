# Optimized YouTube Pipeline — Implementation Plan
**Target: < 10 seconds | Zero quota waste | Full parallelization**

---

## Core Principle: Cache-First Gap Analysis

The entire pipeline is driven by per-type gaps after cache lookup.
Never call YouTube API or AI generation for content already in cache.

```
Gap = ITEMS_PER_TYPE - cached[type].length

videosGap   = max(0, 2 - cached.video.length)   // 0 = skip, 1 = fetch 1, 2 = fetch 2
faqsGap     = max(0, 2 - cached.faq.length)     // 0 = skip, 1 = gen 1, 2 = gen 2
articlesGap = max(0, 2 - cached.article.length)  // 0 = skip, 1 = gen 1, 2 = gen 2
```

---

## The Optimized Per-Skill Pipeline

All three lanes run **simultaneously** via `Promise.all` after the cache lookup.

```
──────────────────────────────────────────────────────────
PHASE 1: Cache Lookup (ALL skills in parallel)
──────────────────────────────────────────────────────────
  Promise.all(skills.map(skill => cacheQuery(skill)))
  Each returns { video: [], faq: [], article: [] }
  
  Time: ~200ms (parallel DB queries for all skills)

──────────────────────────────────────────────────────────
PHASE 2: Per-skill gap fill (3 lanes in parallel per skill)
──────────────────────────────────────────────────────────
  For each skill (all skills already running in parallel):
  
    const [newVideos, newFaqs, newArticles] = await Promise.all([
      videosGap > 0 ? videoLane(skill, videosGap)   : [],
      faqsGap > 0   ? faqLane(skill, faqsGap)       : [],
      articlesGap > 0 ? articleLane(skill, articlesGap) : [],
    ]);

──────────────────────────────────────────────────────────
PHASE 3: Merge + Insert (per skill)
──────────────────────────────────────────────────────────
  final = [...cached.video, ...newVideos,
           ...cached.faq,   ...newFaqs,
           ...cached.article, ...newArticles]
  
  Insert only NEW items (cached items already in DB, no re-insert needed)

──────────────────────────────────────────────────────────
PHASE 4: Cross-skill deduplication + Persist
──────────────────────────────────────────────────────────
  After all skills complete, deduplicate by URL across skill results
  Persist final to user_learning_recommendations
```

---

## Video Lane — Detailed Flow

```
videoLane(skill, count, excludeUrls, weaknesses):

  Step 1: AI generates 3 search queries (500ms)
    Parallel with faqLane and articleLane — no waiting needed.
    Weakness context injected for learning-recommendations.

  Step 2: YouTube search.list — ALL 3 queries in parallel (800ms)
    Promise.all([search(q1), search(q2), search(q3)])
    Each returns up to 5 results = up to 15 candidates total
    Cost: 3 × 100 = 300 units per skill with gap

  Step 3: Batch videos.list enrichment (200ms)
    Collect all unique videoIds from step 2 (up to 15)
    ONE single videos.list call with all IDs → duration, publishedAt
    Cost: 1 unit (regardless of how many IDs, up to 50)
    Filter: duration 3–90 min, publishedAt > 2019

  Step 4: oEmbed verification — ALL candidates in parallel (400ms)
    Promise.all(candidates.map(v => verifyYouTubeUrl(v)))
    Simplified: only <iframe> check + channel blocklist (no keyword guard)
    Take first `count` that pass

  Total video lane: ~1.5s (steps 2-4 are fast; step 1 runs in parallel with FAQ/article)
```

---

## FAQ Lane — Parallel with Video Lane

```
faqLane(skill, count):
  Single OpenAI call requesting `count` FAQ items
  Validate body has ≥ MIN_FAQ_QUESTIONS Q&A pairs
  If validation fails → retry once with higher temperature
  
  Time: ~2-3s (runs in parallel with video lane, no extra wall-clock cost)
```

---

## Article Lane — Parallel with Video Lane

```
articleLane(skill, count):
  Single OpenAI call requesting `count` article items
  Validate body length ≥ MIN_ARTICLE_BODY_CHARS
  If validation fails → retry once
  
  Time: ~2-3s (runs in parallel, no extra wall-clock cost)
```

---

## Timing Breakdown (Target: < 10s)

| Phase | Duration | Notes |
|---|---|---|
| Cache lookup (all skills parallel) | 200ms | Single round-trip to Supabase |
| AI search query generation | 500ms | Runs in parallel with FAQ/article |
| YouTube search.list (3 queries parallel) | 800ms | Only for skills with videosGap > 0 |
| videos.list enrichment (batched) | 200ms | 1 API call for all IDs |
| oEmbed verification (parallel) | 400ms | All candidates at once |
| FAQ + Article AI generation | 2.5s | Runs in parallel with video lane |
| DB insert (new items only) | 200ms | Upsert with onConflict='url' |
| Persist to user_learning_recommendations | 100ms | Single upsert |
| **Total (worst case, all gaps)** | **~4.5s** | Parallelism eliminates sequential stacking |
| **Total (partial cache hit)** | **~3s** | Fewer gaps = fewer API calls |
| **Total (full cache hit all skills)** | **~0.5s** | Cache only, no external calls |

---

## YouTube Quota Optimization

### Quota cost per user session

| Scenario | Quota used |
|---|---|
| Full cache hit (all skills) | 0 units |
| 1 skill needs videos (3 queries + 1 enrich) | 301 units |
| 2 skills need videos | 602 units |
| 4 skills need videos (worst case, new user) | 1204 units |
| Daily capacity at 10,000 units | ~8 new users fully uncached |
| Daily capacity with warm cache (1-2 skills) | ~16-33 sessions |

> [!IMPORTANT]
> **Quota Guard**: Before each YouTube search call, check if quota is likely exhausted
> (track usage in Supabase `system_settings` table or detect 403 quotaExceeded error).
> On quota exhaustion: fall back to AI-generated IDs + oEmbed, never block the user.

### Quota allocation priority
1. `initial-learning-recommendations` (new user onboarding) — highest priority
2. `learning-recommendations` (existing user refresh) — lower priority, uses more cache

---

## Complete Missing Scenarios (Updated)

### 🔴 Critical — Must Fix

#### S1: Partial cache merge (THE core fix)
- **Problem**: Currently ignores partial cache, fetches full count from YouTube
- **Fix**: `videosGap = ITEMS_PER_TYPE - cached.video.length`, pass exact gap to video lane
- **Exclude cached URLs** from YouTube search to prevent duplicates

#### S2: Simplified oEmbed (remove keyword relevance guard)
- **Problem**: Channels like "Confluent" (official Kafka channel) fail the keyword check because
  "confluent" isn't in our genericTechWords list
- **Fix**: Remove `isTechnicallyRelevant` check. Only check `<iframe>` + blocklist.
- **Why safe**: YouTube API search relevance replaces this check entirely

#### S3: videos.list enrichment (real duration)
- **Problem**: `duration_minutes = 15` is hardcoded for every video
- **Fix**: Batch videos.list call after search, parse ISO 8601 duration
- **Filter**: Reject duration < 3 min (teasers) or > 90 min (full courses)
- **Bonus**: Filter publishedAt < 2020 for tech skills (content too outdated)

#### S4: YouTube API quota exhausted (403)
- **Problem**: Silent failure when quota is hit, falls through incorrectly
- **Fix**: Detect `data.error?.errors[0]?.reason === 'quotaExceeded'`
  → Log warning → Fall back to AI-generated IDs + oEmbed
  → AI IDs still fail? → YouTube search link (never behavioral fallback)

#### S5: FAQ/Article generation runs in parallel with Video lane
- **Problem**: Currently sequential (videos then FAQs then articles)
- **Fix**: `Promise.all([videoLane, faqLane, articleLane])` — saves 3-5 seconds

---

### 🟡 High — Should Fix

#### S6: Experience-level cache filter
- **Problem**: Expert user gets foundation-level cached videos
- **Fix**: Add `.in('experience_level', [userLevel, 'any'])` to cache query

#### S7: Weakness-driven sub-topic search queries (learning-recommendations)
- **Problem**: `missedTopics = ["kafka consumer groups"]` → generic "kafka tutorial" query
- **Fix**: Extract the specific sub-concept and inject into query:
  `"kafka consumer group rebalancing tutorial"` not just `"kafka tutorial"`
- **learning-recommendations only**: initial-recs has no interview history

#### S8: Cross-skill URL deduplication after parallel processing
- **Problem**: Skills [Kafka, Data Engineering] processed in parallel may both find
  the same Confluent video
- **Fix**: After all skill results are collected, deduplicate by URL keeping first occurrence
  (highest relevance skill wins)

#### S9: Retry uses different search queries
- **Problem**: If YouTube search returns 0 embeddable results, retry uses same queries = same results
- **Fix**: On retry, broaden to parent domain terms (use `generateTechnicalPivotTerms`)
  and pass to a new YouTube search (different search strings = different results)

#### S10: Cache staleness for video URLs
- **Problem**: Videos cached 6+ months ago may now be deleted/private
- **Fix**: Add `is_active = false` marking to old cached videos when oEmbed returns 404
  (this happens naturally when we verify cached videos before serving them)
- **Or**: Short-circuit — don't verify cached videos on every request. Instead mark them
  stale on first failure when a user actually clicks to play.

---

### 🟢 Medium — Nice to Have

#### S11: Quota sharing strategy (both EFs)
- **Option A (simple)**: Separate YouTube API key per EF (2 Google Cloud projects)
- **Option B**: Quota counter in `system_settings` DB table, checked before each search
- **For now**: `initial-learning-recommendations` gets priority (onboarding must succeed)

#### S12: `forceVideos` / refresh in learning-recommendations
- **Current**: Refresh triggers new AI generation, doesn't re-search YouTube
- **Fix**: On refresh, run videoLane with fresh queries + excludeUrls from current recommendations
  to guarantee new, non-duplicate content

#### S13: YouTube search results include channel videos (not official)
- **Problem**: "kafka tutorial" might return a random person's low-view video
- **Fix**: Add `videoDuration: 'medium'` (already done) but also consider `order: 'viewCount'`
  for well-established skills (more views = more trusted for mainstream tech)
- **Counter**: For niche skills, `order: 'relevance'` is better than viewCount
- **Smart**: Use viewCount ordering only when skill is in `MAINSTREAM_SKILLS` list

---

## Proposed `processSkill` Structure (Pseudocode)

```typescript
async function processSkill(skill, cached, gaps, context):

  const { videosGap, faqsGap, articlesGap } = gaps;

  // All 3 lanes start simultaneously — critical parallelism
  const [newVideos, newFaqs, newArticles] = await Promise.all([

    // LANE 1: YouTube API (only if gap exists)
    videosGap > 0
      ? videoLane(skill, videosGap, context.excludeUrls, context.weaknesses)
      : Promise.resolve([]),

    // LANE 2: AI FAQ generation (only if gap exists)
    faqsGap > 0
      ? faqLane(skill, faqsGap, context)
      : Promise.resolve([]),

    // LANE 3: AI Article generation (only if gap exists)
    articlesGap > 0
      ? articleLane(skill, articlesGap, context)
      : Promise.resolve([]),
  ]);

  // Merge: cached items first (higher priority), new items fill gaps
  const finalVideos   = [...cached.video,   ...newVideos  ].slice(0, ITEMS_PER_TYPE);
  const finalFaqs     = [...cached.faq,     ...newFaqs    ].slice(0, ITEMS_PER_TYPE);
  const finalArticles = [...cached.article, ...newArticles].slice(0, ITEMS_PER_TYPE);

  // Insert ONLY new items — cached items already persisted
  const toInsert = [...newVideos, ...newFaqs, ...newArticles].map(row => ({
    ...row,
    skill_tags: [skill, ...otherSkills].slice(0, 3),
    experience_level: userLevel,
    avg_rating: 4.8,
  }));

  if (toInsert.length > 0) {
    await supabase.from('learning_content').upsert(toInsert, { onConflict: 'url' });
  }

  // Return combined set with recommendation metadata
  return [...finalVideos, ...finalFaqs, ...finalArticles].map(item => ({
    ...item,
    relevance_score: 0.95,
    reason: `Curated for ${skill}`,
    status: 'not_started',
    progress_percentage: 0,
  }));
```

---

## Changes Summary per File

### `initial-learning-recommendations/index.ts`
1. Move cache lookup to Phase 1: all skills queried in parallel before processing begins
2. Compute per-type gaps from cache results
3. Replace sequential `processSkill` internals with `Promise.all([videoLane, faqLane, articleLane])`
4. Implement `videoLane`: parallel YouTube queries → batch videos.list → parallel oEmbed
5. Remove relevance guard from `verifyYouTubeUrl`
6. Add quota exhaustion detection (403 handler)
7. Cross-skill URL deduplication after `Promise.all(skills.map(...))`

### `learning-recommendations/index.ts`
1. Same videoLane + parallel FAQ/article generation
2. Inject specific sub-topics from `missedTopics` into search queries (not just skill name)
3. Add experience-level filter to cache lookup
4. Remove relevance guard from `verifyYouTubeUrl`
5. Retry path uses `generateTechnicalPivotTerms` for different search terms

### Both EFs: `verifyYouTubeUrl`
- Remove: `isTechnicallyRelevant` keyword check (lines ~155-171 in initial-recs)
- Keep: `<iframe>` check + channel blocklist check + `#external` fallback
