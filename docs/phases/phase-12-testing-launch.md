# Phase 12: Testing & Launch

**Status**: ⚪ NOT STARTED

**Goal**: Comprehensive testing of all flows across all layers, staging and production deployment, mobile app store submission, and launch execution.

---

## Chunk 12.1: Integration & E2E Testing

**Deliverable**: All critical user flows are covered by automated tests

### 12.1.1 — Backend Integration Tests

**Flows to test** (against real test database):
- Auth flow: register → login → refresh → logout → refresh token invalidated
- Lesson flow: getTodayLesson → completeLessonService → submitQuiz → streak incremented
- AI generation: generateLesson → lesson stored → served on next request
- Subscription flow: create checkout → webhook → tier upgraded → lesson limits enforced
- Team flow: create team → invite → accept → member sees team lesson paths
- Admin flow: create skill → create path → create lesson → lesson served to user

**Framework**: Jest + Supertest (existing setup)

**Database**: Separate test DB; reset between test runs with `prisma migrate reset`

### 12.1.2 — End-to-End Tests (Web)

**Framework**: Playwright

**Flows**:
- Sign up → onboarding → dashboard → complete lesson → quiz → see streak
- Login → view progress → settings → update notification preferences → logout
- Subscribe to Starter → lesson limits removed → upgrade prompt gone
- Admin login → create lesson → publish → visible to users

**Setup**: `packages/web/e2e/` directory; run against staging environment

### 12.1.3 — Mobile E2E Tests

**Framework**: Detox (for React Native)

**Flows**:
- Sign up → onboarding → lesson tab → complete lesson → quiz
- Login → progress tab → streak visible
- Notification permission request on launch

### 12.1.4 — AI Generation Tests

**Test**: LessonGenerationService with real Ollama (local) in CI

- Generate lesson for each skill × level combination
- Validate output parses to expected schema
- Test fallback when Ollama unavailable

---

## Chunk 12.2: Staging Deployment

**Deliverable**: Staging environment is a full production mirror

### 12.2.1 — Infrastructure

**API** (Render staging):
- Deploy `main` branch → staging service
- Env vars: `DATABASE_URL` (Supabase staging), `JWT_SECRET`, `STRIPE_SECRET_KEY` (Stripe test mode), `OLLAMA_BASE_URL` (staging GPU instance or CPU fallback)

**Web** (Vercel staging):
- Deploy `main` → staging preview URL
- `NEXT_PUBLIC_API_URL` → staging API

**Ollama** (staging):
- Option A: Render GPU instance with Ollama + Mistral
- Option B: CPU instance (slower but cheaper for staging validation)

**Database** (Supabase staging):
- Separate project from production
- Run `prisma migrate deploy` against staging DB
- Run `pnpm db:seed` with realistic test data

### 12.2.2 — Staging Smoke Tests

Run full E2E test suite against staging before every production deploy:
- Auth works
- Lesson generation works (Ollama responding)
- Stripe webhooks work (Stripe test mode)
- Push notifications work (FCM test)
- Team invite flow works

---

## Chunk 12.3: Production Deployment

**Deliverable**: App is live, stable, and monitored

### 12.3.1 — API (Render)

- Deploy from `main` branch
- Configure production Supabase connection
- Set all env vars (JWT_SECRET, STRIPE_SECRET_KEY, OPENAI_API_KEY, SENTRY_DSN, etc.)
- Enable Render health checks: `GET /health`
- Set up Render auto-deploy on push to `main`

### 12.3.2 — Web (Vercel)

- Deploy from `main` branch
- Configure `NEXT_PUBLIC_API_URL` → production API
- Set up custom domain + SSL
- Enable Vercel preview deploys for PRs

### 12.3.3 — Ollama (Production)

- Dedicated GPU instance (DigitalOcean, Render, or RunPod)
- Mistral 7B pre-loaded and health-checked
- API accessible only from production API server (private network or IP whitelist)

### 12.3.4 — CI/CD Pipeline

**File**: `.github/workflows/deploy.yml`

```yaml
on:
  push:
    branches: [main]
jobs:
  test:
    - Run unit tests (jest)
    - Run integration tests
  deploy:
    needs: test
    - Deploy API to Render
    - Deploy web to Vercel
```

---

## Chunk 12.4: Mobile App Store

**Deliverable**: iOS and Android apps available on stores

### 12.4.1 — Build Configuration

**File**: `learning-app/eas.json`

```json
{
  "build": {
    "production": {
      "android": { "buildType": "apk" },
      "ios": { "simulator": false }
    }
  }
}
```

### 12.4.2 — Android (Google Play)

```bash
eas build --platform android --profile production
```

- Create Play Store listing (title, description, screenshots, category)
- Upload APK/AAB to internal testing → closed testing → production
- Set up app signing in EAS

### 12.4.3 — iOS (App Store)

```bash
eas build --platform ios --profile production
```

- Create App Store Connect listing
- Upload via TestFlight → App Store review
- Required: privacy policy URL, support URL
- Review time: 24–72 hours

---

## Chunk 12.5: Launch Activities

**Deliverable**: First users acquired and onboarded

### 12.5.1 — Pre-Launch

- Email waitlist: "You're in — try it free today"
- Set up Mixpanel/Plausible dashboards for launch monitoring
- Verify Sentry is capturing errors from production
- Test Stripe in live mode (real card payment end-to-end)

### 12.5.2 — Launch Day

- Post on ProductHunt
- Post on LinkedIn + Twitter with PM/AI practitioner framing
- Direct outreach to 20–30 PM/AI network contacts
- Monitor: DAU, lesson completions, sign-ups, errors (Sentry)

### 12.5.3 — Success Metrics

- Day 1 retention: >70%
- Day 7 retention: >40%
- Onboarding completion: >80%
- Lesson completion rate: >85%
- Free → paid conversion: >10% within 30 days
- App store rating: >4.5/5

---

## Reference

- See [PROGRESS.md](../PROGRESS.md) for overview
