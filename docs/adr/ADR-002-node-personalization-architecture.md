# ADR-002 — Node-Side Personalization Architecture

**Ticket Reference:** Phase 7 (AI Personalization & Adaptive Learning)
**Status:** accepted
**Date:** 2026-04-17

---

## Context

Phase 7 introduces four personalization systems in the Node backend: a UCB1 recommendation engine, an Elo-based skill tracker, AI coaching feedback wired into the quiz flow, and a learning style classifier. These systems introduce new DB models, algorithm-bearing services, changes to existing API response shapes, and new integration points with the Python AI service.

ADR-001 already governs the Python AI service side (prompt selection, model routing). This ADR governs the Node implementation constraints only.

---

## Decision

Phase 7 personalization features are implemented in Node as pure service-layer additions that slot into the existing Route → Controller → Service → Prisma pattern. No new patterns are introduced. Algorithms (UCB1, Elo) live in a dedicated `packages/api/src/ai/` directory, distinct from `services/`, to signal they are computation units rather than data-access services.

---

## Architectural Constraints for Dev Agent

### 1. Schema — UserProgress engagement fields
`quizScore` **already exists** on `UserProgress` (`Int?`). Do **not** add it again. Only add the three new fields: `completionTime` (`Int?`, seconds), `revisitCount` (`Int @default(0)`), `rating` (`Int?`, -1/0/1 thumbs). Migration must not touch the existing `quizScore` column.

### 2. Schema — UserSkillRating
The `UserSkillRating` model as spec'd in 7.2.1 is correct. Two additions required:
- `onDelete: Cascade` on the `user` relation — already in the spec, must not be omitted.
- The `Skill` model must gain a `userSkillRatings UserSkillRating[]` relation field for Prisma to generate correctly.

### 3. Algorithm services location
UCB1 (`RecommendationEngine.ts`) and Elo (`SkillTracker.ts`) and `LearningStyleClassifier.ts` belong in `packages/api/src/ai/`, not `services/`. These are computation units, not data-access services. They may call Prisma directly for reads; for writes they must go through the appropriate service (e.g. `UserService.updateUserProfile()` for learning style updates — do not write to `UserProfile` directly from `LearningStyleClassifier`).

### 4. Coaching tier enforcement — NOT on the quiz route
The quiz route (`POST /api/lessons/:id/quiz`) serves all tiers. Do **not** add `requirePlan('pro')` middleware to the quiz route. Instead, `CoachingService.generateFeedback()` receives `tier` as a parameter and returns `null` immediately if tier is `free` or `starter`. The coaching field is always present in the quiz response body — `null` for non-pro, string for pro/premium.

### 5. Coaching cache
`CoachingService` must cache coaching responses in the `GeneratedLesson` table (being added in Phase 6 Chunk 6.6) — not in Redis. Redis is reserved for notification/job queues. The unique constraint `(userId, lessonId)` on `GeneratedLesson` is the deduplication key. If a coaching response is already cached, return it without calling the Python AI service.

**Open question**: `GeneratedLesson` currently has no `coachingMessage` field. The dev agent must add a nullable `coachingMessage String?` field to `GeneratedLesson` in the 6.6 migration (or as part of Phase 7 migration if 6.6 is already merged).

### 6. Fallback on coaching — non-blocking
`CoachingService.generateFeedback()` must never throw. Wrap the AI call in try/catch; on any error, log and return `null`. Quiz result delivery must not wait on coaching.

### 7. LessonService.getTodayLesson() delegation
`RecommendationEngine.getNextLesson()` replaces the existing day-order selection logic inside `LessonService`. The response shape of `GET /api/lessons/today` must not change — the recommendation engine is a selection mechanism only. Cold-start (first 3 lessons) must preserve the original day-order behaviour.

### 8. LearningStyleClassifier — profile update via UserService
`LearningStyleClassifier` must not write to `UserProfile` directly. It returns a style string; the caller (quiz submission flow in `LessonService` or `EngagementService`) must call `userService.updateUserProfile(userId, { learningStyle })`. Service layer boundary must not be crossed.

### 9. Phase 6 Chunk 6.6 prerequisite
Phase 7 implementation must not begin until Phase 6 Chunk 6.6 (`GeneratedLesson` model + migration) is complete and merged. `LessonGenerationService._getCachedLesson()` is a stub that returns `null` until 6.6 lands — Phase 7 coaching cache depends on this model existing.

### 10. Shared types
Any new request/response types crossing the Node↔Python boundary (skill_level, learning_style in lesson generation payload) must be added to `packages/shared/src/types/ai.ts` and exported from `@learning/shared`. Do not define them inline in service files.

---

## Consequences

**Benefits**:
- Algorithm services in `ai/` directory are clearly separated from data-access services, making the codebase intent legible.
- Coaching as a non-blocking null-fallback ensures quiz result delivery is never degraded by AI availability.
- Consistent tier enforcement in service layer (not route middleware) keeps routing thin.

**Trade-offs**:
- `GeneratedLesson` carries coaching data as a nullable column — slightly overloaded, but avoids a separate table for MVP.
- `LearningStyleClassifier` writes are indirect (via UserService) — minor overhead, justified by separation of concerns.

---

## Alternatives Considered

**`requirePlan('pro')` middleware on quiz route.** Rejected: breaks quiz access for free/starter users.

**Redis for coaching cache.** Rejected: Redis is for ephemeral job queue/notification data. Coaching responses are durable user data — DB is the right store.

**Separate `CoachingCache` table.** Deferred: `GeneratedLesson` with a nullable `coachingMessage` column is sufficient for MVP. Can extract to a dedicated table in Phase 9+ if needed.

**`LearningStyleClassifier` writing to `UserProfile` directly.** Rejected: violates the service layer boundary; all user profile mutations must go through `UserService`.
