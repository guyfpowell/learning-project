# Phase 5: Stripe Billing

**Status**: ⚪ NOT STARTED

**Goal**: Implement full Stripe subscription billing — replace the stubbed implementation with real checkout, webhook handling, tier enforcement, and plan management.

---

## Chunk 5.1: Stripe Checkout

**Deliverable**: Users can subscribe to a paid plan via Stripe

### 5.1.1 — Stripe Setup

**Config**:
- Add `stripe` package to `packages/api`
- Add `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` to `.env`
- Add `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` to web `.env`
- Create Stripe products/prices in dashboard: Free, Starter ($19), Pro ($49), Premium ($99)
- Seed `SubscriptionPlan` table with Stripe price IDs

### 5.1.2 — Checkout Session API

**File**: `packages/api/src/services/SubscriptionService.ts`

**Methods**:
- `createCheckoutSession(userId, planId)` — Create Stripe checkout session, return URL
- `createCustomerPortalSession(userId)` — Return Stripe portal URL for plan management

**Routes**:
- `POST /api/subscriptions/checkout` — Create checkout session
- `POST /api/subscriptions/portal` — Create customer portal session
- `GET /api/subscriptions/plans` — List all plans (with Stripe price IDs)
- `GET /api/subscriptions/current` — Get user's current subscription

**Tests**: Unit test SubscriptionService with Stripe mocked

---

## Chunk 5.2: Webhook Handling

**Deliverable**: Subscription status stays in sync with Stripe events

**Reference implementation**: `pocketchange` backend at `/Users/guypowell/documents/projects/pocketchange/backend/src/modules/webhooks/stripe.webhook.ts`. Port the webhook signature verification and `StripeEventLog` idempotency pattern. **Do not port the ledger system** — pocketchange's DonorLedger/RecipientLedger/VendorLedger were for multi-party wallet balances; the learning app has no wallet, Stripe is the source of truth.

### 5.2.1 — Webhook Endpoint

**File**: `packages/api/src/controllers/SubscriptionController.ts`

**Route**: `POST /api/subscriptions/webhook` (no auth middleware — Stripe calls this)

**Idempotency** (port from pocketchange):
Add `StripeEventLog` table to Prisma schema:
```prisma
model StripeEventLog {
  id          String   @id  // Stripe event ID (e.g. evt_xxx)
  eventType   String
  processedAt DateTime @default(now())
}
```
At the top of the webhook handler: check if `id` already exists in `StripeEventLog`; if so, return 200 immediately. Insert on first processing. Prevents double-activation when Stripe retries.

**Events to handle**:
- `checkout.session.completed` — Activate subscription, store `stripeCustomerId` + `stripeSubscriptionId`
- `customer.subscription.updated` — Update plan tier and status
- `customer.subscription.deleted` — Downgrade to free tier
- `invoice.payment_failed` — Mark subscription as past_due
- `invoice.payment_succeeded` — Confirm active status

**Validation**: Verify webhook signature with `STRIPE_WEBHOOK_SECRET` using `express.raw()` middleware on this route (before `express.json()`)

**Tests**: Unit test each webhook event handler with fixture payloads; test idempotency (duplicate event returns 200, no side effects)

---

## Chunk 5.3: Tier Enforcement

**Deliverable**: Feature access is gated by subscription tier

### 5.3.1 — Subscription Middleware

**File**: `packages/api/src/middleware/subscription.ts`

**Logic**:
- Fetch user's active subscription on each protected request (cache in Redis, 5min TTL)
- Attach `req.subscription` with plan tier and limits
- Export `requirePlan(minTier)` middleware factory

**Tier Limits**:
- Free: 1 lesson/day, Mistral-generated lessons, basic quiz
- Starter ($19): Unlimited lessons, Mistral-generated lessons
- Pro ($49): Unlimited + AI coaching via OpenAI, advanced quiz explanations
- Premium ($99): Unlimited + OpenAI, priority generation, team features

### 5.3.2 — Lesson Limit Enforcement

**File**: `packages/api/src/services/LessonService.ts`

**Logic**:
- `getTodayLesson()` checks daily lesson count vs. plan limit
- Return `429` with upgrade prompt if limit exceeded
- Track daily lesson count in Redis (key: `lesson_count:{userId}:{date}`, TTL: 24h)

### 5.3.3 — Frontend Upgrade Prompts

**Files**: `packages/web/app/(dashboard)/` pages

**Features**:
- Show upgrade CTA when lesson limit hit
- Show plan tier badge in settings
- Subscription management link → Stripe customer portal
- Display billing page at `/settings/billing`

**Tests**: Unit tests for limit logic; frontend tests for upgrade prompt display

---

## Chunk 5.4: Trial & Promo Management

**Deliverable**: Free trials and promo codes work correctly

### 5.4.1 — Free Trial

**Stripe config**: Set 7-day trial on Starter plan
**Backend**: Handle `trialing` subscription status — grant Starter-tier access during trial
**Frontend**: Show trial expiry countdown in settings

### 5.4.2 — Promo Codes

**Stripe config**: Create promotion codes in dashboard
**Frontend**: Promo code field on checkout page (pass to Stripe checkout session)

---

## Next Phase

[Phase 6: AI Lesson Generation](./phase-6-ai-lesson-generation.md)
