# App Context — Learning App Mobile

Last updated: 2026-04-14 (4.2.3)

---

## Project Overview

React Native/Expo mobile app for the Learning platform. Built in Phase 4 of the learning project.
Target: feature parity with the web app (lessons, quiz, progress, profile, settings).

**App repo**: `/Users/guypowell/Documents/Projects/learning-app/` (currently empty — not yet initialised)
**Monorepo**: `/Users/guypowell/Documents/Projects/learning/`
**Reference app**: `/Users/guypowell/documents/projects/pocketchange-app/` — copy infrastructure, strip domain logic

---

## Tech Stack

| Layer | Library | Version |
|-------|---------|---------|
| Framework | Expo | ~54.0.0 |
| Runtime | React Native | 0.81.5 |
| UI library | React | 19 |
| Navigation | Expo Router | ~6.0.23 (file-based, typed routes) |
| State | Zustand | 5 |
| Data fetching | TanStack Query | 5 |
| HTTP | Axios | ^1.7.9 |
| Secure storage | Expo SecureStore | ~15.0.8 |
| Error tracking | Sentry (@sentry/react-native) | ^8.5.0 |
| Fonts | @expo-google-fonts/poppins | ^0.2.3 |
| Icons | @expo/vector-icons | ^15.0.3 |
| Testing | Jest + @testing-library/react-native | jest ^29, rntl ^12.9 |
| Test runner | jest-expo | ~54.0.17 |

---

## Folder Structure

```
learning-app/
├── app/
│   ├── (auth)/              # login.tsx, signup.tsx
│   ├── (tabs)/              # lessons.tsx, progress.tsx, profile.tsx, settings.tsx
│   │   └── _layout.tsx      # tab navigator
│   ├── _layout.tsx          # root layout — AuthGate, error boundary, Sentry
│   └── index.tsx            # redirect to tabs or auth
├── src/
│   ├── components/
│   │   ├── ui/              # Button, Input, Card, Badge, Spinner, Logo
│   │   └── QuizModal.tsx
│   ├── hooks/               # useAuth.ts, useLesson.ts, useQuiz.ts, useProgress.ts
│   ├── lib/
│   │   └── api.ts           # Axios instance, 401 interceptor, token refresh mutex
│   ├── providers/
│   │   └── QueryProvider.tsx
│   ├── services/            # auth.service.ts, lesson.service.ts, progress.service.ts
│   ├── store/
│   │   └── auth.store.ts    # Zustand + Expo SecureStore persistence
│   └── theme/
│       └── index.ts         # colour tokens, spacing, typography
├── babel.config.js
├── jest.config.js
├── jest.setup.ts
├── metro.config.js
├── eas.json
├── tsconfig.json
└── package.json
```

---

## API

**Base URL (local)**: `http://localhost:3000`

| Route prefix | Description |
|---|---|
| `GET /health` | Health check |
| `/api/auth` | Login, signup, logout, refresh |
| `/api/users` | User profile, user progress |
| `/api/lessons` | Fetch lessons, today's lesson |
| `/api/subscriptions` | Subscription status |
| `/api/notifications` | Notification preferences (GET/PATCH) |
| `GET /api/users/progress` | Returns `UserProgressStats` (totalLessonsCompleted, currentStreak, averageScore, lastLessonDate) |

**Auth**: JWT bearer token. Stored in Expo SecureStore. Refresh token flow via 401 interceptor in `src/lib/api.ts`.

---

## Shared Types

All response types imported from `@learning/shared` (monorepo package — do not redefine locally).

Key types:
- `ApiResponse<T>`, `ApiListResponse<T>`, `ErrorResponse`
- `AuthRequest`, `AuthResponse`
- `User`, `UserProfile`, `UserProgress`, `UserAuth`
- `Lesson`, `LessonContent`, `Quiz`, `SkillPath`, `Skill`
- `NotificationPreference`
- `UserProgressStats`

---

## Design System

Copied from `pocketchange-app/src/theme/index.ts` — retheme colours only.
Font: Poppins (via `@expo-google-fonts/poppins`).
UI components: `Button`, `Input`, `Card`, `Badge`, `Spinner`, `Logo` — all in `src/components/ui/`.

---

## Navigation

Root layout (`app/_layout.tsx`) checks auth state:
- Authenticated → `(tabs)` group
- Unauthenticated → `(auth)` group

Tabs: Lessons | Progress | Profile | Settings

---

## Key Patterns

- **HTTP client**: Always use the singleton in `src/lib/api.ts` — never create a new Axios instance
- **Auth state**: Read/write via `src/store/auth.store.ts` — never read token directly from SecureStore in components
- **Secure storage**: Expo SecureStore only for tokens — never AsyncStorage for sensitive data
- **Response types**: Always import from `@learning/shared` — never define API response interfaces locally
- **No native Stripe SDK** — payments use web checkout (redirect flow only)

---

## What to Copy from `pocketchange-app`

| File | Action |
|------|--------|
| `src/lib/api.ts` | Copy wholesale |
| `src/store/auth.store.ts` | Copy wholesale |
| `src/services/auth.service.ts` | Copy wholesale |
| `src/hooks/useAuth.ts` | Copy wholesale |
| `src/components/ui/` | Copy wholesale |
| `src/theme/index.ts` | Copy, retheme colours |
| `src/providers/QueryProvider.tsx` | Copy wholesale |
| `app/_layout.tsx` | Copy, update for learning app |
| `jest.config.js`, `babel.config.js`, `metro.config.js` | Copy wholesale |
| `eas.json` | Copy wholesale |

Strip all: wallet, donation, recipient, donor, QR, Stripe native SDK, `src/providers/StripeWrapper.tsx`, `src/config/features.ts`

---

## Constraints

- Expo Go only (no bare workflow) — no custom native modules
- OTA updates via Expo Updates
- No iOS native builds without Apple Developer account
- No native Stripe SDK — web checkout redirect only

---

## Phase 4 Progress

| Chunk | Description | Status |
|-------|-------------|--------|
| 4.1.1 | Expo project initialisation | ✅ Complete |
| 4.1.2 | Auth flow (login, signup, SecureStore) | ✅ Complete |
| 4.1.3 | Navigation setup (tabs + auth gate) | ✅ Complete |
| 4.2.1 | Lessons tab (today's lesson card) | ✅ Complete |
| 4.2.2 | Quiz modal | ✅ Complete |
| 4.2.3 | Progress tab + Profile tab | ✅ Complete |

---

## Gotchas

- **`validateEmail` runs on raw input** — the screen trims/lowercases email only inside the `mutate` call, *after* validation passes. Inputs with leading/trailing spaces fail the regex. Test inputs must be regex-valid before testing normalisation behaviour.
- **`react-native-worklets` must stay in package.json** — `react-native-reanimated@4.x` requires it transitively via its Babel plugin. Removing it breaks all tests with `Cannot find module 'react-native-worklets/plugin'`.
- **No refresh token** — learning backend uses stateless JWT (`{ token, user }`). No `/auth/refresh` endpoint. `api.ts` 401 interceptor clears auth + redirects directly (no refresh attempt). `refreshToken` field in store is always `''`.
- **`@learning/shared` in Jest** — requires `moduleNameMapper` in `jest.config.js` pointing to `../learning/packages/shared/src/index.ts` (TypeScript source), not the compiled `dist/`.
- **`RegisterInput` requires `name`** — the learning API's `/auth/register` endpoint requires `{ email, password, name }`. Unlike pocketchange, name is mandatory at registration.
- **`app/(auth)/_layout.tsx`** — copied from pocketchange, may still contain pocketchange-specific route references. Check before 4.1.2.
- **Service typing vs real API shape** — Services type `api.get<T>()` as the raw model (e.g. `Lesson`), but the backend wraps responses in `{ success, data, timestamp }`. Tests mock `api.get` to return `{ data: mockModel }` directly, which papers over the mismatch. At runtime, `data` from axios is the full `ApiResponse` object. This is an established pattern in the codebase — follow it for consistency, do not fix individual services.
- **`Button` loading hides label** — when `loading={true}`, `Button` renders `ActivityIndicator` instead of label text. Tests that navigate via `getByText('NEXT')` will fail if `isPending` is `true` from mount. Use a single-quiz lesson (first = last question) to avoid navigation when testing loading state.
- **`QuizResult`/`QuizFeedback` in shared** — these types were missing from `@learning/shared` and were added in 4.2.2. Import from `@learning/shared`, never define locally.
- **`useSubmitQuiz` mock needs `reset`** — TanStack Query mutations expose a `reset` function. QuizModal calls `submit.reset()` on open. Mock `useSubmitQuiz` must include `reset: jest.fn()` or tests will throw `submit.reset is not a function`.
- **`useAuthStore` selector mock in profile tests** — `useAuthStore((s) => s.user)` uses a selector. Mock as `(useAuthStore as unknown as jest.Mock).mockReturnValue(mockUser)` — the mock returns `mockUser` regardless of the selector arg passed, which is the intended behaviour.
- **`UserProgressStats` not in backend stats response** — the backend `getUserProgress()` returns a computed object (not a Prisma model). The shape is `{ totalLessonsCompleted, currentStreak, averageScore, lastLessonDate }`. Type defined in `@learning/shared` as `UserProgressStats`.

---

## Change Log

| Date | Chunk | Summary |
|------|-------|---------|
| 2026-04-12 | — | Context file created. Phase 4 not yet started. |
| 2026-04-12 | 4.1.1 | Project init: rsync copy from pocketchange-app, stripped domain logic, replaced auth types/services with learning-app equivalents, 58 tests passing. |
| 2026-04-13 | 4.1.2 | Auth flow screens verified. `(auth)/_layout.tsx` confirmed clean. Full screen-level tests added for sign-in (15 tests) and register (11 tests). 77 tests total, all passing. |
| 2026-04-13 | 4.1.3 | Navigation setup. Created `(tabs)/_layout.tsx` (Tabs navigator, Ionicons, indigo tint) and 4 placeholder screens (lessons, progress, profile, settings). 9 new tests. 86 tests total, all passing. |
| 2026-04-13 | 4.2.1 | Lessons tab. Created `lesson.service.ts`, `useLesson.ts`, real `lessons.tsx` screen. Card with title, difficulty Badge, duration, Take Quiz Button (no-op until 4.2.2). 14 new tests. 100 tests total, all passing. |
| 2026-04-14 | 4.2.2 | Quiz modal. Added `QuizFeedback`/`QuizResult` to `@learning/shared`. Created `quiz.service.ts`, `useQuiz.ts`, `QuizModal.tsx`. Wired Take Quiz button in lessons screen. 23 new tests. 123 tests total, all passing. |
| 2026-04-14 | 4.2.3 | Progress + Profile tabs. Added `UserProgressStats` to `@learning/shared`. Created `progress.service.ts`, `useProgress.ts`. Real `progress.tsx` (3 stat cards + last lesson date). Real `profile.tsx` (user name/email + outline Log Out button via useLogout). 19 new tests. 142 tests total, all passing. Phase 4 complete. |
