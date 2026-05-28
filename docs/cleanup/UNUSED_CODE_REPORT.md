# Unused Code Analysis Report
**Generated on**: 2026-02-17
**Status**: Analysis Complete

This report identifies unused code, imports, and resources in the `ai_interview_coach` project. Removing these items will improve build times, reduce bundle size, and maintain code cleanliness.

## 1. Unused Imports
These imports are declared but never used in their respective files.

| File | Unused Import | Reason |
|------|---------------|--------|
| `lib/features/practice/providers/practice_hub_provider.dart` | `core/services/cache_service.dart` | Cache logic likely moved or removed during refactoring. |
| `lib/features/practice/screens/practice_hub_screen.dart` | `package:shimmer/shimmer.dart` | `SkeletonLoader` widget is used instead, which encapsulates shimmer logic. |
| `lib/features/practice/widgets/knowledge_graph_card.dart` | `package:animate_do/animate_do.dart` | Animations might have been removed or moved to the parent widget. |

## 2. Unused Analytics Methods
The following methods in `AnalyticsService` (lib/core/services/analytics_service.dart) have zero references in the codebase.

| Method Name | Signature | Recommendation |
|-------------|-----------|----------------|
| `trackUpgradePromptView` | `({required String userId, required String location})` | Remove if upgrade prompts are tracked elsewhere or not yet implemented. |
| `trackScreenView` | `({required String screenName, String? screenClass})` | `GoRouter` observer or individual page tracking might be used instead. |

## 3. Unused Generated Code
These are generated method signatures from `build_runner` that are not referenced.

| File | Symbol | Reason |
|------|--------|--------|
| `daily_drill_models.g.dart` | `_$DailyDrillQuestionToJson` | `toJson` support was customized or disabled for this model. |
| `daily_drill_models.g.dart` | `_$PaginationInfoToJson` | Serialization for this model is likely not used (read-only). |

## 4. Temporarily Disabled Code
Code blocks that have been commented out but left for future reference.

| File | Logic | Context |
|------|-------|---------|
| `lib/features/practice/screens/video_player_screen.dart` | Web Video State Listener | Commented out due to `youtube_player_iframe` version issues. labeled with TODO. |

## 5. Other Findings
*   `test/widget_test.dart`: Contains a reference to `MyApp` which does not exist (should be `App`). This test file is currently broken.

## Action Plan
1.  **Safe to Delete**:
    *   Remove all unused imports listed in Section 1.
    *   Remove unused `AnalyticsService` methods if no `upgrade_prompt` feature is imminent.
    *   Fix `test/widget_test.dart` to point to `App` class.
2.  **Keep for Now**:
    *   Generated code warnings (these will disappear if the models are updated/regenerated with different settings, or can be ignored).
    *   Commented out video player code (needs to be fixed, not deleted).
