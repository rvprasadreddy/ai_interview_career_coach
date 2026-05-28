# Android Release & Deployment Guide

## 1. Generating the Release Bundle
Run the following command to generate the `.aab` file:
```powershell
flutter build appbundle --release
```
**Output Path:** `build\app\outputs\bundle\release\app-release.aab`

## 2. Google Play Store Upload Steps
1. **Console Access**: Log in to [Google Play Console](https://play.google.com/console/).
2. **Production Release**: Navigate to **Release** > **Production**.
3. **Create Release**: Click **Create new release** at the top right.
4. **Bundle Upload**: Upload the `app-release.aab` file generated in Step 1.
5. **Rollout**: Review and start rollout to production.

## 3. In-App Purchase (IAP) Configuration
These IDs must match the `IapProductIds` class in code exactly.

### Subscriptions (Monetize > Subscriptions)
| Name | Product ID | Frequency | Price (Suggested) |
| :--- | :--- | :--- | :--- |
| Pro Monthly | `interviprep_pro_monthly` | Monthly | ₹299 |
| Pro Quarterly | `interviprep_pro_quarterly` | Quarterly | ₹699 |

### One-Time Products (Monetize > In-app products)
| Name | Product ID | Type | Price (Suggested) |
| :--- | :--- | :--- | :--- |
| JD Precision Mock | `interviprep_addon_jd_mock` | Consumable | ₹99 |
| Resume Rewrite Pro | `interviprep_addon_resume_rewrite` | Non-Consumable | ₹999 |
| Ad-Free Experience | `interviprep_addon_ads_free` | Non-Consumable | ₹99 |
| Elite Tools Pack | `interviprep_addon_elite_tools` | Non-Consumable | ₹1,499 |

## 4. Post-Deployment Checklist
- [x] Run `flutter build appbundle --release`.
- [ ] Upload `.aab` to Google Play Console.
- [ ] Create/Activate all IAP Product IDs in Monetize section.
- [ ] Verify `GOOGLE_SERVICE_ACCOUNT_JSON` is set in Supabase Secrets.
- [ ] Test purchase flow with a "License Test" account.


## Release Notes
Welcome to InterviPrep — Your AI-powered interview coach! 

In this initial release, we've built a complete ecosystem to help you land your dream job:

• AI Interview Coach: Practice with natural-sounding AI voices, customizable accents, and adaptive personalities ranging from supportive to challenging.
• Voice Interviews: Upload your resume or paste a specific Job Description to generate highly relevant, industry-specific mock interviews.
• Real-time Interview Feedback: Receive instant scores, competency breakdowns, and "Deep Dive" insights after every session to identify exactly where to improve.
• Performance Trends: Track your progress with detailed visual charts and historical analysis of your interview performance over time.
• Practice Hub: Learning content recommendations( video, articles, faqs) based on the interview feedback & weak areas.
• Daily Drills: Practice with daily drills to improve your interview skills.
• Pro Experience: Unlock unlimited interviews, advanced AI analysis, and a premium ad-free experience.

Prepare with confidence. Practice with InterviPrep.
