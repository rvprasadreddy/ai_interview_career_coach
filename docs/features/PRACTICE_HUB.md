# Practice Hub Feature Documentation

## Overview
The Practice Hub is an AI-powered learning management system within InterviPrep that provides personalized learning paths based on actual interview performance. It leverages a feedback loop between the Interview Engine and a curated content library.

## Core Components

### 1. Recommendation Engine (Supabase Edge Function)
- **Logic**: Analyzes the last 10 interviews to identify competency gaps.
- **Matching**: Matches categories with scores below 70% to "Foundation" or "Intermediate" content. Matches categories above 85% to "Expert" content.
- **Tiers**:
    - **Free**: One-time recommendation generation. Static content until upgrade.
    - **Pro**: Dynamic refresh. Unlimited recommendations. Advanced analytics.
- **Security**: UUID validation, RLS enforcement, and service-role hardening.

### 2. Knowledge Graph (Spider Chart)
- **Widget**: `KnowledgeGraphCard`
- **Visualization**: Radar chart showing proficiency across all analyzed skills (Python, SQL, System Design, etc.).
- **Data Source**: `allCategoryScores` returned by the recommendation engine.

### 3. Discovery & Search
- **Search**: Targeted keyword search across all recommended topics.
- **Topic Clusters**: Content is grouped by category (e.g., "PYTHON", "SQL") to help users focus on specific domains.
- **Filters**: Real-time filtering by difficulty level (Foundation, Expert) and content format (Video, FAQ, Article).

### 4. Progress Persistence
- **Table**: `user_learning_progress`
- **Status**: `not_started`, `in_progress`, `completed`.
- **Sync**: Automatically marks content as `in_progress` when opened and `completed` when finished (videos) or manually marked (articles).

## Implementation Details

### Tier Management
| Feature | Free Tier | Pro Tier |
|---------|-----------|----------|
| Max Recommendations | 10 | 20 |
| Dynamic Refresh | No (Cached) | Yes (Real-time) |
| Analytics | Basic | Advanced (Knowledge Graph) |
| Content Types | Videos only | All (Videos, Blogs, FAQs) |

### Engineering Standards
- **Logging**: Structured JSON-like logging in Edge Functions.
- **Performance**: Shimmer skeletons for all async operations.
- **Accessibility**: Semantic labels for screen readers.
- **Internationalization**: Centralized string keys (In-progress).

## API Reference
- **Endpoint**: `learning-recommendations`
- **Method**: `POST`
- **Payload**: `{ "userId": "UUID" }`
