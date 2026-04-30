# Build Plan — Phases

This directory contains the detailed implementation plan for each phase of the learning app.

## Structure

Each phase file contains:
- **Goal**: What the phase accomplishes
- **Status**: ✅ Complete / ⚪ Not started
- **Chunks**: Logical units of work (e.g., 3.1, 3.2, 3.3)
- **Deliverables**: What gets built
- **Files**: Specific files to create/modify
- **Testing**: How to verify completeness

## Files

| Phase | File | Status | Scope |
|-------|------|--------|-------|
| 1 | [phase-1-foundation.md](./phase-1-foundation.md) | ✅ Complete | Monorepo, Docker, DB schema |
| 2 | [phase-2-backend.md](./phase-2-backend.md) | ✅ Complete | Auth, lessons, subscriptions API |
| 3 | [phase-3-web-frontend.md](./phase-3-web-frontend.md) | ✅ Complete | Dashboard, lessons, auth UI |
| 4 | [phase-4-mobile.md](./phase-4-mobile.md) | ✅ Complete | React Native/Expo app |
| 5 | [phase-5-stripe-billing.md](./phase-5-stripe-billing.md) | ⚪ Not started | Full Stripe checkout, webhooks, tier enforcement |
| 6 | [phase-6-ai-lesson-generation.md](./phase-6-ai-lesson-generation.md) | ✅ Complete | Ollama/Mistral lesson generation, template engine |
| 7 | [phase-7-ai-personalization.md](./phase-7-ai-personalization.md) | ✅ Complete | Recommendation engine, adaptive difficulty, AI coaching |
| 8 | [phase-8-notifications-habit.md](./phase-8-notifications-habit.md) | ✅ Complete | Push notifications, streak engine, UX polish |
| 9 | [phase-9-admin-cms.md](./phase-9-admin-cms.md) | ✅ Complete | Admin auth, lesson authoring, skill path management |
| 10 | [phase-10-enterprise-team.md](./phase-10-enterprise-team.md) | 🔄 In progress | Team accounts, team analytics, enterprise custom paths (10.2 deferred) |
| 11 | [phase-11-security-analytics.md](./phase-11-security-analytics.md) | ⚪ Not started | Security hardening, Mixpanel, Sentry, performance |
| 12 | [phase-12-testing-launch.md](./phase-12-testing-launch.md) | ⚪ Not started | E2E testing, staging, production, app stores, launch |

## How to Use

1. **On session startup**: Read [../PROGRESS.md](../PROGRESS.md) to see current phase
2. **For active phase**: Read the relevant phase file only
3. **Skip completed phases** unless you need context

## Definition of Done

1. All new code and all code the new code interacts with has full unit test coverage
2. All new code and all code the new code interacts with works with no lint or jest errors
3. The code works and provides the intended customer behaviour

## Next Steps

→ Read [../PROGRESS.md](../PROGRESS.md) for current status
