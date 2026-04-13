# Phase 11: Security, Analytics & Observability

**Status**: ⚪ NOT STARTED

**Goal**: Harden the platform for production — close known security gaps, add analytics to track user behaviour and business metrics, and set up error tracking and monitoring.

---

## Chunk 11.1: Security Hardening

**Deliverable**: All known security gaps from the architectural review are closed

### 11.1.1 — JWT Token Expiry & Refresh

**Current gap**: JWT tokens issued with no expiry — a leaked token is valid forever.

**Reference implementation**: `pocketchange` backend has this production-ready. Read these files before implementing:
- `/Users/guypowell/documents/projects/pocketchange/backend/src/config/jwt.ts` — sign/verify helpers
- `/Users/guypowell/documents/projects/pocketchange/backend/src/modules/auth/auth.service.ts` — token issuance, refresh, revocation
- `/Users/guypowell/documents/projects/pocketchange/backend/src/middleware/authenticate.ts` — Bearer token extraction

**Key pattern to adopt**: pocketchange stores refresh tokens in **Redis** (not a DB table) — this is simpler and faster. The token hash is stored as a Redis key with TTL equal to the refresh token expiry.

**Implementation**:

**File**: `packages/api/src/services/AuthService.ts`

**Changes**:
- Add `expiresIn: '15m'` to access token
- Issue a refresh token (`expiresIn: '30d'`), store hash in Redis (key: `refresh:{userId}:{tokenHash}`, TTL: 30d)
- On refresh: validate token signature + Redis key existence; issue new access token; rotate refresh token
- On logout: delete Redis key

**Routes**:
- `POST /api/auth/refresh` — Validate refresh token, issue new access + refresh tokens
- `POST /api/auth/logout` — Revoke refresh token (delete Redis key)

**Frontend**: Intercept 401 responses in `api-client.ts`, auto-call `/refresh`, retry original request

**Mobile**: pocketchange-app's `src/lib/api.ts` already implements this pattern with mutex deduplication — copy it directly

### 11.1.2 — Rate Limiting

**Package**: `express-rate-limit`

**Apply**:
- Auth endpoints (`/api/auth/register`, `/api/auth/login`): 5 requests/15 min per IP
- AI generation endpoints: 10 requests/min per user (prevent abuse)
- General API: 100 requests/min per user

**File**: `packages/api/src/middleware/rateLimiter.ts`

### 11.1.3 — CORS Hardening

**Current gap**: `Access-Control-Allow-Origin: *`

**Fix**: Whitelist known origins
```typescript
const allowedOrigins = [
  'https://yourdomain.com',
  'http://localhost:3001'  // dev only
]
```

### 11.1.4 — Input Sanitisation

**Current gap**: Lesson content is stored and served without sanitisation — XSS risk if content becomes user-generated.

**Package**: `dompurify` (server-side via `isomorphic-dompurify`) or `sanitize-html`

**Apply**: Sanitise lesson content before storage (in `LessonGenerationService` and admin lesson save)

**Tests**: Unit test rate limiter middleware; test refresh token rotation; test CORS rejection on unknown origin

---

## Chunk 11.2: Analytics Integration

**Deliverable**: Key user activation, retention, and conversion events are tracked

### 11.2.1 — Analytics Provider

**Choose one**: Mixpanel (B2C focused, funnel analysis) or Plausible (privacy-first, simpler)

**Config**: Add API key to `.env`, initialise in `packages/api/src/lib/analytics.ts`

### 11.2.2 — Server-Side Event Tracking

**File**: `packages/api/src/lib/analytics.ts`

**Events to track** (server-side, in service layer):
- `user_registered` — `{ userId, plan: 'free' }`
- `onboarding_completed` — `{ userId, selectedSkills, timezone }`
- `lesson_completed` — `{ userId, lessonId, skillId, streakCount }`
- `quiz_submitted` — `{ userId, lessonId, score, correct }`
- `subscription_created` — `{ userId, plan, mrr }`
- `subscription_cancelled` — `{ userId, plan, tenure_days }`
- `team_created` — `{ teamId, ownerId, seatLimit }`

### 11.2.3 — Frontend Event Tracking

**File**: `packages/web/lib/analytics.ts`

**Events**:
- `page_view` — Each route change
- `cta_clicked` — Upgrade prompts, lesson start buttons
- `quiz_started`, `quiz_abandoned` — Funnel drop-off
- `onboarding_step_completed` — Per step drop-off

---

## Chunk 11.3: Error Tracking (Sentry)

**Deliverable**: All unhandled errors in API, web, and mobile are captured with full context

### 11.3.1 — API Sentry

**Package**: `@sentry/node`

**Init**: In `packages/api/src/index.ts` before route registration

**Capture**:
- All unhandled errors via `Sentry.setupExpressErrorHandler(app)`
- Manual capture in catch blocks for AI generation failures
- User context on authenticated requests (`Sentry.setUser({ id: req.user.userId })`)

### 11.3.2 — Web Sentry

**Package**: `@sentry/nextjs`

**Init**: `sentry.client.config.ts` and `sentry.server.config.ts`

**Capture**: Unhandled React errors, API client failures, page-level errors

### 11.3.3 — Mobile Sentry

**Package**: `@sentry/react-native`

**Init**: In `learning-app/app/_layout.tsx`

**Capture**: Native crashes, JS errors, Expo-specific errors

---

## Chunk 11.4: Performance Monitoring

**Deliverable**: Slow queries and API bottlenecks are visible before they become user problems

### 11.4.1 — API Response Time Logging

**Middleware**: `packages/api/src/middleware/requestLogger.ts`

Log: method, path, status code, response time (ms) — use `morgan` or custom middleware

**Alert threshold**: Log `WARN` for responses > 500ms, `ERROR` for > 2000ms

### 11.4.2 — Database Query Monitoring

**Prisma**: Enable query event logging in dev/staging

```typescript
prisma.$on('query', (e) => {
  if (e.duration > 200) logger.warn(`Slow query: ${e.duration}ms — ${e.query}`)
})
```

### 11.4.3 — AI Generation Latency Tracking

**Track**: Time from generation request to response in `LessonGenerationService`

**Log**: Model used, prompt tokens, generation time — use Sentry performance transactions

---

## Next Phase

[Phase 12: Testing & Launch](./phase-12-testing-launch.md)
