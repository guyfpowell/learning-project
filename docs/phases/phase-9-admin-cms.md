# Phase 9: Admin CMS

**Status**: ⚪ NOT STARTED

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

---

## Next Phase

[Phase 10: Enterprise & Team Features](./phase-10-enterprise-team.md)
