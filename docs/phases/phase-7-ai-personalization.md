# Phase 7: AI Personalization & Adaptive Learning

**Status**: ✅ COMPLETE

**arch-review: required - approved** — [ADR-002](../adr/ADR-002-node-personalization-architecture.md)

**Architecture reference**: [ADR-001 — Python AI Microservice](../adr/ADR-001-python-ai-service.md) (Python side) and [ADR-002 — Node Personalization Architecture](../adr/ADR-002-node-personalization-architecture.md) (Node side). All LLM calls go through the Python AI service. Node determines context (difficulty level, learning style, skill rating) and passes it to Python. Python selects the appropriate prompt and generates content. Node validates responses with Zod.

**Architectural constraints (dev agent must read ADR-002 before implementing)**:
1. `quizScore` already exists on `UserProgress` — do NOT add it again; only add `completionTime`, `revisitCount`, `rating`
2. `Skill` model needs `userSkillRatings UserSkillRating[]` relation field for `UserSkillRating` migration to work
3. Algorithm services (`RecommendationEngine`, `SkillTracker`, `LearningStyleClassifier`) go in `packages/api/src/ai/`, not `services/`
4. Coaching tier enforcement is inside `CoachingService.generateFeedback()`, NOT route-level `requirePlan` middleware on quiz route
5. Coaching cache uses `GeneratedLesson` table (add nullable `coachingMessage String?` field) — not Redis
6. `CoachingService.generateFeedback()` must never throw — wrap in try/catch, return `null` on any error
7. `LearningStyleClassifier` must not write to `UserProfile` directly — return style string, caller invokes `UserService.updateUserProfile()`
8. Phase 6 Chunk 6.6 must be complete before Phase 7 begins

**Goal**: Make lesson delivery adaptive. The system learns from each user's quiz performance, engagement patterns, and learning style to personalise lesson selection, difficulty, and coaching feedback.

---

## Chunk 7.1: Recommendation Engine (Multi-Armed Bandit)

**Deliverable**: Next lesson is selected by the recommendation engine, not static day ordering

### 7.1.1 — Engagement Tracking

**File**: `packages/api/src/services/EngagementService.ts`

**Track per user per lesson**:
- `completionTime` — seconds to complete lesson
- `quizScore` — 0–100
- `revisitCount` — how many times lesson was revisited
- `rating` — optional thumbs up/down (if exposed in UI)

**Store in**: `UserProgress` table (add new fields via migration)

### 7.1.2 — Bandit Algorithm

**File**: `packages/api/src/ai/RecommendationEngine.ts`

**Algorithm**: Upper Confidence Bound (UCB1) — no LLM needed, pure math

**Logic**:
- For each candidate next lesson: compute UCB score = `avg_reward + sqrt(2 * ln(total_pulls) / pulls)`
- `reward` = normalised quiz score + completion rate
- Select lesson with highest UCB score
- Cold-start: serve lessons in order for first 3 lessons, then switch to bandit

**Inputs**: User engagement history + candidate lessons for current skill path

**Methods**:
- `getNextLesson(userId, skillPathId)` — Return recommended next lesson ID
- `recordOutcome(userId, lessonId, score, completionTime)` — Update reward history

### 7.1.3 — API Integration

**Update**: `LessonService.getTodayLesson()` — delegate to `RecommendationEngine.getNextLesson()`

**Tests**: Unit test UCB calculation; test cold-start fallback; test reward updates

---

## Chunk 7.2: Adaptive Difficulty (Bayesian Skill Tracking)

**Deliverable**: Lesson difficulty adjusts based on demonstrated knowledge level

### 7.2.1 — Skill Level Model

**File**: `packages/api/src/ai/SkillTracker.ts`

**Algorithm**: Elo-style rating (simplified Bayesian)

**Logic**:
- Each user has a `skillRating` per skill (starts at 1000)
- After each quiz: update rating using Elo formula
  - Score > 80%: rating increases (user performing above expected level)
  - Score < 50%: rating decreases
- Map rating to difficulty: <800 → beginner, 800–1200 → intermediate, >1200 → advanced

**Store**: `UserSkillRating` table (new model)

```prisma
model UserSkillRating {
  id        String   @id @default(cuid())
  userId    String
  skillId   String
  rating    Int      @default(1000)
  updatedAt DateTime @updatedAt
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  skill     Skill    @relation(fields: [skillId], references: [id])
  @@unique([userId, skillId])
}
```

### 7.2.2 — Difficulty Routing

**Methods**:
- `getCurrentLevel(userId, skillId)` — Return `beginner | intermediate | advanced`
- `updateRating(userId, skillId, quizScore)` — Recalculate after quiz submission

**Integration with Python service**: `LessonGenerationService` calls `SkillTracker.getCurrentLevel()` and passes the result as `skill_level` in the generation request to the Python AI service. Template selection for that difficulty level happens inside Python — Node only provides the level.

**Tests**: Unit test Elo calculation; test level transitions; test that correct `skill_level` is passed in the AI service request payload

---

## Chunk 7.3: AI Coaching Feedback

**Deliverable**: After each quiz, user receives personalised AI-generated feedback (Pro/Premium tier)

**Note**: The coaching endpoint (`POST /api/coaching/message`) is built in Phase 6, Chunk 6.4. This chunk wires it into the quiz flow and adds the contextual prompt construction.

### 7.3.1 — Coaching Service

**File**: `packages/api/src/services/CoachingService.ts`

**Trigger**: After quiz submission for Pro/Premium users

**Logic**:
- Assemble `CoachingRequest` payload for the Python AI service:
  - `messages`: the quiz question, user's answer, whether correct
  - `lesson_context`: lesson title + content summary
  - `user_context`: skill rating, past quiz performance, learning goal from profile
  - `tier`: user's subscription tier (Python uses for model routing only)
- Call `AIServiceClient.coachingMessage()` (built in Phase 6)
- Validate response with `CoachingOutputSchema` (Zod — already defined in Phase 6)
- Cache result per (userId, lessonId) to avoid repeat generation on page refresh

**Methods**:
- `generateFeedback(userId, lessonId, quizResult)` — Return `CoachingOutput` or null on failure
- Tier enforcement via `requirePlan('pro')` middleware on the route — never inside Python

**Fallback**: If AI service unavailable, return `coaching: null` — frontend handles gracefully (no coaching card shown). Do not block quiz result delivery.

### 7.3.2 — API Integration

**Update**: `POST /api/lessons/:id/quiz` response

**Add to response**:
```json
{
  "correct": true,
  "score": 90,
  "explanation": "...",
  "coaching": "..."   // null for free/starter or if AI unavailable; string for pro/premium
}
```

### 7.3.3 — Frontend Display

**Files**:
- `packages/web/app/(dashboard)/lessons/[id]/quiz.tsx`
- `learning-app/app/(tabs)/lessons.tsx` (mobile quiz modal)

**Display coaching message below quiz explanation** with distinct visual treatment (coaching card). Render nothing if `coaching` is null.

**Tests**: Unit test CoachingService with `AIServiceClient` mocked; test tier gating; test null fallback on AI service error; test frontend coaching card renders/hides correctly

---

## Chunk 7.4: Learning Style Detection

**Deliverable**: System detects user learning style and adapts lesson format accordingly

### 7.4.1 — Style Signals

**Track implicitly** (no user input needed):
- Lesson completion time vs. estimated time → fast/slow reader
- Revisit rate → needs repetition
- Quiz first-attempt accuracy → conceptual vs. applied learner
- Time of day pattern → morning/evening learner (already in profile)

### 7.4.2 — Style Classification

**File**: `packages/api/src/ai/LearningStyleClassifier.ts`

**Simple rule-based classifier** (no LLM):
- Fast + high accuracy → `visual-concise` style (shorter lessons, bullet points)
- Slow + low accuracy → `detailed-narrative` style (more context, examples)
- High revisit → `reinforcement` style (recap-focused lessons)

### 7.4.3 — Context Injection

**Update**: `LessonGenerationService` — include `learning_style` field in the `user_context` object passed to Python AI service.

Python uses the style value to select prompt variant (concise vs. detailed) inside its template system. Node's only responsibility is passing the classified style string — it does not select or modify prompts.

**Update**: `UserProfile.learningStyle` — auto-update from classifier on each quiz submission.

**Tests**: Unit test classifier rules; test profile update flow; test that `learning_style` is included in the AI service request payload

---

---

## What Was Implemented (2026-04-17)

### Schema (via Phase 6 Chunk 6.6 migration)
- `UserSkillRating` model — Elo ratings per (userId, skillId), unique constraint, cascade on user delete
- `UserProgress` — added `completionTime Int?`, `revisitCount Int @default(0)`, `rating Int?` (quizScore pre-existed)
- `GeneratedLesson.coachingMessage String?` — coaching cache per (userId, skillId)
- `Skill.userSkillRatings` and `User.skillRatings` backrelations

### Node Backend (`packages/api/src/`)

**`src/ai/` (new directory — computation units)**:
- `SkillTracker.ts` — Elo rating system; `getCurrentLevel(userId, skillId)` → `beginner|intermediate|advanced`; `updateRating(userId, skillId, quizScore)` — 8 tests
- `RecommendationEngine.ts` — UCB1 bandit; cold-start (≤3 completed → day order), post-cold-start → UCB score per incomplete lesson; `getNextLesson(userId, skillPathId)` + `recordOutcome()` — 4 tests
- `LearningStyleClassifier.ts` — rule-based: `visual-concise` (fast+accurate), `detailed-narrative` (slow/low accuracy), `reinforcement` (high revisit), `general` (default) — 6 tests

**`src/services/` (new files)**:
- `EngagementService.ts` — `recordEngagement()` upserts UserProgress engagement fields; `getEngagementSignals()` computes ratios for classifier — 4 tests
- `CoachingService.ts` — `generateFeedback()`: returns null for free/starter; checks cache first; calls `aiServiceClient.coachingMessage()`; never throws; caches result in `GeneratedLesson.coachingMessage` — 7 tests

**`LessonService` updates**:
- `getTodayLesson()` → delegates to `RecommendationEngine.getNextLesson()` (handles cold-start internally)
- `submitQuiz(userId, lessonId, answers, tier?)` → scores quiz → fires `skillTracker.updateRating()` + `engagementService.recordEngagement()` + `_updateLearningStyle()` (fire-and-forget); `await coachingService.generateFeedback()` (never throws); returns `{ score, feedbacks, lesson, coaching }`
- 13 tests (updated)

**`middleware/plan.ts`**: Added `detectPlan` — non-blocking plan detection (sets `req.planName` without gating tier)

**`routes/lessons.ts`**: Quiz route gets `detectPlan` middleware so controller can pass tier to service

**`LessonController.submitQuiz`**: Passes `req.planName ?? 'free'` as tier — 10 tests (updated)

### Shared Types
- `QuizResult.coaching: string | null` added to `packages/shared/src/types/lesson.ts`

### Web Frontend (`packages/web`)
- `QuizSubmitResponse` in `api-client.ts` updated to `{ score, feedbacks, lesson, coaching }`
- `submitQuiz` request body fixed: `{ answers: { [quizId]: answer } }` (was incorrect format)
- `quiz.tsx` results view updated to render per-feedback cards + coaching card (shown if `coaching !== null`, hidden if null) — 2 new api-client tests; jest config fixed (.cjs rename)

### Mobile (`learning-app`)
- `QuizModal.tsx` results view: `coaching` destructured from `submit.data`; coaching card rendered conditionally below feedback list — 2 new tests (18 total in QuizModal suite)

### Test Summary
| Suite | Tests |
|-------|-------|
| Node API (packages/api) | 100 passing |
| Web frontend (packages/web) | 12 passing |
| Mobile (learning-app) | 144 passing |

## Next Phase

[Phase 8: Notifications & Habit Engine](./phase-8-notifications-habit.md)
