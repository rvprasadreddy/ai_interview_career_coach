# InterviPrep: Daily Drill Question Assignment Criteria

This document details how the system "intelligently" selects a Daily Drill question for each user every 24 hours.

---

## 1. The Assignment Strategy (Hierarchy)

The system uses a **Tiered Logic** to ensure the most relevant question is picked, while always having a fallback to prevent "empty states".

### **Execution Priority Table**

| Tier | Priority | Filter Criteria | Purpose |
| :--- | :--- | :--- | :--- |
| **1** | **Primary Match** | `Skills` + `Role` + `Difficulty` + `Gap Focus` | Matches current skills AND addresses identified weaknesses. |
| **2** | **Skill Expansion** | `Skills` + `Role` | Ensures questions remain within the user's domain expertise. |
| **3** | **General Growth** | `Role` + `Difficulty` + `Gap Focus` | Fallback for when all skill-specific questions are exhausted. |
| **4** | **Safety Fallback** | `Role` | Broadest match for the user's career level. |
| **5** | **AI Generation** | **Real-time Generation** | If the Master Bank contains NO questions for the user's niche skill set, the system invokes AI to generate custom questions instantly and assigns one. |

---

## 2. Skill Selection Criteria

The assignment logic distinguishes between fresh users and seasoned practitioners:

### **Scenario A: New User (No Interviews Taken)**
*   **Driver**: `Primary Skill Set`
*   **Source**: The `skills` array defined during onboarding or in the User Profile.
*   **Goal**: Alignment with the user's self-proclaimed expertise.

### **Scenario B: Active User (Interviews Taken)**
*   **Driver**: `Primary Skills` + `Interview Skills`
*   **Source**: Profile Skills + any skills mentioned in previous `interviews.config`.
*   **Goal**: Reinforcing skills the user has actually practiced in simulated "heat of the moment" scenarios.

---

## 3. "Generation on Demand" Logic

If a user specifies a very niche skill (e.g., "Quantum Computing with Q#") and the pre-loaded library has 0 matches:
1.  The system identifies the "Empty State".
2.  The `daily-drill-notifier` Edge Function calls the **AI Content Architect**.
3.  **3 new questions** are generated specifically for that skill level and role.
4.  The questions are stored in the `daily_drill_questions` table for the community.
5.  One of these is assigned to the user immediately.

---

## 4. Dynamic Factors

| Factor | Logic |
| :--- | :--- |
| **Difficulty Level** | Driven by `readiness_score`. Score < 40 = Easy, < 70 = Medium, > 70 = Hard. |
| **Target Role** | Map experience years: <1 (Intern), <3 (Junior), <6 (Mid), <12 (Senior), 12+ (Lead). |
| **Gap Focus** | The system analyzes performance. Categories with <60% success are tagged as `weak_categories` and pushed to the top of selection. |

---

## 5. Deployment Information

*   **Database Function**: `public.get_or_assign_daily_question`
*   **Edge Function**: `daily-drill-notifier` (Action: `get_today_question`)
