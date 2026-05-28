# Practice Hub Recommendation Engine Documentation

The Practice Hub is powered by a sophisticated backend engine implemented as a Supabase Edge Function. It analyzes user profile data and interview performance to generate a personalized learning roadmap.

## 1. Core Logic: The 4 Scenario Model

The engine operates on four distinct scenarios to ensure every user gets value, regardless of their history:

### Scenario 1: Profile-Only (Cold Start)
*   **Trigger**: User has NO completed interviews.
*   **Logic**: Matches Content Categories against the skills listed in the user's profile.
*   **Difficulty**: Maps years of experience to entry-level Difficulty (e.g., 0-2 yrs -> Foundation/Intermediate).
*   **Goal**: Get the user started with relevant "Foundation" material.

### Scenario 2: Data-Driven (Main Mode)
*   **Trigger**: User has at least one completed interview.
*   **Logic**: 
    1.  Identifies "Weak Areas" (Category score < 70%).
    2.  Prioritizes content for these weak areas.
    3.  Boosts "Critical" and "Essential" priority content.
    4.  Fills remaining slots with "Growth Content" (Advanced topics for areas where the user is already strong).
*   **Relevance Scoring**: Items are ranked by a relevance score (0.0 to 1.0) based on how well they match the user's specific performance gaps.

### Scenario 3: AI-Augmented (Backup)
*   **Trigger**: Not enough content found in the database for a specific gap.
*   **Logic**: Uses OpenAI (GPT-4) to generate a customized "Roadmap Summary" or specific FAQ/Article snippets in real-time.

### Scenario 4: Central Repository Matching
*   **Logic**: A strict matching filter that aligns content `experience_level` with user trajectory.

---

## 2. Refresh & Regeneration Logic

To balance performance with fresh insights, we use a hybrid model:

### Auto-Fetch (Passive)
*   Triggered whenever the Practice Hub is opened.
*   **Behavior**: Checks if "New Interview Data" is available since the last roadmap was generated.
*   If new data exists: Regenerates automatically.
*   If NO new data: Serves cached recommendations for sub-second load times.

### Manual Refresh (Active)
*   Triggered by "Pull-to-Refresh" on the mobile app.
*   **Bypass Cooldown**: Manual refresh *always* triggers regeneration IF:
    1.  The user has moved to a new mastery stage (e.g. Foundation -> Intermediate).
    2.  The time-based cooldown has expired (Free: 24h, Paid: 6h).
*   **Response**: Provides a specialized success message ("Your personalized roadmap has been successfully refreshed!") so the UI can confirm the action.

---

## 3. Analytics Integration

### Personalized Analysis
*   **Strengths**: Top 3 categories with scores >= 80%. (Fallback: Top scores >= 70% if no 80%+ exist).
*   **Weaknesses**: Top 3 categories with scores < 70%.

### Learning Impact
*   Tracks the "Overall Readiness Score" over the last 5 interviews.
*   Displays a trend line showing how the user's readiness is improving as they consume the recommended content.

### Knowledge Graph
*   A spider chart visualizing competency across different dimensions (DSA, Behavioral, System Design, etc.).

---

## 4. Identified Gaps & Future Improvements

1.  **Content Variety**: Currently heavily weighted towards Videos and Articles; more interactive "Drills" can be integrated.
2.  **Cross-Platform Viewing**: Articles from external domains (Medium, LinkedIn) sometimes block iframes (X-Frame-Options); we handle this with an "Open External" fallback.
3.  **Real-time Tracking**: Progress is tracked on content close; deeper "Active Time" tracking could improve recommendation weighting.
