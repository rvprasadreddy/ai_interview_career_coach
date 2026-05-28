# Gap Analysis: Old vs New Edge Functions

## initial-learning-recommendations (v19 → v20)

### 🔴 Critical Gaps — Must Fix

#### GAP-1: `buildPrompt` generates videos AND faqs AND articles together (old), now split
**Old:** `buildPrompt` asked AI for all 3 types in one call (video + faq + article). The FAQ/article AI
prompt was specifically crafted with domain guidance, excluded URLs, sequence_index, retry notes etc.
**New:** FAQ lane and Article lane call `callOpenAI(key, buildPrompt(...))` using the same `buildPrompt`
that was designed for the mixed-type response. The AI receives a prompt asking for all 3 types but
**the video items returned are now silently discarded** (filtered out by `.filter(it.type === 'faq')`).
This means **2 of every 3 AI output tokens are wasted** generating videos we throw away.

**Fix:** Add a dedicated FAQ-only and Article-only prompt variant to `buildPrompt`.

---

#### GAP-2: `buildPrompt` in new initial-recs is missing `domainGuidance` and `sequencing` sections
**Old `buildPrompt` included:**
- `SEQUENTIAL LEARNING` instruction (sequence_index: 1=Foundation, 2=Application, 3=Advanced)
- `domainGuidance` block with industry-specific channel hints and niche pivot rules
- `excludeNote` (injected excluded URLs into prompt)
- `retryNote` with Concept Substitution instruction when niche skills fail

**New `buildPrompt`:**
```typescript
// Only has: basic role/industry/level context + behavioural prohibition + excludeNote + retryNote
// Missing: sequencing instructions, domainGuidance, channel suggestions
```
**Impact:** AI generates content without sequence_index progression and without domain-specific channel guidance.

---

#### GAP-3: Stage 3b (Domain Broadening via `generateTechnicalPivotTerms`) — logic broken in new version
**Old:** If `videoCount < 2`, used AI pivot terms then called `buildPrompt` on pivot term + verified via oEmbed.
**New:** Stage 3b calls `videoLane` with `skill: term` but `videoLane` calls `generateYouTubeSearchQueries`
which already handles niche pivoting. This double-pivots and may return empty if the term is too abstract.
**More importantly:** The Stage 3b in new code only fires if `resolvedVideos.length === 0` (zero),
not `< ITEMS` (less than 2). So if YouTube API returns 1 video out of needed 2, Stage 3b never runs.

---

#### GAP-4: Stage 3c (Targeted retry with pivot prompt) missing in new version
**Old Stage 3c:** If 0 videos after Stage 3b, retried with a broadened `buildPrompt(skill, ..., isRetry=true)`.
**New:** Goes directly to YouTube search link fallback. The AI retry path for zero-result skills is gone.

---

#### GAP-5: `skill_tags` assignment uses wrong field name in new `processSkill`
**Old:** `skill_tags: skillSpecificTags` (= `[skill, ...otherSkills].slice(0, 3)`)
**New:** `skill_tags: skillTags` — the variable is named correctly but check line 197:
```typescript
const skillTags = [skill, ...skills.filter(s => s !== skill)].slice(0, 3);
```
This is correct. ✅ No gap here — just verifying.

---

#### GAP-6: `experience_level` mapping missing `'any'` for `foundation` level in insert
**Old:** `experience_level: experienceLevel === 'foundation' ? 'any' : experienceLevel`
**New:** Same logic present on line ~197. ✅ No gap.

---

### 🟡 Medium Gaps

#### GAP-7: Old `processSkill` returned cached items with `id` from DB for roadmap linking
**Old:** Full cache hit returned `{...item, relevance_score, reason, status, progress_percentage}` — items had DB `id`.
**New:** Same pattern. ✅ No gap.

#### GAP-8: `avg_rating: 4.8` applied to inserts ✅ present in new version.

#### GAP-9: Old version also cached `searchSkill` variant (`.ilike.${searchSkill}`)
**Old cache query:**
```sql
.or(`category.ilike.${skill},category.ilike.${searchSkill},skill_tags.cs.{${skill}},skill_tags.cs.{${searchSkill}}`)
```
**New cache query:**
```sql
.or(`category.ilike.${skill},category.ilike.${searchSkill},skill_tags.cs.{${skill}}`)
```
Missing: `skill_tags.cs.{${searchSkill}}` — minor cache miss for dot-notation skills.

---

### ✅ Correctly Preserved
- Run-once protection (`already_initialized` check)
- Profile null guard (404)
- FALLBACK_SKILLS for no-skills users
- Parallel profile + progress fetch (`Promise.all`)
- Cross-skill `Promise.all` processing
- YouTube search link last resort (no behavioral injection)
- `upsert onConflict: 'url'`
- `globalSeenUrls` cross-skill dedup

---

## learning-recommendations (v122 → v123)

### 🔴 Critical Gaps — Must Fix

#### GAP-10: Behavioral fallback `verifyYouTubeUrl` call uses old 3-arg signature
**Old (line 957):**
```typescript
const safeUrl = await verifyYouTubeUrl(v.url, 'Behavioral', userId);
```
**New:** `verifyYouTubeUrl` is imported from `_shared/youtube.ts` with signature `(url, userId, log)`.
**But the old call `verifyYouTubeUrl(v.url, 'Behavioral', userId)` is still in the file!**
This will compile but pass `'Behavioral'` as `userId` and `userId` as `log` — **runtime bug**.

**Fix:** Change to `verifyYouTubeUrl(v.url, uid, log)`.

---

#### GAP-11: `validateArticleBody` used in `generateAIContent` but not in `_shared`
**Old:** `validateArticleBody` (separate from `validateFaqBody`) checked article body length ≥ 1500 chars.
**New:** `_shared/ai-helpers.ts` only has `validateFaqBody`. Articles in `generateAIContent` are validated
by `validateFaqBody` which uses FAQ Q&A pattern matching — **articles will always fail this check**.

Looking at current `generateAIContent` article validation:
```typescript
// Does it use validateArticleBody? Let's check...
```
Actually `validateArticleBody` is still defined locally in `learning-recommendations/index.ts` (line 131).
✅ No gap IF `generateAIContent` still references it. Need to verify.

---

#### GAP-12: `probeArticleUrl` removed from `_shared` but still used in `generateAIContent`
**Old:** `probeArticleUrl` checked article URL accessibility. Still defined locally (line 136). ✅ No gap.

---

#### GAP-13: `insertValidatedItems` sets `experience_level: 'any'` for ALL items
**Old:** Same behavior. ✅ No gap.

---

#### GAP-14: `avg_rating` sort in `pickMultiplePerType`
**Old line 411:** `candidates.sort((a, b) => (b.avg_rating ?? 4.5) - (a.avg_rating ?? 4.5))`
This is still in the current file. ✅ No gap.

---

### 🟡 Medium Gaps

#### GAP-15: `generateAIContent` — video block now calls `videoLane` but misses `skill_tags` on returned rows
**Old:** `insertValidatedItems` stamped `skill_tags` from the outer `skills` array.
**New:** `videoLane` rows have `tags: [skill.toLowerCase(), 'tutorial']` but no `skill_tags` field.
`insertValidatedItems` adds `experience_level: 'any', avg_rating: 4.8` but not `skill_tags`.
**Impact:** Videos inserted by `videoLane` path have no `skill_tags` → `pickMultiplePerType` cache lookup
by `skill_tags.cs.{skill}` won't find them on next request.

---

#### GAP-16: `verifyYouTubeUrl` in behavioral fallback (GAP-10) also called in Stage 3c
**Old line 957:** `verifyYouTubeUrl(v.url, 'Behavioral', userId)` — wrong args with new import.

---

### ✅ Correctly Preserved
- Full `buildRecs` 5-stage pipeline (missed topics, weak areas, profile skills, strong areas, hardening)
- `hydrateContent` skeleton resolution
- `insertValidatedItems` DB insert with fallback
- Smart cache bypass on refresh flags
- `buildTrend`, `buildStats`, `calcJourney`, `analyzeCategoryPerformance`
- `fetchInsights` for missed/covered topics
- Flutter response shape (overallReadiness, journey, stats, readinessTrend, scenario, message)
- Parallel profile + interviews + progress fetch

---

## Priority Fix Order

| # | Gap | File | Severity | Fix |
|---|---|---|---|---|
| 1 | GAP-10 | learning-recs | 🔴 Runtime bug | `verifyYouTubeUrl(v.url, uid, log)` |
| 2 | GAP-1 | initial-recs | 🔴 Token waste | Add faq-only/article-only prompt variants |
| 3 | GAP-2 | initial-recs | 🟡 Quality | Add sequencing + domainGuidance to buildPrompt |
| 4 | GAP-3 | initial-recs | 🟡 Logic | Stage 3b: trigger on `< ITEMS` not `=== 0` |
| 5 | GAP-15 | learning-recs | 🟡 Cache miss | Add `skill_tags` to videoLane rows |
| 6 | GAP-9 | initial-recs | 🟢 Minor | Add `skill_tags.cs.{${searchSkill}}` to cache query |
| 7 | GAP-4 | initial-recs | 🟢 Minor | Stage 3c AI retry for zero-result skills |
