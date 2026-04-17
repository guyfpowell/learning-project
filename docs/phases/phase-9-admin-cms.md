# Phase 9: Admin CMS

**Status**: ✅ COMPLETE (Chunks 9.1–9.4)

**Goal**: Build an internal admin interface for authoring lessons, managing skill paths, and overseeing platform content — without needing to touch the database directly.

---

## Chunk 9.1: Admin Auth & Role System

**Deliverable**: Admin users can log in to a protected admin area

### 9.1.1 — Admin Role

**Schema update**: Add `role` field to `User` model

```prisma
enum UserRole {
  user
  admin
}
// Add to User: role UserRole @default(user)
```

**Middleware**: `packages/api/src/middleware/adminGuard.ts` — checks `req.user.role === 'admin'`, returns 403 otherwise

**Seed**: Add first admin user via `prisma/seed.ts` (email + hashed password, role = admin)

### 9.1.2 — Admin Routes

**Base path**: `/api/admin/*` — all routes protected by `authMiddleware` + `adminGuard`

**Routes to add**:
- `GET /api/admin/skills` — List all skills
- `POST /api/admin/skills` — Create skill
- `GET /api/admin/skill-paths` — List all skill paths
- `POST /api/admin/skill-paths` — Create skill path
- `GET /api/admin/lessons` — List all lessons (paginated)
- `POST /api/admin/lessons` — Create lesson
- `PATCH /api/admin/lessons/:id` — Update lesson
- `DELETE /api/admin/lessons/:id` — Soft delete lesson (set `published = false`)
- `GET /api/admin/users` — List users (paginated, with subscription status)
- `GET /api/admin/stats` — Platform stats (total users, DAU, MRR)

### 9.1.3 — Admin Frontend Route Group

**File**: `packages/web/app/(admin)/layout.tsx`

**Features**:
- Separate layout from dashboard (no tab nav — use sidebar)
- Admin auth check: redirect to `/login` if not admin role
- Sidebar navigation: Skills, Lessons, Users, Stats

### ✅ 9.1 Implementation Summary (2026-04-17)

- ✅ **Prisma schema**: `UserRole` enum (`user` | `admin`), `role UserRole @default(user)` on `User`, `published Boolean @default(true)` + `publishedAt DateTime?` on `Lesson`
- ✅ **JWT**: `TokenPayload` now includes `role`; `AuthService.register` + `login` both include `role` in token
- ✅ **Shared types**: `User` interface in `@learning/shared` now includes `role: 'user' | 'admin'`
- ✅ **`/auth/me`**: returns `role` in response
- ✅ **`packages/api/src/middleware/adminGuard.ts`**: returns 403 for non-admin or unauthenticated requests
- ✅ **`packages/api/src/services/AdminService.ts`**: `listSkills`, `createSkill`, `listSkillPaths`, `createSkillPath`, `listLessons` (paginated + filterable), `createLesson`, `updateLesson`, `deleteLesson` (soft delete via `published=false`), `listUsers` (paginated with subscription), `getStats` (MRR stubbed at 0 pending Phase 5)
- ✅ **`packages/api/src/controllers/AdminController.ts`**: HTTP handlers with input validation for all service methods
- ✅ **`packages/api/src/routes/admin.ts`**: all `/api/admin/*` routes protected by `authMiddleware` + `adminGuard`
- ✅ **Seed**: admin user upserted (`admin@learning.app`, role=admin)
- ✅ **`packages/web/app/(admin)/layout.tsx`**: sidebar with Skills, Lessons, Users, Stats nav; redirects non-admins to `/login`
- ✅ **Tests**: 143 API tests (3 new files: `adminGuard.test.ts`, `AdminService.test.ts`, `AdminController.test.ts`), all passing

---

## Chunk 9.2: Lesson Authoring UI

**Deliverable**: Admins can create, edit, and publish lessons through the UI

### 9.2.1 — Lesson List Page

**File**: `packages/web/app/(admin)/lessons/page.tsx`

**Features**:
- Paginated table of all lessons (title, skill, level, published status, created date)
- Search/filter by skill and level
- "New Lesson" button → lesson editor
- Toggle published/unpublished inline

### 9.2.2 — Lesson Editor

**File**: `packages/web/app/(admin)/lessons/[id]/page.tsx` (also handles create via `new`)

**Fields**:
- Title (text input)
- Skill path (dropdown)
- Day number (number input)
- Content (rich text editor — use `react-quill` or `tiptap`)
- Estimated duration (minutes)
- Media URL (optional)
- Published toggle

**Quiz section** (inline on same page):
- Question text
- 4 answer options (A–D)
- Correct answer selector
- Explanation text

**Actions**: Save draft, Publish, Delete

### 9.2.3 — Lesson Schema Update

**Add to `Lesson` model**:
```prisma
published   Boolean  @default(true)
publishedAt DateTime?
```

**Update lesson delivery**: `getTodayLesson()` only serves `published = true` lessons

### ✅ 9.2 Implementation Summary (2026-04-17)

- ✅ **Tiptap**: `@tiptap/react`, `@tiptap/pm`, `@tiptap/starter-kit` installed in `packages/web`
- ✅ **`LessonService.getTodayLesson()`**: filters `p.lesson.published !== false` on incomplete progress records; `findFirst` fallback uses `where: { published: true }`
- ✅ **`packages/web/lib/api-client.ts`**: Added `adminListSkillPaths`, `adminListLessons`, `adminCreateLesson`, `adminUpdateLesson`, `adminDeleteLesson`; exported `AdminLesson`, `AdminLessonsResponse`, `AdminSkillPath`, `AdminCreateLessonInput`, `AdminUpdateLessonInput`
- ✅ **`packages/web/app/(admin)/admin/lessons/page.tsx`**: Paginated table (title, skill, level, day, status, date), skill/level filters, inline publish/unpublish toggle, "New Lesson" button
- ✅ **`packages/web/app/(admin)/admin/lessons/[id]/page.tsx`**: Lesson editor with Tiptap rich text (bold/italic/h2/bullet toolbar), skill path dropdown, day/duration/difficulty fields, media URL, published toggle, quiz section (question, options A–D, correct answer, explanation), Save/Publish/Delete actions; handles create (`/new`) and edit by ID
- ✅ **Route note**: Admin pages placed under `app/(admin)/admin/` (not `app/(admin)/`) to resolve to `/admin/...` URLs without conflicting with `app/(dashboard)/lessons/[id]`
- ✅ **Tests**: 145 API tests (+2 LessonService published-filter tests), 26 web tests (+7 lesson-list +5 lesson-editor), all passing
- ✅ **Build**: `/admin/lessons` (static) and `/admin/lessons/[id]` (dynamic) confirmed clean

---

## Chunk 9.3: Skill Path Management

**Deliverable**: Admins can create and organise skill paths and skills

### 9.3.1 — Skills Page

**File**: `packages/web/app/(admin)/skills/page.tsx`

**Features**:
- List all skills with path counts
- Create new skill (name, description, icon, category)
- Create skill path under a skill (level: beginner/intermediate/advanced)

### 9.3.2 — Lesson Ordering

**File**: `packages/web/app/(admin)/skills/[id]/page.tsx`

**Features**:
- List lessons in a skill path ordered by day
- Drag-to-reorder (update `day` field)
- Add/remove lessons from path

### ✅ 9.3 Implementation Summary (2026-04-17)

- ✅ **`AdminService.listSkills()`**: now includes `_count: { select: { skillPaths: true } }` for path counts
- ✅ **`AdminService.listSkillPaths(skillId?)`**: accepts optional `skillId` filter for per-skill listing
- ✅ **`AdminService.listLessons()`**: added `skillPathId` option — filters by skill path and orders by `day: asc`
- ✅ **`AdminController`**: `listSkillPaths` reads `skillId` query param; `listLessons` reads `skillPathId` query param
- ✅ **`packages/web/lib/api-client.ts`**: Added `adminListSkills`, `adminCreateSkill`, `adminCreateSkillPath`, `adminListLessonsByPath`; updated `adminListSkillPaths(skillId?)` to accept optional filter; exported `AdminSkill`, `AdminCreateSkillInput`, `AdminCreateSkillPathInput`
- ✅ **`packages/web/app/(admin)/admin/skills/page.tsx`**: Lists all skills with path counts and categories; inline "New Skill" form (name, description, category); inline "New Path" form (skill dropdown, level dropdown, durationHours)
- ✅ **`packages/web/app/(admin)/admin/skills/[id]/page.tsx`**: Lists lessons for a skill path ordered by day; drag-to-reorder via HTML5 drag API + move-up/move-down buttons; "Save Order" patches changed `day` values; "Remove" soft-deletes; "Add Lesson" links to `/admin/lessons/new`
- ✅ **Tests**: 147 API tests (+2 AdminService tests), 40 web tests (+7 skills page +7 skill-path detail), all passing
- ✅ **Build**: `/admin/skills` (static) and `/admin/skills/[id]` (dynamic) confirmed clean

---

## Chunk 9.4: Platform Stats Dashboard

**Deliverable**: Admin can see key platform metrics at a glance

### 9.4.1 — Stats API

**File**: `packages/api/src/services/AdminStatsService.ts`

**Methods**:
- `getTotalUsers()` — User count
- `getDAU()` — Users who completed a lesson today
- `getActiveSubscribers()` — Count by tier (free, starter, pro, premium)
- `getLessonCompletionRate()` — Completions / lesson serves (last 7 days)
- `getMRR()` — Sum of active subscription MRR from Stripe

### 9.4.2 — Stats Page

**File**: `packages/web/app/(admin)/stats/page.tsx`

**Display**: Cards for total users, DAU, MRR, lesson completion rate

**Tests**: Unit test AdminStatsService queries; test admin guard blocks non-admin requests

### ✅ 9.4 Implementation Summary (2026-04-17)

- ✅ **`AdminService.getStats()`**: added `lessonCompletionRate` — computes completions in last 7 days divided by lesson starts in last 7 days; returns 0 when no starts
- ✅ **`api-client.ts`**: Added `adminGetStats()` method (`GET /admin/stats`); exported `AdminStats` interface (`totalUsers`, `dau`, `activeSubscribers`, `mrr`, `lessonCompletionRate`)
- ✅ **`packages/web/app/(admin)/admin/stats/page.tsx`**: Five stat cards — Total Users, Daily Active Users, Active Subscribers, Lesson Completion Rate (last 7 days, shown as %), MRR ($0 with "pending Stripe integration" note); loading and error states
- ✅ **Tests**: 148 API tests (was 147 + 1 new `getStats` lessonCompletionRate test), 47 web tests (was 40 + 7 stats page tests), all passing
- ✅ **Build**: `/admin/stats` (static) confirmed clean

---

## Next Phase

[Phase 10: Enterprise & Team Features](./phase-10-enterprise-team.md)
