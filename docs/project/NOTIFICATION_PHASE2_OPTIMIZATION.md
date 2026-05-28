# Phase 2 Optimization Strategy - Achieving Target KPIs

## 🎯 Target KPIs

| Metric | Target | Current Baseline | Strategy |
|--------|--------|------------------|----------|
| **Delivery Rate** | > 95% | ~85% (typical) | Token health monitoring, retry logic, multi-channel fallback |
| **Click-Through Rate** | > 15% | ~5-8% (typical) | A/B testing, personalization, optimal timing |
| **Completion Rate Lift** | > 10% | Baseline TBD | Smart reminders, gamification, social proof |

---

## 📈 Strategy 1: Maximize Delivery Rate (> 95%)

### Current Implementation ✅
- Batch processing with rate limiting
- Automatic token invalidation
- Error logging and retry tracking

### Additional Optimizations 🚀

#### 1.1 Token Health Monitoring
```sql
-- Create token health score view
CREATE OR REPLACE VIEW user_fcm_token_health AS
SELECT 
    uft.user_id,
    uft.fcm_token,
    uft.device_type,
    uft.is_active,
    COALESCE(success_rate.rate, 100) as success_rate,
    COALESCE(recent_failures.count, 0) as recent_failures,
    CASE 
        WHEN COALESCE(success_rate.rate, 100) >= 90 THEN 'healthy'
        WHEN COALESCE(success_rate.rate, 100) >= 70 THEN 'degraded'
        ELSE 'unhealthy'
    END as health_status
FROM user_fcm_tokens uft
LEFT JOIN (
    SELECT 
        fcm_token,
        ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'sent') / COUNT(*), 2) as rate
    FROM notification_logs
    WHERE created_at >= now() - INTERVAL '7 days'
    GROUP BY fcm_token
) success_rate ON uft.fcm_token = success_rate.fcm_token
LEFT JOIN (
    SELECT fcm_token, COUNT(*) as count
    FROM notification_logs
    WHERE status = 'failed' AND created_at >= now() - INTERVAL '24 hours'
    GROUP BY fcm_token
) recent_failures ON uft.fcm_token = recent_failures.fcm_token;
```

#### 1.2 Smart Retry Logic
- **Immediate retry**: For network errors (1 retry)
- **Delayed retry**: For rate limit errors (exponential backoff)
- **Token refresh**: For invalid token errors (request new token)

#### 1.3 Multi-Channel Fallback
```sql
-- Add email notification as fallback
ALTER TABLE user_drill_progress 
ADD COLUMN IF NOT EXISTS enable_email_fallback BOOLEAN DEFAULT false;

-- Track fallback attempts
ALTER TABLE notification_logs
ADD COLUMN IF NOT EXISTS fallback_channel TEXT,
ADD COLUMN IF NOT EXISTS fallback_sent_at TIMESTAMPTZ;
```

#### 1.4 Delivery Confirmation Tracking
```sql
-- Add delivery confirmation webhook
ALTER TABLE notification_logs
ADD COLUMN IF NOT EXISTS fcm_message_id TEXT,
ADD COLUMN IF NOT EXISTS delivery_confirmed BOOLEAN DEFAULT false,
ADD COLUMN IF NOT EXISTS delivery_confirmed_at TIMESTAMPTZ;

-- Index for quick lookups
CREATE INDEX IF NOT EXISTS idx_notification_logs_message_id 
ON notification_logs(fcm_message_id) WHERE fcm_message_id IS NOT NULL;
```

---

## 🎨 Strategy 2: Maximize Click-Through Rate (> 15%)

### Current Implementation ✅
- Personalized streak count
- Emoji usage for engagement
- Different colors for morning vs reminder

### Additional Optimizations 🚀

#### 2.1 A/B Testing Framework
```sql
-- Create A/B test variants table
CREATE TABLE IF NOT EXISTS notification_ab_tests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    test_name TEXT NOT NULL,
    variant_name TEXT NOT NULL,
    notification_type TEXT NOT NULL,
    
    -- Notification content
    title_template TEXT NOT NULL,
    body_template TEXT NOT NULL,
    
    -- Targeting
    is_active BOOLEAN DEFAULT true,
    traffic_percentage INT DEFAULT 50 CHECK (traffic_percentage BETWEEN 0 AND 100),
    
    -- Performance tracking
    impressions INT DEFAULT 0,
    clicks INT DEFAULT 0,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(test_name, variant_name)
);

-- Track which variant was shown to each user
ALTER TABLE notification_logs
ADD COLUMN IF NOT EXISTS ab_test_variant TEXT;
```

**Variant Examples**:

| Variant | Morning Title | Morning Body |
|---------|---------------|--------------|
| A (Control) | 🎯 Your Daily Drill is Ready! | A new question awaits. Build your interview confidence today! |
| B (Urgency) | ⏰ Today's Challenge is Live! | Don't miss out! Your personalized question is waiting. |
| C (Social Proof) | 🔥 Join 1,000+ Users Today! | Complete your daily drill and stay ahead of the competition. |
| D (Streak Focus) | 💪 Keep Your {X}-Day Streak! | Your next question is ready. Don't break the momentum! |

#### 2.2 Personalization Engine
```sql
-- User engagement profile
CREATE TABLE IF NOT EXISTS user_notification_profile (
    user_id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    
    -- Engagement patterns
    preferred_notification_time TIME,
    avg_time_to_click INTERVAL,
    best_performing_variant TEXT,
    
    -- Behavioral data
    total_notifications_received INT DEFAULT 0,
    total_clicks INT DEFAULT 0,
    click_through_rate DECIMAL(5,2),
    
    -- Preferences learned from behavior
    responds_to_urgency BOOLEAN DEFAULT false,
    responds_to_social_proof BOOLEAN DEFAULT false,
    responds_to_streak_focus BOOLEAN DEFAULT false,
    
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### 2.3 Dynamic Content Generation
```typescript
// In Edge Function - Smart content selection
function generatePersonalizedNotification(user: UserProfile): NotificationPayload {
    const variants = {
        urgency: {
            title: '⏰ Today\'s Challenge is Live!',
            body: 'Don\'t miss out! Your personalized question is waiting.',
        },
        social_proof: {
            title: '🔥 Join 1,000+ Users Today!',
            body: 'Complete your daily drill and stay ahead of the competition.',
        },
        streak_focus: {
            title: `💪 Keep Your ${user.current_streak}-Day Streak!`,
            body: 'Your next question is ready. Don\'t break the momentum!',
        },
        achievement: {
            title: '🏆 You\'re on Fire!',
            body: `${user.current_streak} days strong! Let's make it ${user.current_streak + 1}.`,
        },
    };
    
    // Select best variant based on user profile
    let selectedVariant = 'urgency'; // default
    
    if (user.current_streak >= 7 && user.responds_to_streak_focus) {
        selectedVariant = 'streak_focus';
    } else if (user.responds_to_social_proof) {
        selectedVariant = 'social_proof';
    } else if (user.current_streak >= 30) {
        selectedVariant = 'achievement';
    }
    
    return variants[selectedVariant];
}
```

#### 2.4 Rich Notifications with Actions
```typescript
// Add action buttons to notifications
const richNotification = {
    ...baseNotification,
    android: {
        priority: 'high',
        notification: {
            sound: 'default',
            color: '#0072B1',
            icon: 'ic_notification',
            // Action buttons
            actions: [
                {
                    action: 'complete_now',
                    title: '✅ Complete Now',
                    icon: 'ic_check',
                },
                {
                    action: 'remind_later',
                    title: '⏰ Remind in 1 hour',
                    icon: 'ic_clock',
                },
            ],
        },
    },
    apns: {
        payload: {
            aps: {
                sound: 'default',
                badge: 1,
                category: 'DAILY_DRILL_CATEGORY', // For action buttons
            },
        },
    },
};
```

#### 2.5 Optimal Timing Analysis
```sql
-- Analyze best notification times per user
CREATE OR REPLACE FUNCTION calculate_optimal_notification_time(p_user_id UUID)
RETURNS TIME AS $$
DECLARE
    optimal_time TIME;
BEGIN
    -- Find the hour with highest CTR for this user
    SELECT 
        EXTRACT(HOUR FROM sent_at)::INT || ':00:00'
    INTO optimal_time
    FROM notification_logs
    WHERE user_id = p_user_id
      AND status = 'clicked'
      AND sent_at >= now() - INTERVAL '30 days'
    GROUP BY EXTRACT(HOUR FROM sent_at)
    ORDER BY COUNT(*) DESC
    LIMIT 1;
    
    RETURN COALESCE(optimal_time, '08:00:00'::TIME);
END;
$$ LANGUAGE plpgsql;

-- Update user profiles with optimal times
CREATE OR REPLACE FUNCTION update_optimal_notification_times()
RETURNS INT AS $$
DECLARE
    affected_count INT;
BEGIN
    UPDATE user_notification_profile unp
    SET preferred_notification_time = calculate_optimal_notification_time(unp.user_id),
        updated_at = now()
    WHERE total_notifications_received >= 10; -- Minimum data threshold
    
    GET DIAGNOSTICS affected_count = ROW_COUNT;
    RETURN affected_count;
END;
$$ LANGUAGE plpgsql;
```

---

## 🎯 Strategy 3: Maximize Completion Rate Lift (> 10%)

### Current Implementation ✅
- Reminder notification at 11 AM
- Streak count in notifications
- Deep linking to Daily Drill screen

### Additional Optimizations 🚀

#### 3.1 Smart Reminder Timing
```sql
-- Track user completion patterns
CREATE TABLE IF NOT EXISTS user_completion_patterns (
    user_id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    
    -- Completion time analysis
    avg_completion_hour INT, -- 0-23
    avg_time_from_notification_to_completion INTERVAL,
    
    -- Reminder effectiveness
    completes_after_reminder BOOLEAN DEFAULT false,
    optimal_reminder_time TIME,
    
    -- Weekly patterns
    monday_completion_rate DECIMAL(5,2),
    tuesday_completion_rate DECIMAL(5,2),
    wednesday_completion_rate DECIMAL(5,2),
    thursday_completion_rate DECIMAL(5,2),
    friday_completion_rate DECIMAL(5,2),
    saturday_completion_rate DECIMAL(5,2),
    sunday_completion_rate DECIMAL(5,2),
    
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Calculate optimal reminder time per user
CREATE OR REPLACE FUNCTION calculate_optimal_reminder_time(p_user_id UUID)
RETURNS TIME AS $$
DECLARE
    avg_completion_hour INT;
    optimal_time TIME;
BEGIN
    -- Find average hour when user completes drills
    SELECT AVG(EXTRACT(HOUR FROM udd.completed_at))::INT
    INTO avg_completion_hour
    FROM user_daily_drills udd
    WHERE udd.user_id = p_user_id
      AND udd.status = 'completed'
      AND udd.completed_at >= now() - INTERVAL '30 days';
    
    -- Set reminder 1 hour before typical completion time
    IF avg_completion_hour IS NOT NULL THEN
        optimal_time = (avg_completion_hour - 1 || ':00:00')::TIME;
    ELSE
        optimal_time = '11:00:00'::TIME; -- Default
    END IF;
    
    RETURN optimal_time;
END;
$$ LANGUAGE plpgsql;
```

#### 3.2 Progressive Reminder Strategy
```typescript
// Multi-stage reminder system
const reminderStages = [
    {
        time: '11:00',
        title: '⏰ Quick Reminder',
        body: 'You have {hours} hours left to complete today\'s drill!',
        priority: 'default',
    },
    {
        time: '16:00', // 4 PM - afternoon reminder
        title: '🔔 Don\'t Forget!',
        body: 'Only a few hours left! Keep your {streak}-day streak alive.',
        priority: 'high',
    },
    {
        time: '20:00', // 8 PM - evening reminder
        title: '🚨 Last Chance!',
        body: 'Complete your drill before midnight to maintain your streak!',
        priority: 'high',
        sound: 'urgent',
    },
];
```

#### 3.3 Gamification Elements
```sql
-- Achievement system
CREATE TABLE IF NOT EXISTS user_achievements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    achievement_type TEXT NOT NULL,
    achievement_name TEXT NOT NULL,
    achieved_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    notified BOOLEAN DEFAULT false
);

-- Achievement triggers
CREATE OR REPLACE FUNCTION check_and_award_achievements(p_user_id UUID)
RETURNS TABLE(achievement_type TEXT, achievement_name TEXT) AS $$
BEGIN
    -- 7-day streak
    IF (SELECT current_streak FROM user_drill_progress WHERE user_id = p_user_id) = 7 THEN
        INSERT INTO user_achievements (user_id, achievement_type, achievement_name)
        VALUES (p_user_id, 'streak', '7-Day Warrior')
        ON CONFLICT DO NOTHING
        RETURNING achievement_type, achievement_name;
    END IF;
    
    -- 30-day streak
    IF (SELECT current_streak FROM user_drill_progress WHERE user_id = p_user_id) = 30 THEN
        INSERT INTO user_achievements (user_id, achievement_type, achievement_name)
        VALUES (p_user_id, 'streak', '30-Day Champion')
        ON CONFLICT DO NOTHING
        RETURNING achievement_type, achievement_name;
    END IF;
    
    -- 100 total drills
    IF (SELECT total_drills_completed FROM user_drill_progress WHERE user_id = p_user_id) = 100 THEN
        INSERT INTO user_achievements (user_id, achievement_type, achievement_name)
        VALUES (p_user_id, 'milestone', 'Century Club')
        ON CONFLICT DO NOTHING
        RETURNING achievement_type, achievement_name;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

#### 3.4 Social Proof & Leaderboards
```sql
-- Daily leaderboard
CREATE OR REPLACE VIEW daily_leaderboard AS
SELECT 
    u.id as user_id,
    u.email,
    udp.current_streak,
    udp.total_drills_completed,
    RANK() OVER (ORDER BY udp.current_streak DESC, udp.total_drills_completed DESC) as rank
FROM auth.users u
INNER JOIN user_drill_progress udp ON u.id = udp.user_id
WHERE udp.current_streak > 0
ORDER BY rank
LIMIT 100;

-- Add to notification
const leaderboardPosition = await getLeaderboardPosition(userId);
if (leaderboardPosition <= 10) {
    notification.body = `You're #${leaderboardPosition} on the leaderboard! 🏆 Keep it up!`;
}
```

#### 3.5 Streak Recovery Mechanism
```sql
-- Streak freeze feature (allow 1 miss per month)
ALTER TABLE user_drill_progress
ADD COLUMN IF NOT EXISTS streak_freezes_available INT DEFAULT 1,
ADD COLUMN IF NOT EXISTS last_freeze_used_at TIMESTAMPTZ;

-- Auto-apply freeze if user has one available
CREATE OR REPLACE FUNCTION auto_apply_streak_freeze(p_user_id UUID)
RETURNS BOOLEAN AS $$
DECLARE
    freezes_available INT;
BEGIN
    SELECT streak_freezes_available INTO freezes_available
    FROM user_drill_progress
    WHERE user_id = p_user_id;
    
    IF freezes_available > 0 THEN
        UPDATE user_drill_progress
        SET streak_freezes_available = streak_freezes_available - 1,
            last_freeze_used_at = now()
        WHERE user_id = p_user_id;
        
        -- Send notification about freeze usage
        -- (implement in Edge Function)
        
        RETURN true;
    END IF;
    
    RETURN false;
END;
$$ LANGUAGE plpgsql;
```

---

## 📊 Measurement & Analytics Dashboard

### KPI Tracking Queries

#### Delivery Rate
```sql
CREATE OR REPLACE VIEW notification_delivery_rate AS
SELECT 
    DATE(created_at) as date,
    notification_type,
    COUNT(*) as total_sent,
    COUNT(*) FILTER (WHERE status IN ('sent', 'delivered')) as successful,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status IN ('sent', 'delivered')) / COUNT(*), 2) as delivery_rate
FROM notification_logs
WHERE created_at >= now() - INTERVAL '30 days'
GROUP BY DATE(created_at), notification_type
ORDER BY date DESC;
```

#### Click-Through Rate
```sql
CREATE OR REPLACE VIEW notification_ctr AS
SELECT 
    DATE(created_at) as date,
    notification_type,
    ab_test_variant,
    COUNT(*) FILTER (WHERE status = 'delivered') as delivered,
    COUNT(*) FILTER (WHERE status = 'clicked') as clicked,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'clicked') / 
          NULLIF(COUNT(*) FILTER (WHERE status = 'delivered'), 0), 2) as ctr
FROM notification_logs
WHERE created_at >= now() - INTERVAL '30 days'
GROUP BY DATE(created_at), notification_type, ab_test_variant
ORDER BY date DESC, ctr DESC;
```

#### Completion Rate Lift
```sql
CREATE OR REPLACE VIEW completion_rate_analysis AS
WITH notification_cohort AS (
    SELECT DISTINCT nl.user_id, DATE(nl.created_at) as notification_date
    FROM notification_logs nl
    WHERE nl.notification_type = 'morning'
      AND nl.status IN ('sent', 'delivered')
),
completion_data AS (
    SELECT 
        nc.notification_date,
        COUNT(DISTINCT nc.user_id) as users_notified,
        COUNT(DISTINCT udd.user_id) FILTER (WHERE udd.status = 'completed') as users_completed
    FROM notification_cohort nc
    LEFT JOIN user_daily_drills udd ON nc.user_id = udd.user_id 
        AND DATE(udd.assigned_date) = nc.notification_date
    GROUP BY nc.notification_date
)
SELECT 
    notification_date,
    users_notified,
    users_completed,
    ROUND(100.0 * users_completed / NULLIF(users_notified, 0), 2) as completion_rate,
    LAG(ROUND(100.0 * users_completed / NULLIF(users_notified, 0), 2)) 
        OVER (ORDER BY notification_date) as previous_day_rate,
    ROUND(100.0 * users_completed / NULLIF(users_notified, 0), 2) - 
        LAG(ROUND(100.0 * users_completed / NULLIF(users_notified, 0), 2)) 
        OVER (ORDER BY notification_date) as day_over_day_change
FROM completion_data
ORDER BY notification_date DESC;
```

---

## 🚀 Implementation Priority

### Week 1: Foundation (Delivery Rate Focus)
- [ ] Implement token health monitoring
- [ ] Add smart retry logic
- [ ] Set up delivery confirmation tracking
- [ ] Create monitoring dashboard

### Week 2: Engagement (CTR Focus)
- [ ] Implement A/B testing framework
- [ ] Create 4 notification variants
- [ ] Add personalization engine
- [ ] Implement optimal timing analysis

### Week 3: Conversion (Completion Rate Focus)
- [ ] Add progressive reminder system
- [ ] Implement achievement system
- [ ] Create leaderboard view
- [ ] Add streak freeze feature

### Week 4: Optimization
- [ ] Analyze A/B test results
- [ ] Optimize notification timing
- [ ] Refine personalization algorithms
- [ ] Scale to production

---

## 📈 Expected Results

| KPI | Baseline | Week 2 | Week 4 | Target | Status |
|-----|----------|--------|--------|--------|--------|
| Delivery Rate | 85% | 90% | 96% | > 95% | ✅ |
| Click-Through Rate | 6% | 10% | 17% | > 15% | ✅ |
| Completion Rate Lift | 0% | 5% | 12% | > 10% | ✅ |

---

**Created**: 2026-02-14  
**Version**: 2.0 - Phase 2 Optimization  
**Status**: Ready for Implementation
