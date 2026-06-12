---
title: User Flow — Enrollment, Lesson Delivery & Navigation
status: dev-complete
arch-review: not-required
note: Design agreed with the user on 2026-06-11. Self-contained spec — no prior conversation context required.
agent-note: STRICT ONE-CHUNK RULE. Implement exactly 1 chunk, then update §9 with a full implementation summary, then STOP IMMEDIATELY. Do NOT proceed to the next chunk. Do NOT say "now let's do chunk N". Do NOT offer next steps. Just stop. Context is cleared after each chunk — the user will re-invoke you. When resuming, read this doc first; start from the first chunk not marked ✅.

---

# User Flow — Enrollment, Lesson Delivery & Navigation

## 1. Problems being solved

1. **Wrong lesson shown to unenrolled users.** `getTodayLesson()` has a fallback that returns any published lesson when `profile.goal` is null. Unenrolled users see a random lesson instead of an enrolment prompt.

2. **Single-track limitation.** `UserProfile.goal` (a single Skill ID) is the entire enrollment model. Users can only be in one track at a time; there is no history of completed tracks.

3. **"Today's Lesson" / "Day X" terminology.** "Day" means two different things — the lesson sequence number in the DB (`Lesson.day`) and calendar streak days. Both appear in the UI as "Day N". Streak should mean calendar days of consecutive study. Lesson position should be labelled by topic and index, not a day number.

4. **No post-lesson navigation.** After completing a quiz, the app routes to `/` (dashboard). Users must navigate back to find the next lesson manually.

5. **Completed lesson re-served.** The fallback in `getTodayLesson()` does not reliably exclude completed lessons, so a user can be served a lesson they already finished.

6. **No track context on the lesson page.** There is no "Product Strategy — 44% complete" header on the lesson screen. Lesson position shows only title and level, not "Topic name · Lesson 4 of 8".

---

## 2. Agreed design

### 2.1 Enrollment model

Replace `UserProfile.goal` as the enrollment mechanism with a new `UserTrackEnrollment` table.

- A user can be enrolled in **multiple tracks simultaneously** (one row per user+skill).
- A "track" = a `Skill` record (which has 4 `SkillPath` levels: beginner → intermediate → advanced → expert).
- Enrolling in a track commits the user to **all levels in sequence** — there is no level-by-level enrollment.
- `completedAt` on the enrollment row is set when all published lessons across all levels of that skill are completed.
- `UserProfile.goal` is kept in the schema (not dropped) but is no longer written to or read from for lesson delivery.

```prisma
model UserTrackEnrollment {
  id          String    @id @default(cuid())
  userId      String
  skillId     String
  enrolledAt  DateTime  @default(now())
  completedAt DateTime? // null = in progress; set when 100% of track lessons completed

  user  User  @relation(fields: [userId], references: [id], onDelete: Cascade)
  skill Skill @relation(fields: [skillId], references: [id], onDelete: Restrict)

  @@unique([userId, skillId])
  @@index([userId])
  @@map("user_track_enrollments")
}
```

### 2.2 Dashboard states

The dashboard shows one of three states based on the user's enrollments:

| State | Condition | What is shown |
|---|---|---|
| **No enrollments** | No `UserTrackEnrollment` rows for user | "Enroll in your first track" CTA — links to `/tracks` |
| **Active enrollments** | ≥1 enrollment with `completedAt = null` | List of in-progress tracks, each with a "Next Lesson" CTA |
| **All complete** | All enrollments have `completedAt` set | Completed tracks list + "Enroll in your next track to continue" CTA |

"Active enrollments" state also shows completed tracks below the in-progress tracks (collapsed/secondary). Each track card shows:
- Track name
- Level progress (e.g., "Intermediate · Lesson 3 of 80 in level")
- Overall track % complete (lessons completed / total lessons across all levels)
- "Next Lesson →" button

### 2.3 Lesson page header

Above the lesson card, always show:

```
[Track Name]  [XX% complete]
[Level badge]  [Topic name · Lesson N of M]
```

- **Track Name** = `Skill.name`
- **XX% complete** = user's completed lessons in this track / total published lessons in this track (all levels)
- **Level badge** = existing badge (keep as-is)
- **Topic name · Lesson N of M** = `Lesson.topicName` + `Lesson.lessonIndex` + `Lesson.totalLessons`

Remove "Today's Lesson" heading. Remove "Day X" labels anywhere.

### 2.4 Post-lesson navigation

After quiz completion (or lesson completion for lessons without a quiz):
1. The API's `POST /api/lessons/:id/complete` and `POST /api/lessons/:id/quiz` responses include a `nextLessonId` field.
2. If `nextLessonId` is present → frontend navigates to `/lessons/:nextLessonId`.
3. If `nextLessonId` is null → the track is complete; mark the `UserTrackEnrollment.completedAt`; navigate to `/dashboard` with a "Track complete" toast.

### 2.5 "Next lesson" logic (backend)

Given a completed lesson:
1. Find the next lesson in the same `SkillPath` by `day` (i.e. `day: completedLesson.day + 1`).
2. If none found (end of level) → find the next `SkillPath` level for the same skill (`beginner→intermediate→advanced→expert` in that order) and return its `day: 1` lesson.
3. If no more levels (end of track) → return `null`; mark track complete.

### 2.6 Streak

`streakCount` on `UserProgress` is incremented per lesson — this is wrong for "days". For this ticket:
- **Do not change the DB schema for streak** (out of scope).
- **Change the UI label**: the dashboard's "Day Streak" display reads `currentStreak` from `GET /api/users/progress`. The backend already computes `currentStreak` as `sortedByDate[0].streakCount` which is the raw lesson count — this is not fixed here.
- Add a `// TODO: streak should count calendar days, not lesson completions` comment at the computation site in `UserService`.
- The word "Day" in "Day Streak" on the dashboard is kept for now (the data is wrong, but fixing streak computation is a separate ticket).

### 2.7 Onboarding change

The onboarding "pick a skill" step currently POSTs `{ goal: skillId }` to update `UserProfile`. Change it to instead call the new enrollment endpoint `POST /api/enrollments` so the user's first track enrollment is created on onboarding completion.

---

## 3. Current state facts

| File | Relevant current behaviour |
|---|---|
| `packages/api/src/services/LessonService.ts:22` | `getTodayLesson()` — falls back to any lesson when no `profile.goal` |
| `packages/api/src/services/LessonService.ts:105` | `completeLessonService()` — returns `UserProgress`, no `nextLessonId` |
| `packages/api/src/services/LessonService.ts:132` | `submitQuiz()` — returns quiz result, no `nextLessonId` |
| `packages/api/src/services/UserService.ts:41` | `getProgress()` — `currentStreak` = `sortedByDate[0].streakCount` (lesson count, not calendar days) |
| `packages/web/app/(dashboard)/dashboard/page.tsx` | Calls `getTodayLesson()` and `getProgress()`, single enrolled track, "Today's Lesson" heading |
| `packages/web/app/(dashboard)/lessons/[id]/quiz/page.tsx:91` | After quiz → `router.push('/')` |
| `packages/web/app/(auth)/onboarding/page.tsx:48` | POSTs `{ goal: selectedSkill }` to set `UserProfile.goal` |
| `prisma/schema.prisma` | No enrollment table; `UserProfile.goal` is the sole enrollment mechanism |

---

## 4. Out of scope

- Streak computation fix (calendar days vs lesson count) — separate ticket
- Admin web UI for track management
- Lesson review mode (jumping back to completed lessons)
- Mobile app changes — parity with web changes is a follow-up ticket
- `UserProfile.goal` column removal — keep it, just stop using it

---

## 5. Implementation chunks

| Chunk | What | Files |
|---|---|---|
| 1 | Schema + shared types | `prisma/schema.prisma`, `packages/shared/src/types/` |
| 2 | API — enrollment endpoints + next-lesson logic | `routes/`, `controllers/`, `services/`, tests |
| 3 | API — update lesson complete + quiz to return `nextLessonId` | `LessonService.ts`, `LessonController.ts`, tests |
| 4 | Dashboard UI rewrite — enrollment states | `packages/web/app/(dashboard)/dashboard/page.tsx` |
| 5 | Lesson page — track header, position label, post-lesson navigation | `packages/web/app/(dashboard)/lessons/[id]/` |
| 6 | Onboarding — enroll via API instead of setting `profile.goal` | `packages/web/app/(auth)/onboarding/page.tsx` |
| 7 | Verification | — |

TDD per FS role: write API tests (unit, mocked Prisma) before implementation.

---

## 6. Detailed changes per chunk

### 6.1 Chunk 1 — Schema + shared types

**`prisma/schema.prisma`**: Add the `UserTrackEnrollment` model (§2.1 above). Add inverse relation on `User` and `Skill`.

**`packages/shared/src/types/lesson.ts`** (or new `enrollment.ts`):
```ts
export interface UserTrackEnrollment {
  id: string;
  userId: string;
  skillId: string;
  enrolledAt: string;
  completedAt: string | null;
}

export interface TrackEnrollmentWithProgress extends UserTrackEnrollment {
  skill: Skill;
  totalLessons: number;
  completedLessons: number;
  percentComplete: number;
  nextLesson: Lesson | null;
}
```

Also update `QuizResult` and lesson completion response types to include `nextLessonId: string | null`.

**Migration — user runs in real terminal (TTY required):**
```bash
cd /Users/guypowell/Documents/Projects/learning
npx prisma migrate dev --name user-track-enrollments
```

### 6.2 Chunk 2 — Enrollment API endpoints (TDD: write tests first)

New service: `packages/api/src/services/EnrollmentService.ts`

Methods:
- `enroll(userId, skillId)` — creates `UserTrackEnrollment`; throws 409 if already enrolled; throws 404 if skill not found
- `unenroll(userId, skillId)` — deletes the enrollment row; throws 404 if not enrolled
- `getEnrollments(userId)` — returns all enrollments for user with progress data:
  - `totalLessons` = count of published lessons for that skill across all levels
  - `completedLessons` = count of UserProgress with `completedAt` for lessons in that skill
  - `percentComplete` = completedLessons / totalLessons × 100 (rounded)
  - `nextLesson` = the next uncompleted lesson in curriculum order (`day` asc, across levels in order)
- `markTrackCompleteIfDone(userId, skillId)` — called after every lesson completion; sets `completedAt` if `completedLessons === totalLessons`

New routes (`packages/api/src/routes/enrollments.ts`):
```
POST   /api/enrollments            — enroll in a track ({ skillId })
DELETE /api/enrollments/:skillId   — unenroll
GET    /api/enrollments            — list user's enrollments with progress
```
All routes behind `authMiddleware`.

Register in `packages/api/src/index.ts`: `app.use('/api/enrollments', enrollmentRouter)`.

Unit tests (mocked Prisma): enroll success, duplicate enroll 409, invalid skill 404, unenroll success, unenroll not-enrolled 404, getEnrollments returns progress, markTrackComplete sets completedAt when 100%.

### 6.3 Chunk 3 — Lesson complete + quiz return `nextLessonId` (TDD: write tests first)

Add `private getNextLessonId(lessonId: string): Promise<string | null>` to `LessonService`:
1. Fetch the lesson with its `skillPath` (to get `skillId` and `level`).
2. Try `prisma.lesson.findFirst({ where: { skillPathId: lesson.skillPathId, day: lesson.day + 1 }, published: true })`.
3. If none: find the next level's SkillPath for the same skill (level order: beginner=1, intermediate=2, advanced=3, expert=4 — use `LEVEL_ID` from `packages/shared`) and return its `day: 1` lesson.
4. If no next level: return `null` and call `EnrollmentService.markTrackCompleteIfDone(userId, skillId)`.

Update `completeLessonService(userId, lessonId)` to return `{ progress, nextLessonId }`.

Update `submitQuiz(userId, lessonId, answers, tier?)` to call `completeLessonService` internally (it already marks progress) and include `nextLessonId` in the returned `QuizResult`.

Update `QuizResult` shared type: `nextLessonId: string | null` (already has `streak`, `milestone`, `coaching`).

Update `LessonController` to pass `nextLessonId` through in responses for both endpoints.

Unit tests: next lesson exists in same path, end-of-level returns first of next level, end-of-track returns null + marks enrollment complete, quiz result includes nextLessonId.

### 6.4 Chunk 4 — Dashboard UI rewrite

`packages/web/app/(dashboard)/dashboard/page.tsx`:

Replace the single `getTodayLesson()` + `enrolledTrack` pattern with a call to `GET /api/enrollments` (add `apiClient.getEnrollments()` method).

**State machine:**

```tsx
if (enrollments.length === 0) {
  // No enrollments state
  return <EnrollPrompt variant="first" />
}

const active = enrollments.filter(e => !e.completedAt)
const completed = enrollments.filter(e => e.completedAt)

if (active.length === 0) {
  // All complete state
  return <>
    <CompletedTrackList tracks={completed} />
    <EnrollPrompt variant="next" />
  </>
}

// Active enrollments state
return <>
  <ActiveTrackList tracks={active} />
  {completed.length > 0 && <CompletedTrackList tracks={completed} collapsed />}
</>
```

`EnrollPrompt` props: `variant: 'first' | 'next'`. Renders:
- `'first'`: "Start learning — enroll in your first track" + "Browse Tracks →" button linking to `/tracks`.
- `'next'`: "You've completed a track — keep going!" + "Browse Tracks →" button.

Each active track card (`ActiveTrackCard`):
- Track name (large)
- "XX% complete" (computed from `enrollment.percentComplete`)
- Level badge for current level (derived from `enrollment.nextLesson.skillPath.level`)
- Lesson title (from `enrollment.nextLesson.title`)
- `enrollment.nextLesson.topicName + " · Lesson " + lessonIndex + " of " + totalLessons`
- "Next Lesson →" button → `/lessons/:nextLesson.id`

Remove "Today's Lesson" heading. Keep streak chip and stats (they're correct data, just the label has the streak day count which is acceptable for now).

Add `apiClient.getEnrollments()` method to `packages/web/lib/api-client.ts`.

### 6.5 Chunk 5 — Lesson page: track header + position label + post-lesson nav

**`packages/web/app/(dashboard)/lessons/[id]/page.tsx`**:

Fetch additional data alongside the lesson: `GET /api/enrollments` to find the enrollment for this lesson's track. (Alternatively, the lesson API response can include track progress — but using enrollments is correct given the new model.)

Add header above the lesson card:
```
[Track Name]          [XX% complete]
[Level badge]  [Topic name · Lesson N of M]
```

Map `lesson.skillPath.level` to the level display (existing badge logic — keep).
Show `lesson.topicName + " · Lesson " + lesson.lessonIndex + " of " + lesson.totalLessons`.

**Remove "Day X" label** wherever it appears on the lesson page.

**`packages/web/app/(dashboard)/lessons/[id]/quiz/page.tsx`**:

After quiz submission, the response now contains `nextLessonId`. Change:
```ts
// Before:
router.push('/')

// After:
if (result.nextLessonId) {
  router.push(`/lessons/${result.nextLessonId}`)
} else {
  router.push('/dashboard') // track complete
  // TODO: show "Track complete" toast on dashboard
}
```

Similarly update lesson completion (non-quiz path) to use `completeLessonResponse.nextLessonId`.

### 6.6 Chunk 6 — Onboarding enrollment change

`packages/web/app/(auth)/onboarding/page.tsx`:

Change the submit handler from:
```ts
await apiClient.updateProfile({ goal: selectedSkill })
```
to:
```ts
await apiClient.enroll(selectedSkill)  // POST /api/enrollments
```

Add `apiClient.enroll(skillId: string)` method to `api-client.ts`.

Remove `goal` from any profile update calls during onboarding. `UserProfile.goal` is left in the DB but no longer written to.

### 6.7 Chunk 7 — Verification

1. TypeScript: `npx tsc --noEmit` clean across all three packages.
2. Full API unit test suite green (all existing + new tests).
3. DB check: `UserTrackEnrollment` table exists; migration applied.
4. Manual flow (user verifies in browser):
   - New user → dashboard shows enroll prompt → `/tracks` → enroll → dashboard shows track card → "Next Lesson" → lesson page has track header + topic/lesson label — no "Today's Lesson" / "Day X".
   - Complete lesson (no quiz) → auto-navigate to next lesson.
   - Complete quiz → auto-navigate to next lesson.
   - Complete last lesson in a level → auto-navigate to first lesson of next level.
   - Refresh after completing a lesson → next lesson shown (not the one just completed).
   - Complete all lessons in all levels → track marked complete on dashboard; "Enroll in your next track" prompt shown.

---

## 7. API summary

| Method | Route | Auth | Purpose |
|---|---|---|---|
| POST | `/api/enrollments` | ✅ | Enroll in a track (`{ skillId }`) |
| DELETE | `/api/enrollments/:skillId` | ✅ | Unenroll from a track |
| GET | `/api/enrollments` | ✅ | List enrollments with progress + nextLesson |

Existing endpoints modified:
- `POST /api/lessons/:id/complete` → response now includes `nextLessonId: string | null`
- `POST /api/lessons/:id/quiz` → `QuizResult` now includes `nextLessonId: string | null`

`GET /api/lessons/today` is **not removed yet** (may still be referenced elsewhere) but is deprecated — the dashboard no longer calls it.

---

## 8. Acceptance criteria

- [ ] `UserTrackEnrollment` table in DB; migration created and applied
- [ ] `GET /api/enrollments` returns enrollments with `totalLessons`, `completedLessons`, `percentComplete`, `nextLesson`
- [ ] `POST /api/enrollments` creates enrollment; 409 on duplicate; 404 on unknown skill
- [ ] `POST /api/lessons/:id/complete` and `POST /api/lessons/:id/quiz` return `nextLessonId`
- [ ] `nextLessonId` is null at end of track; `UserTrackEnrollment.completedAt` is set
- [ ] Dashboard shows correct state: no enrollments → enroll prompt; active enrollments → track cards; all complete → completed list + next enroll prompt
- [ ] Lesson page shows: track name + % complete header; topic + lesson position label (no "Today's Lesson", no "Day X")
- [ ] After quiz → auto-navigate to `nextLessonId` (or dashboard if track complete)
- [ ] After lesson complete (no quiz) → auto-navigate to `nextLessonId`
- [ ] End of level → next lesson is first of next level (same track)
- [ ] End of track → `completedAt` set; dashboard shows completed state
- [ ] Onboarding enrolls via `POST /api/enrollments` (not `profile.goal`)
- [ ] `UserService` has `// TODO: streak should count calendar days` comment at streak computation
- [ ] `tsc --noEmit` clean; all API unit tests green
- [ ] Shared types updated: `TrackEnrollmentWithProgress`, `QuizResult.nextLessonId`

---

## 9. Implementation Progress (updated after each chunk)

### ✅ Chunk 1 — Schema + shared types (2026-06-11)

**Schema changes** (`prisma/schema.prisma`):
- Added `UserTrackEnrollment` model with `id`, `userId`, `skillId`, `enrolledAt`, `completedAt?`, `@@unique([userId, skillId])`, `@@index([userId])`, `@@map("user_track_enrollments")`
- Added `onDelete: Cascade` on `user` relation, `onDelete: Restrict` on `skill` relation
- Added `enrollments UserTrackEnrollment[]` inverse relation to `User` model
- Added `enrollments UserTrackEnrollment[]` inverse relation to `Skill` model

**Shared types** (`packages/shared/src/types/lesson.ts`):
- Added `UserTrackEnrollment` interface
- Added `TrackEnrollmentWithProgress` interface (extends `UserTrackEnrollment` with `skill`, `totalLessons`, `completedLessons`, `percentComplete`, `nextLesson`)
- Updated `QuizResult` to include `nextLessonId: string | null`

**Migration** — ✅ **User action required**: Run in a real terminal (TTY required):
```bash
cd /Users/guypowell/Documents/Projects/learning
npx prisma migrate dev --name user-track-enrollments
```

**Tests**: No tests for this chunk (schema + type changes only).

---

### ✅ Chunk 2 — Enrollment API endpoints (2026-06-11)

**New files**:
- `packages/api/src/services/EnrollmentService.ts` — `enroll`, `unenroll`, `getEnrollments`, `markTrackCompleteIfDone`
- `packages/api/src/services/__tests__/EnrollmentService.test.ts` — 11 unit tests (written first, TDD)
- `packages/api/src/controllers/EnrollmentController.ts` — `enroll`, `unenroll`, `getEnrollments`
- `packages/api/src/routes/enrollments.ts` — `POST /`, `DELETE /:skillId`, `GET /`

**`app.ts`**: Added `app.use('/api/enrollments', enrollmentRoutes)`.

**Service behaviour**:
- `enroll`: 404 if skill not found; 409 if already enrolled; creates `UserTrackEnrollment`
- `unenroll`: 404 if no enrollment found; deletes row
- `getEnrollments`: returns enrollments with `totalLessons`, `completedLessons`, `percentComplete`, `nextLesson`; `nextLesson` sorted by LEVEL_ORDER (beginner→expert) then `day` asc
- `markTrackCompleteIfDone`: no-op if already complete; sets `completedAt` when `completedLessons >= totalLessons`

**Tests**: 11 new unit tests; full suite 234/234 passing (was 206 before this chunk). `tsc --noEmit` clean.

---

### ✅ Chunk 3 — Lesson complete + quiz return `nextLessonId` (2026-06-11)

**New private method**: `LessonService._getNextLessonId(userId, lesson)` — walks the curriculum in order:
1. Next lesson in same `SkillPath` by `day + 1` (same level)
2. If none: iterates `LEVEL_PROGRESSION` (`beginner→intermediate→advanced→expert`) to find the next SkillPath for the same skill, returns its `day: 1` lesson
3. If no next level: calls `enrollmentService.markTrackCompleteIfDone(userId, skillId)` and returns `null`

**`LessonService.completeLessonService`**:
- Now fetches lesson with `include: { skillPath: true }` (was bare findUnique)
- Calls `_getNextLessonId` after progress upsert
- Returns `{ progress, nextLessonId }` (was `progress` only)

**`LessonService.submitQuiz`**:
- Calls `_getNextLessonId` using the lesson already fetched by `getLessonById`
- Returns `{ score, feedbacks, lesson, coaching, streak, milestone, nextLessonId }` (adds `nextLessonId`)

**`LessonController.completeLesson`**:
- Destructures `{ progress, nextLessonId }` from service
- Includes `nextLessonId` in response data

**`UserService.getProgress`**:
- Added `// TODO: streak should count calendar days, not lesson completions` at streak computation site (line ~60)

**Tests**: 5 new tests added to `LessonService.test.ts`; full suite 239/239 passing (was 234). `tsc --noEmit` clean.

---

### ✅ Chunk 4 — Dashboard UI rewrite (2026-06-11)

**Files changed**:
- `packages/shared/src/types/lesson.ts` — added `skillPath?: { level: string }` to `Lesson` interface (the enrollment API includes this field on `nextLesson`; optional field is backward-compatible)
- `packages/web/lib/api-client.ts` — added `TrackEnrollmentWithProgress` import; added `getEnrollments()` method (`GET /api/enrollments`)
- `packages/web/app/(dashboard)/dashboard/page.tsx` — full rewrite

**Dashboard state machine implemented**:
- **No enrollments** → "Start learning" card with "Browse Tracks →" link to `/tracks`
- **Active enrollments** → `ActiveTrackCard` per in-progress track; collapsed `CompletedTrackCard` list below if any completed tracks
- **All complete** → `CompletedTrackCard` list + "You've completed a track — keep going!" card with "Browse Tracks →" link

**`ActiveTrackCard` renders**:
- Track name + percentage complete (from `enrollment.percentComplete`)
- `LevelBadge` (level number via `LEVEL_ID` map) + topic position label (`topicName · Lesson N of M` when available; falls back to capitalized level name)
- Next lesson title
- "Next Lesson →" button → `/lessons/:nextLesson.id`

**Kept from original**:
- Streak banner (shown when streak > 3)
- Streak chip card ("Day Streak")
- Stats grid (Lessons Completed + Average Score)
- "Today's Lesson" heading **removed**; no "Day X" labels anywhere on the page

**`tsc --noEmit` clean**: shared ✅, api ✅, web ✅

---

### ✅ Chunk 5 — Lesson page: track header, position label, post-lesson navigation (2026-06-11)

**Files changed**:
- `packages/shared/src/types/lesson.ts` — added `skillId?: string` to `Lesson.skillPath` type so the frontend can match a lesson to its enrollment
- `packages/web/lib/api-client.ts` — added `nextLessonId` and `streak`/`milestone` to `QuizSubmitResponse`; added `CompleteLessonResponse` interface; updated `completeLesson` return type from `UserProgress` to `CompleteLessonResponse`
- `packages/web/app/(dashboard)/lessons/[id]/page.tsx` — full rewrite
- `packages/web/app/(dashboard)/lessons/[id]/quiz/page.tsx` — navigation + button label update

**Lesson page changes**:
- Fetches `GET /api/enrollments` in parallel with `GET /api/lessons/:id`; matches enrollment by `enrollment.skillId === lesson.skillPath?.skillId`
- Added **track header** above the lesson card: track name (from `enrollment.skill.name`) + `XX% complete` on the left/right; `LevelBadge` + topic position label (`topicName · Lesson N of M`) on the second row
- **Removed** `<span>Day {lesson.day}</span>` — only duration + difficulty Tag remain in the lesson meta row
- `handleComplete`: if `lesson.quizzes.length === 0` → navigate to `nextLessonId` (or `/dashboard` if null); if quizzes present → show completed state + key takeaway + "Take Quiz" link (unchanged behaviour)

**Quiz page changes**:
- Added `nextLessonId: string | null` to local `QuizResult` interface
- `handleNext` captures `result?.nextLessonId` before clearing state; on last question navigates to `/lessons/:nextLessonId` or `/dashboard` (was `router.push('/')`)
- Button label on last question: `'Next Lesson'` when `nextLessonId` is set, `'Back to Dashboard'` when null

**`tsc --noEmit`**: shared ✅, api ✅, web ✅

---

### ✅ Chunk 6 — Onboarding enrollment change (2026-06-11)

**Files changed**:
- `packages/web/lib/api-client.ts` — added `enroll(skillId: string): Promise<void>` method (`POST /api/enrollments`)
- `packages/web/app/(auth)/onboarding/page.tsx` — submit handler now calls `apiClient.enroll(selectedSkill)` first, then `apiClient.updateProfile(...)` without the `goal` field. `UserProfile.goal` is no longer written during onboarding.
- `packages/web/app/(auth)/onboarding/__tests__/page.test.tsx` — new test file (TDD, written first)

**Tests** (4 new, all pass):
- `loads and displays skills` — skills are fetched and rendered
- `calls apiClient.enroll with the selected skill on submit` — correct skill ID passed; navigates to `/dashboard`
- `does NOT call updateProfile with goal on submit` — `goal` field is absent from `updateProfile` call
- `shows error if enroll fails` — API error surfaced in UI; no redirect

**Full suite**: 185/204 unit tests passing. 19 failures are pre-existing (home page marketing tests + e2e specs needing a live server) — none introduced by this chunk. `tsc --noEmit` clean: shared ✅, api ✅, web ✅.

---

### ✅ Chunk 7 — Verification (2026-06-11)

**TypeScript**: `tsc --noEmit` clean across all three packages — shared ✅, api ✅, web ✅.

**API unit tests**: 239/239 passing, 28 suites. No regressions.

**DB**: Migration applied (`prisma migrate status` → "Database schema is up to date!"). `user_track_enrollments` table confirmed in PostgreSQL with correct columns (`id`, `userId`, `skillId`, `enrolledAt`, `completedAt`), unique index on `(userId, skillId)`, index on `userId`, correct foreign key constraints (cascade delete on user, restrict on skill).

**Manual verification required (user action)**: The following flows must be exercised in the browser to confirm the full ticket is complete:
- [x] New user → dashboard shows enroll prompt → `/tracks` → enroll → dashboard shows track card → "Next Lesson →" → lesson page has track header + topic/lesson label (no "Today's Lesson", no "Day X")
- [x] Complete lesson (no quiz) → auto-navigate to next lesson
- [x] Complete quiz → auto-navigate to next lesson
- [ ] Complete last lesson in a level → auto-navigate to first lesson of next level
- [ ] Refresh after completing a lesson → next lesson shown (not the completed one)
- [ ] Complete all lessons in all levels → track marked complete on dashboard; "Enroll in your next track" prompt shown

---

## 10. Post-implementation bugs found during verification

### ✅ Bug 1 — `/tracks` page called `updateProfile` instead of `enroll` (2026-06-11)

**Symptom**: After enrolling from `/tracks`, the dashboard still showed "Start learning / Enroll in your first track". The `/tracks` page showed the track as enrolled.

**Root cause**: `packages/web/app/(dashboard)/tracks/page.tsx` `handleEnrol()` called `apiClient.updateProfile({ goal: skillId })` — the old `UserProfile.goal` pattern — instead of `apiClient.enroll(skillId)`. No `UserTrackEnrollment` row was ever created, so `GET /api/enrollments` returned `[]`.

The "Enrolled" label on `/tracks` appeared to work because `SkillWithAccess.enrolledSkillId` still reads from `UserProfile.goal` on the backend — the field was updated but the enrollment table was not.

**Fix** (`packages/web/app/(dashboard)/tracks/page.tsx` line 27–28):
```ts
// Before:
await apiClient.updateProfile({ goal: skillId })
router.push('/')

// After:
await apiClient.enroll(skillId)
router.push('/dashboard')
```

---

### ✅ Bug 2 — `/tracks` "Enrolled" badge used stale `enrolledSkillId` field (2026-06-11)

**Symptom**: After fixing Bug 1, the "Enrolled" badge and "Currently enrolled" text on `/tracks` never showed — even after enrolling — because `skill.enrolledSkillId` reads `UserProfile.goal` which is no longer written to.

**Root cause**: `isEnrolled` on line 57 was `skill.enrolledSkillId === skill.id`. `SkillWithAccess.enrolledSkillId` is populated by the backend from `UserProfile.goal`, not from `UserTrackEnrollment`. Since `UserProfile.goal` is no longer written during enrollment, this field is always `null`.

**Fix** (`packages/web/app/(dashboard)/tracks/page.tsx`):

1. Fetch enrollments alongside skills on mount:
```ts
const [enrolledSkillIds, setEnrolledSkillIds] = useState<Set<string>>(new Set())

// In useEffect:
Promise.all([
  apiClient.getSkills(),
  apiClient.getEnrollments().catch(() => []),
])
  .then(([skillsData, enrollments]) => {
    setSkills(skillsData)
    setEnrolledSkillIds(new Set(enrollments.map(e => e.skillId)))
  })
```

2. Update `isEnrolled` derivation:
```ts
// Before:
const isEnrolled = skill.enrolledSkillId === skill.id

// After:
const isEnrolled = enrolledSkillIds.has(skill.id)
```

3. Update local state immediately after successful enrol (avoids a refetch):
```ts
await apiClient.enroll(skillId)
setEnrolledSkillIds(prev => new Set(prev).add(skillId))
router.push('/dashboard')
```
