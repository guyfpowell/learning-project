---
title: FE Agent Context — Learning App Web Frontend
last_updated: 2026-04-12
---

# Learning App — Frontend Context

## Tech Stack

**Framework**: Next.js 14 (App Router)
**React**: 18.2.0 with TypeScript 5.3.3 (strict mode)
**Styling**: TailwindCSS 3.4.1 + Lucide icons
**State Management**: React Context API (useAuth hook)
**HTTP Client**: Custom fetch wrapper (apiClient)
**Testing**: Jest 29 + React Testing Library 14
**Package Manager**: pnpm workspaces

---

## Project Structure

```
packages/web/
├── app/
│   ├── layout.tsx              # Root layout with AuthProvider
│   ├── page.tsx                # Landing page (redirects to dashboard if auth)
│   ├── globals.css             # TailwindCSS imports
│   ├── (auth)/                 # Route group: URLs /login, /signup, /onboarding
│   │   ├── layout.tsx          # Centered form layout
│   │   ├── login/page.tsx      # Email + password login
│   │   ├── signup/page.tsx     # Registration with password confirmation
│   │   └── onboarding/page.tsx # Skill selection + preferences
│   └── (dashboard)/            # Protected route group
│       ├── layout.tsx          # Sidebar + header layout
│       ├── page.tsx            # Dashboard home (streak + today's lesson)
│       ├── progress.tsx        # Stats + completed lessons
│       ├── settings.tsx        # Profile + notification preferences
│       └── lessons/
│           └── [id]/
│               ├── page.tsx    # Lesson detail + content
│               └── quiz.tsx    # Quiz questions + results
├── lib/
│   ├── auth-context.tsx        # Auth state provider + useAuth hook
│   ├── api-client.ts           # Fetch wrapper with JWT injection
│   ├── protected-route.tsx     # HOC for route protection
│   └── __tests__/              # Auth + API client tests
├── components/
│   ├── common/
│   │   └── LoadingSpinner.tsx  # Reusable loading spinner
│   ├── forms/                  # (placeholder for form components)
│   ├── lesson/                 # (placeholder for lesson components)
│   └── ...                     # Other UI components
└── public/
    └── assets/                 # Static assets
```

---

## Key Implementation Patterns

### 1. Authentication Flow

**Token Storage**: localStorage key `auth_token`

**Auth Context**:
```typescript
interface AuthContextType {
  user: User | null
  isAuthenticated: boolean
  loading: boolean
  login(email, password): Promise<void>
  register(email, password, name): Promise<void>
  logout(): void
}
```

**Auto-login on Mount**: 
- Checks localStorage for token
- Calls GET `/api/auth/me` to validate
- Sets user state on success, clears token on 401

**Usage in Components**:
```typescript
const { user, isAuthenticated, loading, login, register, logout } = useAuth()
```

### 2. API Client

**Base URL**: `process.env.NEXT_PUBLIC_API_URL` (default: `http://localhost:3000/api`)

**JWT Injection**: Auto-includes `Authorization: Bearer <token>` on authenticated requests

**Error Handling**: Throws Error with message from response (can be caught in try/catch)

**Available Methods**:
- `register(email, password, name)` → AuthResponse
- `login(email, password)` → AuthResponse
- `getMe()` → User
- `getSkills()` → Skill[]
- `getTodayLesson()` → Lesson
- `getLesson(id)` → Lesson
- `completeLesson(id)` → UserProgress
- `submitQuiz(lessonId, quizId, answer)` → QuizSubmitResponse
- `getProgress()` → ProgressResponse
- `updateProfile(profile)` → User
- `getNotificationPreferences()` → NotificationPreference
- `updateNotificationPreferences(prefs)` → NotificationPreference

### 3. Protected Routes

**Pattern**: Wrap dashboard pages with `<ProtectedRoute>` HOC

**Behavior**:
- Shows loading spinner while auth check completes
- Redirects unauthenticated users to `/login`
- All pages under `(dashboard)/` must use this

### 4. Form Validation

**Client-side Only**:
- Email: format check (regex)
- Password: minimum 8 characters
- Confirmations: field matching

**Validation Timing**: On submit, before API call

**Error Display**: Show in red box above form

### 5. Styling Convention

**CSS Framework**: TailwindCSS (no inline styles)

**Responsive Breakpoints**: Default mobile-first (375px → 768px → 1440px)

**Color Scheme**: 
- Primary: Blue-600 (buttons, links)
- Success: Green-600
- Error: Red-600
- Text: Gray-900 (dark), Gray-600 (muted)
- Backgrounds: Gray-50 or white

**Common Patterns**:
- Cards: `bg-white rounded-lg shadow-md p-6`
- Buttons: `px-6 py-2/3 bg-blue-600 text-white font-semibold rounded-lg hover:bg-blue-700 transition`
- Inputs: `px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500`
- Spinners: `animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600`

---

## Data Models & API Contracts

### User
```typescript
{
  id: string
  email: string
  name: string
  profile?: {
    id: string
    userId: string
    goal?: string
    preferredTime?: 'morning' | 'afternoon' | 'evening'
    timezone?: string
    learningStyle?: 'visual' | 'text' | 'mixed'
  }
  subscriptions?: Subscription[]
}
```

### Lesson
```typescript
{
  id: string
  skillPathId: string
  day: number
  title: string
  content: string (markdown/html)
  durationMinutes: number
  mediaUrl?: string
  difficulty: 'beginner' | 'intermediate' | 'advanced'
  quizzes: Quiz[]
}
```

### Quiz
```typescript
{
  id: string
  lessonId: string
  type: 'multiple-choice' | 'short-answer'
  question: string
  options: string[]
  correctAnswer: string
  explanation: string
}
```

### All API Responses Follow Envelope
```json
{
  "success": boolean,
  "data": <any>,
  "error"?: string,
  "code"?: string,
  "timestamp": ISO8601
}
```

---

## Environment Configuration

**File**: `packages/web/.env.local` (copied from `.env.example`)

**Required Variables**:
```
NEXT_PUBLIC_API_URL=http://localhost:3000/api
```

**Usage in Code**: `process.env.NEXT_PUBLIC_API_URL`

---

## Development Commands

```bash
# Install deps (from packages/web)
pnpm install

# Start dev server (runs on http://localhost:3001)
pnpm dev

# Build for production
pnpm build

# Run production server
pnpm start

# Run tests
pnpm test

# Watch tests
pnpm test:watch

# Test coverage
pnpm test:coverage
```

---

## Testing Strategy

**Framework**: Jest + React Testing Library

**Coverage Target**: 80%+ on critical paths (auth, api-client, key pages)

**Test Location**: `__tests__/` folders next to files being tested

**No Snapshots**: Test behavior, not implementation

**Mock Strategy**:
- Mock all API calls (using jest.mock on fetch)
- Test error paths: invalid input, network errors, 401 responses
- Mock real response shapes from backend

---

## Type Safety

**Strict Mode**: Yes, all TS files compiled with `strict: true`

**Type Imports**: Import from `@learning/shared`:
```typescript
import { User, Lesson, Skill, UserProgress } from '@learning/shared'
```

**No `any` Type**: All parameters and returns explicitly typed

---

## Known Gotchas

1. Route groups don't appear in URL: `(auth)/login/page.tsx` → `/login`
2. `useAuth` must be in client components (mark with `'use client'`)
3. Protected routes check auth at render time
4. Token required for authenticated endpoints, not for `/lessons/skills`
5. Form validation is client-side only
6. Always show loading states (spinner + disabled buttons)
7. localStorage not available server-side
8. Dynamic routes use square brackets: `[id]/page.tsx`

---

## Build Output

**Completed** (2026-04-12):
- ✅ Config files (tsconfig, next.config, tailwind, jest, postcss)
- ✅ Auth system (context + useAuth hook)
- ✅ API client (typed fetch wrapper)
- ✅ All pages (landing, auth flow, dashboard, lessons, progress, settings)
- ✅ Components (LoadingSpinner, forms)
- ✅ Tests (auth-context, api-client)
- ✅ TailwindCSS responsive design
- ✅ TypeScript strict mode passes

**Design system migration** (2026-06-09):
- ✅ Tailwind removed, design system tokens wired (ticket 1)
- ✅ Core UI components ported to TSX in `components/ui/` (ticket 2)
- ✅ Form components ported to TSX in `components/ui/` (ticket 3)
- ✅ Feedback components ported; old Toast.tsx replaced (ticket 4)
- ✅ Learning components ported to `components/learning/` (ticket 5)
- ✅ Marketing site at `/` — all 8 sections + AuthModal (ticket 6)
- ✅ All app pages restyled — auth, dashboard, admin (ticket 7)
- ✅ Utility components restyled — Skeleton.tsx, LoadingSpinner.tsx (ticket 8)

## App Pages — Design System Patterns (ticket 7)

- Spinners: `<div style={{ borderRadius: '50%', border: '3px solid var(--neutral-200)', borderTopColor: 'var(--brand-600)', animation: 'spin 0.8s linear infinite' }} />` — needs `data-testid` for tests
- Link-as-button: use `<Link className="asc-btn asc-btn--primary">` (Button renders `<button>`, not anchor)
- `Tag` component uses `level` prop (`beginner | intermediate | advanced`), not `tone`
- `LevelBadge` expects `level: number` — not suitable for string difficulty labels; use `Tag` instead
- `ProgressBar` is in `@/components/ui`, not `@/components/learning`
- `Switch` component requires the label and switch to be wired manually (`htmlFor` + `id`); no built-in label slot
- `Card` `as="section"` supported for semantic HTML
- All inline loading states: use `var(--neutral-200)` / `var(--surface-sunken)` skeleton backgrounds
- Admin sidebar uses `var(--neutral-950)` background with `rgba(255,255,255,0.08)` dividers
- Dashboard sidebar uses `var(--surface-inverse)`
- `Skeleton` component: accepts `style?: CSSProperties` (not `className`); background `var(--surface-sunken)`, uses `skeleton-pulse` keyframe in globals.css
- `LoadingSpinner`: inline styles only, uses `spin` keyframe, has `data-testid="loading-spinner"`

## Marketing Site (app/page.tsx)

- All sections in one file: Container + Logo helpers, Nav, Hero, HeroMock, Stat, Tracks, How, AIBand, Pricing/PlanCard, FinalCTA, Footer, AuthModal, Home
- `@keyframes spin` added to `globals.css`
- Auth modal is full-screen overlay (not a separate route) — `useState<'login' | 'signup' | null>`
- `TRACK_ICONS` and `HOW_ICONS` are lookup objects mapping icon name string → `React.ReactElement`
- `Badge` component uses `tone` prop (not `variant`)
- `TRACKS` array typed as `Array<{ ... featured?: boolean }>` (not `as const`)
- Tests in `app/__tests__/page.test.tsx` (19 tests)

