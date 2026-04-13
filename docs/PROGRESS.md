# Build Plan — Progress Tracker

Last updated: 2026-04-12.

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
| **5** | Stripe Billing | ⚪ Not started | 5.1–5.4 | After Phase 4 |
| **6** | AI Lesson Generation | ⚪ Not started | 6.1–6.5 | After Phase 5 |
| **7** | AI Personalization & Adaptive Learning | ⚪ Not started | 7.1–7.4 | After Phase 6 |
| **8** | Notifications & Habit Engine | ⚪ Not started | 8.1–8.4 | After Phase 7 |
| **9** | Admin CMS | ⚪ Not started | 9.1–9.4 | After Phase 8 |
| **10** | Enterprise & Team Features | ⚪ Not started | 10.1–10.4 | After Phase 9 |
| **11** | Security, Analytics & Observability | ⚪ Not started | 11.1–11.4 | After Phase 10 |
| **12** | Testing & Launch | ⚪ Not started | 12.1–12.5 | After Phase 11 |

---

## Current Phase: Phase 4 — Mobile App

**What's being built**: React Native/Expo app with full feature parity to the web app.

**Chunking**:
- **4.1**: Expo project setup, auth flow, navigation → [phase-4-mobile.md](./phases/phase-4-mobile.md#chunk-41-mobile-project-setup--auth)
- **4.2**: Lessons tab, quiz modal, progress tab, profile tab → [phase-4-mobile.md](./phases/phase-4-mobile.md#chunk-42-mobile-features)

**Progress within Phase 4**:
- [x] 4.1.1 — Expo project initialisation
- [x] 4.1.2 — Auth flow (login, signup, Expo Secure Store)
- [x] 4.1.3 — Navigation setup (tabs + root auth check)
- [x] 4.2.1 — Lessons tab (today's lesson card)
- [x] 4.2.2 — Quiz modal
- [x] 4.2.3 — Progress tab + Profile tab

---

## Quick Links

### Completed Phases (Archive)
- [Phase 1: Foundation & Infrastructure](./phases/phase-1-foundation.md)
- [Phase 2: Core Backend API](./phases/phase-2-backend.md)
- [Phase 3: Web Frontend](./phases/phase-3-web-frontend.md)

### Active Phase
- [Phase 4: Mobile App](./phases/phase-4-mobile.md) ← **READ THIS NEXT**

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
