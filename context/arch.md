---
name: Arch Context — Learning App
description: Architectural knowledge and decision log for the Learning App MVP
last_updated: 2026-04-12
---

# Learning App — Architectural Context

## Project Overview

**Product**: AI-native micro-learning platform generating personalized 3-minute daily lessons for product managers, product leaders, and AI practitioners.

**Current Status**: Phase 3 (Web Frontend) ✅ COMPLETE. All auth, dashboard, lesson, quiz, progress, and settings pages functional. Ready for Phase 4 (Mobile App).

**Architecture**: Monorepo (pnpm) with TypeScript, Express backend, Next.js 14 frontend, PostgreSQL + Redis.

---

## System Architecture Layers

### 1. Data Layer (PostgreSQL + Prisma ORM)

**Key Models**:
- **User** — Authentication, email, password hash. Has UserProfile, UserProgress, Subscriptions, Notifications, NotificationPreference.
- **UserProfile** — Extended user data: goal, timezone, learningStyle, preferredTime.
- **Skill** — Skill domains (e.g., "product-management", "ai-engineering"). Linked via SkillPath.
- **SkillPath** — Progression levels (beginner/intermediate/advanced) within a skill. Contains Lessons.
- **Lesson** — Individual lessons (day 1, day 2, etc.) with title, content, media URL. Links to Quizzes and UserProgress.
- **Quiz** — Multiple-choice questions tied to lessons with correctAnswer + explanation.
- **UserProgress** — Tracks completion, quiz score (0-100), streak count, last lesson date per user-lesson pair.
- **Subscription** / **SubscriptionPlan** — Billing model (free, starter, pro, premium). Tracks Stripe IDs, status, expiry.
- **NotificationPreference** — Per-user toggle settings for daily reminders, streak notifications, lesson available alerts.
- **Notification** — Audit log of sent notifications with type + click tracking.

**Schema Patterns**:
- Cascading deletes on user deletion (profile, progress, subscriptions, notifications all cascade).
- Unique constraints: (userId, lessonId) in UserProgress; (userId) in Subscription; (skillId, level) in SkillPath.
- Indexes on userId for fast lookups (UserProgress, Subscription, Notification, NotificationPreference).

---

### 2. Backend API (Express + TypeScript)

**Architecture**: Route → Controller → Service → Prisma

**Key Routes**:
- **Auth** (`/api/auth`): `POST /register`, `POST /login`, `GET /me` (protected)
- **Users** (`/api/users`): GET/PATCH profile, user info (protected)
- **Lessons** (`/api/lessons`): GET today's lesson, GET lesson by ID, POST complete lesson, POST quiz submission (all protected)
- **Subscriptions** (`/api/subscriptions`): GET plans, POST create subscription, webhook handlers (protected)
- **Notifications** (`/api/notifications`): GET/PATCH user notification preferences (protected)

**Auth Pattern**:
- JWT access tokens (15m expiry) stored in Authorization header (Bearer token).
- Refresh tokens (30d) stored as SHA-256 hashes in Redis (`refresh:{userId}`).
- `POST /api/auth/refresh` rotates both tokens. `POST /api/auth/logout` revokes refresh token.
- authMiddleware extracts JWT, validates, sets `req.user` (id + email + role), and sets Sentry user context.
- Web + mobile both have 401 interceptors that auto-refresh silently before redirecting to login.

**Error Handling**:
- AppError class: `new AppError(code, message, statusCode)`.
- Controllers delegate errors to `next(error)` → caught by error-handler middleware.
- Middleware returns structured JSON error responses.

**Service Layer** (Business Logic):
- **AuthService**: register, login, getUserById. Bcryptjs for password hashing.
- **UserService**: getUserProfile, updateUserProfile.
- **LessonService**: getTodayLesson, getNextLesson, completeLessonService (marks progress), submitQuiz (stores score, returns result).
- **SubscriptionService**: getPlanById, getUserSubscription, createSubscription (Stripe stub).
- **NotificationPreferenceService**: getPreferences (fetch or create defaults), updatePreferences.

**Database Seeding**:
- `prisma/seed.ts` creates initial skills (e.g., "Product Strategy", "AI Engineering") with SkillPaths (beginner → lessons).
- Sample lessons added with quizzes.
- Seed runs on `pnpm db:seed`.

---

### 3. Frontend (Next.js 14 + React 18 + TailwindCSS)

**Architecture**: Pages (App Router) → Components → Hooks → API Client

**Key Pages**:
- **Root** (`/`): Landing page with auth checks (redirects to dashboard if logged in, login if not).
- **Auth Routes** (`(auth)` route group — URLs `/login`, `/signup`, `/onboarding`):
  - `login/page.tsx` — Email/password login with API submission.
  - `signup/page.tsx` — Email/password signup with validation (8+ chars, match passwords).
  - `onboarding/page.tsx` — Post-signup: fetches skills from `/api/skills`, user selects skills + timezone + time preference.
- **Dashboard Routes** (`(dashboard)` route group — protected by ProtectedRoute HOC):
  - `page.tsx` — Dashboard home: streak, today's lesson card, start lesson button.
  - `lessons/[id]/page.tsx` — Lesson detail: fetch + display lesson content, complete button.
  - `lessons/[id]/quiz.tsx` — Quiz screen: fetch quiz, display MCQ, submit answer, show score + explanation.
  - `progress.tsx` — Stats card (current streak, total lessons, avg score) + calendar view of completion dates.
  - `settings.tsx` — User profile, notification preferences toggles (daily reminder + time, streak notifications, lesson available alerts), logout.

**Auth State Management**:
- **auth-context.tsx**: Stores JWT in localStorage. On app mount, auto-login if token exists. `useAuth()` hook provides token + setToken + logout.
- AuthProvider wraps app in layout.tsx.
- ProtectedRoute HOC checks auth, redirects unauthenticated users to `/login`.

**API Client**:
- **api-client.ts**: Fetch wrapper that auto-injects Bearer token from auth context.
- Methods: `register()`, `login()`, `getMe()`, `getSkills()`, `getTodayLesson()`, `completeLesson()`, `submitQuiz()`, `getProgress()`, `getNotificationPreferences()`, `updateNotificationPreferences()`.
- Error handling: catches 4xx/5xx responses, throws typed errors for UI to display.

**UI Components** (TailwindCSS + Lucide icons):
- Reusable form inputs (text, email, password, select, checkbox, textarea).
- Cards, buttons, badges for lessons + progress.
- Responsive design: mobile (375px) → tablet (768px) → desktop (1440px). Grids use `grid-cols-1 md:grid-cols-3`, stacked layouts on mobile.

**Type Safety**:
- All types imported from `@learning/shared` (package alias in tsconfig).
- Shared types: User, Lesson, Quiz, UserProgress, Subscription, SubscriptionPlan, NotificationPreference, API response envelopes.

---

## Architectural Patterns & Decisions

### 1. System of Record Principle
**Decision**: All critical state (user auth, lesson completion, quiz scores, subscription status) stored in PostgreSQL. Frontend reads from API only — no client-side caching of state.

**Why**: Prevents desync between clients. Single source of truth for billing, progress, streaks.

### 2. Thin Client (Presentation Layer)
**Decision**: Frontend contains only UI logic + form validation. All business logic (streak calculation, lesson delivery rules, quiz scoring) in backend services.

**Why**: Prevents cheating (client-side score manipulation), ensures consistency across web/mobile, easier to change rules server-side.

### 3. Route → Controller → Service Separation
**Decision**: Express routes define endpoints, controllers validate + call services, services own business logic.

**Why**: Testable (mock services in controller tests), decoupled from HTTP layer, reusable services across routes.

### 4. JWT Authentication (Stateless)
**Decision**: No session store. JWT token issued at login, client stores in localStorage, sends with every API request.

**Why**: Scales horizontally (no session affinity), simple to implement, works well for SPA + mobile.

**Limitation**: No built-in logout (token valid until expiry). **TODO for Phase 6**: Add token expiry claims + refresh token flow to enable revocation.

### 5. Cascading Deletes
**Decision**: Deleting a User cascades to Profile, UserProgress, Subscriptions, Notifications, NotificationPreference.

**Why**: Data integrity. Deleting a user cleans up all dependent data atomically.

### 6. Skill → SkillPath → Lesson Hierarchy
**Decision**: Skills are domains (e.g., "Product Management"). SkillPaths are progression levels within a skill (Beginner → Intermediate → Advanced). Lessons are daily activities within a path.

**Why**: Supports adaptive difficulty. Users select a skill, get assigned a path based on assessment, then progress through lessons.

### 7. Subscription Model (Stripe stubs retained)
**Decision**: Phase 5 (Stripe billing) was cancelled — not required for this product. `Subscription`, `SubscriptionPlan`, `StripeEventLog` models and nullable stripe ID fields remain in schema but are never populated by real payments.

**Why retained**: Subscription tier model (`free/starter/pro/premium`) is still used for AI coaching tier enforcement (`detectPlan` middleware). Stripe fields are harmless stubs.

### 9. Personalization Layer (ADR-002 — accepted, Phase 7)
**Algorithm services** (`packages/api/src/ai/`): `RecommendationEngine` (UCB1 bandit, lesson selection), `SkillTracker` (Elo rating, difficulty routing), `LearningStyleClassifier` (rule-based, style detection). These are computation units — they may read from Prisma but route all writes through the service layer.

**New DB models**: `UserSkillRating` (userId+skillId unique, Elo rating, cascade on user delete); `GeneratedLesson` (Phase 6 6.6 — adds nullable `coachingMessage` field in Phase 7). `UserProgress` gains `completionTime`, `revisitCount`, `rating` fields — `quizScore` already exists.

**Coaching flow**: `CoachingService.generateFeedback()` is called post-quiz for Pro/Premium; returns `null` immediately for free/starter; wraps AI call in try/catch so quiz delivery is never blocked. Cached in `GeneratedLesson.coachingMessage`.

**Quiz response**: `POST /api/lessons/:id/quiz` response gains a `coaching` field (`string | null`). Never omitted — null for free/starter or AI failure. Shape change is backwards-compatible.

### 8. Python AI Microservice (ADR-001 — accepted)
**Decision**: All LLM interactions (lesson generation, quiz generation, AI coaching) handled by a dedicated FastAPI Python service in repo `/Users/guypowell/Documents/Projects/learning-ai`. Node backend calls it over HTTP and validates every response with Zod. Node is system of record; Python is best-effort.

**Stack**: FastAPI + uvicorn (dev), uvicorn + gunicorn (prod, Render). Local models via Ollama (`llama3:8b` default, `mistral:7b` available). Production via Google Vertex AI. Provider switched via `AI_PROVIDER` env var.

**Trust model**: Python = best effort. Node + Zod = source of truth. All responses validated with `schema.safeParse()` before storage or serving. Failures trigger fallback chain: DB cache → static content → 503.

**Reliability**: 30s timeout (AbortController), 3 retries with exponential backoff (~150/300/600ms), circuit breaker via `opossum` (opens after 5 failures in 10s, half-open after 30s).

**Auth**: `X-Internal-API-Key` header shared between services via env var.

**Response mode**: Batch only (not streaming) — required for Zod validation invariant.

**Zod contract**: `packages/shared/src/types/ai.ts` — exported from `@learning/shared`. Single source of truth for Node↔Python data contract.

**Context flow**: Node assembles context (tier, skill_level, user_context, learning_style) → passes to Python. Python selects model and prompt variant. Node never has prompt templates or model clients.

**Full ADR**: `docs/adr/ADR-001-python-ai-service.md`

---

## Module Interactions

### Authentication Flow
1. User submits email + password to `/api/auth/register` or `/api/auth/login`.
2. AuthService hashes password (bcryptjs), validates, generates JWT.
3. Frontend stores JWT in localStorage, adds to Authorization header.
4. Protected routes checked by authMiddleware → validates JWT → sets req.user.

### Lesson Delivery Flow
1. Frontend calls `GET /api/lessons/today` (with auth).
2. LessonService fetches next uncompleted lesson for user based on selected skill path.
3. Frontend displays lesson content + "Complete" button.
4. User clicks → frontend calls `POST /api/lessons/:id/complete`.
5. LessonService marks UserProgress.completedAt, updates streak.
6. Frontend displays quiz screen.
7. User submits quiz → frontend calls `POST /api/lessons/:id/quiz`.
8. LessonService validates answer, stores quiz score, returns feedback.
9. Progress page fetches stats from `GET /api/users/progress`, displays streak + calendar.

### Notification Preferences Flow
1. Settings page loads → calls `GET /api/notifications/preferences`.
2. NotificationPreferenceService fetches or creates defaults.
3. User toggles options → calls `PATCH /api/notifications/preferences`.
4. Backend updates database. Frontend shows success message.
5. (Phase 5): Cron job reads NotificationPreference to send emails/push notifications.

---

## Data Flow & State Management

### Frontend State
- **Auth State** (auth-context): JWT token, user ID, email. Persists to localStorage.
- **Page State** (local useState): Form inputs, loading states, API responses.
- **No Redux/Zustand**: Simple useAuth hook + API client. Sufficient for MVP.

### Backend State
- **In-Memory**: Express app, middleware stack, Prisma client instance.
- **Persistent**: All user data in PostgreSQL.
- **Cache (future)**: Redis for job queues, notification scheduling (Phase 5).

---

## Integration Points

### Stripe (Stubbed, Phase 2+ scope)
- SubscriptionService.createSubscription() should call Stripe API.
- Webhook endpoint at `POST /api/subscriptions/webhook` to handle Stripe events.
- Currently mocked; no real payment processing.

### OpenAI / Mistral (Future, Phase 2+)
- Lesson content generated by LLM (not hardcoded).
- LessonService.generateLesson() will call OpenAI/Mistral API (not yet implemented).

### Email Notifications (Future, Phase 5)
- Cron job reads NotificationPreference.
- Calls email service (SendGrid or similar).
- Tracks sent notifications in Notification model.

---

## Architectural Gaps & Technical Debt

| Issue | Impact | Severity | Phase | Status |
|-------|--------|----------|-------|--------|
| **No token expiry/refresh** | Leaked tokens valid forever | MEDIUM | 11 | ✅ FIXED — 15m access / 30d refresh via Redis |
| **No rate limiting** | API vulnerable to abuse | MEDIUM | 11 | ✅ FIXED — authLimiter / aiLimiter / generalLimiter |
| **CORS open (*)** | Any origin can call API | MEDIUM | 11 | ✅ FIXED — ALLOWED_ORIGINS allowlist |
| **No input sanitisation** | XSS risk on lesson content | MEDIUM | 11 | ✅ FIXED — sanitize-html on admin lesson save |
| **Stripe integration stubbed** | Can't charge users | N/A | — | 🚫 CANCELLED — not required |
| **No analytics** | Can't track user behaviour | LOW | 11.2 | ⚪ DEFERRED — provider not chosen |
| **No email notifications** | Can't remind users | MEDIUM | 12 | Push notifications live (Phase 8); email deferred |
| **No CI/CD** | No quality gate on push | MEDIUM | 12 | Phase 12 scope |

---

## Performance Considerations

### Database Queries
- UserProgress queries on userId (indexed). Fast for "lessons completed by user".
- Subscription queries on userId (unique). Single read per user.
- Indexes prevent N+1 queries on Lesson → Quiz relationships.

### API Response Size
- Lessons include full content (markdown). May benefit from pagination (future optimization).
- Quiz responses small (question + options).

### Frontend Bundle Size
- Next.js 14 with dynamic imports on route groups. Auto-code-splitting.
- TailwindCSS (purged on build). Lucide icons (tree-shakeable).

---

## Security Posture

### Input Validation
- Controllers validate email format, password length (8+ chars), required fields.
- Frontend validates form inputs before submission.
- Backend validates all request bodies (no blind trust).

### Authentication
- Passwords hashed with bcryptjs (salted).
- JWT signed (HMAC-SHA256). Secret stored in .env.

### Sensitive Data
- JWT tokens in Authorization header (not cookies — prevents CSRF but requires manual handling in mobile).
- Refresh tokens stored hashed in Redis, never in DB or client logs.
- Database connection string in .env.

### Hardened in Phase 11
- ✅ CORS locked to `ALLOWED_ORIGINS` allowlist (was `*`).
- ✅ Rate limiting on auth (5/15min), AI (10/min), general (100/min).
- ✅ Admin lesson content sanitised with `sanitize-html` before storage.
- ✅ Sentry error tracking wired (API + web + mobile); no-ops without DSN.
- ✅ Request logger + Prisma slow query logging + AI latency logging.

### Remaining gaps
- No HTTPS enforcement (dev only — handled at infra layer in prod).
- No CI/CD quality gate (Phase 12 scope).

---

## Change Log

**2026-06-08** — Phase 11: Security, Analytics & Observability (11.1, 11.3, 11.4 complete; 11.2 deferred)
- JWT auth upgraded: 15m access tokens + 30d refresh tokens stored hashed in Redis
- New routes: `POST /api/auth/refresh`, `POST /api/auth/logout`
- Rate limiting middleware added (auth/AI/general tiers)
- CORS hardened to allowlist (`ALLOWED_ORIGINS` env var)
- Lesson content sanitised with `sanitize-html` on admin save
- Sentry wired across API, web, mobile (graceful no-op without DSN)
- Request logger, Prisma slow query logging, AI latency logging added
- Phase 5 (Stripe) cancelled; Phase 10.2 (Team Stripe) cancelled
- API test count: 206 (was 184)

**2026-04-17** — ADR-003: Push Notification Architecture (Phase 8)
- Mobile: Expo Push Service (`expo-server-sdk`) — no Firebase account required
- Web: Web Push API (`web-push` npm) + VAPID keys — no external account required
- New `PushToken` model: `(userId, platform, deviceId)` unique, cascade on user delete — supports multi-device
- Unified `PushNotificationService` abstraction routes by platform — never throws (notification failure is non-blocking)
- `node-cron` for MVP (daily + evening jobs in `packages/api/src/jobs/`) — Bull deferred to Phase 11
- New `StreakService`: pure-function milestone/message helpers + `isStreakAtRisk()` (reads UserProgress + timezone)
- Cron idempotency: 20-hour guard via `Notification` table before any send
- VAPID keys: `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` in env — never committed
- `POST /api/notifications/push-token` added to notifications router (protected)
- Full ADR: `docs/adr/ADR-003-push-notification-architecture.md`

**2026-04-17** — ADR-002: Node Personalization Architecture (Phase 7)
- `quizScore` already exists on `UserProgress` — only add `completionTime`, `revisitCount`, `rating`
- Algorithm services (`RecommendationEngine`, `SkillTracker`, `LearningStyleClassifier`) in `packages/api/src/ai/` (computation units, not data-access services)
- Coaching tier enforcement in `CoachingService.generateFeedback()` not route middleware — quiz route open to all tiers
- Coaching cache in `GeneratedLesson.coachingMessage` (nullable String) — not Redis
- `LearningStyleClassifier` returns style string; caller invokes `UserService.updateUserProfile()` — no direct DB writes
- `Skill` model needs `userSkillRatings UserSkillRating[]` backrelation for migration
- Phase 6 Chunk 6.6 prerequisite: `GeneratedLesson` model must exist before Phase 7 coaching cache

**2026-04-14** — ADR-001: Python AI Microservice
- Accepted architecture for all AI interactions: FastAPI Python service in `learning-ai` repo
- Ollama (llama3:8b + mistral:7b) local; Vertex AI prod; switched via AI_PROVIDER env var
- Node trust model: Zod validates all Python responses; fallback chain on failure
- Reliability: 30s timeout, 3 retries exp backoff, opossum circuit breaker
- Phase 6 (AI generation) and Phase 7 (personalization) updated to reflect this architecture
- Phase 5 (Stripe billing) unchanged
- ADR written at `docs/adr/ADR-001-python-ai-service.md`

**2026-04-12** — Initialization Complete
- Read fs.md (full stack context).
- Read Prisma schema: 11 models with cascading deletes, unique constraints, indexes.
- Read API structure: 5 route modules, service layer, error handling middleware.
- Read frontend structure: Next.js 14 with route groups, auth context, API client.
- Documented system architecture, module interactions, data flows.
- Identified architectural patterns (system of record, thin client, JWT auth, cascading deletes).
- Identified gaps (token expiry, rate limiting, Stripe stubbed, AI generation stubbed, email notifications).
- All decisions documented with reasoning and Phase assignments.

---
