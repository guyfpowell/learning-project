# Code Review — Learning App (2026-04-12)

## Test Results

| Component | Status | Details |
|-----------|--------|---------|
| Backend Build | ✅ PASS | TypeScript compiles cleanly |
| Backend Tests | ✅ PASS | 40 tests passing |
| Frontend Build | ✅ PASS | Next.js 14 build succeeds |
| Frontend Tests | ❌ FAIL | 4 tests failing |
| Database | ✅ PASS | Schema synced, all migrations applied |

---

## Issues Found

### HIGH SEVERITY

1. **Frontend test mocks don't properly throw errors**
   - Files: `packages/web/lib/__tests__/api-client.test.ts`, `packages/web/lib/__tests__/auth-context.test.tsx`
   - 4 tests failing because mock setup doesn't replicate actual error behavior

2. **153 `any` type casts in frontend code**
   - Files: `packages/web/lib/auth-context.tsx`, `packages/web/lib/api-client.ts`, throughout frontend
   - TypeScript type checking bypassed

3. **No validation on API responses**
   - File: `packages/web/lib/auth-context.tsx:34`
   - apiClient.getMe() result assigned directly to state without validation

### MEDIUM SEVERITY

4. **Mock tests don't match real API schema**
   - File: `packages/web/lib/__tests__/api-client.test.ts:14-21`
   - Mock response structure differs from actual ApiResponse<T>

5. **API error response format inconsistency**
   - Files: `packages/api/src/middleware/error-handler.ts`, `packages/web/lib/api-client.ts:44-45`
   - Frontend and backend error handling may not align

6. **Lesson delivery ignores skill paths**
   - File: `packages/api/src/services/LessonService.ts:27-40`
   - getTodayLesson() returns first incomplete lesson, ignores user's selected skill path

7. **Auth context auto-login race condition**
   - File: `packages/web/lib/auth-context.tsx:28-46`
   - Multiple components can call useAuth() before initial auth check completes

### LOW SEVERITY

8. **Missing endpoint: GET /api/skills**
   - Onboarding calls apiClient.getSkills() but endpoint doesn't exist
   - Frontend will crash when user reaches onboarding

9. **Stripe integration stubbed**
   - File: `packages/api/src/controllers/SubscriptionController.ts`
   - Stripe API calls commented out, no webhook handlers

10. **Request validation missing**
    - File: `packages/api/src/controllers/AuthController.ts`
    - Controllers don't validate input format (email, password, etc.)

---

## Phased Fix Plan

### Phase 1: Type Safety & Tests

#### 1.1 Fix Frontend Test Mocks
**Files**: `packages/web/lib/__tests__/api-client.test.ts`, `packages/web/lib/__tests__/auth-context.test.tsx`

**Current behavior**: 
- Tests mock `global.fetch` but don't properly simulate error responses
- `api-client.test.ts` line 42-49: expects `.rejects.toThrow()` but apiClient doesn't throw
- `auth-context.test.tsx` line 137: mocks login error but doesn't propagate it

**Fix required**:
- Line 42-49 in api-client.test.ts: When `ok: false`, apiClient.request() should throw the error object. Currently swallows it at line 44-45.
- Rewrite mock to: `mockResolvedValueOnce({ ok: false, json: async () => ({ error, code }) })` → should throw
- Line 137 in auth-context.test.tsx: Ensure mock error propagates through `login()` method

**Success criteria**: All 4 frontend tests pass. Error paths tested end-to-end.

#### 1.2 Remove `any` Type Casts
**Files**: `packages/web/lib/auth-context.tsx`, `packages/web/lib/api-client.ts`, form components

**Current behavior**: 153 instances of `as any` or `any` type annotations

**Specific locations**:
- `auth-context.tsx`: `apiClient` imported but typed loosely, `userData` from API has no type
- `api-client.ts`: Generic `<ApiResponse<any>>` throughout, request/response payloads untyped
- Form handlers: `e.target` cast as `any`

**Fix required**:
- Import `User`, `Lesson`, `Quiz`, `UserProgress` from `@learning/shared`
- Type all API response returns: `async register(...): Promise<{ token: string; user: User }>`
- Type form event: `e: React.ChangeEvent<HTMLInputElement>` (not `as any`)
- Update AuthContextType to use `User` type from shared

**Success criteria**: TypeScript strict mode passes with 0 type errors. No `as any` in codebase.

#### 1.3 Add Response Validation
**File**: `packages/web/lib/auth-context.tsx:34`

**Current behavior**: 
```typescript
const userData = await apiClient.getMe()
setUser(userData)  // No validation
```

**Fix required**:
- After API call, validate response shape before setting state
- Use type guard: Check `userData.id`, `userData.email` exist and are strings
- Or: Import Zod, create `UserSchema = z.object({ id: z.string(), email: z.string(), name: z.string() })`, validate with `.parse(userData)`
- If validation fails, clear token and redirect to login

**Success criteria**: Malformed API response handled gracefully. No crashes from invalid data.

---

### Phase 2: Fill Missing Gaps

#### 2.1 Implement GET /api/skills Endpoint
**Missing endpoint**: `GET /api/skills` (called by onboarding page)

**Files to create/modify**:
- `packages/api/src/routes/lessons.ts`: Add route `router.get('/skills', ...)`
- `packages/api/src/controllers/LessonController.ts`: Add method `getSkills(req, res, next)`
- `packages/api/src/services/LessonService.ts`: Add method `getAllSkills()` → query Prisma `skill.findMany()`

**Implementation**:
```typescript
// Route: GET /api/lessons/skills (or /api/skills if separate)
async getSkills(req, res, next) {
  try {
    const skills = await lessonService.getAllSkills()
    res.json({ success: true, data: skills, timestamp: new Date() })
  } catch (error) {
    next(error)
  }
}

// Service
async getAllSkills() {
  return await prisma.skill.findMany({
    select: { id: true, name: true, description: true, category: true }
  })
}
```

**Success criteria**: 
- `GET /api/lessons/skills` returns list of skills
- Onboarding page fetches and displays skills without crashing
- Frontend test passes

#### 2.2 Fix Lesson Delivery Logic
**File**: `packages/api/src/services/LessonService.ts:12-56`

**Current behavior**: 
```typescript
async getTodayLesson(userId: string) {
  const allProgress = await prisma.userProgress.findMany({ where: { userId } })
  const incompleteLessons = allProgress.filter((p) => !p.completedAt)
  if (incompleteLessons.length > 0) {
    return this.getLessonById(incompleteLessons[0].lessonId)  // First incomplete, ignores skill path
  }
  // If all complete, return first lesson again (wrong)
}
```

**Fix required**:
- Fetch user's selected skill path from `user.profile.skillPathId`
- Get lessons for that skill path only: `prisma.lesson.findMany({ where: { skillPath: { id: skillPathId } } })`
- Return first incomplete lesson from that path (ordered by `day` field)
- If all lessons in path complete, mark path complete, suggest next path

**Success criteria**: 
- User progresses through lessons in their selected skill path
- Doesn't skip around or repeat
- Tests verify lesson ordering by `day` field

#### 2.3 Add Request Validation
**File**: `packages/api/src/controllers/AuthController.ts`

**Current behavior**: No validation of email format, password strength, etc.

**Fix required**:
- Option A: Add validation middleware (joi or zod)
- Option B: Validate in each controller before calling service

**Example (Option B - simpler)**:
```typescript
async register(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const { email, password, name } = req.body
    
    // Validate
    if (!email || !email.includes('@')) throw new AppError('INVALID_EMAIL', 'Invalid email', 400)
    if (!password || password.length < 8) throw new AppError('WEAK_PASSWORD', 'Password must be 8+ chars', 400)
    if (!name || name.length < 2) throw new AppError('INVALID_NAME', 'Name required', 400)
    
    const response = await authService.register({ email, password, name })
    res.json({ success: true, data: response, timestamp: new Date() })
  } catch (error) {
    next(error)
  }
}
```

**Success criteria**: Backend rejects invalid inputs. Frontend and backend validation rules match (8+ password, valid email).

---

### Phase 3: Robustness

#### 3.1 Fix Auth Context Race Condition
**File**: `packages/web/lib/auth-context.tsx:24-46`

**Current behavior**:
```typescript
export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    const checkAuth = async () => {
      try {
        const token = localStorage.getItem('auth_token')
        if (token) {
          const userData = await apiClient.getMe()  // Async, may not complete before children render
          setUser(userData)
        }
      } catch (error) {
        localStorage.removeItem('auth_token')
      } finally {
        setLoading(false)
      }
    }
    checkAuth()
  }, [])

  return (
    <AuthContext.Provider value={{ user, loading, ... }}>
      {children}  // Children render immediately, may call useAuth() before auth check completes
    </AuthContext.Provider>
  )
}
```

**Fix required**:
- Keep `loading` state during auth check
- Root layout (`app/layout.tsx` or `app/providers.tsx`) should check `loading` and show spinner
- Or wrap protected routes in Suspense boundary that waits for auth check

**Example**:
```typescript
// In root layout
{loading ? <LoadingSpinner /> : <AuthProvider>{children}</AuthProvider>}
```

**Success criteria**: No race conditions. Auth check completes before any page renders.

#### 3.2 Audit Error Handling
**Files**: `packages/api/src/middleware/error-handler.ts`, `packages/web/lib/api-client.ts:43-45`

**Current behavior**:
- Backend throws `AppError` → error-handler middleware → JSON response
- Frontend expects `{ error, code }` shape
- No guarantee they match

**Fix required**:
- Read `error-handler.ts` and confirm response shape (should be `{ success: false, error: string, code: string, timestamp: string }`)
- Update `api-client.ts:44-45` to parse this exact shape
- Add test: throw an error from backend, verify frontend receives correct shape

**Success criteria**: Error responses consistent format. Frontend error handling tested against real API errors.

#### 3.3 Integration Tests
**Create**: `packages/api/src/__tests__/integration.test.ts`

**Test full flow**:
1. Register user → verify user created in database
2. Login → verify token returned
3. Get today's lesson → verify lesson from selected skill path
4. Complete lesson → verify UserProgress.completedAt set
5. Submit quiz → verify quizScore stored

**Success criteria**: All critical user flows tested against real database. No mocks.

---

### Phase 4: Polish

#### 4.1 Code Review & Refactor
- Review all changes for consistency
- Remove dead code (empty middleware package, unused imports)
- Ensure error messages user-friendly

#### 4.2 Security Hardening
**Not required for MVP but flag**:
- Rate limiting on `/auth/register` and `/auth/login` (prevent brute force)
- Input sanitization if lesson content ever user-generated
- HTTPS enforcement in production

#### 4.3 Performance
- Verify database queries use indexes (UserProgress.userId, Subscription.userId)
- Check N+1 queries in lesson loading
- Profile bundle size (already good at 97KB)

---

## Salvageability Assessment

**Is the project salvageable?** Yes.

Backend is solid (40 tests pass, architecture clean). Frontend needs type safety fixes and missing endpoints. Not worth rewriting — fix the current codebase.
