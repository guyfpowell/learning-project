# Phase 7: AI Personalization & Adaptive Learning

**Status**: ⚪ NOT STARTED

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

**Update**: `LessonGenerationService.selectTemplate()` — use `SkillTracker.getCurrentLevel()` to pick correct template difficulty

**Methods**:
- `getCurrentLevel(userId, skillId)` — Return `beginner | intermediate | advanced`
- `updateRating(userId, skillId, quizScore)` — Recalculate after quiz submission

**Tests**: Unit test Elo calculation; test level transitions; test template selection routing

---

## Chunk 7.3: AI Coaching Feedback

**Deliverable**: After each quiz, user receives personalised AI-generated feedback (Pro/Premium tier)

### 7.3.1 — Coaching Service

**File**: `packages/api/src/services/CoachingService.ts`

**Trigger**: After quiz submission for Pro/Premium users

**Prompt**: Build contextual prompt with:
- Question + user's answer (right or wrong)
- User's history on this topic (skill rating, past quiz performance)
- Learning goal from user profile

**Output**: 2–3 sentence coaching message explaining the correct answer in the context of the user's goal

**Model**: OpenAI (Pro/Premium only) — gated by `requirePlan('pro')` middleware

**Methods**:
- `generateFeedback(userId, lessonId, quizResult)` — Return coaching message string
- Cached per (userId, lessonId) to avoid repeat generation on page refresh

### 7.3.2 — API Integration

**Update**: `POST /api/lessons/:id/quiz` response

**Add to response**:
```json
{
  "correct": true,
  "score": 90,
  "explanation": "...",
  "coaching": "..."   // null for free/starter; AI-generated for pro/premium
}
```

### 7.3.3 — Frontend Display

**Files**:
- `packages/web/app/(dashboard)/lessons/[id]/quiz.tsx`
- `learning-app/app/(tabs)/lessons.tsx` (mobile quiz modal)

**Display coaching message below quiz explanation** with distinct visual treatment (coaching card)

**Tests**: Unit test CoachingService with OpenAI mocked; test tier gating; test frontend rendering of coaching card

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

### 7.4.3 — Template Influence

**Update**: `LessonGenerationService.buildUserContext()` — inject detected learning style into prompt

**Template variation**: Each template has style variants (concise vs. detailed prompt suffixes)

**Update**: `UserProfile.learningStyle` — auto-update from classifier on each quiz submission

**Tests**: Unit test classifier rules; test profile update flow; test prompt style injection

---

## Next Phase

[Phase 8: Notifications & Habit Engine](./phase-8-notifications-habit.md)
