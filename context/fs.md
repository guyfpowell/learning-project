---
title: FS Agent Context — Learning App
last_updated: 2026-04-11
---

# Learning App — Full Stack Context

## Business Vision & Market Opportunity

**Product**: AI-native micro-learning platform that generates, personalizes, and adapts daily 3-minute lessons in real-time.

**Target Audience (Phase 1)**: Product managers, product leaders, and AI practitioners needing bite-sized learning on AI product strategy, prompt engineering, AI governance, and AI-first product patterns.

**Differentiation**: 
- AI-generated lessons (not curated libraries)
- Real-time adaptation to quiz performance and learning style
- PM/AI vertical focus with thought leadership credibility
- Enterprise team learning analytics (Phase 2+)

**Revenue Model** (B2C freemium):
- Free: 1 lesson/day, basic quiz (Mistral 7B generated)
- Starter: $19/month (unlimited lessons, AI generation)
- Pro: $49/month (includes AI coaching via OpenAI API budget)
- Premium: $99/month (unlimited API budget)

**Go-To-Market**: Email waitlist → ProductHunt launch → PM network activation (LinkedIn, Twitter) → Enterprise partnerships

---

## Development Phases & Status

| Phase | Scope | Status |
|-------|-------|--------|
| **1** | Foundation & Infrastructure (monorepo, Docker, DB schema) | ✅ COMPLETE |
| **2** | Core Backend API (auth, lesson delivery, subscriptions) | ✅ COMPLETE |
| **3** | Web Frontend (auth, dashboard, lessons, progress) | ✅ COMPLETE |
| **4** | Mobile App (React Native/Expo parity) | ✅ COMPLETE |
| **5** | Stripe Billing | ⚪ Not started |
| **6** | AI Lesson Generation | ✅ COMPLETE |
| **7** | AI Personalization & Adaptive Learning | ✅ COMPLETE |
| **8** | Notifications & Habit Engine | ✅ COMPLETE |
| **9** | Admin CMS | ⚪ NEXT |

**MVP Target**: Phases 1–2 complete. Phase 3 (web frontend) is the immediate next step. Full MVP = 12 weeks (Phases 1–3 + core Phase 6).

---

## Architecture

**Stack**:
- **Backend**: Express 4.18 + TypeScript 5 (`packages/api`)
- **Frontend**: Next.js 14 + React 18 + TailwindCSS (`packages/web`)
- **Database**: PostgreSQL (Supabase for prod, Docker local)
- **ORM**: Prisma 5
- **Cache/Queue**: Redis (for future job queues, notifications)
- **Auth**: JWT + bcryptjs
- **Package Manager**: pnpm monorepo

**Project Structure**:
```
learning/
├── packages/
│   ├── api/              # Backend (routes → controllers → services)
│   │   ├── src/
│   │   │   ├── index.ts              # Express app setup
│   │   │   ├── db.ts                 # Prisma client instance
│   │   │   ├── routes/               # auth.ts, lessons.ts, users.ts, subscriptions.ts
│   │   │   ├── controllers/          # AuthController, LessonController, UserController, etc.
│   │   │   ├── services/             # AuthService, LessonService, SubscriptionService
│   │   │   └── middleware/           # auth.ts (JWT guard), error-handler.ts
│   ├── web/              # Frontend (pages → components → hooks → lib)
│   │   ├── app/          # Next.js app router
│   │   │   ├── page.tsx              # Landing/home (checks auth, shows login/signup/dashboard)
│   │   │   ├── (auth)/               # Auth layout group
│   │   │   │   ├── login/page.tsx
│   │   │   │   ├── signup/page.tsx
│   │   │   │   └── onboarding/page.tsx  # Post-signup profile setup
│   │   │   └── (dashboard)/          # Protected dashboard layout
│   │   │       ├── page.tsx          # Dashboard home (streak, today's lesson)
│   │   │       ├── lessons/[id]/page.tsx    # Lesson detail
│   │   │       ├── lessons/[id]/quiz.tsx    # Quiz screen
│   │   │       ├── progress.tsx      # Progress/stats
│   │   │       └── settings.tsx      # User settings
│   │   ├── components/   # Reusable UI components (to be built)
│   │   ├── lib/          # Utilities
│   │   │   ├── api-client.ts         # Fetch wrapper, auto-JWT header
│   │   │   ├── auth-context.tsx      # Auth state management
│   │   │   └── protected-route.tsx   # Route guard HOC
│   │   └── public/       # Static assets
│   ├── shared/           # Shared types (User, Lesson, Subscription, API envelopes)
│   │   └── src/types/    # lesson.ts, user.ts, subscription.ts, api.ts
│   └── middleware/       # Empty placeholder (can repurpose or remove)
├── prisma/
│   ├── schema.prisma     # Database schema (User, Skill, SkillPath, Lesson, Quiz, UserProgress, Subscription, Notification)
│   └── seed.ts           # Seed script for initial skills + plans
├── docker-compose.yml    # PostgreSQL 5433, Redis 6380, optional Ollama
├── pnpm-workspace.yaml
└── package.json          # Root with shared scripts + dev deps (Prisma, TypeScript, Prettier)
```

## Phase 1 Completion Summary

**What's done**:
- ✅ Monorepo structure (pnpm workspaces)
- ✅ Express backend skeleton with routes, controllers, services, middleware structure
- ✅ Next.js 14 web frontend with basic home page + auth layout structure
- ✅ Prisma ORM with comprehensive schema (User, Skill, SkillPath, Lesson, Quiz, UserProgress, Subscription, Notification)
- ✅ Docker Compose (PostgreSQL 5433, Redis 6380)
- ✅ Auth middleware (JWT-based)
- ✅ Error handling pattern (AppError class + middleware)

**Code Status**:
- Backend: ~1,470 lines of TypeScript (routes, controllers, auth logic)
- Frontend: ~1,885 lines of React/TypeScript (pages, layouts, auth context)
- Both packages compile and run locally

## Phase 2 Completion Summary

**API Endpoints (Implemented)**:

| Route | Method | Auth | Status |
|-------|--------|------|--------|
| `/api/auth/register` | POST | ❌ | ✅ |
| `/api/auth/login` | POST | ❌ | ✅ |
| `/api/auth/me` | GET | ✅ | ✅ |
| `/api/users/*` | * | ✅ | Routes defined, logic TBD |
| `/api/lessons/today` | GET | ✅ | Routes defined, logic TBD |
| `/api/lessons/:id` | GET | ✅ | Routes defined, logic TBD |
| `/api/lessons/:id/complete` | POST | ✅ | Routes defined, logic TBD |
| `/api/lessons/:id/quiz` | POST | ✅ | Routes defined, logic TBD |
| `/api/subscriptions/*` | * | ✅ | Routes defined, logic TBD |

**Services Skeleton**:
- `AuthService` — register, login, getUserById
- `LessonService` — getTodayLesson, getNextLesson, completeLessonService, submitQuiz
- `SubscriptionService` — (routes exist, service TBD)

**Backend Architecture Notes**:
- Controllers handle validation, delegate to services, catch/forward errors
- Services contain business logic, throw AppError with code + message
- Middleware wraps auth checks; authMiddleware extracts JWT and sets `req.user`
- All errors routed through error-handler middleware (returns structured JSON)

---

## Phase 3 (Next) — Web Frontend

**Deliverables**:
1. Auth flow (login/signup/onboarding pages fully functional)
2. Protected dashboard layout with navigation
3. Lesson screen (fetch + display from API)
4. Quiz screen (submit answers, show score + feedback)
5. Progress page (streaks, stats, completed lessons)
6. Settings page (profile, logout)
7. Responsive design (mobile 375px, tablet 768px, desktop 1440px)

**Key Frontend Patterns**:
- **Auth state**: `auth-context.tsx` (localStorage token, auto-login on mount)
- **API calls**: `api-client.ts` (fetch wrapper, auto-includes JWT, handles errors)
- **Protected routes**: `protected-route.tsx` (HOC that redirects to login if no auth)
- **Components**: Build reusable UI components (forms, cards, buttons) with TailwindCSS + Lucide icons
- **Type safety**: Import types from `@learning/shared`

**Frontend Checklist** (from BUILD_PLAN):
- [ ] 3.1.1 — Auth pages (login, signup, onboarding)
- [ ] 3.1.2 — Protected layout + navigation
- [ ] 3.1.3 — Onboarding form submission
- [ ] 3.2.1 — Dashboard home page
- [ ] 3.2.2 — Lesson screen + completion
- [ ] 3.2.3 — Quiz screen + results
- [ ] 3.3.1 — Progress page
- [ ] 3.3.2 — Settings page
- [ ] 3.3.3 — Responsive design testing

## Known Issues & Blockers for Phase 3

| Issue | Impact | Priority | Note |
|-------|--------|----------|------|
| **Frontend pages mostly empty** | Phase 3 starting from skeleton; need to build auth/dashboard UIs | HIGH — core work | This is the work for Phase 3 |
| **Service layer partially complete** | Some route handlers call services that don't exist yet | HIGH | Implement as needed during Phase 3 |
| **No tests** | Can't refactor safely; "tests pending" in both packages | MEDIUM | Defer to Phase 6; Phase 3 focuses on feature completeness |
| **No CI/CD** | No quality gate on push | MEDIUM | Add in Phase 6 (testing chunk) |
| **Not in git** | No version history for learning project itself | LOW | Initialize git, create baseline commit before Phase 3 work |
| **middleware package empty** | Dead code; potential confusion | LOW | Can keep as placeholder or remove later |
| **Auth token expiry/refresh** | JWT generated, no expiry/refresh strategy | MEDIUM | Add token exp claim before Phase 6 launch |
| **Stripe integration stubbed** | Subscription routes exist, but Stripe webhook not fully wired | MEDIUM | Wire up in Phase 2 completion or Phase 3 stretch |

**Clarification**: The issues I noted in my earlier critique (no tests, no CI) are intentional—they're deferred to Phase 6. The BUILD_PLAN prioritizes feature completeness first, then testing/polish/deployment. This is reasonable for MVP development.

## Local Development Environment

**Ports:**
- PostgreSQL: 5433 (localhost)
- Redis: 6380 (localhost)
- API: 3000 (http://localhost:3000/health)
- Web: 3001 (http://localhost:3001)

**DB Credentials (local)**:
- User: `learning_user`
- Password: `learning_password`
- Database: `learning_app`
- Connection string: `postgresql://learning_user:learning_password@localhost:5433/learning_app`

**Environment Files**:
- `packages/api/.env.local` — API config (DATABASE_URL, JWT_SECRET)
- `packages/web/.env.local` — Web config (NEXT_PUBLIC_API_URL)

**Setup Commands**:
```bash
# Install workspace dependencies
pnpm install

# Start Docker services (Postgres + Redis)
docker-compose up -d

# Run development servers (both API + Web in parallel)
pnpm dev

# Build all packages
pnpm build

# Format code
pnpm format

# Database commands
pnpm db:push      # Sync Prisma schema
pnpm db:seed      # Run seed script
pnpm db:reset     # Reset database
```

---

## Code Patterns & Implementation Rules

### Backend (Express)
- **Route → Controller → Service** pattern:
  - Routes: Define endpoints, extract/validate req.body
  - Controllers: Validate input, call service, format response
  - Services: Business logic, throw AppError for failures
- Controllers delegate errors to `next(error)` → caught by error-handler middleware
- AppError structure: `new AppError(code, message, statusCode)`
- Auth middleware extracts JWT from `Authorization: Bearer <token>` → sets `req.user`
- All DB calls use Prisma ORM via `db` instance

### Frontend (Next.js 14 + React 18)
- **Pages use 'use client'** for client-side rendering (auth checks, API calls)
- **Auth state**: `auth-context.tsx` manages JWT token (localStorage)
- **API client**: `api-client.ts` wraps fetch, auto-includes Authorization header, handles errors
- **Protected routes**: `protected-route.tsx` HOC checks auth, redirects unauthenticated users to `/login` (route group paths don't include prefix)
- **Components**: Reusable UI with TailwindCSS + Lucide icons
- **Type imports**: Import from `@learning/shared` for API contracts

### Shared Types
- Defined in `packages/shared/src/types/`
- Exported via `packages/shared/src/index.ts`
- Used by both API (request/response validation) and Web (form/API typing)

---

## Phase 3 Dependencies & Readiness

**Before starting Phase 3 frontend work**:
1. ✅ Backend API is running (`pnpm dev` in packages/api)
2. ✅ Database is seeded with initial skills/plans
3. ✅ Auth endpoints are tested and working (register, login, /me)
4. ✅ Lesson routes exist (even if logic TBD — can mock responses for frontend)

**Assumption for Phase 3**: Backend API endpoints are available; frontend just needs to consume them (with mocked/static responses if service logic incomplete).

---

## Implementation Approach

**When building each chunk, follow this order**:
1. **Types first** — Define request/response types in `packages/shared`
2. **Database layer** — Prisma queries and models
3. **Service logic** — Business logic independent of routes/UI
4. **API routes** — Call services, handle errors, return responses
5. **UI/components** — Consume API endpoints

**Principles**:
- **Type-safety**: Always define types before implementing. Share types between API and UI.
- **Error handling**: Every async operation needs try/catch → return consistent error responses
- **Testing**: Unit test services first (easier than testing routes)
- **Decoupling**: Services should not depend on routes or UI logic

**Must-Have (MVP)**:
- User auth (register, login, JWT)
- Lesson delivery (static lessons OK)
- Quiz submission (basic MCQ)
- Subscription checkout (Stripe integration)
- Web & mobile apps (functional on both)
- Daily reminders (email or push)

**Nice-to-Have (Post-MVP)**:
- AI lesson generation (Ollama/Mistral)
- Adaptive difficulty (Bayesian skill tracking)
- Leaderboards & social features
- Team/enterprise licensing

**Out of Scope**:
- Live Q&A sessions
- Instructor reviews
- Certification/badges (unless driving retention)
- Complex integrations (Slack, Calendly)

---

## Quick Navigation

**On context reset**: Read these files in order:
1. [docs/PROGRESS.md](../docs/PROGRESS.md) — Current phase status (100 lines, 1 min read)
2. [docs/phases/phase-3-web-frontend.md](../docs/phases/phase-3-web-frontend.md) — Active phase details (only read what you need)
3. Skip completed phase files (they're archived for reference only)

**DO NOT READ**: `docs/BUILD_PLAN.md` (it's been split into the phase files above — use those instead)

---

## Chunk 3.1 Implementation Summary (2026-04-12)

**What was completed**: Full authentication & onboarding flow for Phase 3 web frontend
- Auth context with auto-login on mount using localStorage tokens
- API client with typed fetch wrapper and JWT token auto-injection
- Login, signup, and onboarding pages with form validation
- Skills fetching from backend API in onboarding flow
- Protected routes with redirect to /login for unauthenticated users
- Dashboard layout with navigation and logout functionality
- Testing infrastructure (Jest + React Testing Library) set up

**Key implementation patterns established**:
1. **Auth Flow**: localStorage → auth-context → useAuth hook in components
2. **API Calls**: All routes use apiClient which auto-adds Bearer token
3. **Protected Pages**: Wrapped in ProtectedRoute HOC that checks auth state
4. **Form Validation**: Client-side validation before API submission
5. **Skills Selection**: Fetched dynamically from `/api/skills` on onboarding page
6. **Error Handling**: API errors caught and displayed to user with user-friendly messages

**Build verified**: ✅ Next.js build succeeds with no TypeScript errors, all 7 pages prerendered

**Next steps**: Chunk 3.2 (Dashboard, Lesson Screen, Quiz Screen) can proceed immediately — auth layer is solid and tested

---

## Change Log

**2026-04-12** — ROUTE GROUP FIX (Best Practice Alignment)
- Fixed all frontend page routes to follow Next.js App Router best practices
- Route groups `(auth)/` prevent folder name from appearing in URL path
- Corrected paths: `/auth/login` → `/login`, `/auth/signup` → `/signup`, `/auth/onboarding` → `/onboarding`
- Updated links in: `app/page.tsx`, `app/(auth)/login/page.tsx`, `app/(auth)/signup/page.tsx`
- Updated redirects in: login page (after successful auth), signup page (after registration)
- ✅ Next.js build succeeds with correct route structure
- Reference: [Next.js Route Groups](https://nextjs.org/docs/14/app/building-your-application/routing/route-groups)

**2026-04-12** — CHUNK 3.3 COMPLETE
- Added NotificationPreference model to Prisma schema
- Created NotificationPreferenceService with getPreferences/updatePreferences
- Created NotificationPreferenceController with get/update endpoints
- Added API routes `/api/notifications/preferences` (GET, PATCH)
- Created 11 frontend unit tests for Progress and Settings pages
- Updated Progress page with month calendar view and responsive layout
- Updated Settings page to save notification preferences to backend API
- Added API client methods for notification preference endpoints
- ✅ All 40 backend tests passing (29 existing + 11 new)
- ✅ All 11 frontend tests passing
- ✅ Next.js build succeeds with no errors
- ✅ Phase 3 (Web Frontend) is now 100% COMPLETE

**2026-04-12** — CHUNK 3.2 COMPLETE
- Set up Jest testing infrastructure for backend (ts-jest, jest.config.cjs)
- Wrote 29 comprehensive backend unit tests (LessonService, UserService, controllers)
- ✅ All 29 backend tests passing
- Verified end-to-end flow: Register → Get Lesson → Complete → Submit Quiz → See Results
- Added 3 sample lessons with quizzes to seed data
- Updated Prisma seed script to create lessons (not just skills/plans)
- Verified Next.js build succeeds with no TypeScript errors
- Verified both API and Web servers running correctly
- Tested full lesson completion flow manually with curl
- Updated PROGRESS.md and phase documentation with completion summary

**2026-04-17** — PHASE 8 COMPLETE (Notifications & Habit Engine)
- Schema: `PushToken` model (userId, token, platform "expo"|"web", deviceId, unique on all 3, cascade delete)
- `PushNotificationService`: unified Expo + web-push send, never throws, removes stale tokens
- `StreakService`: `getMilestone()`, `getStreakMessage()`, `isStreakAtRisk()` (timezone-aware)
- Cron jobs: `dailyReminderJob.ts` (hourly, matches preferredTime to UTC hour, 20h idempotency), `streakAtRiskJob.ts` (evening, at-risk check)
- `POST /api/notifications/push-token` added to notifications router
- `QuizResult` shared type: `streak: number`, `milestone: string | null`
- Milestone notifications wired into `LessonService.submitQuiz()` (fire-and-forget, idempotent)
- Web: service worker (`public/sw.js`), `useWebPush` hook, `useDarkMode` hook, `useToast` hook
- Web components: `Toast`/`ToastProvider`, `OfflineBanner`, `Confetti`, `Skeleton` variants
- Web pages fully dark-mode aware (`dark:` Tailwind classes), ARIA throughout
- Tailwind: `darkMode: 'class'`, custom animations (pulse-green, shake, count-up, slide-in, progress-fill)
- Mobile: `useNotifications` hook (expo-notifications, permissions, token registration), global mocks for expo-notifications + expo-device
- Mobile QuizModal: `Animated.spring` score, milestone card, streak row, coloured feedback borders
- Tests: 122 API + 12 web + 149 mobile, all passing

**2026-04-17** — PHASE 6 CHUNK 6.6 + PHASE 7 COMPLETE
- Prisma schema: `GeneratedLesson` (AI lesson cache, skillId key, `coachingMessage`), `UserSkillRating` (Elo), `UserProgress` engagement fields (`completionTime`, `revisitCount`, `rating`)
- `src/ai/`: `SkillTracker` (Elo), `RecommendationEngine` (UCB1), `LearningStyleClassifier` (rule-based)
- `src/services/`: `EngagementService`, `CoachingService` (never throws, null fallback, cache in `GeneratedLesson`)
- `LessonService.submitQuiz()` wires all personalisation; `getTodayLesson()` delegates to `RecommendationEngine`
- `middleware/plan.ts`: `detectPlan` non-blocking middleware added
- `QuizResult.coaching: string | null` in shared types
- Web quiz.tsx: coaching card below explanation (null-safe)
- Mobile QuizModal.tsx: coaching card in results view (null-safe)
- jest.config.cjs renamed in web package (ESM/CJS fix)
- 100 Node API + 12 Web + 144 Mobile tests all passing

**2026-04-12** — CHUNK 3.1 COMPLETE
- Set up Jest + React Testing Library testing infrastructure
- Implemented auth context with auto-login on mount
- Implemented API client with skill fetching endpoint
- Implemented login page with email/password validation  
- Implemented signup page with password strength (8+ chars) and confirmation
- Implemented onboarding page that fetches skills from API, collects timezone/time preferences
- Implemented dashboard layout with protected routes and logout
- Updated root layout with AuthProvider wrapper
- Verified Next.js build succeeds with no errors
- Updated PROGRESS.md and phase documentation with complete details

**2026-04-12** — Earlier
- Refactored BUILD_PLAN.md into separate phase files to save tokens on context reset
- Created [docs/PROGRESS.md](../docs/PROGRESS.md) — Central status tracker (updated after each chunk completion)
- Created [docs/phases/](../docs/phases/) — One file per phase, only read what's active
- Token savings: ~10k per reset (was reading full 12k-line BUILD_PLAN → now reading 2-3k from active phase only)
- **Current Status**: Phase 1 ✅, Phase 2 ✅, Phase 3 ⚪ IN PROGRESS (Chunk 3.1 active — auth pages)

**2026-04-11**
- Initial context created
- Reviewed BUILD_PLAN.md and business plan
- Understood: This is a startup MVP with 6 phases and 12-week target
- Fixed Docker setup: ports 5433/6380 (isolated from pocketchange)
- Fixed TypeScript config: tsconfig.json module → ES2020 (was commonjs)
- Fixed dev scripts: tsx for API, next dev -p 3001 for web (no port conflicts)
