# Phase 8: Notifications & Habit Engine

**Status**: ⚪ NOT STARTED

**Goal**: Deliver daily reminders at user-preferred times across web and mobile. Reinforce the daily learning habit with streaks, milestones, and motivational messaging.

---

## Chunk 8.1: Push Notifications (Web)

**Deliverable**: Web users receive daily reminders at their preferred time

### 8.1.1 — Notification Provider Setup

**Choose one**: Firebase Cloud Messaging (FCM) or OneSignal

**Config**:
- Add FCM credentials to `.env` (`FIREBASE_SERVICE_ACCOUNT_KEY`)
- Install `firebase-admin` in `packages/api`

### 8.1.2 — Device Token Storage

**Schema update**: Add `pushToken` field to `User` or `NotificationPreference` model

**API endpoint**: `POST /api/notifications/push-token` — store web push token for authenticated user

### 8.1.3 — Daily Reminder Cron Job

**File**: `packages/api/src/jobs/dailyReminderJob.ts`

**Logic**:
- Run every hour (cron: `0 * * * *`)
- Query users where `NotificationPreference.dailyReminderEnabled = true` AND `preferredTime` matches current UTC hour
- For each user: check if they've already completed today's lesson (skip if done)
- Send push notification: "Your daily lesson is ready 📚"

**Cron runner**: `node-cron` or Bull queue (Redis-backed for reliability)

**Tests**: Unit test reminder logic (filtering, skip-if-done); mock push sending

---

## Chunk 8.2: Push Notifications (Mobile)

**Deliverable**: Mobile users get push notifications via Expo Notifications

### 8.2.1 — Expo Notification Setup

**File**: `learning-app/hooks/useNotifications.ts`

**Logic**:
- Request push permissions on app launch
- Get Expo push token (`Notifications.getExpoPushTokenAsync()`)
- Send token to backend: `POST /api/notifications/push-token`
- Handle incoming notification while app is foregrounded (navigate to lesson screen)

### 8.2.2 — Background Notification Handler

**File**: `learning-app/app/_layout.tsx`

**Setup**:
- Register background notification handler
- On tap: navigate to today's lesson

**Tests**: Unit test useNotifications hook with Expo SDK mocked

---

## Chunk 8.3: Habit Reinforcement

**Deliverable**: Users are motivated to maintain and extend their learning streak

### 8.3.1 — Streak Service Enhancements

**File**: `packages/api/src/services/StreakService.ts`

**Add**:
- `getMilestone(streakCount)` — Return milestone label (7, 14, 30, 60, 100, 365 days)
- `isStreakAtRisk(userId)` — True if user hasn't done today's lesson and it's past 6 PM local time
- `getStreakMessage(streakCount)` — Return motivational message string appropriate to streak length

### 8.3.2 — Milestone Notifications

**Trigger**: After lesson completion, if streak hits a milestone
- Send in-app notification + push notification: "🔥 30-day streak! You're unstoppable."
- Track milestone sent in `Notification` table (avoid duplicate sends)

### 8.3.3 — "Streak at Risk" Reminder

**Job**: Evening cron (e.g. 6 PM user local time) — send reminder if streak at risk and `dailyReminderEnabled`
- Message: "Don't break your 14-day streak! Your lesson takes 3 minutes."

### 8.3.4 — Frontend Celebrations

**Files**: Web dashboard, mobile lessons tab

**Animations**:
- Confetti on milestone streaks (7, 30, 100 days)
- Streak counter increment animation on lesson completion
- "Keep the streak alive" banner when streak > 3 days

**Tests**: Unit test milestone detection; test "at risk" logic with timezone handling

---

## Chunk 8.4: UX Polish

**Deliverable**: App feels complete, responsive, and polished across all screens

### 8.4.1 — Loading States

- Skeleton loaders on lesson fetch, quiz load, progress stats
- Spinner on all async button actions (complete, submit quiz, save settings)

### 8.4.2 — Error Handling

- User-friendly error messages for all API failures
- Toast notifications for success/failure actions
- Offline detection banner (web + mobile)

### 8.4.3 — Accessibility

- ARIA labels on all interactive elements (web)
- Screen reader support for lesson content
- Keyboard navigation through quiz options

### 8.4.4 — Animations

- Quiz answer feedback animation (correct = green pulse, incorrect = red shake)
- Lesson completion checkmark animation
- Progress bar fill animation on stats page

### 8.4.5 — Dark Mode (Web)

- TailwindCSS dark mode (`dark:` variants)
- Respect system preference (`prefers-color-scheme`)
- Toggle in settings

---

## Next Phase

[Phase 9: Admin CMS](./phase-9-admin-cms.md)
