# Pre-existing Failing Tests

Recorded 2026-06-12 during REQ-002 chunk 002.3 implementation. Confirmed pre-existing by stashing all changes and re-running the suite — same failures appeared.

**Scope**: web unit tests only (`packages/web`, `--testPathIgnorePatterns=e2e`). API unit tests: 256 passing, 0 failing.

---

## Failing Test Suites (2 suites, 25 tests)

### 1. `packages/web/app/__tests__/page.test.tsx`

**Suite**: `Home — marketing site › section rendering`

| Test | Status |
|---|---|
| renders Nav logo and auth buttons | FAIL |
| renders Hero headline | FAIL |
| renders all track cards | FAIL |
| renders How It Works section | FAIL |
| renders AIBand section | FAIL |

5 failures. Likely caused by a recent structural change to the home/marketing page that the tests haven't been updated to reflect.

---

### 2. `packages/web/app/(dashboard)/tracks/__tests__/page.test.tsx`

20 failures (exact test names not captured — were not printed in the test run output reviewed). Likely caused by a recent change to the tracks dashboard page or its API contract.

---

## Notes

- These failures were present before any REQ-002 work began.
- Neither suite is related to admin functionality.
- No new failures were introduced by REQ-002 chunks 002.1–002.3.

---

## Pre-existing Type Errors (non-test)

### 1. `prisma/seed.ts` — `bcryptjs` missing type declarations

**Error**: `TS7016 — Could not find a declaration file for module 'bcryptjs'`  
**File**: `prisma/seed.ts` line 2  
**First noticed**: REQ-003 chunk 2 (2026-06-12)  
**Impact**: Type error only — seed runs correctly at runtime. No test failure.  
**Fix**: Install `@types/bcryptjs` — deferred, not blocking.

---

## Policy

Going forward, record any pre-existing errors or failures discovered during REQ implementation here before starting chunk work, so they are not mistakenly attributed to new changes.
