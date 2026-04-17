# ADR-003 — Push Notification Architecture (Phase 8)

**Ticket Reference:** Phase 8 — Notifications & Habit Engine
**Status:** accepted
**Date:** 2026-04-17

## Context

Phase 8 requires daily push reminders, milestone notifications, and "streak at risk" alerts delivered to both web and mobile clients. The system must store device tokens server-side, schedule sends via cron, and support two client platforms (Next.js web app, Expo/React Native mobile app). Minimal external account setup is a hard constraint for this phase.

Three provider options were assessed:

1. **Expo Push Service (mobile) + Web Push API / VAPID (web)** — two standard APIs, no external accounts beyond an Expo account. `expo-server-sdk` (Node) sends to Expo; `web-push` (npm) sends to browser.
2. **OneSignal unified** — single provider for both platforms, but requires account creation, vendor lock-in, and introduces a third-party SDK in the client.
3. **FCM directly** — unified but requires a Firebase project, `firebase-admin` SDK, and client-side Firebase SDK on web. Heavy setup with no material advantage over option 1 at this scale.

## Decision

**Mobile**: Expo Push Service via `expo-server-sdk` Node package.
**Web**: Web Push API (W3C standard) via `web-push` npm package + VAPID key pair.
**Cron scheduler**: `node-cron` (MVP-appropriate; Bull deferred to Phase 11).
**Token storage**: New `PushToken` model (one row per device per user, supports multi-device).

This is a split-provider approach with a unified `PushNotificationService` abstraction in the backend that handles both token types transparently.

## Architectural Constraints for Dev Agent

### Schema
1. Add a new `PushToken` model to `prisma/schema.prisma`:
   - Fields: `id`, `userId`, `token String`, `platform String` ("expo" | "web"), `deviceId String?` (browser fingerprint or Expo device ID), `createdAt`, `updatedAt`
   - Unique constraint: `(userId, platform, deviceId)` — prevents duplicate token rows per device
   - Cascade delete on user deletion
   - Index on `userId`
2. Do **not** add a `pushToken` field to `NotificationPreference` — the `PushToken` table is the correct home (supports multi-device).

### Backend Services
3. Create `packages/api/src/services/PushNotificationService.ts`:
   - Single `send(userId, payload)` method that queries `PushToken` table, routes to Expo or `web-push` based on `platform` field
   - Expo sends must batch in groups of ≤100 (Expo API limit)
   - On `DeviceNotRegistered` or `InvalidCredentials` errors from Expo: delete stale token from DB (do not retry)
   - On web-push 410 Gone: delete stale token from DB
   - Service must never throw — catch all send errors, log, and continue (notification failure must not block lesson completion)

4. Create `packages/api/src/jobs/dailyReminderJob.ts`:
   - Runs on `node-cron` schedule `0 * * * *` (every hour on the hour)
   - Queries users where `NotificationPreference.enableDailyReminder = true` AND `UserProfile.preferredTime` maps to current UTC hour
   - Before sending: check `Notification` table for `type = 'daily-reminder'` sent in the past 20 hours (idempotency guard — prevents duplicate sends on cron overlap)
   - Skips users who have already completed today's lesson
   - Calls `PushNotificationService.send()`, then records send in `Notification` table
   - Cron is registered in `packages/api/src/index.ts` after the Express server starts

5. Create `packages/api/src/services/StreakService.ts`:
   - `getMilestone(streak: number): string | null` — returns label for 7, 14, 30, 60, 100, 365; null otherwise
   - `isStreakAtRisk(userId: string): Promise<boolean>` — true if user has no completed lesson today AND current time in user's timezone is past 18:00
   - `getStreakMessage(streak: number): string` — returns motivational string appropriate to streak length
   - All methods are pure (stateless) except `isStreakAtRisk` which reads `UserProgress` and `UserProfile.timezone`

### API Endpoints
6. Add `POST /api/notifications/push-token` (protected) to `packages/api/src/routes/notifications.ts`:
   - Body: `{ token: string, platform: "expo" | "web", deviceId?: string }`
   - Upsert into `PushToken` table (update token if `(userId, platform, deviceId)` already exists)

### Configuration
7. VAPID keys for web push stored in env as `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` (`mailto:admin@example.com`)
8. VAPID keys are **not** committed to git — generate once per environment with `npx web-push generate-vapid-keys`
9. No Firebase project required. No OneSignal account required.

### Mobile (Expo)
10. `learning-app/hooks/useNotifications.ts` — request permissions, get Expo push token, POST to `/api/notifications/push-token` with `platform: "expo"`
11. Background tap handler in `learning-app/app/_layout.tsx` — navigates to `/(tabs)/lessons` on notification tap
12. Expo token must be obtained with `projectId` from `app.json` (required by Expo SDK 50+)

### Web
13. Service worker file at `packages/web/public/sw.js` — listens for `push` event, calls `self.registration.showNotification()`
14. VAPID public key exposed to frontend via `NEXT_PUBLIC_VAPID_PUBLIC_KEY` env var
15. `useWebPush` hook in `packages/web` — subscribes browser, sends subscription object to `/api/notifications/push-token` with `platform: "web"`

### Testing
16. Unit test `PushNotificationService` with Expo SDK and `web-push` fully mocked
17. Unit test cron job filtering logic (time-zone matching, skip-if-done, idempotency guard) with Prisma mocked
18. Unit test `StreakService` methods — `isStreakAtRisk` must test timezone boundary conditions

### Deferred to Phase 11
- Migrate cron to Bull queue (Redis-backed) for reliability and retry guarantees
- Dead letter queue for failed push sends
- Push analytics (open rates, click-through)

## Consequences

**Positive**:
- No new external accounts required to start development
- Expo Push Service handles APNs + FCM certificate complexity for mobile
- Web Push is a W3C standard — works in all modern browsers without vendor SDK
- Multi-device support built into schema from day one
- `PushNotificationService` abstraction means swapping to FCM/OneSignal later touches only one file

**Negative**:
- Two code paths in `PushNotificationService` (Expo vs web-push) — more test surface
- Expo Push Service is a third-party relay for mobile (Expo company) — acceptable for MVP, should evaluate direct APNs/FCM for Phase 11
- VAPID key rotation requires re-subscribing all browser clients (operational cost at scale)
- `node-cron` has no retry guarantee; a crashed process silently skips sends — acceptable for MVP

## Alternatives Considered

**OneSignal**: Rejected — vendor lock-in, requires account creation, client SDK adds bundle weight, limited control over token lifecycle.

**FCM directly**: Rejected — requires Firebase project, `firebase-admin` service account key, and Firebase client SDK on web frontend. No material benefit over Web Push API + Expo at this scale. Revisit if native Android app is built outside Expo (Phase 12+).

**Bull queue instead of node-cron**: Deferred — adds operational complexity (Bull UI, Redis queue monitoring) not warranted for MVP user volume. Architecture supports migration.
