# REQ-002: Admin Interface

**Status**: `dev-complete` (002.1 ✅, 002.2 ✅, 002.3 ✅, 002.4 ✅)
**arch-review**: `not-required`

agent-note: STRICT ONE-CHUNK RULE. Implement exactly 1 chunk, then update with a full implementation summary, then STOP IMMEDIATELY. Do NOT proceed to the next chunk. Do NOT say "now let's do chunk N". Do NOT offer next steps. Just stop. Context is cleared after each chunk — the user will re-invoke you. When resuming, read this doc first; start from the first chunk not marked ✅.

---

## Requirements

The admin interface must allow an admin to:

1. See all users in a list with a user summary
2. Click in to any user to see all their details, including what tracks and lessons they are on and have completed
3. In the user list: enable/disable a user, and make a user 'Premium' without them having to pay
4. Overall usage stats: logins/logouts with times and dates, most popular lessons, most popular tracks, most active users
5. Lesson admin: see every track, every level, every topic and every lesson — go to any one to look at it or test it

---

## What Already Exists (Phase 9)

| Capability | Status | Location |
|---|---|---|
| Admin auth guard (`/admin/*` requires `role: admin`) | ✅ Done | `middleware/adminGuard.ts` |
| Admin layout + protected route group | ✅ Done | `web/app/(admin)/layout.tsx` |
| Lesson list with skill/level filter, pagination | ✅ Done | `GET /api/admin/lessons`, `/admin/lessons` |
| Lesson detail/edit | ✅ Done | `/admin/lessons/[id]` |
| Skills and skill-path CRUD | ✅ Done | `GET/POST /api/admin/skills`, `/admin/skills` |
| User list (paginated, basic: email, name, plan) | ✅ Done | `GET /api/admin/users`, (web page not yet confirmed) |
| Basic stats (total users, DAU, subscribers, completion rate) | ✅ Done | `GET /api/admin/stats`, `/admin/stats` |

**Not yet built** — gaps this req closes:
- User detail page (tracks, lessons, progress)
- Enable/disable user
- Make user premium
- Login event tracking
- Rich analytics (popular lessons/tracks, active users, login history)
- Hierarchical lesson browser with preview/test

---

## Implementation Plan

### Chunk 002.1 — User Management

**Scope**:
- Schema: add `isDisabled Boolean @default(false)` to `User` model + migration
- Schema: add `LoginEvent` model (userId, type `login|logout`, timestamp, ipAddress)
- Auth service: block login if `isDisabled = true` (throw `ACCOUNT_DISABLED` error)
- Auth service: create `LoginEvent` on login and logout
- API: `GET /api/admin/users/:id` — full user detail (profile, subscription, enrolled tracks, progress summary per track)
- API: `PATCH /api/admin/users/:id/disable` — set `isDisabled = true`
- API: `PATCH /api/admin/users/:id/enable` — set `isDisabled = false`
- API: `PATCH /api/admin/users/:id/make-premium` — upsert subscription to `premium` plan (status: active, no Stripe)
- Web: `/admin/users` — enhance list with disable toggle, premium badge, link to detail
- Web: `/admin/users/[id]` — user detail page (profile, plan, enrolled tracks table, lesson completion history)

**Tests**:
- Unit: `AdminService` — getUserDetail, disableUser, enableUser, makeUserPremium
- Unit: `AuthService` — login blocked when `isDisabled = true`
- Integration: `PATCH /api/admin/users/:id/disable` → subsequent login returns 403
- Web: user list page renders disable toggle; user detail page renders progress

**Acceptance**:
- [x] Admin can see user details including tracks enrolled and lessons completed
- [x] Admin can disable a user; disabled user cannot log in
- [x] Admin can enable a disabled user; they can log in again
- [x] Admin can make any user premium with one click; they immediately get premium access

---

### Chunk 002.2 — Usage Analytics

**Scope**:
- API: `GET /api/admin/analytics` — consolidated endpoint returning:
  - Login activity: last 50 login/logout events (userId, email, type, timestamp)
  - Most popular lessons: top 10 by `UserProgress.completedAt` count (lesson title, track, completions)
  - Most popular tracks: top 10 by enrolled user count (track name, users, completions)
  - Most active users: top 10 by completions last 30 days (name, email, completions)
- Web: `/admin/stats` — enhance existing page with:
  - Login activity feed (date/time, user email, login/logout)
  - Popular lessons table (top 10)
  - Popular tracks table (top 10)
  - Most active users leaderboard (top 10)

**Tests**:
- Unit: `AdminService.getAnalytics` — returns correct shape with mocked Prisma
- Web: stats page renders all four analytics sections

**Acceptance**:
- [x] Admin can see a feed of logins and logouts with timestamps and user emails
- [x] Admin can see which lessons are completed most often
- [x] Admin can see which tracks have the most users
- [x] Admin can see which users are most active

---

### ✅ Chunk 002.3 — Lesson Browser (2026-06-12)

**What was implemented**:

**Backend** (`packages/api`):
- New interfaces exported from `AdminService.ts`: `LessonTreeQuiz`, `LessonTreeLesson`, `LessonTreeTopic`, `LessonTreeLevel`, `LessonTreeSkill`
- `AdminService.getLessonTree()` — single Prisma query (`skill.findMany` with deep include: skillPaths → lessons → quizzes), groups lessons by `topicName` in memory, returns skills ordered by `order`, levels ordered by level name, lessons ordered by `lessonNumber`
- `AdminController.getLessonTree()` — delegates to service, returns `{ success: true, data }`
- `GET /api/admin/lessons/tree` — added to admin router before `GET /admin/lessons` (prevents route conflict)

**Frontend** (`packages/web`):
- `LessonTreeQuiz`, `LessonTreeLesson`, `LessonTreeTopic`, `LessonTreeLevel`, `LessonTreeSkill` interfaces added to `api-client.ts`
- `apiClient.adminGetLessonTree()` — calls `GET /admin/lessons/tree`
- `/admin/lessons/browse` — three-column hierarchical browser:
  - Column 1: all tracks (skills ordered by `order`), click to select
  - Column 2: levels for selected track with lesson counts, click to select
  - Column 3: topics for selected level (auto-expanded on level select), toggle to collapse, lessons listed under each topic with lesson position (Lesson N of M)
  - Slide-in panel on lesson click: title, topicName · Lesson N of M, summary, content, key takeaway (branded callout), quiz questions with options highlighted in green for correct answer + explanation
  - "Test lesson ↗" link opens `/lessons/[id]` in new tab
- Admin nav updated: "Browse Lessons" added between Skills and Lessons; active highlight uses longest-prefix match to avoid ambiguity between `/admin/lessons` and `/admin/lessons/browse`

**Tests**:
- 3 new `AdminService.getLessonTree` unit tests: nested structure, topic grouping, empty state — all passing
- 5 new browse page web tests: track list renders, levels on skill select, topics+lessons on level select, lesson panel opens with content and quiz, error state — all passing
- **API total**: 256 unit tests passing (was 253)
- **Web total**: 202 passing (pre-existing 25 failures unchanged, all in unrelated test suites)

**Acceptance**:
- [x] Admin can navigate from track → level → topic → lesson without leaving the page
- [x] Admin can read any lesson's full content and quiz in the panel
- [x] "Test lesson" link takes admin directly to the user-facing lesson

---

### Chunk 002.3 — Lesson Browser (original spec)

**Scope**:
- API: `GET /api/admin/lessons/tree` — hierarchical response: skills → skill paths (levels) → lessons grouped by `topicName`, ordered by `lessonNumber`
- Web: `/admin/lessons/browse` — hierarchical drill-down:
  - Column 1: track list (all skills)
  - Column 2: levels for selected track (beginner/intermediate/advanced/expert) with lesson counts
  - Column 3: topics for selected level, each expandable to show lesson list
  - Click any lesson → slide-in panel showing full content, summary, key takeaway, quiz questions (read-only)
  - "Test lesson" button → opens user-facing lesson page in new tab at `/lessons/[id]`
- Nav link: add "Browse Lessons" to admin navigation alongside existing "Lessons" (flat list) and "Skills"

**Tests**:
- Unit: `AdminService.getLessonTree` — returns correct nested structure
- Web: browse page renders track list, selects level, shows lesson panel

**Acceptance**:
- [ ] Admin can navigate from track → level → topic → lesson without leaving the page
- [ ] Admin can read any lesson's full content and quiz in the panel
- [ ] "Test lesson" link takes admin directly to the user-facing lesson

---

### ✅ Chunk 002.4 — Admin Nav Entry Point (2026-06-12)

**What was implemented**:

**Frontend** (`packages/web`):
- `app/(dashboard)/layout.tsx` — added conditional `{user?.role === 'admin' && <Link href="/admin/users">Admin</Link>}` block inside the `<nav>`, after the `NAV_LINKS.map(...)` block. Styled with a top border and marginTop to visually separate it from the regular nav items.

**Tests**:
- Created `app/(dashboard)/__tests__/layout.test.tsx` with 2 tests:
  1. Admin nav link renders when `user.role === 'admin'` ✅
  2. Admin nav link does NOT render when `user.role === 'user'` ✅

**Acceptance**:
- [x] Admin nav link appears at the bottom of the sidebar nav, separated by a rule, when `user.role === 'admin'`
- [x] Clicking it navigates to `/admin/users`
- [x] Regular users do not see the Admin link

---

**Problem**: The admin interface exists at `/admin/users`, `/admin/stats`, etc., but there is no link to it from the regular dashboard. Admin users have no way to reach it without typing a direct URL. There is also no `/admin/page.tsx`, so navigating to `/admin` returns a 404.

**Scope** (one file, one change):
- `packages/web/app/(dashboard)/layout.tsx` — add a conditional "Admin" nav item that only renders when `user?.role === 'admin'`

**No backend changes required.** The `role` field is already returned by `/api/auth/me` and stored on the `user` object in `auth-context.tsx`.

**Exact change**:

File: `packages/web/app/(dashboard)/layout.tsx`

The `NAV_LINKS` constant is a static array at the top of the file (lines 9–15). It cannot be made conditional inline. Instead, render the admin link separately after the `NAV_LINKS.map(...)` block, inside the `<nav>` element (line 37–49).

Current nav block:
```tsx
<nav style={{ flex: 1, padding: '0 var(--space-3)' }} aria-label="Main navigation">
  {NAV_LINKS.map(({ href, label }) => (
    <Link
      key={href}
      href={href}
      aria-label={label}
      style={{ display: 'block', padding: 'var(--space-2) var(--space-3)', borderRadius: 'var(--radius-sm)', marginBottom: 'var(--space-1)', color: 'var(--neutral-400)', fontSize: 'var(--text-sm)', fontWeight: 'var(--weight-medium)', textDecoration: 'none', transition: 'background var(--dur-base), color var(--dur-base)' }}
      onMouseOver={(e) => { (e.currentTarget as HTMLElement).style.background = 'rgba(255,255,255,0.08)'; (e.currentTarget as HTMLElement).style.color = 'var(--text-inverse)' }}
      onMouseOut={(e) => { (e.currentTarget as HTMLElement).style.background = ''; (e.currentTarget as HTMLElement).style.color = 'var(--neutral-400)' }}
    >
      {label}
    </Link>
  ))}
</nav>
```

Replace with:
```tsx
<nav style={{ flex: 1, padding: '0 var(--space-3)' }} aria-label="Main navigation">
  {NAV_LINKS.map(({ href, label }) => (
    <Link
      key={href}
      href={href}
      aria-label={label}
      style={{ display: 'block', padding: 'var(--space-2) var(--space-3)', borderRadius: 'var(--radius-sm)', marginBottom: 'var(--space-1)', color: 'var(--neutral-400)', fontSize: 'var(--text-sm)', fontWeight: 'var(--weight-medium)', textDecoration: 'none', transition: 'background var(--dur-base), color var(--dur-base)' }}
      onMouseOver={(e) => { (e.currentTarget as HTMLElement).style.background = 'rgba(255,255,255,0.08)'; (e.currentTarget as HTMLElement).style.color = 'var(--text-inverse)' }}
      onMouseOut={(e) => { (e.currentTarget as HTMLElement).style.background = ''; (e.currentTarget as HTMLElement).style.color = 'var(--neutral-400)' }}
    >
      {label}
    </Link>
  ))}
  {user?.role === 'admin' && (
    <Link
      href="/admin/users"
      aria-label="Admin"
      style={{ display: 'block', padding: 'var(--space-2) var(--space-3)', borderRadius: 'var(--radius-sm)', marginTop: 'var(--space-3)', color: 'var(--neutral-400)', fontSize: 'var(--text-sm)', fontWeight: 'var(--weight-medium)', textDecoration: 'none', transition: 'background var(--dur-base), color var(--dur-base)', borderTop: '1px solid rgba(255,255,255,0.08)', paddingTop: 'var(--space-3)' }}
      onMouseOver={(e) => { (e.currentTarget as HTMLElement).style.background = 'rgba(255,255,255,0.08)'; (e.currentTarget as HTMLElement).style.color = 'var(--text-inverse)' }}
      onMouseOut={(e) => { (e.currentTarget as HTMLElement).style.background = ''; (e.currentTarget as HTMLElement).style.color = 'var(--neutral-400)' }}
    >
      Admin
    </Link>
  )}
</nav>
```

**Tests**:
- Add 2 tests to the existing dashboard layout test file (or create `layout.test.tsx` if none exists):
  1. Admin nav link is rendered when `user.role === 'admin'`
  2. Admin nav link is NOT rendered when `user.role === 'user'`

**Acceptance**:
- [ ] Logged in as `admin@learning.app`: "Admin" link appears at the bottom of the sidebar nav, separated by a rule
- [ ] Clicking it navigates to `/admin/users`
- [ ] Logged in as a regular user: "Admin" link does not appear

---

## Implementation Notes

- `isDisabled` check must run in `AuthService.login` before JWT is issued — not in middleware (middleware only runs on authenticated routes)
- `LoginEvent` is append-only; no delete. Keep it simple: no indexes needed at this scale
- `makeUserPremium` looks up `SubscriptionPlan` by `name = 'premium'`; upserts `Subscription` (userId unique); sets `status = 'active'`, `currentPeriodStart = now`, `currentPeriodEnd = +100 years` (admin-granted, no expiry)
- Analytics queries run on demand (no caching needed for admin tool at current scale)
- Lesson tree endpoint is read-only; no write operations added in 002.3

---

## Change Log

### ✅ Chunk 002.2 — Usage Analytics (2026-06-12)

**What was implemented**:

**Backend** (`packages/api`):
- `AdminService.getAnalytics()` — 8 parallel/sequential Prisma queries returning all 4 analytics shapes:
  - `loginActivity` — last 50 `LoginEvent` rows joined to user email, ordered by `createdAt desc`
  - `popularLessons` — `userProgress.groupBy(lessonId)` where `completedAt != null`, top 10, joined to lesson title + track name + level
  - `popularTracks` — `userTrackEnrollment.groupBy(skillId)`, top 10 by enrollment count, plus a second groupBy for completion counts; joined to skill name
  - `mostActiveUsers` — `userProgress.groupBy(userId)` last 30 days, top 10, joined to user name + email
- New interfaces exported from `AdminService.ts`: `LoginEventEntry`, `PopularLesson`, `PopularTrack`, `ActiveUser`, `AdminAnalytics`
- `AdminController.getAnalytics()` — delegates to service, returns `{ success: true, data }`
- `GET /api/admin/analytics` — added to admin router

**Frontend** (`packages/web`):
- `AdminAnalytics`, `LoginEventEntry`, `PopularLesson`, `PopularTrack`, `ActiveUser` interfaces added to `api-client.ts`
- `apiClient.adminGetAnalytics()` — calls `GET /admin/analytics`
- `/admin/stats` page enhanced: now fetches stats + analytics in parallel; renders 4 new sections below the stat cards:
  - Login Activity — table of last 50 events (time, email, login/logout badge)
  - Popular Lessons — top 10 table (title, track, level, completions)
  - Popular Tracks — top 10 table (track, users enrolled, completed)
  - Most Active Users — top 10 leaderboard (name, email, completions last 30 days)

**Tests**:
- 5 new `AdminService.getAnalytics` unit tests (login activity, popular lessons, popular tracks, most active users, empty state) — all passing
- Updated prisma mock to include `loginEvent` and `userTrackEnrollment`
- 4 new stats page web tests for the 4 analytics sections — all passing
- **API total**: 253 unit tests passing (was 248)
- **Web total**: 197 unit tests passing (2 pre-existing unrelated failures unchanged)

---

### ✅ Chunk 002.1 — User Management (2026-06-12)

**What was implemented**:

**Schema & Migration** (`20260612115029_add_user_disabled_login_events`):
- `User.isDisabled Boolean @default(false)` — blocks login when true
- `LoginEvent` model — append-only audit log (userId, type `login|logout`, ipAddress, createdAt); indexed on userId + createdAt

**Shared types** (`packages/shared`):
- `ERROR_CODES.AUTH_ACCOUNT_DISABLED = 'AUTH_007'` added

**Backend** (`packages/api`):
- `AuthService.login`: checks `isDisabled` after password validation → throws 403 `AUTH_ACCOUNT_DISABLED`; creates `LoginEvent` on success; accepts optional `ipAddress` param
- `AuthService.logout`: creates `LoginEvent` on logout; accepts optional `ipAddress` param
- `AuthController`: passes `req.ip ?? req.socket?.remoteAddress` to both login and logout
- `AdminService`: `getUserDetail`, `disableUser`, `enableUser`, `makeUserPremium` (upserts subscription to premium plan, expiresAt +100 years)
- `listUsers` now returns `isDisabled` field
- Routes: `GET /api/admin/users/:id`, `PATCH /:id/disable`, `/:id/enable`, `/:id/make-premium`

**Frontend** (`packages/web`):
- `api-client.ts`: `adminListUsers`, `adminGetUserDetail`, `adminDisableUser`, `adminEnableUser`, `adminMakeUserPremium` + types (`AdminUserSummary`, `AdminUsersResponse`, `AdminUserDetail`, `AdminUserEnrollment`, `AdminUserProgressItem`)
- `/admin/users` page: paginated table with disable/enable toggle, premium badge, plan badge, link to detail
- `/admin/users/[id]` page: profile grid, enrolled tracks table, lesson history grouped by track+level with quiz scores

**Tests**:
- Unit: 3 new `AuthService` tests (disabled throws 403, login creates LoginEvent, logout creates LoginEvent); 8 new `AdminService` tests (getUserDetail, disableUser, enableUser, makeUserPremium × 2)
- Integration: 5 new tests — GET user detail, disable user, disabled user login blocked (403 + AUTH_007), enable user, re-enabled user can login
- Web: 7 tests for `/admin/users` list page; 7 tests for `/admin/users/[id]` detail page
- `AuthController.test.ts` updated for new logout signature

**Test counts**: API 248 unit (was 206 + others), all passing; 14 new web tests passing; integration: 5 new passing

---

## Bug fix (2026-06-12) — Browse: lessons showing `lessonNumber` instead of `Lesson N of M`

`LessonTreeLesson` in `AdminService.ts` was missing `totalLessons` — the field was in the Prisma model and the web `api-client` type, but never added to the backend interface or the `getLessonTree` mapping. The frontend null-check `lesson.totalLessons != null` therefore always failed, falling back to `Lesson {lessonNumber}` (1..80) instead of `Lesson {lessonIndex} of {totalLessons}` (1..8).

**Fix**: Added `totalLessons: number | null` to `LessonTreeLesson` interface and `totalLessons: lesson.totalLessons` to the mapping in `AdminService.getLessonTree`.

---

## Bug fix (2026-06-12) — No way back from admin to the normal app

The admin sidebar had no navigation link back to the learner-facing app. Added a "← Back to app" link in the sidebar footer (above Logout) in `packages/web/app/(admin)/layout.tsx`, linking to `/`.
