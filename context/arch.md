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
- JWT tokens stored in Authorization header (Bearer token).
- authMiddleware extracts JWT, validates, and sets `req.user` (contains userId + email).
- Protected routes check authMiddleware before controller execution.
- No token expiry/refresh yet (TODO for Phase 6 launch).

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

### 7. Subscription Stripe Integration (Stubbed)
**Decision**: SubscriptionPlan models exist with pricing. Subscription model has stripeCustomerId + stripeSubscriptionId fields. Webhook routes exist but not fully wired.

**Why**: Prepared for Stripe integration (Phase 2+ stretch goal). Can mock subscriptions for frontend development.

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

| Issue | Impact | Severity | Phase |
|-------|--------|----------|-------|
| **No token expiry/refresh** | Leaked tokens valid forever | MEDIUM | 6 (launch) |
| **No rate limiting** | API vulnerable to abuse | MEDIUM | 6 (launch) |
| **Stripe integration stubbed** | Can't charge users | HIGH | 2+ (stretch) |
| **AI lesson generation stubbed** | Lessons hardcoded in DB | MEDIUM | 2+ (stretch) |
| **No email notifications** | Can't remind users | MEDIUM | 5 (polish) |
| **No mobile app** | iOS/Android unavailable | HIGH | 4 (scope) |
| **No analytics/observability** | Can't track user behavior | LOW | Future |

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
- Database connection string in .env.
- Stripe API key in .env (future).

### Missing Mitigations (TODO for Phase 6)
- No HTTPS enforcement (dev only).
- No CORS whitelist (currently `*`).
- No rate limiting on auth endpoints (DDoS risk).
- No input sanitization (XSS risk if lesson content user-generated).

---

## Change Log

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
