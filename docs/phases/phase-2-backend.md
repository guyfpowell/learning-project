# Phase 2: Core Backend API

**Status**: ✅ COMPLETE

**Goal**: Build essential API endpoints with authentication, lesson delivery, and subscription management.

---

## Chunk 2.1: Authentication & User Management ✅

**Status**: Completed  
**Deliverable**: Users can register/login; JWT tokens in responses

**API Endpoints**:
- `POST /api/auth/register` — Create user with email/password/name
- `POST /api/auth/login` — Login, return JWT token
- `GET /api/auth/me` — Get current user (Protected)
- `PATCH /api/users/profile` — Update profile (goal, timezone) (Protected)
- `GET /api/users/progress` — Get user progress stats (Protected)

**Files**:
- `src/middleware/jwt.ts` (signToken, verifyToken, extractToken)
- `src/middleware/passwords.ts` (hashPassword, comparePassword)
- `src/middleware/auth.ts` (authMiddleware, optionalAuthMiddleware)
- `src/middleware/error-handler.ts` (AppError class, error handling)
- `src/services/AuthService.ts` (register, login, getUserById)
- `src/services/UserService.ts` (updateProfile, getUserProgress)
- `src/controllers/AuthController.ts` (request handlers)
- `src/controllers/UserController.ts` (request handlers)
- `src/routes/auth.ts` (route definitions)
- `src/routes/users.ts` (route definitions)

**Features**:
- Passwords hashed with bcrypt (10 rounds)
- JWT tokens signed and verified
- Protected routes check auth middleware
- Consistent error responses with error codes

---

## Chunk 2.2: Lesson Delivery Engine ✅

**Status**: Completed  
**Deliverable**: API can serve daily lessons and accept quiz submissions

**API Endpoints**:
- `GET /api/lessons/today` — Fetch today's lesson for user (Protected)
- `GET /api/lessons/:id` — Get specific lesson with quizzes (Protected)
- `POST /api/lessons/:id/complete` — Mark lesson complete, update streak (Protected)
- `POST /api/lessons/:id/quiz` — Submit quiz answers, get score + feedback (Protected)
- `GET /api/lessons` — Get upcoming lessons (Protected)

**Files**:
- `src/services/LessonService.ts`:
  - `getTodayLesson(userId)` — Fetch user's current lesson
  - `getLessonById(lessonId)` — Get specific lesson
  - `completeLessonService(userId, lessonId)` — Mark complete, update streak
  - `submitQuiz(userId, lessonId, answers)` — Score quiz, return feedback
  - `getUpcomingLessons(userId, limit)` — List upcoming lessons
- `src/controllers/LessonController.ts` (request handlers)
- `src/routes/lessons.ts` (route definitions)

**Features**:
- Flexible lesson ordering
- Quiz scoring (0-100)
- Streak tracking on completion
- Multiple quiz questions with explanations

---

## Chunk 2.3: Subscriptions & Billing ✅

**Status**: Completed  
**Deliverable**: API can manage subscriptions (Stripe integration ready)

**API Endpoints**:
- `GET /api/subscriptions/plans` — List all plans (Public)
- `GET /api/subscriptions/status` — Get current subscription (Protected)
- `POST /api/subscriptions/checkout` — Create checkout session (Protected)
- `POST /api/subscriptions/upgrade` — Upgrade subscription (Protected)
- `POST /api/subscriptions/cancel` — Cancel subscription (Protected)
- `GET /api/subscriptions/history` — Subscription event history (Protected)

**Files**:
- `src/services/SubscriptionService.ts`:
  - `getAllPlans()` — List all subscription plans
  - `getUserSubscription(userId)` — Get current (defaults to free)
  - `createCheckoutSession(userId, planId)` — Initiate checkout
  - `upgradeSubscription(userId, planId)` — Upgrade plan
  - `cancelSubscription(userId)` — Cancel subscription
  - `getSubscriptionHistory(userId)` — View events
- `src/controllers/SubscriptionController.ts` (request handlers)
- `src/routes/subscriptions.ts` (route definitions)

**Plans** (from seed):
- Free: $0, 1 lesson/day
- Starter: $19/month, 3 lessons/day
- Pro: $49/month, 10 lessons/day, AI coaching
- Premium: $99/month, unlimited, AI coaching

**Features**:
- Users default to free plan
- Upgrade/downgrade between plans
- Track billing history via events
- Mock Stripe integration ready for production

---

## Testing the Backend

**Start server**:
```bash
cd packages/api
pnpm dev
# API running on http://localhost:3000
```

**Test auth**:
```bash
# Register
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123","name":"Test User"}'

# Login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# Get current user (replace TOKEN)
curl -X GET http://localhost:3000/api/auth/me \
  -H "Authorization: Bearer TOKEN"
```

**Test lessons**:
```bash
# Get today's lesson
curl -X GET http://localhost:3000/api/lessons/today \
  -H "Authorization: Bearer TOKEN"

# Submit quiz
curl -X POST http://localhost:3000/api/lessons/LESSON_ID/quiz \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"answers":{"QUIZ_ID":"answer"}}'
```

---

## Reference

- See [PROGRESS.md](../PROGRESS.md) for overview
- See [Phase 3](./phase-3-web-frontend.md) for next phase (Web Frontend)

