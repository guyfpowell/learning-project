# Phase 8: Notifications & Habit Engine

**Status**: ✅ COMPLETE
**arch-review**: required - approved
**ADR**: [ADR-003 — Push Notification Architecture](../adr/ADR-003-push-notification-architecture.md)

**Goal**: Deliver daily reminders at user-preferred times across web and mobile. Reinforce the daily learning habit with streaks, milestones, and motivational messaging.

---

## Architectural Constraints (from ADR-003)

**Provider decision**:
- Mobile: Expo Push Service via `expo-server-sdk` (no Firebase account needed)
- Web: Web Push API (W3C standard) via `web-push` npm + VAPID keys

**Schema**: New `PushToken` model — do NOT add pushToken to `NotificationPreference`

**Key constraints for dev agent** (full detail in ADR-003):
1. `PushToken(userId, platform, deviceId)` unique — supports multi-device
2. `PushNotificationService.send()` must never throw — swallow all send errors
3. Expo batch limit: ≤100 tokens per send call
4. Delete stale tokens on `DeviceNotRegistered` / HTTP 410 responses
5. Cron idempotency: check `Notification` table before sending (20-hour window)
6. VAPID keys in env (`VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT`) — never committed
7. `node-cron` for MVP — Bull deferred to Phase 11
8. Cron registered in `index.ts` after server starts
9. Expo token requires `projectId` from `app.json` (SDK 50+ requirement)

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

## Implementation Summary (2026-04-17)

### Chunk 8.1 — Push Notifications (Backend + Web)
- **Schema**: New `PushToken` model (`userId, token, platform, deviceId` — unique on `(userId, platform, deviceId)`, cascade delete, indexed)
- **PushNotificationService** (`packages/api/src/services/`): unified send for Expo + Web Push API (VAPID). Never throws. Removes stale tokens on `DeviceNotRegistered` / HTTP 410.
- **API**: `POST /api/notifications/push-token` (protected) — upserts device token
- **Daily reminder cron** (`packages/api/src/jobs/dailyReminderJob.ts`): `node-cron` hourly, matches `UserProfile.preferredTime` to UTC hour, skips users who already completed today, 20-hour idempotency guard via `Notification` table
- **Service worker** (`packages/web/public/sw.js`): handles `push` + `notificationclick` events, navigates to lesson on tap
- **`useWebPush` hook** (`packages/web/lib/`): registers SW, requests permission, POSTs VAPID subscription to backend
- **Settings page**: browser push subscribe button + status indicator

### Chunk 8.2 — Push Notifications (Mobile)
- **`useNotifications` hook** (`learning-app/src/hooks/`): requests permissions, gets `expo-notifications` token, POSTs to backend. No-ops silently in simulator or on permission denial.
- **`_layout.tsx`**: `addNotificationResponseReceivedListener` for background tap → navigates to `/(tabs)` or `/(tabs)/progress`
- **Tabs layout**: calls `useNotifications()` on mount (authenticated scope only)
- **Global mocks**: `__mocks__/expo-notifications.ts` + `expo-device.ts` for test isolation

### Chunk 8.3 — Habit Reinforcement (Backend + Frontend)
- **`StreakService`** (`packages/api/src/services/`): `getMilestone(streak)`, `getStreakMessage(streak)`, `isStreakAtRisk(userId)` (timezone-aware, past-6PM check)
- **Milestone notifications**: wired into `LessonService.submitQuiz()` — fire-and-forget, idempotent per `streak-milestone-{n}` type in `Notification` table
- **Evening at-risk cron** (`packages/api/src/jobs/streakAtRiskJob.ts`): hourly, uses `isStreakAtRisk`, 20-hour idempotency guard
- **`QuizResult` shared type** extended: `streak: number`, `milestone: string | null`
- **Web dashboard**: "Keep the streak alive" banner (streak > 3), animated streak counter (`animate-count-up`)
- **Web quiz page**: milestone celebration banner + confetti (`Confetti` component, CSS keyframes), streak display in results
- **Mobile `QuizModal`**: animated score (`Animated.spring`), milestone card (orange gradient), streak row, left-border feedback cards (green/red)

### Chunk 8.4 — UX Polish (Web)
- **Skeleton loaders**: `DashboardSkeleton`, `QuizSkeleton`, `ProgressSkeleton` (replaces spinners on all load states)
- **Toast system** (`components/Toast.tsx`): `ToastProvider` + `useToast()` hook — success/error/info toasts with slide-in animation, wired globally in `app/layout.tsx`
- **Offline banner**: `OfflineBanner` component detects `online`/`offline` events, shown globally
- **Tailwind animations**: `animate-pulse-green` (correct answer), `animate-shake` (incorrect answer), `animate-count-up` (streak), `animate-slide-in` (toasts), `animate-progress-fill` (progress bars)
- **Dark mode**: `darkMode: 'class'` in Tailwind, `useDarkMode` hook (localStorage + `prefers-color-scheme`), toggle in Settings, `dark:` variants across dashboard/quiz/progress/settings/layout
- **ARIA**: `role="progressbar"`, `role="alert"`, `aria-live`, `aria-label`, `aria-checked` (toggle switches), `aria-busy` (skeleton states), `accessibilityLabel` on all mobile interactives, `fieldset`/`legend` on quiz options, `<header>`/`<main>`/`<nav>` semantic elements
- **Settings page**: toggle switch component (replaces checkboxes), styled select with ARIA, loading skeleton

### Tests
- **API**: 122 tests passing (18 suites). New: `PushNotificationService.test.ts` (4), `StreakService.test.ts` (8), `dailyReminderJob.test.ts` (5)
- **Web**: 12 tests passing
- **Mobile**: 149 tests passing (25 suites). New: `useNotifications.test.ts` (5)

---

## Next Phase

[Phase 9: Admin CMS](./phase-9-admin-cms.md)
