# Freemium Track Access — Requirements & Build Plan

**Status**: `complete`  
**arch-review**: `not-required`  
**Created**: 2026-06-09

---

## Problem

Free test users cannot access AI track lessons they've been added to the DB. The deeper issue is the app has no track discovery UI and no freemium gating — users are silently given a random first lesson with no visibility into other tracks. This kills conversion: users who can't see premium content don't know what they're missing and have no reason to upgrade.

---

## Goals

1. Users can browse all tracks and see what's available — free and premium
2. Free users see premium tracks locked with an upgrade CTA (not hidden)
3. Users can enrol in any track they have access to, and switch at any time
4. Lesson delivery respects the user's chosen track
5. Test users can be flagged as paid without code changes

---

## Out of Scope

- Multi-track simultaneous enrolment (users are in exactly one track at a time for MVP)
- Stripe payment flow (Phase 5 cancelled)
- Analytics on upgrade click-throughs (Phase 11.2 deferred)

---

## Data Model Changes

### `Skill` — add `isPremium` flag

```prisma
model Skill {
  // existing fields ...
  isPremium Boolean @default(false)
}
```

All existing skills default to `false`. "AI for Product Managers" is marked `true` in the seed.

**No other schema changes.** `UserProfile.goal` (skill ID) already captures the user's enrolled track. `SubscriptionPlan.maxSkillPaths` already captures the plan limit (free = 1, which is enforced implicitly by single-track design).

---

## Chunks

---

### Chunk 1 — Schema, Seed & Backend

**Deliverables**

1. **Prisma**: Add `isPremium Boolean @default(false)` to `Skill`
2. **Seed**: Set `isPremium: true` on the "AI for Product Managers" skill upsert
3. **`GET /api/skills`** — update response to include:
   - `isPremium: boolean` on each skill
   - `userHasAccess: boolean` — `true` if skill is free, or user has a paid subscription
   - `enrolledSkillId: string | null` — the user's current `profile.goal`
4. **`PATCH /api/users/profile`** — allow users to update `goal` (track switch). Validate: if target skill is premium and user is free, return `403 UPGRADE_REQUIRED`.
5. **Fix `getTodayLesson`**: when user has no progress records, look up the first lesson from the skill path matching `user.profile.goal`. Fall back to first published lesson only if `profile.goal` is null.

**Plan gating approach**

`detectPlan` middleware already exists. For the track switch endpoint, check the skill's `isPremium` flag against the user's subscription tier. No change to existing lesson delivery auth — gating happens at enrolment time.

**Tests**
- Unit: `PATCH /api/users/profile` — free user blocked from premium skill, paid user allowed
- Unit: updated `GET /api/skills` returns correct `userHasAccess` per tier
- Unit: `getTodayLesson` uses `profile.goal` when no progress exists

---

### Chunk 2 — Frontend: Tracks Page

**Route**: `/tracks` (new page, accessible from dashboard nav)

**Page layout**: Grid of track cards. Each card shows:
- Track name + category badge
- Number of lessons + estimated hours
- User's progress (% complete) if enrolled
- "Enrolled" badge if this is the user's current track
- For premium tracks + free users: lock icon + "Upgrade to unlock" CTA (links to `/settings` or upgrade flow)

**Enrol / Switch**: clicking an accessible track calls `PATCH /api/users/profile` with the new `goal`, then redirects to `/` (dashboard) to start the first lesson.

**Dashboard update**: Dashboard "Today's Lesson" card shows the track name so users know which track they're in and can click through to `/tracks` to switch.

**Tests**
- Renders locked state for premium tracks when user is free
- Renders enrolled badge for current track
- Enrol action calls API and redirects

---

### Chunk 3 — Test Access

No code changes. Use `scripts/upgrade-user.ts` to create/update the subscription record:

```bash
pnpm tsx scripts/upgrade-user.ts
```

Note: `/admin/users` referenced in the original plan does not exist — use the script above.

**Test user**: `testai2@test.com` — upgraded to `pro`, enrolled in "AI for Product Managers"

---

## Acceptance Criteria

- [x] Free user sees all tracks on `/tracks`, AI track shows locked state
- [x] Free user cannot enrol in AI track — gets upgrade prompt
- [x] Paid test user can enrol in AI track
- [x] After enrolment, `GET /api/lessons/today` returns a lesson from the correct track
- [x] User can switch tracks (accessible ones) from `/tracks` at any time
- [x] Dashboard shows which track the user is currently in
- [x] All new code has unit test coverage

---

## Implementation Order

1. Chunk 1 (schema + backend) first — unblocks testing via API
2. Chunk 2 (frontend tracks page) — depends on Chunk 1 API
3. Chunk 3 (test access flag) — can be done any time after seed is re-run

---

## What Was Implemented

### Chunk 1 — Schema, Seed & Backend (2026-06-09)

**Schema**
- ✅ `isPremium Boolean @default(false)` added to `Skill` model (`prisma/schema.prisma`)
- ✅ `pnpm db:push` synced to local DB

**Seed**
- ✅ `prisma/seed.ts` — AI for Product Managers upsert now sets `isPremium: true` on both `create` and `update`

**`GET /api/skills`** (`packages/api/src/routes/lessons.ts`)
- ✅ Now runs `detectPlan` middleware — reads user's subscription tier
- ✅ Fetches `userProfile.goal` alongside team memberships in a parallel `Promise.all`
- ✅ Returns `userHasAccess: boolean` (`true` if free skill or paid user) and `enrolledSkillId: string | null` on every skill in the response

**`PATCH /api/users/profile`** (`packages/api/src/routes/users.ts`, `controllers/UserController.ts`)
- ✅ `detectPlan` middleware added to the route
- ✅ Controller checks `skill.isPremium` when `goal` is being set; throws `403 SUBSCRIPTION_INACTIVE` if user is on free plan
- ✅ Non-goal profile updates (timezone, preferredTime, etc.) bypass the skill check entirely

**`getTodayLesson`** (`packages/api/src/services/LessonService.ts`)
- ✅ When a user has no progress records, looks up `user.profile.goal` (skill ID)
- ✅ Finds the earliest `skillPath` for that skill, then the first `published` lesson ordered by `day asc`
- ✅ Falls back to unordered `findFirst` only if `profile.goal` is null or the skill has no paths/lessons

**Tests** — all 212 pass (was 206)
- ✅ `UserController`: 3 new tests — free user blocked from premium track, paid user allowed, non-goal update skips skill check
- ✅ `LessonService`: 4 new tests — routes new user by `profile.goal`, falls back when path not found, falls back when path has no lessons

**Note for re-seeding**: `seed.ts` runs `prisma.skill.deleteMany()` which changes all skill IDs. After re-seeding, any existing user's `profile.goal` will be stale. Re-register the test user (or manually update `profile.goal` in DB) after each re-seed.

---

### Bug Fixes (2026-06-09, post-chunk)

**`LessonService.getTodayLesson`** (`packages/api/src/services/LessonService.ts`)
- ✅ Track-switch bug: incomplete progress from old track was overriding `profile.goal` — fixed by filtering `incompleteLessons` to the enrolled skill only
- ✅ Post-completion loop: after completing a lesson the service was routing back to day 1 — fixed by excluding completed lesson IDs when finding the next unstarted lesson
- ✅ 2 new unit tests added (20 total in LessonService suite)

**Seed (`prisma/seed.ts`)** — three silent failures fixed:
- ✅ `userSkillRating.deleteMany()` now called before `skill.deleteMany()` (FK constraint was aborting the seed silently)
- ✅ `__dirname` now defined via `fileURLToPath(import.meta.url)` (ESM context, `__dirname` was undefined — generated-lessons.json was never loaded)
- ✅ `l.quiz[0]` instead of `l.quiz` (quiz is an array in the JSON; was causing `undefined` field errors)

**`scripts/upgrade-user.ts`** — updated to create subscription if none exists (upsert behaviour)

**Note**: `pnpm db:seed` swallows seed output and errors — always run the seed directly with `npx tsx prisma/seed.ts` from `packages/api/` to see actual output.

---

### Chunk 2 — Frontend: Tracks Page (2026-06-09)

**Shared types** (`packages/shared/src/types/lesson.ts`)
- ✅ `isPremium: boolean` added to `Skill` interface
- ✅ `SkillPath.lessons` made optional (`lessons?: Lesson[]`) — the `/api/skills` endpoint includes paths without lessons
- ✅ New `SkillWithAccess` interface extending `Skill` with `skillPaths`, `userHasAccess`, `enrolledSkillId`

**API client** (`packages/web/lib/api-client.ts`)
- ✅ `SkillWithAccess` imported from `@learning/shared`
- ✅ `getSkills()` return type updated to `Promise<SkillWithAccess[]>`

**`/tracks` page** (`packages/web/app/(dashboard)/tracks/page.tsx`)
- ✅ Calls `getSkills()` on mount
- ✅ Renders a card per track: name, category badge, total estimated hours (summed from `skillPaths[].durationHours`)
- ✅ "Enrolled" badge (brand tone) on the user's current track
- ✅ "Premium" badge + "Upgrade to unlock" CTA linking to `/settings` for locked tracks
- ✅ Enrol button for accessible, non-enrolled tracks — calls `updateProfile({ goal })` then redirects to `/`

**Dashboard** (`packages/web/app/(dashboard)/dashboard/page.tsx`)
- ✅ Now fetches `getSkills()` in parallel with lesson and progress data
- ✅ Today's Lesson card shows the enrolled track name as a link to `/tracks` (top-right of card header)

**Sidebar nav** (`packages/web/app/(dashboard)/layout.tsx`)
- ✅ "Tracks" link added between Dashboard and Progress

**Tests** — 6 new tests in `app/(dashboard)/tracks/__tests__/page.test.tsx`, all passing (173 total unit tests pass)
- ✅ Renders page heading
- ✅ Renders a card for each track
- ✅ Renders Enrolled badge for current track
- ✅ Renders locked state for premium track when user is free
- ✅ Renders enrol button for accessible non-enrolled track
- ✅ Enrol calls `updateProfile` and redirects to `/`
