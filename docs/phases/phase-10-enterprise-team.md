# Phase 10: Enterprise & Team Features

**Status**: 🔄 IN PROGRESS (Chunks 10.1, 10.3, 10.4 complete; 10.2 deferred to after Phase 5)

**Goal**: Enable B2B sales by supporting team accounts, team-level analytics, and org admin capabilities. Target: small PM teams (5–20 people) at $499/month.

---

## Chunk 10.1: Team Account Model

**Deliverable**: Users can belong to a team; team admins can manage members

### 10.1.1 — Schema

**Add to `prisma/schema.prisma`**:

```prisma
model Team {
  id          String       @id @default(cuid())
  name        String
  slug        String       @unique
  ownerId     String
  createdAt   DateTime     @default(now())
  owner       User         @relation("TeamOwner", fields: [ownerId], references: [id])
  members     TeamMember[]
  subscription TeamSubscription?
}

model TeamMember {
  id        String   @id @default(cuid())
  teamId    String
  userId    String
  role      TeamRole @default(member)
  joinedAt  DateTime @default(now())
  team      Team     @relation(fields: [teamId], references: [id], onDelete: Cascade)
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([teamId, userId])
}

enum TeamRole {
  admin
  member
}

model TeamSubscription {
  id                   String   @id @default(cuid())
  teamId               String   @unique
  stripeCustomerId     String
  stripeSubscriptionId String
  seatLimit            Int      @default(20)
  status               String   @default("active")
  team                 Team     @relation(fields: [teamId], references: [id], onDelete: Cascade)
}
```

**Update `User`**: Add `teamMembers TeamMember[]`

### 10.1.2 — Team API Routes

**Base path**: `/api/teams`

**Endpoints**:
- `POST /api/teams` — Create team (owner becomes first admin)
- `GET /api/teams/:id` — Get team details + members
- `POST /api/teams/:id/invite` — Send invite email to new member
- `POST /api/teams/:id/accept-invite` — Accept invite (token-based)
- `DELETE /api/teams/:id/members/:userId` — Remove member (team admin only)
- `GET /api/teams/:id/subscription` — Team billing info

### 10.1.3 — Invite Flow

**Invite token**: JWT with `{ teamId, email, expiresIn: '7d' }`

**Email**: Send invite via SendGrid — "You've been invited to join [Team] on Learning App"

**Accept**: Validate token, add user to team (create user account if email not registered)

---

## Chunk 10.2: Team Stripe Billing

**Deliverable**: Teams subscribe to the Team plan; seat usage enforced

### 10.2.1 — Team Plan

**Stripe product**: Team plan at $499/month (up to 20 seats)

**Webhook events to handle**:
- Same as individual subscriptions (completed, updated, deleted, payment failed)
- But updates `TeamSubscription` not `Subscription`

### 10.2.2 — Seat Enforcement

- On member join: check `TeamSubscription.seatLimit` vs. current `TeamMember` count
- If at limit: reject with 402 + "seat limit reached, contact admin to upgrade"

### 10.2.3 — Lesson Access via Team Membership

- If user has no individual subscription but has active `TeamMembership`: grant Starter-tier access
- Team subscription overrides individual (team members get full lesson access)

---

## Chunk 10.3: Team Analytics Dashboard

**Deliverable**: Team admins can track their team's learning progress

### 10.3.1 — Team Stats API

**File**: `packages/api/src/services/TeamAnalyticsService.ts`

**Methods**:
- `getTeamSummary(teamId)` — Total completions, avg streak, avg quiz score for team
- `getMemberProgress(teamId)` — Per-member: lessons completed, streak, current skill, last active
- `getSkillGapAnalysis(teamId)` — Which skills have lowest avg scores across team members
- `getTeamLeaderboard(teamId)` — Ranked by streak and lessons completed

**Routes**:
- `GET /api/teams/:id/analytics` — Summary stats
- `GET /api/teams/:id/members/progress` — Member breakdown
- `GET /api/teams/:id/skill-gaps` — Skill gap analysis

### 10.3.2 — Team Dashboard Page

**File**: `packages/web/app/(dashboard)/team/page.tsx`

**Visible to**: Team admins only

**Sections**:
- Team summary cards (avg streak, total completions, avg score)
- Member progress table (name, streak, lessons done, last active, skill)
- Skill gap heatmap (skills vs. team avg score)
- Leaderboard (top 5 by streak)

---

## Chunk 10.4: Custom Skill Paths (Enterprise)

**Deliverable**: Enterprise clients can have custom skill tracks tailored to their org

### 10.4.1 — Custom Path Assignment

**Schema**: Add `teamId` to `SkillPath` model (nullable — null = global, set = team-specific)

**Admin route**: `POST /api/admin/skill-paths` — allow setting `teamId` for team-specific path creation

### 10.4.2 — Path Visibility Rules

- `teamId = null`: visible to all users (global paths)
- `teamId = X`: only visible to members of team X

**Update**: `GET /api/skills` — filter paths by user's team membership

### 10.4.3 — Custom Content Seeding

**Admin UI** (Phase 9): When creating a lesson/path, optionally scope to a team

**Tests**: Unit test team analytics queries; test seat enforcement; test invite token validation; test path visibility filtering

---

---

## ✅ Implementation Summary (2026-04-30)

**Chunks completed**: 10.1, 10.3, 10.4 (Chunk 10.2 deferred — requires Phase 5 Stripe)

### Chunk 10.1 — Team Account Model
- **Schema**: `TeamRole` enum, `Team`, `TeamMember`, `TeamSubscription` models added; `User.teamMembers` + `User.ownedTeams` relations added; `db:push` applied
- **`TeamService`**: `createTeam` (auto-generates slug, adds owner as admin member), `getTeam` (member-only access), `inviteMember` (JWT invite token, email stubbed), `acceptInvite` (token validation + email match check), `removeMember` (admin-only, blocks owner removal), `getTeamSubscription`
- **`TeamController`**: wraps all service methods with validation and HTTP responses
- **`routes/teams.ts`**: `POST /api/teams`, `GET /api/teams/:id`, `POST /api/teams/:id/invite`, `POST /api/teams/:id/accept-invite`, `DELETE /api/teams/:id/members/:userId`, `GET /api/teams/:id/subscription` — all protected by `authMiddleware`
- **Tests**: 17 TeamService + 12 TeamController = 29 new API tests, all passing

### Chunk 10.3 — Team Analytics Dashboard
- **`TeamAnalyticsService`**: `getTeamSummary` (completions, avg score, avg streak, member count), `getMemberProgress` (per-member breakdown: lessons, score, streak, lastActive, currentSkill), `getSkillGapAnalysis` (skills ranked by avg score ascending), `getTeamLeaderboard` (top 10 by streak then lessons)
- **Analytics routes** added to `routes/teams.ts`: `GET /api/teams/:id/analytics`, `GET /api/teams/:id/members/progress`, `GET /api/teams/:id/skill-gaps`, `GET /api/teams/:id/leaderboard`
- **`packages/web/app/(dashboard)/team/page.tsx`**: summary stat cards, member progress table (name, streak, lessons, score, current skill), leaderboard (rank badges, streak), skill gap heatmap (coloured progress bars)
- **Team nav link** added to dashboard layout sidebar
- **Team API client methods + types** added to `api-client.ts`: `createTeam`, `getTeam`, `inviteTeamMember`, `acceptTeamInvite`, `removeTeamMember`, `getTeamAnalytics`, `getTeamMemberProgress`, `getTeamSkillGaps`, `getTeamLeaderboard`; exported `Team`, `TeamMember`, `TeamSummary`, `MemberProgress`, `SkillGap`, `LeaderboardEntry` interfaces
- **Tests**: 7 TeamAnalyticsService + 7 web TeamPage = 14 new tests, all passing

### Chunk 10.4 — Custom Skill Paths
- **Schema**: `teamId String?` added to `SkillPath` (nullable, indexed); `db:push` applied
- **`GET /api/lessons/skills`**: new endpoint — fetches skills with paths filtered by team visibility (`teamId = null` OR `teamId in user's teams`); user's team memberships resolved from JWT
- **`AdminService.CreateSkillPathInput`**: `teamId?: string` field added so admin can scope new paths to a team
- **Tests**: all 184 API + 54 web tests passing

### Test Counts
| Package | Before | After | New |
|---------|--------|-------|-----|
| API | 148 | 184 | +36 |
| Web | 47 | 54 | +7 |

## Next Phase

[Phase 11: Security, Analytics & Observability](./phase-11-security-analytics.md)
