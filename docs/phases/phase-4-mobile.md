# Phase 4: Mobile App

**Status**: 🔵 IN PROGRESS

**Goal**: Build React Native/Expo app with same core functionality as web.

---

## Foundation: Use `pocketchange-app` as Starting Point

**Decision**: Do not scaffold from scratch. Copy `pocketchange-app` (`/Users/guypowell/documents/projects/pocketchange-app`) as the foundation and replace the domain logic.

**What to copy wholesale:**
- `src/lib/api.ts` — Axios instance with 401 interceptor, token refresh mutex, Sentry breadcrumbs
- `src/store/auth.store.ts` — Zustand auth store with Expo SecureStore persistence
- `src/services/auth.service.ts` — Login, register, logout, refresh
- `src/hooks/useAuth.ts` — TanStack Query mutations for auth
- `src/components/ui/` — Button, Input, Card, Badge, Spinner
- `src/theme/index.ts` — Color tokens, spacing, typography (retheme colours only)
- `src/providers/QueryProvider.tsx` — TanStack Query singleton
- `app/_layout.tsx` — Root layout with AuthGate, error boundary, Sentry
- `jest.config.js`, `babel.config.js`, `metro.config.js` — build config
- `eas.json` — EAS Build config

**What to strip out:**
- `src/services/wallet.service.ts`, `donation.service.ts`, `recipient.*.service.ts`
- `src/hooks/useWallet.ts`, `useDonation.ts`, `useRecipient*.ts`
- `src/components/donor/`, `src/components/scan/`
- `app/(donor)/`, `app/(recipient)/`
- `app/recipient/[id].tsx`, `app/donate/[id].tsx`, `app/donation/[id].tsx`
- `src/providers/StripeWrapper.tsx` (replace with learning-app's own subscription flow)
- `src/config/features.ts` (replace with learning-app feature flags)

**What to replace with learning-app equivalents:**
- Tab groups: `(donor)` → `(tabs)` with Lessons, Progress, Profile, Settings
- Auth group: keep structure, update API base URL to learning API (`localhost:3000`)
- Services: replace with lesson, quiz, progress services
- Hooks: replace with useLesson, useQuiz, useProgress hooks

**Stack (confirmed from pocketchange-app):**
- Expo ~54, React Native 0.81.5, React 19
- Expo Router ~6 (file-based, typed routes)
- Zustand 5 + TanStack Query 5
- Expo SecureStore (tokens)
- Axios (API client)
- Sentry (error tracking — already wired)
- Jest + @testing-library/react-native (tests)

---

## Chunk 4.1: Mobile Project Setup & Auth

### 4.1.1 — Expo Project Initialization

**Deliverable**: Expo project boots in Expo Go

**Setup**:
```bash
npx create-expo-app learning-app
cd learning-app
npx expo install expo-router
```

**Folder Structure**:
```
learning-app/
├── app/
│   ├── (auth)/          # login, signup, onboarding
│   ├── (tabs)/          # lessons, progress, profile, settings
│   └── _layout.tsx      # root navigator with auth check
├── components/          # reusable UI components
├── hooks/               # custom hooks (useAuth, etc.)
├── services/            # API client, storage
└── package.json
```

### 4.1.2 — Auth Flow

**Status**: ✅ COMPLETE

**Files**:
- `app/(auth)/login.tsx` — Login screen
- `app/(auth)/signup.tsx` — Signup screen
- `hooks/useAuth.ts` — Auth state + Expo Secure Store for tokens

**Features**:
- Use Expo Secure Store (instead of localStorage)
- Same auth logic as web (login, register, logout)
- Auto-login on app launch

### 4.1.3 — Navigation Setup

**Status**: ✅ COMPLETE

**File**: `app/(tabs)/_layout.tsx`

**Tabs**:
- Lessons
- Progress
- Profile
- Settings

**Root Navigation**: Check auth state, show (auth) or (tabs) based on login status

---

## Chunk 4.2: Mobile Features

### 4.2.1 — Lessons Tab

**Status**: ✅ COMPLETE

**File**: `app/(tabs)/lessons.tsx`

**Display**:
- Today's lesson card
- Lesson duration, difficulty
- Button to take quiz

### 4.2.2 — Quiz Tab / Quiz Modal

**Status**: ✅ COMPLETE

**File**: `components/QuizModal.tsx`

**Features**:
- Display quiz questions
- Submit answers
- Show results + feedback

### 4.2.3 — Progress & Profile Tabs

**Status**: ✅ COMPLETE

**Files**:
- `app/(tabs)/progress.tsx` — Streak counter, calendar view, stats
- `app/(tabs)/profile.tsx` — User info, settings, logout

---

## Chunk 4.2.3 Implementation Summary (2026-04-14)

**Status**: ✅ COMPLETE

### Shared types added:
- **`@learning/shared: UserProgressStats`** — `{ totalLessonsCompleted, currentStreak, averageScore, lastLessonDate }`

### Files Created:
- **`src/services/progress.service.ts`** — `progressService.getProgress()` → `GET /users/progress`
- **`src/hooks/useProgress.ts`** — `useProgress()` TanStack Query hook (`queryKey: ['progress']`)

### Files Modified (replaced placeholders):
- **`app/(tabs)/progress.tsx`** — 3 stat cards (streak, lessons done, avg score) + last lesson date card; loading/error/empty states
- **`app/(tabs)/profile.tsx`** — User name + email from auth store; outline Log Out button via `useLogout`

### Tests:
- `src/services/__tests__/progress.service.test.ts` — 3 tests (GET call, returns data, throws on error)
- `src/hooks/__tests__/useProgress.test.ts` — 4 tests (success, loading, error, called once)
- `app/(tabs)/__tests__/progress.test.tsx` — 10 tests (render, heading, error, null, loading, streak, lessons, score, last date, no date when null)
- `app/(tabs)/__tests__/profile.test.tsx` — 6 tests (render, heading, name, email, logout button, logout called on press)

### Test counts:
- **142 tests, 24 suites — all passing**

---

## Next Phase

[Phase 5: Stripe Billing](./phase-5-stripe-billing.md)

---

## Chunk 4.1.1 Implementation Summary (2026-04-12)

**Status**: ✅ COMPLETE

**Approach**: Copied `pocketchange-app` wholesale via rsync (excluding `node_modules`, `.expo`, `dist`), then stripped domain logic and replaced with learning-app equivalents. Did NOT scaffold from scratch.

### Files Stripped (deleted):
- `src/services/wallet.service.ts`, `donation.service.ts`, `recipient.self.service.ts`, `recipient.service.ts`
- `src/hooks/useWallet.ts`, `useDonation.ts`, `useRecipient.ts`, `useRecipientSelf.ts`
- `src/providers/StripeWrapper.tsx`
- `src/config/features.ts`
- `app/(donor)/`, `app/(recipient)/`, `app/recipient/`, `app/donate/`, `app/donation/`
- `src/components/donor/`, `src/components/scan/`
- `app/(auth)/set-password.tsx` (mustChangePassword flow — not in learning backend)
- All corresponding pocketchange-specific tests
- `plan-app.md`, `CLAUDE.md`, `README.md`, `knowledge/`, `docs/`, `scripts/`

### Files Modified:
- **`package.json`**: name → `learning-app`; stripped `@pocketchange/shared`, `@stripe/stripe-react-native`, `@gorhom/bottom-sheet`, `expo-camera`, `expo-brightness`, `react-native-qrcode-svg`, `react-native-svg`, `react-native-worklets` (as explicit dep, still needed transitively via reanimated); added `@learning/shared: file:../learning/packages/shared`; removed `deploy:ota` script. NOTE: `react-native-worklets` must remain as `react-native-reanimated@4.x` depends on it at runtime.
- **`app.json`**: name → `Learning`, slug → `learning-app`, bundle IDs → `com.learning.app`, stripped camera/Stripe plugins, updated Sentry project, bg colour → `#F8FAFC`
- **`babel.config.js`**: Removed `react-native-worklets/plugin` from plugins (it was explicit; reanimated plugin loads it transitively)
- **`eas.json`**: Stripped Stripe env vars, updated `EXPO_PUBLIC_API_URL` → `https://api.learning.app/api`
- **`jest.config.js`**: Removed `@gorhom` from transformIgnorePatterns, added `@learning/shared` moduleNameMapper pointing to `../learning/packages/shared/src/index.ts`, removed `/node_modules/react-native-reanimated/plugin/` ignore (not needed)
- **`src/theme/index.ts`**: Recoloured to indigo primary (`#4F46E5`, `#3730A3`, `#6366F1`), sky blue accent, lighter background (`#F8FAFC`), darker textDark (`#1E293B`)
- **`src/store/auth.store.ts`**: Replaced pocketchange `AuthUser` (role/walletBalance) with `UserAuth` from `@learning/shared`; removed `mustChangePassword` field and `setMustChangePassword` action; changed storage key from `pocketchange-auth` → `learning-auth`; `setAuth` signature simplified to 3 args (no mustChangePassword)
- **`src/lib/api.ts`**: Default BASE_URL → `http://localhost:3000/api`; removed token refresh flow entirely (learning backend has no `/auth/refresh` endpoint — stateless JWT); 401 interceptor now clears auth + redirects to sign-in directly; `logoutHandled` guard resets after tick via `setTimeout(..., 0)` to prevent blocking future requests
- **`src/services/auth.service.ts`**: Complete rewrite — imports `AuthResponse`, `UserAuth` from `@learning/shared`; `login/register` return `{ user, token }` (single token); `logout` is a no-op (stateless JWT, no server call); `RegisterInput` includes `name` field (required by learning backend)
- **`src/services/user.service.ts`**: Updated to use `UserAuth`, `UserProfile` from `@learning/shared`; added `getProfile()` for `/users/profile`
- **`src/types/index.ts`**: Re-exports from `@learning/shared` only — no locally-defined API response types
- **`src/hooks/useAuth.ts`**: Removed `useSetPassword`; `useLogin`/`useRegister` call `setAuth(user, token, '')`; `useLogout` calls `authService.logout()` (no-op) + `clearAuth()` in `onSettled`
- **`app/_layout.tsx`**: Removed `StripeWrapper`, `BottomSheetModalProvider`, `@gorhom/bottom-sheet` import; simplified `AuthGate` — no role-based routing, no mustChangePassword; authenticated users go to `/(tabs)`, unauthenticated to `/(auth)/sign-in`; Stack only declares `index`, `(auth)`, `(tabs)` screens
- **`app/(auth)/sign-in.tsx`**: Branding updated (POCKET CHANGE → LEARNING, updated tagline/sub-text)
- **`app/(auth)/register.tsx`**: Added `name` field (required by learning API); updated branding; `register.mutate` now passes `{ name, email, password }`

### Tests:
- **58 tests, 11 suites, all passing** (`npx jest --ci`)
- `app/__tests__/AuthGate.test.tsx` — 7 tests: hydration guard, unauthenticated redirect, no-loop, authenticated → tabs, mid-session token clear
- `src/store/__tests__/auth.store.test.ts` — 4 tests: initial state, setAuth, clearAuth, setHasHydrated
- `src/lib/__tests__/api.test.ts` — 7 tests: default URL, custom URL, prod throw, token injection, no-token, 401 clear+redirect, auth-endpoint 401 bypass, network error pass-through
- `src/services/__tests__/auth.service.test.ts` — 6 tests: login POST, login returns user+token, login throws, register POST (includes name), register returns user+token, logout no-op
- `src/services/__tests__/user.service.test.ts` — 2 tests: getMe, getProfile
- `src/hooks/__tests__/useAuth.test.ts` — 8 tests: login calls service, login sets store, register calls service with name, register sets store, logout clears store, logout clears even on failure
- `src/components/ui/__tests__/` — 18 tests: Button (6), Input (6), Card (2), Badge (4)
- `app/(auth)/__tests__/sign-in.test.tsx` — 7 tests: extractError helper

### Key Decisions / Gotchas:
1. **`react-native-worklets` must stay** — `react-native-reanimated@4.x` requires it as a peer dep via its Babel plugin. Removing it from package.json broke the entire test suite with `Cannot find module 'react-native-worklets/plugin'`.
2. **No refresh token** — The learning backend uses a stateless JWT (single `token` field). `auth.store.ts` keeps `refreshToken` field (always stored as `''`) for future compatibility, but the 401 interceptor in `api.ts` does not attempt a refresh.
3. **`@learning/shared` in Jest** — needs `moduleNameMapper` pointing directly to the TypeScript source (`../learning/packages/shared/src/index.ts`) because the compiled `dist/` may not exist in CI.
4. **`AuthUser` interface removed** — was pocketchange-specific. All user types now come from `@learning/shared`.

### What's left in app/(auth):
- `sign-in.tsx` ✅ (updated branding, uses `useLogin`)
- `register.tsx` ✅ (updated branding, name field added, uses `useRegister`)
- `_layout.tsx` — needs to be checked (may have pocketchange routing)

### Next chunk: 4.1.2 — Auth flow screens (confirm `(auth)/_layout.tsx`, verify login/register work end-to-end, Expo SecureStore persistence on app launch)

---

## Chunk 4.1.2 Implementation Summary (2026-04-13)

**Status**: ✅ COMPLETE

**Approach**: Core implementation was already delivered as part of 4.1.1. This chunk confirmed the `(auth)/_layout.tsx` is clean and added full screen-level test coverage for the sign-in and register screens.

### Files Verified / Confirmed:
- `app/(auth)/_layout.tsx` — Clean Stack navigator, no pocketchange-specific routes
- `app/(auth)/sign-in.tsx` — Login screen with form validation, error banner, loading state, SecureStore-backed auth via `useLogin`
- `app/(auth)/register.tsx` — Signup screen with name/email/password/confirm fields, validation, success/error banners, `useRegister`
- `src/hooks/useAuth.ts` — `useLogin`, `useRegister`, `useLogout` — all wired to auth store
- `src/store/auth.store.ts` — Zustand + Expo SecureStore persistence; hydration guard via `_hasHydrated`
- `app/_layout.tsx` — `AuthGate` handles auto-login on app launch (waits for hydration, routes to tabs or sign-in)

### Tests Added:
- `app/(auth)/__tests__/sign-in.test.tsx` — Expanded from 7 (extractError only) to **15 tests**: 7 extractError unit tests + 8 screen tests (render, validation, submit, error banner, loading state, nav to register)
- `app/(auth)/__tests__/register.test.tsx` — New file: **11 tests** (render, empty-form validation, password-too-short, password-mismatch, submit payload, name-trim+email-lowercase, error banner, success banner, loading state, nav back)

### Test counts:
- **77 tests, 12 suites — all passing**

### Key gotcha confirmed:
- `validateEmail` runs on raw (un-trimmed) input before `mutate` is called. The `email.trim().toLowerCase()` normalisation happens inside the `mutate` call only. Test inputs must be valid for the regex before testing normalisation behaviour.

### Next chunk: 4.1.3 — Navigation setup (tabs + auth gate)

---

## Chunk 4.1.3 Implementation Summary (2026-04-13)

**Status**: ✅ COMPLETE

**Approach**: Created tab navigator layout and four placeholder screens. Auth gate (root routing) was already delivered in 4.1.1 via `app/_layout.tsx`.

### Files Created:
- **`app/(tabs)/_layout.tsx`** — Tabs navigator with Ionicons, indigo active tint (`colors.teal`), Poppins medium label font
- **`app/(tabs)/lessons.tsx`** — Placeholder: "Lessons" heading + "Coming soon"
- **`app/(tabs)/progress.tsx`** — Placeholder: "Progress" heading + "Coming soon"
- **`app/(tabs)/profile.tsx`** — Placeholder: "Profile" heading + "Coming soon"
- **`app/(tabs)/settings.tsx`** — Placeholder: "Settings" heading + "Coming soon"

### Tests Added:
- `app/(tabs)/__tests__/TabsLayout.test.tsx` — 1 test (renders without error)
- `app/(tabs)/__tests__/lessons.test.tsx` — 2 tests (renders, heading visible)
- `app/(tabs)/__tests__/progress.test.tsx` — 2 tests (renders, heading visible)
- `app/(tabs)/__tests__/profile.test.tsx` — 2 tests (renders, heading visible)
- `app/(tabs)/__tests__/settings.test.tsx` — 2 tests (renders, heading visible)

### Test counts:
- **86 tests, 17 suites — all passing**

### Next chunk: 4.2.1 — Lessons tab (today's lesson card)

---

## Chunk 4.2.1 Implementation Summary (2026-04-13)

**Status**: ✅ COMPLETE

### Files Created:
- **`src/services/lesson.service.ts`** — `lessonService.getTodayLesson()` calls `GET /lessons/today`, returns `Lesson`
- **`src/hooks/useLesson.ts`** — `useTodayLesson()` TanStack Query hook (`queryKey: ['lesson', 'today']`)
- **`app/(tabs)/lessons.tsx`** — Replaced placeholder with real screen

### Screen behaviour:
- Loading: full-screen `Spinner`
- Error: "Unable to load lesson. Please try again." error text
- Null lesson (no lesson scheduled): "No lesson scheduled for today." muted text
- Data: `Card` with lesson title, `Badge` (difficulty → success/warning/error variant), duration in minutes, "Take Quiz" `Button` (onPress no-op — wired in 4.2.2)

### Difficulty → Badge variant mapping:
| difficulty | variant |
|---|---|
| beginner | success (green) |
| intermediate | warning (yellow) |
| advanced | error (red) |

### Tests:
- `src/services/__tests__/lesson.service.test.ts` — 3 tests (GET call, returns data, throws on error)
- `src/hooks/__tests__/useLesson.test.ts` — 4 tests (success, loading, error, called once)
- `app/(tabs)/__tests__/lessons.test.tsx` — 9 tests (render, heading, title, duration, badge, quiz button, error, null state, no card while loading)

### Test counts:
- **100 tests, 19 suites — all passing**

### Next chunk: 4.2.2 — Quiz modal

---

## Chunk 4.2.2 Implementation Summary (2026-04-14)

**Status**: ✅ COMPLETE

### Shared types added:
- **`@learning/shared: QuizFeedback`** — per-question result: quizId, question, userAnswer, correctAnswer, isCorrect, explanation
- **`@learning/shared: QuizResult`** — score, feedbacks[], lesson

### Files Created:
- **`src/services/quiz.service.ts`** — `quizService.submitQuiz(lessonId, answers)` → `POST /lessons/:id/quiz`
- **`src/hooks/useQuiz.ts`** — `useSubmitQuiz()` TanStack Query mutation
- **`src/components/QuizModal.tsx`** — full quiz modal component

### Files Modified:
- **`app/(tabs)/lessons.tsx`** — wired "Take Quiz" button to `QuizModal`; added `quizVisible` state
- **`app/(tabs)/__tests__/lessons.test.tsx`** — added mock for `QuizModal`; added "opens modal on press" test

### QuizModal behaviour:
- **Quiz view**: questions one at a time; progress indicator ("Question X of Y"); multiple-choice options (Pressable rows) or short-answer (Input); Next/Submit button (disabled until answer selected); error banner on submission failure
- **Results view**: score %, "X of Y correct", feedback list per question (correct/incorrect indicator, explanation); Done button
- **No quiz state**: "No quiz available for this lesson." message
- **Reset on open**: `useEffect` resets questionIndex, answers, and mutation state when `visible` toggles to `true`

### Tests:
- `src/services/__tests__/quiz.service.test.ts` — 3 tests
- `src/hooks/__tests__/useQuiz.test.ts` — 3 tests
- `src/components/__tests__/QuizModal.test.tsx` — 16 tests

### Test counts:
- **123 tests, 22 suites — all passing**

### Gotcha:
- When `isPending: true`, `Button` renders `ActivityIndicator` (not label text). Navigation tests that call `fireEvent.press(getByText('NEXT'))` will fail if `isPending` is set before navigation. Test must use single-quiz lesson to avoid navigation when testing loading state.

### Next chunk: 4.2.3 — Progress tab + Profile tab

