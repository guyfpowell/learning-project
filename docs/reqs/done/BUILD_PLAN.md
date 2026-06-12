# Learning App — Detailed Build Plan

> ⚠️ **This file has been archived.** See [PROGRESS.md](./PROGRESS.md) for current status and [docs/phases/](./phases/) for detailed phase documentation.


## Overview

This document breaks down the MVP build into meaningful, sequenced chunks. **Target: 12 weeks to MVP launch** with the following stack:

- **Monorepo** (`learning/`): Backend (Express/TypeScript), Web Frontend (Next.js/React/TypeScript), Shared Types (Prisma, TypeScript)
- **Mobile Repo** (`learning-app/`): React Native with Expo
- **Infrastructure**: Vercel (web), Render (API + Redis), Supabase (PostgreSQL), Ollama (self-hosted Mistral 7B for AI lesson generation)
- **Local Dev**: Docker Compose (all services locally)

---

## Phase 1: Foundation & Infrastructure

**Goal**: Set up development environment, infrastructure, and skeleton code.

### Chunk 1.1: Local Development Setup

#### 1.1 Monorepo Structure (`learning/`)
- [ ] Create folder structure:
  - `packages/api` (Express backend)
  - `packages/web` (Next.js frontend)
  - `packages/middleware` (shared middleware & auth)
  - `packages/shared` (TypeScript types, constants)
  - `prisma/` (schema.prisma, migrations)
  - `.github/workflows/` (CI/CD)
- [ ] Create root `package.json` with pnpm workspaces
- [ ] Create `tsconfig.json` (root config)
- [ ] Create `docker-compose.yml` (local: PostgreSQL, Redis, optional Ollama)

**Deliverable**: Monorepo folder structure ready for code.

### Chunk 1.2: Backend Skeleton (`packages/api`)
- [ ] Initialize Express/TypeScript project
- [ ] Create basic folder structure:
  - `src/routes/` (empty)
  - `src/controllers/` (empty)
  - `src/services/` (empty)
  - `src/middleware/` (empty)
  - `src/index.ts` (basic Express app)
- [ ] Set up TypeScript compilation
- [ ] Create `package.json` with: `express`, `typescript`, `@types/express`, `nodemon`, `dotenv`

**Deliverable**: Express server boots (no routes yet).

### Chunk 1.3: Web Frontend Skeleton (`packages/web`)
- [ ] Initialize Next.js project (TypeScript)
- [ ] Create basic folder structure:
  - `app/layout.tsx` (root layout)
  - `app/page.tsx` (home page, placeholder)
  - `components/` (empty)
  - `lib/` (utilities)
- [ ] Install: `react`, `next`, `tailwindcss`, `@shadcn/ui`
- [ ] Configure Tailwind + Shadcn

**Deliverable**: Next.js app builds and runs locally.

### Chunk 1.4: Shared Types (`packages/shared`)
- [ ] Create TypeScript package with shared types:
  - `src/types/lesson.ts` (Lesson, SkillPath, LessonContent)
  - `src/types/user.ts` (User, UserProfile, UserProgress)
  - `src/types/subscription.ts` (SubscriptionPlan, Subscription)
  - `src/types/api.ts` (API request/response envelopes)
  - `src/constants/skills.ts` (initial skill path definitions)
  - `src/constants/error-codes.ts` (app error codes)
- [ ] Configure exports in `package.json`

**Deliverable**: Shared types package can be imported by API and web.

### Chunk 1.5: Docker Compose Setup
- [ ] Create `docker-compose.yml`:
  - PostgreSQL 15 (port 5432)
  - Redis 7 (port 6379)
  - Optional: Ollama placeholder (port 11434)
- [ ] Create `.env.local` template
- [ ] Test: `docker-compose up` starts all services

**Deliverable**: All services run locally with one command.

### Chunk 1.6: Database & Prisma

#### 1.6.1 Prisma Schema
- [ ] Create `prisma/schema.prisma`:
  - User model (id, email, passwordHash, name, createdAt, updatedAt)
  - UserProfile model (userId, goal, preferredTime, timezone, learningStyle)
  - Skill model (id, name, description, category)
  - SkillPath model (id, skillId, level, durationHours)
  - Lesson model (id, skillPathId, day, title, content, durationMinutes, mediaUrl, difficulty)
  - UserProgress model (userId, lessonId, completedAt, quizScore, streakCount)
  - Subscription model (userId, planId, status, startedAt, renewedAt, expiresAt)
  - Notification model (id, userId, type, sentAt, clickedAt)
- [ ] Configure `datasource` (PostgreSQL via Supabase connection string in `.env`)
- [ ] Run `prisma migrate dev --name init` (creates migration)

**Deliverable**: Database schema version-controlled, migration generated.

#### 1.6.2 Prisma Client & Database Utility
- [ ] Create `src/db.ts` in `packages/api`:
  - Export Prisma client instance
  - Add logging in development
- [ ] Test connection to local PostgreSQL (via Docker)

**Deliverable**: API can connect to database.

#### 1.6.3 Seed Script (Optional)
- [ ] Create `prisma/seed.ts`:
  - Seed initial skills (AI Product Strategy, Prompt Engineering, etc.)
  - Seed subscription plans (Free, Starter $19, Pro $49, Premium $99)
- [ ] Add seed command to `package.json`: `prisma db seed`

**Deliverable**: Database can be reset and re-seeded easily.

---

## Phase 2: Core Backend API

**Goal**: Build essential API endpoints with authentication, lesson delivery, and subscription management.

### Chunk 2.1: Authentication & User Management

#### 2.1.1 Auth Middleware
- [ ] Create `packages/middleware/src/auth/`:
  - `jwt.ts` (sign/verify JWT)
  - `passwords.ts` (hash/compare using bcrypt)
  - `middleware.ts` (Express middleware for Protected routes)
- [ ] Create auth routes in `packages/api/src/routes/auth.ts`:
  - `POST /auth/register` (email, password, name → create User + UserProfile)
  - `POST /auth/login` (email, password → JWT token)
  - `POST /auth/logout` (invalidate token)
- [ ] Create controller: `packages/api/src/controllers/AuthController.ts`

**Deliverable**: Users can register/login. JWT tokens in responses.

#### 2.1.2 User Profile Routes
- [ ] Create `packages/api/src/routes/users.ts`:
  - `GET /users/me` (Protected: fetch current user + profile)
  - `PATCH /users/me` (Protected: update user profile)
  - `GET /users/me/progress` (Protected: user progress data)
- [ ] Create controller: `packages/api/src/controllers/UserController.ts`

**Deliverable**: Authenticated users can fetch/update their profile.

#### 2.1.3 Error Handling Middleware
- [ ] Create `packages/middleware/src/errors/`:
  - `AppError.ts` (custom error class)
  - `error-handler.ts` (Express error middleware)
- [ ] Integrate into API

**Deliverable**: API returns consistent error responses.

### Chunk 2.2: Lesson Delivery Engine

#### 2.2.1 Lesson Routes
- [ ] Create `packages/api/src/routes/lessons.ts`:
  - `GET /lessons/today` (Protected: fetch today's lesson for user)
  - `GET /lessons/:id` (Protected: fetch specific lesson by ID)
  - `POST /lessons/:id/complete` (Protected: mark lesson complete)
  - `POST /lessons/:id/quiz` (Protected: submit quiz answer, get feedback)
- [ ] Create controller: `packages/api/src/controllers/LessonController.ts`

**Deliverable**: API can serve daily lessons and accept quiz submissions.

#### 2.2.2 Lesson Service Logic
- [ ] Create `packages/api/src/services/LessonService.ts`:
  - `getTodayLesson(userId)`: fetch user's today lesson (based on skill path + day counter)
  - `getNextLesson(userId)`: determine next lesson (basic logic: increment day, or use multi-armed bandit)
  - `completeLessonService(userId, lessonId)`: mark complete, update streaks
  - `submitQuiz(userId, lessonId, answers)`: evaluate answers, return score + feedback
- [ ] Integrate Bayesian skill tracking (simple ELO-style rating)

**Deliverable**: Lesson logic can be tested independently.

#### 2.2.3 Template-Based Lesson Generation (Optional for MVP)
- [ ] Create `packages/api/src/services/LessonGenerationService.ts`:
  - `generateLessonVariants(skillPathId)`: call Ollama API (or mock for testing)
  - Caching layer: store generated lessons in PostgreSQL
- [ ] Mock implementation: return static template-based lessons for MVP

**Deliverable**: Framework for AI lesson generation (mock for now).

### Chunk 2.3: Subscriptions & Billing

#### 2.3.1 Subscription Routes
- [ ] Create `packages/api/src/routes/subscriptions.ts`:
  - `GET /subscriptions/plans` (Public: list available plans)
  - `POST /subscriptions/checkout` (Protected: initiate Stripe checkout)
  - `GET /subscriptions/status` (Protected: current subscription status)
  - `POST /subscriptions/cancel` (Protected: cancel subscription)
- [ ] Create controller: `packages/api/src/controllers/SubscriptionController.ts`

**Deliverable**: API can manage subscriptions.

#### 2.3.2 Stripe Integration (Basic)
- [ ] Create `packages/api/src/services/StripeService.ts`:
  - `createCheckoutSession(userId, planId)`: call Stripe API
  - `verifyWebhook(event)`: handle Stripe webhooks (payment_intent.succeeded, etc.)
- [ ] Create webhook route: `POST /webhooks/stripe`
- [ ] Update Subscription model on webhook

**Deliverable**: Checkout flow can be tested (Stripe test mode).

#### 2.3.3 Plan Configuration
- [ ] Define subscription plans in code/database:
  - Free: 1 lesson/day, basic quiz
  - Starter: $19/month, unlimited lessons
  - Pro: $49/month, includes AI coaching
  - Premium: $99/month (future)

**Deliverable**: Plan data available via API.

## Phase 3: Web Frontend

**Goal**: Build responsive Next.js web app for desktop/tablet users.

### Chunk 3.1: Authentication & Onboarding

#### 3.1.1 Auth Pages
- [ ] Create `app/(auth)/` layout:
  - `app/(auth)/signup.tsx` (Sign up form)
  - `app/(auth)/login.tsx` (Login form)
  - `app/(auth)/onboarding.tsx` (Goal/time preference selection)
- [ ] Create `lib/api-client.ts`:
  - Typed fetch wrapper for API calls
  - Handle JWT token storage (localStorage or httpOnly cookie)
  - Auto-attach token to requests

**Deliverable**: Users can sign up and log in.

#### 3.1.2 Protected Layout & Navigation
- [ ] Create `app/(dashboard)/` layout:
  - Protected routes (require JWT token)
  - Navigation sidebar/header
  - User menu
- [ ] Create `components/ProtectedRoute.tsx` (client-side route guard)

**Deliverable**: Authenticated users see dashboard; unauthenticated redirects to login.

#### 3.1.3 Onboarding Flow
- [ ] `app/(auth)/onboarding.tsx`:
  - Skill selection (dropdown or cards)
  - Time preference (morning, afternoon, evening)
  - Submit to `PATCH /users/me`

**Deliverable**: New users can complete onboarding.

### Chunk 3.2: Dashboard & Lesson Screen

#### 3.2.1 Dashboard/Home Page
- [ ] `app/(dashboard)/page.tsx`:
  - Display user's current streak
  - Show today's lesson (if available)
  - Display next lesson date/time
  - Link to today's lesson

**Deliverable**: Users see their progress at a glance.

#### 3.2.2 Lesson Screen
- [ ] `app/(dashboard)/lessons/today.tsx`:
  - Fetch today's lesson via `GET /lessons/today`
  - Display lesson content (title, description, learning points)
  - Show duration (3 minutes)
  - Button: "Complete & Take Quiz" → Mark complete + show quiz

**Deliverable**: Users can read lesson and access quiz.

#### 3.2.3 Quiz Screen
- [ ] `app/(dashboard)/lessons/[id]/quiz.tsx`:
  - Display quiz questions (MCQ, short answer)
  - Submit answers via `POST /lessons/:id/quiz`
  - Show results + feedback + score
  - Show next lesson recommendation

**Deliverable**: Users can take quiz and see results.

### Chunk 3.3: Progress & Settings

#### 3.3.1 Progress Page
- [ ] `app/(dashboard)/progress.tsx`:
  - Display current streak
  - Calendar view of completed lessons
  - Stats: total lessons, average quiz score
  - Skill breakdown (which skills have been completed)

**Deliverable**: Users can see learning progress.

#### 3.3.2 Profile & Settings
- [ ] `app/(dashboard)/settings.tsx`:
  - Display user profile (email, name, goal)
  - Notification preferences (time, frequency)
  - Logout button
  - Subscription info + upgrade link

**Deliverable**: Users can manage settings.

#### 3.3.3 Responsive Design
- [ ] Test on mobile (375px), tablet (768px), desktop (1440px)
- [ ] Ensure touch-friendly buttons and readable text

**Deliverable**: Web app works on all screen sizes.

## Phase 4: Mobile App

**Goal**: Build React Native/Expo app with same core functionality as web.

### Chunk 4.1: Mobile Project Setup & Auth

#### 4.1.1 Expo Project Initialization
- [ ] Create `learning-app/` with Expo:
  - `npx create-expo-app learning-app`
  - Set up TypeScript
  - Install dependencies: `react-native`, `expo`, `expo-router`
- [ ] Create folder structure:
  - `app/(auth)/` (login, signup, onboarding)
  - `app/(tabs)/` (lessons, progress, profile, settings)
  - `components/` (reusable components)
  - `hooks/` (custom hooks)
  - `services/` (API client, storage)

**Deliverable**: Expo project boots in Expo Go.

#### 4.1.2 Auth Flow
- [ ] Create `app/(auth)/login.tsx` and `app/(auth)/signup.tsx`
- [ ] Create `hooks/useAuth.ts` (manage auth state + token storage)
- [ ] Use Expo Secure Store for token storage (instead of localStorage)

**Deliverable**: Users can log in on mobile.

#### 4.1.3 Navigation Setup
- [ ] Create `app/(tabs)/_layout.tsx` (bottom tab navigation)
- [ ] Tabs: Lessons, Progress, Profile, Settings
- [ ] Create `app/_layout.tsx` (root navigator with auth check)

**Deliverable**: Navigation structure in place.

### Chunk 4.2: Mobile Features

#### 4.2.1 Lessons Tab
- [ ] `app/(tabs)/lessons.tsx`:
  - Fetch today's lesson
  - Display lesson card (same as web)
  - Button to take quiz
  - Show lesson duration, difficulty

**Deliverable**: Users can see lessons on mobile.

#### 4.2.2 Quiz Tab / Quiz Modal
- [ ] Create `components/QuizModal.tsx`:
  - Display quiz questions
  - Submit answers
  - Show results + feedback
- [ ] Integrate into lessons tab

**Deliverable**: Users can take quiz on mobile.

#### 4.2.3 Progress & Profile Tabs
- [ ] `app/(tabs)/progress.tsx`:
  - Streak counter
  - Calendar view (calendar library)
  - Stats
- [ ] `app/(tabs)/profile.tsx`:
  - User info, settings, logout

**Deliverable**: All core screens functional on mobile.

## Phase 5: Notifications & Polish

**Goal**: Add daily reminders, push notifications, and UX refinements.

### Chunk 5.1: Push Notifications (Web)
- [ ] Set up Firebase Cloud Messaging (FCM) / OneSignal
- [ ] Store device token in Subscription table
- [ ] Create `src/services/NotificationService.ts`:
  - `sendDailyReminder(userId)`: send at user's preferred time
- [ ] Webhook: trigger notification at 2 AM UTC (batch process all users)

**Deliverable**: Users receive daily reminders at preferred time.

### Chunk 5.2: Push Notifications (Mobile)
- [ ] `hooks/useNotifications.ts`:
  - Request permission on launch
  - Store FCM/Expo token to backend
  - Handle incoming notifications (launch lesson screen)

**Deliverable**: Mobile users get push notifications.

### Chunk 5.3: Habit Reinforcement
- [ ] Add streak counter with visual emphasis
- [ ] Add "keep the streak alive" messaging on every lesson
- [ ] Add celebration on milestone streaks (7, 14, 30, 100 days)

**Deliverable**: Users feel motivated to maintain daily habit.

### Chunk 5.4: UX Polish
- [ ] Loading states on all async operations
- [ ] Error messages (user-friendly)
- [ ] Success toasts on lesson completion
- [ ] Animations: streak counter increment, quiz answer feedback
- [ ] Accessibility: ARIA labels, keyboard navigation

**Deliverable**: App feels polished and responsive.

## Phase 6: Testing & Launch

**Goal**: Test fully, deploy, and make live.

### Chunk 6.1: Testing
- [ ] Unit tests for critical services:
  - LessonService (getTodayLesson, completeLessonService)
  - AuthService (register, login)
  - SubscriptionService (billing logic)
- [ ] Integration tests for API endpoints
- [ ] End-to-end tests (login → lesson → quiz flow)

**Deliverable**: Core flows tested and pass.

### Chunk 6.2: Staging Deployment
- [ ] Deploy API to Render (staging branch)
- [ ] Deploy web to Vercel (staging branch)
- [ ] Configure Supabase staging database (or use prod for MVP)
- [ ] Test full flow: sign up, lesson, subscription

**Deliverable**: Staging environment mirrors production.

### Chunk 6.3: Production Deployment
- [ ] Deploy API to Render (main branch)
- [ ] Deploy web to Vercel (main branch)
- [ ] Set up Supabase production database
- [ ] Configure production environment variables
- [ ] Test in production

**Deliverable**: App is live at domain.

### Chunk 6.4: Mobile App Store
- [ ] Build APK for Android (Expo):
  - `eas build --platform android`
  - Upload to Google Play
- [ ] Build IPA for iOS (Expo):
  - `eas build --platform ios`
  - Upload to App Store (or TestFlight first)
- [ ] Set up store listings, screenshots, description

**Deliverable**: Native apps available on stores.

### Chunk 6.5: Launch Activities
- [ ] Email waitlist (first lesson is free)
- [ ] Post on ProductHunt
- [ ] Share with PM network (LinkedIn, Twitter)
- [ ] Monitor analytics + errors (Sentry)

**Deliverable**: First users acquired.

## Implementation Priorities

### Must-Have (MVP)
- User auth (register, login, JWT)
- Lesson delivery (static lessons for MVP, template-based)
- Quiz submission (basic MCQ)
- Subscription checkout (Stripe integration)
- Web & mobile apps (working on both platforms)
- Daily reminders (email or push)

### Nice-to-Have (Post-MVP, Weeks 13+)
- AI lesson generation (Ollama/Mistral integration)
- Adaptive difficulty (Bayesian skill tracking)
- Leaderboards & social features
- Team/enterprise licensing
- Advanced analytics

### Out of Scope (Future)
- Live Q&A sessions
- Instructor reviews
- Certification/badges (unless driving retention)
- Complex integrations (Slack, Calendly, etc.)

---

## Chunks Checklist

| Phase | Chunk | Status | Dependencies |
|-------|-------|--------||
| 1 | Monorepo + Docker setup | ✅ | None |
| 1 | Backend skeleton | ✅ | 1.1 |
| 1 | Web frontend skeleton | ✅ | 1.1 |
| 1 | Shared types | ✅ | 1.1 |
| 1 | Docker Compose | ✅ | 1.2, 1.3 |
| 1 | Database & Prisma | ✅ | 1.5 |
| 2 | Auth & user management | ✅ | 1.6 |
| 2 | Lesson delivery engine | ✅ | 2.1 |
| 2 | Subscriptions & billing | ✅ | 2.1 |
| 3 | Web auth & onboarding | ⚪ | 2.1, 1.4 |
| 3 | Web dashboard & lesson | ⚪ | 3.1, 2.2 |
| 3 | Web progress & settings | ⚪ | 3.2 |
| 4 | Mobile project & auth | ⚪ | 2.1, 1.4 |
| 4 | Mobile features | ⚪ | 4.1, 2.2, 2.3 |
| 5 | Notifications | ⚪ | 2.1 |
| 5 | Polish & UX | ⚪ | 3, 4 |
| 6 | Testing | ⚪ | 2, 3, 4 |
| 6 | Staging deployment | ⚪ | 6.1 |
| 6 | Production deployment | ⚪ | 6.2 |
| 6 | Mobile app store | ⚪ | 6.2 |
| 6 | Launch | ⚪ | 6.3, 6.4 |

---

## Success Metrics (Post-Launch)

- **Day 1 retention**: >70% of sign-ups complete first lesson
- **Day 7 retention**: >40% return for second lesson
- **Conversion to paid**: >10% of free users upgrade within 30 days
- **Lesson completion**: >85% of assigned lessons completed
- **App rating**: >4.5/5 on stores
- **Streak sustainability**: Average streak >7 days

---

## Notes for Haiku

When coding each phase:
1. Start with types (from `packages/shared`)
2. Build database layer (Prisma queries)
3. Build service logic (independent of routes)
4. Build API routes (call services)
5. Build UI/components (consume API)

This keeps code testable and decoupled.

**Type-safety first**: Always define types before implementing. Share types between API and UI.

**Error handling**: Every async operation should have try/catch → return consistent error responses.

**Testing**: Write unit tests for services; they're easier to test than routes.

---

## Completion Log

### ✅ Chunk 1.1: Local Development Setup
**Status**: Completed  
**Files Created**:
- `package.json` (root monorepo with pnpm workspaces)
- `tsconfig.json` (root TypeScript config with path aliases)
- `docker-compose.yml` (PostgreSQL 16 + Redis 7, optional Ollama)
- Folder structure: `packages/{api,web,middleware,shared}`, `prisma/`, `.github/workflows/`

**What It Does**: Sets up monorepo foundation with TypeScript compilation and Docker services for local development.

---

### ✅ Chunk 1.2: Backend Skeleton (`packages/api`)
**Status**: Completed  
**Files Created**:
- `packages/api/package.json` (Express, TypeScript, Prisma client, dev tools)
- `packages/api/tsconfig.json` (extends root config)
- `packages/api/src/index.ts` (Express app with /health and /api endpoints)
- `packages/api/.env.example` (environment template)
- Folder structure: `src/{routes,controllers,services,middleware}`

**What It Does**: Express server boots on port 3000 with basic health check. Ready to add routes.

---

### ✅ Chunk 1.3: Web Frontend Skeleton (`packages/web`)
**Status**: Completed  
**Files Created**:
- `packages/web/package.json` (Next.js 14, React 18, Tailwind, ESLint)
- `packages/web/tsconfig.json` (extends root config)
- `packages/web/tailwind.config.ts` (Tailwind CSS configuration)
- `packages/web/postcss.config.js` (PostCSS setup)
- `packages/web/next.config.js` (Next.js configuration)
- `packages/web/app/layout.tsx` (root layout)
- `packages/web/app/globals.css` (Tailwind directives + global styles)
- `packages/web/app/page.tsx` (home page placeholder)
- `packages/web/.eslintrc.json` (Next.js linting)
- `packages/web/.env.example` (environment template)
- `.gitignore` (Next.js defaults)
- Folder structure: `app/`, `lib/`, `components/`, `public/`

**What It Does**: Next.js app with Tailwind CSS configured. Home page displays placeholder. Ready to add components and auth flows.

---

### ✅ Chunk 1.4: Shared Types (`packages/shared`)
**Status**: Completed  
**Files Created**:
- `packages/shared/package.json` (TypeScript types package)
- `packages/shared/tsconfig.json` (extends root config with declaration flag)
- `packages/shared/src/index.ts` (barrel export)
- `packages/shared/src/types/lesson.ts` (Lesson, LessonContent, Skill, SkillPath, Quiz)
- `packages/shared/src/types/user.ts` (User, UserProfile, UserProgress, UserAuth)
- `packages/shared/src/types/subscription.ts` (SubscriptionPlan, Subscription, SubscriptionEvent)
- `packages/shared/src/types/api.ts` (ApiResponse, ApiListResponse, AuthRequest, AuthResponse, ErrorResponse)
- `packages/shared/src/constants/skills.ts` (SKILLS object, SKILL_CATEGORIES, DIFFICULTY_LEVELS)
- `packages/shared/src/constants/error-codes.ts` (ERROR_CODES, ERROR_MESSAGES)

**What It Does**: Shared types package can be imported by API and web. Provides single source of truth for data structures across all apps.

---

### ✅ Chunk 1.5: Docker Compose Setup
**Status**: Completed  
**Files Created**:
- `docker-compose.yml` (PostgreSQL 5433, Redis 6380, optional Ollama 11435)
- `packages/api/.env.local` (from .env.example)
- `packages/web/.env.local` (from .env.example)
- `.gitignore` (root-level for node_modules, .env.local, dist, etc.)
- `README.md` (local dev quick start guide)

**Ports Configured**:
- PostgreSQL: 5433 (doesn't conflict with pocketchange on 5432)
- Redis: 6380 (doesn't conflict with pocketchange on 6379)
- Ollama: 11435 (optional, for local LLM inference)
- API: 3000
- Web: 3001

**What It Does**: All services can be started with `docker-compose up`. PostgreSQL and Redis configured with health checks. Separate from existing pocketchange containers.

**Startup**:
```bash
# Install dependencies
pnpm install

# Start Docker services
docker-compose up -d

# Start development servers
pnpm dev
```

---

### ✅ Chunk 1.6: Database & Prisma
**Status**: Completed  
**Files Created**:
- `prisma/schema.prisma` (complete database schema: User, Skill, Lesson, Quiz, Subscription, Notification models)
- `packages/api/src/db.ts` (Prisma client singleton with development logging)
- `prisma/seed.ts` (seed script: creates 6 skills, 4 subscription plans, 1 skill path)
- Updated `package.json` with Prisma scripts (`db:push`, `db:seed`, `db:reset`)

**Database Schema Includes**:
- **User & Profile**: Accounts, preferences, timezone, learning style
- **Skills & Lessons**: 6 initial PM/AI skills, skill paths (beginner/intermediate/advanced), daily lessons with quizzes
- **Progress**: User progress per lesson, streak counting, quiz scores
- **Subscriptions**: Free/Starter/Pro/Premium plans, Stripe integration hooks
- **Notifications**: Daily reminders, milestone celebrations, new lessons
- All relationships configured with cascade deletes (except Restrict on subscription plan)

**Setup**:
```bash
# Install dependencies (includes Prisma)
pnpm install

# Set up database (create tables)
pnpm db:push

# Seed with initial data (6 skills, 4 plans)
pnpm db:seed

# Reset database (careful!)
pnpm db:reset
```

**Ports**: PostgreSQL on 5433 (isolated from pocketchange on 5432)

---

### ✅ Chunk 2.1: Authentication & User Management
**Status**: Completed  
**Files Created**:
- **Middleware**:
  - `src/middleware/jwt.ts` (signToken, verifyToken, extractToken)
  - `src/middleware/passwords.ts` (hashPassword, comparePassword with bcrypt)
  - `src/middleware/auth.ts` (authMiddleware, optionalAuthMiddleware for Express)
  - `src/middleware/error-handler.ts` (AppError class, errorHandler, notFoundHandler)
- **Services**:
  - `src/services/AuthService.ts` (register, login, getUserById with full validation)
  - `src/services/UserService.ts` (updateProfile, getUserProgress with stats)
- **Controllers**:
  - `src/controllers/AuthController.ts` (register, login, me endpoints)
  - `src/controllers/UserController.ts` (updateProfile, getProgress endpoints)
- **Routes**:
  - `src/routes/auth.ts` (POST /register, POST /login, GET /me)
  - `src/routes/users.ts` (PATCH /profile, GET /progress)
- **Updated**:
  - `src/index.ts` (integrated auth + user routes, error handling)
  - `package.json` (added jsonwebtoken, bcryptjs dependencies)

**API Endpoints Created**:
- `POST /api/auth/register` - Register new user (creates User + UserProfile)
- `POST /api/auth/login` - Login (returns JWT token)
- `GET /api/auth/me` - Get current user (Protected)
- `PATCH /api/users/profile` - Update profile (goal, timezone, etc.) (Protected)
- `GET /api/users/progress` - Get user progress stats (Protected)

**What It Does**: 
- Users can register with email/password/name
- Password hashed with bcrypt (10 rounds)
- JWT tokens signed and verified
- Protected routes check auth middleware
- All errors return consistent AppError format with error codes

**Test Endpoints**:
```bash
# Register
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123","name":"Test User"}'

# Login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# Get current user (using token from login)
curl -X GET http://localhost:3000/api/auth/me \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

---

### ✅ Chunk 2.2: Lesson Delivery Engine
**Status**: Completed  
**Files Created**:
- `src/services/LessonService.ts`:
  - `getTodayLesson(userId)` - Fetch user's current lesson (first incomplete)
  - `getLessonById(lessonId)` - Get specific lesson with quizzes
  - `completeLessonService(userId, lessonId)` - Mark lesson complete, update streak
  - `submitQuiz(userId, lessonId, answers)` - Score quiz, return feedback
  - `getUpcomingLessons(userId, limit)` - List upcoming lessons
- `src/controllers/LessonController.ts` - All lesson endpoints with validation
- `src/routes/lessons.ts` - Protected lesson routes
- **Updated**: `src/index.ts` - Integrated lesson routes

**API Endpoints Created**:
- `GET /api/lessons/today` - Get today's lesson for user (Protected)
- `GET /api/lessons/:id` - Get specific lesson (Protected)
- `POST /api/lessons/:id/complete` - Mark lesson complete (Protected)
- `POST /api/lessons/:id/quiz` - Submit quiz answers, get score (Protected)
- `GET /api/lessons` - Get upcoming lessons with limit param (Protected)

**What It Does**:
- Users can fetch their daily lesson
- Submit quiz answers and get immediate feedback with scores (0-100)
- Streak tracking on lesson completion
- Quiz includes multiple questions with explanations
- Flexible lesson ordering for MVP

**Test Endpoints**:
```bash
# Get today's lesson
curl -X GET http://localhost:3000/api/lessons/today \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# Get specific lesson
curl -X GET http://localhost:3000/api/lessons/LESSON_ID \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# Complete a lesson
curl -X POST http://localhost:3000/api/lessons/LESSON_ID/complete \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# Submit quiz answers (answers must match quiz IDs)
curl -X POST http://localhost:3000/api/lessons/LESSON_ID/quiz \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{"answers":{"QUIZ_ID_1":"correct_answer_1","QUIZ_ID_2":"correct_answer_2"}}'
```

---

### ✅ Chunk 2.3: Subscriptions & Billing
**Status**: Completed  
**Files Created**:
- `src/services/SubscriptionService.ts`:
  - `getAllPlans()` - List all subscription plans
  - `getUserSubscription(userId)` - Get current subscription (defaults to free)
  - `createCheckoutSession(userId, planId)` - Initiate checkout
  - `upgradeSubscription(userId, planId)` - Upgrade to new plan
  - `cancelSubscription(userId)` - Cancel subscription
  - `getSubscriptionHistory(userId)` - View all subscription events
- `src/controllers/SubscriptionController.ts` - All subscription endpoints
- `src/routes/subscriptions.ts` - Subscription routes (public plans, protected user routes)
- **Updated**: `src/index.ts` - Integrated subscription routes

**API Endpoints Created**:
- `GET /api/subscriptions/plans` - List all plans (Public)
- `GET /api/subscriptions/status` - Current subscription (Protected)
- `POST /api/subscriptions/checkout` - Create checkout session (Protected)
- `POST /api/subscriptions/upgrade` - Upgrade subscription (Protected)
- `POST /api/subscriptions/cancel` - Cancel subscription (Protected)
- `GET /api/subscriptions/history` - Subscription event history (Protected)

**Available Plans** (from seed):
- Free: $0, 1 lesson/day, 1 skill path
- Starter: $19/month, 3 lessons/day, 3 skill paths
- Pro: $49/month, 10 lessons/day, 10 skill paths, AI coaching
- Premium: $99/month, unlimited, AI coaching

**What It Does**:
- Users can view all subscription plans
- Defaults to free plan if no subscription exists
- Upgrade/downgrade between plans
- Track billing history via subscription events
- Mock Stripe integration ready for production

**Test Endpoints**:
```bash
# Get all plans (public)
curl -X GET http://localhost:3000/api/subscriptions/plans

# Get current subscription
curl -X GET http://localhost:3000/api/subscriptions/status \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# Upgrade to Pro plan
curl -X POST http://localhost:3000/api/subscriptions/upgrade \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{"planId":"PLAN_ID"}'

# Cancel subscription
curl -X POST http://localhost:3000/api/subscriptions/cancel \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# View subscription history
curl -X GET http://localhost:3000/api/subscriptions/history \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

---

### ✅ Chunk 3.1: Web Authentication & Onboarding
**Status**: Completed  
**Files Created**:
- `packages/web/lib/api-client.ts` - Typed fetch wrapper with all API endpoints, token management, error handling
- `packages/web/lib/auth-context.tsx` - React Context for auth state (user, loading), useAuth hook, login/register/logout methods, localStorage persistence
- `packages/web/lib/protected-route.tsx` - Route guard component that redirects unauthenticated users to /login
- `packages/web/app/(auth)/login.tsx` - Login form page (email, password inputs, form submission, error display, link to signup)
- `packages/web/app/(auth)/signup.tsx` - Signup form page (email, password, name inputs, password validation, error handling, link to login)
- `packages/web/app/(auth)/onboarding.tsx` - Onboarding form page (skill selection, preferred time selection, timezone dropdown, save to profile, skip button)

**API Client Methods**:
- Authentication: `register()`, `login()`, `getMe()`
- Users: `updateProfile()`, `getUserProgress()`
- Lessons: `getTodayLesson()`, `getLessonById()`, `completeLessonService()`, `submitQuiz()`, `getUpcomingLessons()`
- Subscriptions: `getAllPlans()`, `getUserSubscription()`, `createCheckout()`, `upgradeSubscription()`, `cancelSubscription()`, `getSubscriptionHistory()`

**Auth Features**:
- Users can register with email/password/name
- Password validation (minimum 8 characters)
- Users can login with email/password
- JWT tokens stored in localStorage
- Auto-login on page refresh (if token valid)
- useAuth hook provides user state, loading state, and auth methods to all components
- Protected routes check authentication and redirect to /login if needed

**Onboarding Flow**:
- Users taken to /onboarding after signup
- Select primary skill to learn (from 6 available skills)
- Select preferred learning time (morning/afternoon/evening)
- Select timezone (13 common timezones, auto-detects browser timezone)
- Skip button for optional personalization
- Saves preferences to UserProfile via updateProfile() API call

**What It Does**:
- Complete authentication flow from signup to onboarding
- Type-safe API client for all backend endpoints
- Auth state management using React Context
- Protected routes with automatic redirects
- Tailwind-styled pages consistent with design system

**Setup**:
```bash
# Ensure backend is running (API on port 3000)
cd packages/api
pnpm dev

# In another terminal, start web frontend
cd packages/web
pnpm dev
# Web frontend runs on http://localhost:3001
```

**Testing User Flow**:
```bash
# 1. Sign up at http://localhost:3001/signup
#    Email: testuser@example.com
#    Password: password123 (min 8 chars)
#    Name: Test User

# 2. Redirected to http://localhost:3001/onboarding
#    Select skill, time, timezone
#    or click "Skip for now"

# 3. Should proceed to /dashboard (next chunk to implement)

# 4. Login at http://localhost:3001/login
#    Can also test logout functionality
```

**Structure**:
```
packages/web/
├── lib/
│   ├── api-client.ts (typed fetch wrapper)
│   ├── auth-context.tsx (React Context)
│   └── protected-route.tsx (route guard)
├── app/
│   ├── (auth)/ (route group for auth pages)
│   │   ├── login.tsx
│   │   ├── signup.tsx
│   │   └── onboarding.tsx
│   ├── layout.tsx (root layout)
│   ├── page.tsx (home page)
│   └── globals.css
```

---

### ✅ Chunk 3.2: Dashboard & Lesson Screen
**Status**: Completed  
**Files Created**:
- `packages/web/app/(dashboard)/layout.tsx` - Protected dashboard layout with header navigation
- `packages/web/app/(dashboard)/page.tsx` - Dashboard home page displaying user stats and today's lesson
- `packages/web/app/(dashboard)/lessons/[id]/page.tsx` - Individual lesson page with content and completion button
- `packages/web/app/(dashboard)/lessons/[id]/quiz.tsx` - Interactive quiz screen with answer submission

**Dashboard Features**:
- Displays user's current streak (days in a row)
- Shows lessons completed count
- Shows average quiz score
- Displays today's lesson card with title and duration
- Lesson image/media preview
- "Start Lesson & Quiz" call-to-action button
- Next steps section with guidance

**Lesson Screen Features**:
- Full lesson content display (title, description, learning points)
- Lesson duration and difficulty badge
- Featured image/media
- Learning objectives section
- "Complete & Take Quiz" button to mark lesson complete and proceed to quiz
- Skip button to go back
- Info tip about quiz benefits

**Quiz Screen Features**:
- Displays quiz questions from lesson data
- Supports MCQ (multiple choice) and text answer formats
- Radio buttons for MCQ options, text input for open-ended questions
- Submit button to post answers to backend
- Real-time answer tracking
- Result page with:
  - Score display (0-100)
  - Pass/fail indicator (70% = passing)
  - Feedback message based on performance
  - "Continue to Next Lesson" (if passed) or "Try Again" button (if failed)
  - Link to progress page

**Styling & UX**:
- Consistent Tailwind CSS theming with blue primary color
- Gradient hero sections for visual interest
- Card-based layout for lesson information
- Progress indicators and visual hierarchy
- Responsive design (mobile-first)
- Loading states and error handling throughout
- Protected routes with auth check

**API Integration**:
- Calls `GET /api/lessons/today` to fetch today's lesson from dashboard
- Calls `GET /api/users/progress` to fetch user stats (streak, completed count, average score)
- Calls `GET /api/lessons/:id` to fetch full lesson content
- Calls `POST /api/lessons/:id/complete` when user completes lesson
- Calls `POST /api/lessons/:id/quiz` to submit quiz answers
- All calls include JWT auth token

**Test Flow**:
```bash
# 1. Login to dashboard at http://localhost:3001/dashboard
# 2. See today's lesson, user stats (streak, completed, score)
# 3. Click "Start Lesson & Quiz" to view lesson content
# 4. Click "Complete & Take Quiz" to proceed to quiz
# 5. Answer quiz questions and click "Submit Quiz"
# 6. See results with score and feedback
# 7. Click "Continue to Next Lesson" or "Try Again"
```

**Structure**:
```
packages/web/
├── app/
│   ├── (dashboard)/ (protected route group)
│   │   ├── layout.tsx (header + navigation)
│   │   ├── page.tsx (home/dashboard)
│   │   ├── progress.tsx (progress page)
│   │   ├── settings.tsx (settings page)
│   │   └── lessons/
│   │       └── [id]/
│   │           ├── page.tsx (lesson display)
│   │           └── quiz.tsx (quiz form & results)
```

---

### ✅ Chunk 3.3: Progress & Settings Pages
**Status**: Completed  
**Files Created**:
- `packages/web/app/(dashboard)/progress.tsx` - Comprehensive progress tracking page
- `packages/web/app/(dashboard)/settings.tsx` - User settings and preferences page

**Progress Page Features**:
- Current streak display (days in a row)
- Total lessons completed counter
- Average quiz score
- Learning level calculation (based on lessons completed)
- 30-day activity calendar (contribution view)
- Time investment tracking (minutes vs goal)
- Quiz performance chart (7-week history with visual bars)
- Achievements/badges section (Hot Streak, Bookworm, Perfect Score, Speed Learner)
- Call-to-action section based on user status
- Responsive grid layout for all stats

**Settings Page Features**:
- Profile section (email and name - read-only, with support contact info)
- Learning preferences:
  - Primary skill selection (6 skills in card layout)
  - Preferred learning time (morning/afternoon/evening)
  - Timezone selection (13 timezones + auto-detect)
  - Save preferences button (calls `PATCH /api/users/profile`)
- Notification preferences:
  - Email notifications toggle (ON/OFF)
  - Frequency selection (daily/weekly/never)
  - Visual indicators for notification status
- Subscription section:
  - Current plan display (name and price)
  - Billing cycle info
  - Upgrade plan button
  - Cancel subscription button (for paid plans)
- Danger zone:
  - Logout button
  - Delete account button (placeholder for future)
- Support section:
  - Contact support email link

**API Integration**:
- Progress page calls `GET /api/users/progress` to fetch user stats
- Settings page calls `GET /api/subscriptions/status` to fetch subscription
- Settings page calls `PATCH /api/users/profile` to save learning preferences
- All calls include JWT auth token

**Styling & UX**:
- Consistent Tailwind CSS design system
- Gradient backgrounds for visual interest
- Cards with colored left borders (orange, blue, green, purple)
- Responsive grid layouts (1 col mobile, 2-4 cols desktop)
- Toggle buttons with color states
- Color-coded sections (green for achievements, red for danger zone)
- Loading states and error handling
- Success messages on save
- Smooth transitions and hover states

**Responsive Design**:
- Mobile (375px): Single column layout, stacked cards
- Tablet (768px): 2-column grids where appropriate
- Desktop (1440px): Full 4-column grid for stats, 2-column for other sections
- Touch-friendly buttons and inputs
- Readable text sizes across all devices

**Test Flow**:
```bash
# 1. Navigate to http://localhost:3001/progress
# 2. See all user stats (streak, lessons, score, level)
# 3. View 30-day activity calendar with completed dates
# 4. Check time investment progress bar
# 5. See quiz performance chart over 7 weeks
# 6. View achievements section

# 7. Navigate to http://localhost:3001/settings
# 8. View profile information (email, name - read-only)
# 9. Update learning preferences (skill, time, timezone)
# 10. Click "Save Preferences" (shows success message)
# 11. Toggle email notifications on/off
# 12. Select notification frequency
# 13. View current subscription plan
# 14. Click "Logout" to end session
```

**Complete Phase 3 Summary**:
✅ **Chunk 3.1**: Auth pages (login, signup, onboarding)
✅ **Chunk 3.2**: Dashboard and lesson/quiz screens
✅ **Chunk 3.3**: Progress tracking and settings management

**Web Frontend Complete** — All authenticated user pages finished with full functionality.

---
