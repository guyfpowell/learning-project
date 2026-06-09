# Build Plan — Progress Tracker

Last updated: 2026-06-08.

**Implementer instructions:**
1. Context is cleared regularly, this document shows current status
2. This document MUST be kept up to date with any and all progress
3. Progress MUST be explicitly and completely described, both here and in the appropriate phase document — no shortcuts or omissions

## Phase Status Overview

| Phase | Scope | Status | Chunks | Next Action |
|-------|-------|--------|--------|--------------|
| **1** | Foundation & Infrastructure | ✅ COMPLETE | 1.1–1.6 | Archive (reference only) |
| **2** | Core Backend API | ✅ COMPLETE | 2.1–2.3 | Archive (reference only) |
| **3** | Web Frontend | ✅ COMPLETE | 3.1–3.3 | Archive (reference only) |
| **4** | Mobile App | ✅ COMPLETE | 4.1–4.2 | Archive (reference only) |
| **5** | Stripe Billing | 🚫 CANCELLED | — | Not required |
| **6** | AI Lesson Generation | ✅ COMPLETE | 6.1–6.6 | Archive (reference only) |
| **7** | AI Personalization & Adaptive Learning | ✅ COMPLETE | 7.1–7.4 | Archive (reference only) |
| **8** | Notifications & Habit Engine | ✅ COMPLETE | 8.1–8.4 | Archive (reference only) |
| **9** | Admin CMS | ✅ COMPLETE | 9.1–9.4 | Archive (reference only) |
| **10** | Enterprise & Team Features | ✅ COMPLETE (10.2 cancelled) | 10.1 ✅, 10.3 ✅, 10.4 ✅, 10.2 🚫 | Archive (reference only) |
| **11** | Security, Analytics & Observability | 🔄 In progress (11.2 deferred) | 11.1 ✅, 11.3 ✅, 11.4 ✅ / 11.2 ⚪ | 11.2 after analytics provider chosen |
| **12** | Testing & Launch | 🔄 In progress | 12.1 ✅ / 12.2–12.5 ⚪ | 12.2 — Staging Deployment |

---

## Current Phase: Phase 12 — Testing & Launch (12.1 complete)

**Previous phases**: Phases 1–4, 6–11 (partially) complete. Phase 5 (Stripe) cancelled. Phase 10 chunks 10.1, 10.3, 10.4 complete; 10.2 cancelled. Phase 11 chunks 11.1, 11.3, 11.4 complete; 11.2 deferred.

**Phase 12 status**: Chunk 12.1 (Integration & E2E Testing) complete. Chunks 12.2–12.5 not started.

---

## Phase 12 — Testing & Launch (IN PROGRESS)

### Chunk 12.1 — Integration & E2E Testing — COMPLETE (2026-06-09)

**12.1.1 — Backend Integration Tests**
- ✅ Extracted `packages/api/src/app.ts` (pure Express config) from `index.ts` (server start + jobs) — enables Supertest imports without side effects
- ✅ Rate limiters skipped in `NODE_ENV=test` (`app.ts`)
- ✅ `jest.integration.cjs` — separate config, real Prisma (test DB), mocked Redis (in-memory), 30s timeout, `--runInBand`
- ✅ `packages/api/.env.test` — `DATABASE_URL` → `learning_app_test` on port 5433
- ✅ `globalSetup.ts` — creates test DB and runs `prisma db push` before suite
- ✅ Auth integration tests (5 suites): register → login → `/me` → refresh → logout → refresh-after-logout
- ✅ Lesson integration tests (4 suites): `GET /today` → `GET /:id` → `POST /:id/complete` → `POST /:id/quiz` (score 100 + streak check)
- ✅ Team integration tests (4 suites): create team → get team → invite member → accept invite → duplicate-accept rejected
- ✅ Admin integration tests (5 suites): admin-only skill/path/lesson creation; 403 for non-admin; lesson served to regular user; unpublish toggle
- ✅ Subscription flow skipped (Phase 5 cancelled)
- ✅ `test:integration` script added to `packages/api/package.json`
- ✅ Unit tests still pass: 206 tests, 27 suites
- ✅ Integration tests: 32 tests, 5 suites, all passing

**12.1.4 — AI Generation Integration Tests** (bundled with 12.1.1)
- ✅ `aiGeneration.integration.test.ts` — tests against real Prisma DB
- ✅ Static fallback path tested (serves lesson from DB when AI service unavailable)
- ✅ Cache assertion conditional on `AI_SERVICE_API_KEY` being set (skips when key absent)
- ✅ All 3 skill levels (beginner/intermediate/advanced) exercise the fallback path
- ✅ Graceful fallback when AI URL is unreachable

**12.1.2 — Web E2E Tests (Playwright)**
- ✅ `@playwright/test` installed in `packages/web`
- ✅ `playwright.config.ts` — Chromium, `http://localhost:3001`, auto-starts dev server locally, `E2E_BASE_URL` override for staging
- ✅ `e2e/auth-lesson.spec.ts`: sign-up → onboarding → dashboard → lesson → quiz flow; login → progress → settings → save → logout flow
- ✅ `e2e/admin.spec.ts`: admin login → create lesson → verify lesson visible to regular user
- ✅ `test:e2e` and `test:e2e:ui` scripts added
- **To run**: API + web servers must be running; requires `npx playwright install chromium` on first run

**12.1.3 — Mobile E2E Tests (Detox)**
- ✅ `detox` + `@config-plugins/detox` installed in `learning-app`
- ✅ `.detoxrc.js` — iOS simulator (iPhone 16) + Android emulator configs
- ✅ `e2e/jest.config.js` — Detox Jest runner
- ✅ `e2e/auth-lesson.e2e.ts` — sign-up flow, lesson tab, progress tab with streak, notification permission
- ✅ `testID` attributes added to all key UI elements: `signin-email`, `signin-password`, `signin-submit`, `register-name/email/password/confirm/submit`, `lesson-card`, `lesson-title`, `lesson-quiz-btn`, `progress-streak`, `progress-lessons-count`, `progress-avg-score`
- ✅ `e2e:build:ios`, `e2e:build:android`, `e2e:test:ios`, `e2e:test:android` scripts added
- **To run**: requires `expo prebuild` → native build first (will be done in chunk 12.4)

---

## Phase 11 — Security, Analytics & Observability (PARTIALLY COMPLETE)

### Chunks 11.1, 11.3, 11.4 — COMPLETE (2026-06-08)

**Security Hardening (11.1)**
- ✅ JWT short-lived access tokens (15m) + long-lived refresh tokens (30d) — stored hashed in Redis
- ✅ `POST /api/auth/refresh` + `POST /api/auth/logout` endpoints added
- ✅ Web `api-client.ts`: 401 interceptor with mutex queue — auto-refresh + retry + fallback clear
- ✅ Web `auth-context.tsx`: stores `refresh_token` in localStorage, logout calls API
- ✅ Mobile `api.ts`: 401 interceptor replaced simple logout with refresh flow + queue + fallback sign-out
- ✅ Rate limiting: `authLimiter` (5/15min), `aiLimiter` (10/min), `generalLimiter` (100/min)
- ✅ CORS locked to `ALLOWED_ORIGINS` env var (defaults `http://localhost:3001`)
- ✅ Lesson content sanitised with `sanitize-html` on every admin save

**Sentry Error Tracking (11.3)**
- ✅ `@sentry/node` in API, `@sentry/nextjs` in web, `@sentry/react-native` already in mobile
- ✅ API: `initSentry()` gracefully no-ops without `SENTRY_DSN`; user context set on auth; unhandled errors captured
- ✅ Web: `sentry.client.config.ts` + `sentry.server.config.ts`; `next.config.js` wrapped with `withSentryConfig`
- ✅ AI client: manual capture on circuit-open and network errors

**Performance Monitoring (11.4)**
- ✅ `requestLogger` middleware: logs method/path/status/ms; WARN >500ms, ERROR >2000ms
- ✅ Prisma slow query logging (>200ms, dev only)
- ✅ AI latency logging in `AIServiceClient` (`generateLesson`, `coachingMessage`)

### Chunk 11.2 — DEFERRED

Analytics provider not yet chosen. Defer until Mixpanel or Plausible is selected.

**Test totals**: API 206 (was 184), 27 suites, all passing

---

## Phase 10 — Enterprise & Team Features (COMPLETE — 10.2 cancelled)

### Chunks 10.1, 10.3, 10.4 — COMPLETE (2026-04-30)

**Team Account Model (10.1)**
- ✅ Schema: `TeamRole` enum, `Team`, `TeamMember`, `TeamSubscription` models; `User.teamMembers` + `User.ownedTeams` relations
- ✅ `TeamService`: createTeam, getTeam, inviteMember (JWT token, email stubbed), acceptInvite, removeMember, getTeamSubscription
- ✅ `TeamController` + `routes/teams.ts`: all 6 endpoints protected by `authMiddleware`
- ✅ 29 new tests (17 service + 12 controller), all passing

**Team Analytics Dashboard (10.3)**
- ✅ `TeamAnalyticsService`: getTeamSummary, getMemberProgress, getSkillGapAnalysis, getTeamLeaderboard
- ✅ 4 analytics routes added to `routes/teams.ts`
- ✅ `packages/web/app/(dashboard)/team/page.tsx`: summary cards, member table, leaderboard, skill gap heatmap
- ✅ Team nav link in dashboard sidebar
- ✅ Team API client methods + types added to `api-client.ts`
- ✅ 14 new tests (7 service + 7 web), all passing

**Custom Skill Paths (10.4)**
- ✅ `SkillPath.teamId String?` added (nullable, indexed)
- ✅ `GET /api/lessons/skills` with team-aware visibility filtering
- ✅ `AdminService.CreateSkillPathInput.teamId` optional field

**Test totals**: API 184 (was 148), Web 54 (was 47)

---

## Phase 9 — Admin CMS (COMPLETE)

### Chunk 9.4 — COMPLETE (2026-04-17)

**Platform Stats Dashboard**

- ✅ **`AdminService.getStats()`**: added `lessonCompletionRate` — completions / starts in last 7 days; returns 0 when no starts
- ✅ **`api-client.ts`**: Added `adminGetStats()` method + `AdminStats` interface (`totalUsers`, `dau`, `activeSubscribers`, `mrr`, `lessonCompletionRate`)
- ✅ **`app/(admin)/admin/stats/page.tsx`**: Five stat cards — Total Users, DAU, Active Subscribers, Lesson Completion Rate (% last 7 days), MRR ($0 pending Phase 5)
- ✅ **Tests**: 148 API tests (was 147 + 1 new getStats test), 47 web tests (was 40 + 7 stats page tests), all passing
- ✅ **Build**: `/admin/stats` (static) confirmed clean

### Chunk 9.3 — COMPLETE (2026-04-17)

**Skill Path Management**

- ✅ **`AdminService.listSkills()`**: now includes `_count: { select: { skillPaths: true } }` for path counts
- ✅ **`AdminService.listSkillPaths(skillId?)`**: accepts optional `skillId` filter for per-skill path listing
- ✅ **`AdminService.listLessons()`**: added `skillPathId` option — filters by path and orders by `day: asc`
- ✅ **`AdminController`**: `listSkillPaths` reads `skillId` query param; `listLessons` reads `skillPathId` query param
- ✅ **`api-client.ts`**: Added `adminListSkills`, `adminCreateSkill`, `adminCreateSkillPath`, `adminListLessonsByPath`; updated `adminListSkillPaths(skillId?)` to accept optional filter; exported `AdminSkill`, `AdminCreateSkillInput`, `AdminCreateSkillPathInput` types
- ✅ **`app/(admin)/admin/skills/page.tsx`**: Lists all skills with path counts and categories; inline "New Skill" form (name, description, category); inline "New Path" form (skill dropdown, level dropdown, durationHours)
- ✅ **`app/(admin)/admin/skills/[id]/page.tsx`**: Lists lessons for a skill path ordered by day; drag-to-reorder via HTML5 drag API + move-up/move-down buttons; "Save Order" patches changed `day` values via `adminUpdateLesson`; "Remove" soft-deletes via `adminDeleteLesson`; "Add Lesson" links to `/admin/lessons/new`
- ✅ **Tests**: 147 API tests (was 145 + 2 new AdminService tests), 40 web tests (was 26 + 7 skills page + 7 skill path detail), all passing
- ✅ **Build**: Next.js build clean — `/admin/skills` (static) and `/admin/skills/[id]` (dynamic) confirmed

### Chunk 9.2 — COMPLETE (2026-04-17)

**Lesson Authoring UI**

- ✅ **Tiptap**: `@tiptap/react`, `@tiptap/pm`, `@tiptap/starter-kit` installed in `packages/web`
- ✅ **`LessonService.getTodayLesson()`**: now filters `p.lesson.published !== false` on incomplete progress records; `findFirst` fallback uses `where: { published: true }`
- ✅ **`api-client.ts`**: Added `adminListSkillPaths`, `adminListLessons`, `adminCreateLesson`, `adminUpdateLesson`, `adminDeleteLesson` methods + exported `AdminLesson`, `AdminLessonsResponse`, `AdminSkillPath`, `AdminCreateLessonInput`, `AdminUpdateLessonInput` types
- ✅ **`app/(admin)/admin/lessons/page.tsx`**: Paginated lesson table (title, skill, level, day, status, date), skill/level filters, inline publish/unpublish toggle, "New Lesson" button
- ✅ **`app/(admin)/admin/lessons/[id]/page.tsx`**: Lesson editor with Title, Skill Path dropdown, Day, Duration, Difficulty, Media URL, Tiptap rich text editor (bold/italic/h2/bullet toolbar), Published toggle, Quiz section (question, options A–D, correct answer selector, explanation), Save/Publish/Delete actions; create mode (`/new`) and edit mode by `id`
- ✅ **Route structure fix**: Admin pages placed under `app/(admin)/admin/` to resolve to `/admin/...` URLs, avoiding conflict with `app/(dashboard)/lessons/[id]`
- ✅ **Tests**: 145 API tests (was 143 + 2 new LessonService published-filter tests), 26 web tests (was 14 + 7 lesson-list + 5 lesson-editor), all passing
- ✅ **Build**: Next.js build clean — `/admin/lessons` (static) and `/admin/lessons/[id]` (dynamic) routes confirmed

### Chunk 9.1 — COMPLETE (2026-04-17)

**Admin Auth & Role System**

- ✅ **Prisma schema**: `UserRole` enum (`user` | `admin`), `role UserRole @default(user)` on `User`, `published Boolean @default(true)` + `publishedAt DateTime?` on `Lesson`
- ✅ **JWT**: `TokenPayload` now includes `role`; `AuthService.register` + `login` both include `role` in token
- ✅ **Shared types**: `User` interface now includes `role: 'user' | 'admin'`
- ✅ **AuthController `/me`**: now returns `role` in response
- ✅ **`adminGuard.ts`**: 403 for non-admin or unauthenticated requests
- ✅ **`AdminService`**: `listSkills`, `createSkill`, `listSkillPaths`, `createSkillPath`, `listLessons` (paginated + filterable by skill/level), `createLesson`, `updateLesson`, `deleteLesson` (soft delete via `published=false`), `listUsers` (paginated with subscription), `getStats` (MRR stubbed at 0 pending Phase 5)
- ✅ **`AdminController`**: wraps all service methods with HTTP handlers and input validation
- ✅ **`routes/admin.ts`**: `/api/admin/*` routes all protected by `authMiddleware` + `adminGuard`
- ✅ **`index.ts`**: admin routes mounted at `/api/admin`
- ✅ **Seed**: admin user upserted (`admin@learning.app`, `admin-change-me`, role=admin)
- ✅ **Frontend `app/(admin)/layout.tsx`**: sidebar with Skills, Lessons, Users, Stats; redirects non-admins to `/login`
- ✅ **Tests**: 143 API tests (was 138), all passing — 3 new test files: `adminGuard.test.ts` (3), `AdminService.test.ts` (9), `AdminController.test.ts` (6)

---

## Quick Links

### Completed Phases (Archive)
- [Phase 1: Foundation & Infrastructure](./phases/phase-1-foundation.md)
- [Phase 2: Core Backend API](./phases/phase-2-backend.md)
- [Phase 3: Web Frontend](./phases/phase-3-web-frontend.md)

### Active Phase
- [Phase 9: Admin CMS](./phases/phase-9-admin-cms.md) ← **READ THIS NEXT**

### Upcoming Phases
- [Phase 5: Stripe Billing](./phases/phase-5-stripe-billing.md)
- [Phase 6: AI Lesson Generation](./phases/phase-6-ai-lesson-generation.md)
- [Phase 7: AI Personalization & Adaptive Learning](./phases/phase-7-ai-personalization.md)
- [Phase 8: Notifications & Habit Engine](./phases/phase-8-notifications-habit.md)
- [Phase 9: Admin CMS](./phases/phase-9-admin-cms.md)
- [Phase 10: Enterprise & Team Features](./phases/phase-10-enterprise-team.md)
- [Phase 11: Security, Analytics & Observability](./phases/phase-11-security-analytics.md)
- [Phase 12: Testing & Launch](./phases/phase-12-testing-launch.md)

### Reference
- [Business Plan & Go-to-Market](../plan-automatedMicroLearningApp.md) (read once, strategic context)

---

## How to Use This Tracker

**On session startup**:
1. Read this file (PROGRESS.md) to see what phase is active
2. Jump to the relevant phase file
3. Skip archived phases unless you need context

**When starting a chunk**:
1. Find the chunk in the phase file
2. Read the deliverables and checklist
3. Follow the step-by-step instructions

**When completing a chunk**:
1. Check off the box in PROGRESS.md (this file)
2. Update the phase file with completion summary
3. Move to the next chunk in the same phase or next phase

**When context resets**:
- Only re-read the current phase file, not all phase files
- This saves tokens per session

---

## Definition of Done
1. All new code and all code the new code interacts with has full unit test coverage
2. All new code and all code the new code interacts with works with no lint or jest errors
3. The code works and provides the intended customer behaviour

---

## Reference Projects

Two existing codebases can be used as foundations. Read the relevant phase doc for specific guidance on what to copy vs. replace.

### `pocketchange` — `/Users/guypowell/documents/projects/pocketchange`

Production-quality Express/Prisma/PostgreSQL/Redis backend. Relevant to:
- **Phase 4 (Mobile)**: Auth patterns (JWT config, authenticate middleware)
- **Phase 5 (Stripe)**: Webhook handler with idempotency — port `backend/src/modules/webhooks/stripe.webhook.ts`
- **Phase 11 (Security)**: JWT refresh token flow — port `backend/src/modules/auth/auth.service.ts` and `backend/src/config/jwt.ts`

Stack: TypeScript, Express, Prisma 6, PostgreSQL, Redis, Stripe SDK, npm workspaces

### `pocketchange-app` — `/Users/guypowell/documents/projects/pocketchange-app`

Production-quality React Native/Expo mobile app. **Use as the direct starting foundation for `learning-app` (Phase 4).**
- Copy: `src/lib/api.ts`, `src/store/auth.store.ts`, `src/services/auth.service.ts`, `src/hooks/useAuth.ts`, `src/components/ui/`, `src/theme/index.ts`, `src/providers/`, `app/_layout.tsx`
- Strip: all donation/wallet/vendor/QR domain logic
- Replace: tab screens with Lessons, Progress, Profile, Settings

Stack: Expo 54, React Native 0.81.5, React 19, Expo Router 6, Zustand 5, TanStack Query 5, Expo SecureStore, Axios, Sentry

---

## Environment Setup (Reference)

**Ports**:
- PostgreSQL: 5433
- Redis: 6380
- Ollama: 11434
- API: 3000
- Web: 3001

**Startup** (from learning project root):
```bash
pnpm install
docker-compose up -d
pnpm dev  # Runs API + Web in parallel
```

**Access**:
- API: http://localhost:3000/health
- Web: http://localhost:3001

---

## Chunk 3.3 Implementation Summary (2026-04-12)

**What was completed**: Progress page with calendar, Settings page with notification preferences, and backend notification preference API

### Backend Changes:
- ✅ **Prisma Schema**: Added `NotificationPreference` model with fields for daily reminders, streak notifications, lesson availability alerts
- ✅ **Shared Types**: Added `NotificationPreference` interface exported from `@learning/shared`
- ✅ **NotificationPreferenceService** (6 unit tests, all passing):
  - `getPreferences(userId)` — fetches or creates default preferences
  - `updatePreferences(userId, input)` — upserts notification preferences
- ✅ **NotificationPreferenceController** (5 unit tests, all passing):
  - `getPreferences()` — GET /api/notifications/preferences (protected)
  - `updatePreferences()` — PATCH /api/notifications/preferences (protected)
- ✅ **API Routes**: New `/api/notifications` endpoints with auth middleware
- ✅ **Database**: `pnpm db:push` synced schema successfully

### Frontend Changes:
- ✅ **API Client**: Added `getNotificationPreferences()` and `updateNotificationPreferences()` methods
- ✅ **Progress Page** (`app/(dashboard)/progress.tsx`): Streak card, lessons card, quiz score card, month calendar with navigation
- ✅ **Settings Page** (`app/(dashboard)/settings.tsx`): Notification preference toggles, save to API

### Test Coverage:
- ✅ **Backend**: 40 total tests (29 existing + 11 new) — all passing
- ✅ **Frontend**: 11 new tests — all passing

**Phase Complete**: Phase 3 web frontend is now 100% complete with all chunks delivered and tested.
