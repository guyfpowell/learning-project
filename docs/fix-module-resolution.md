# Fix Module Resolution — API ES Modules

**Status**: Ready for implementation  
**Assigned to**: Backend agent (`/be`)  
**Complexity**: Low  
**Time estimate**: 15 minutes  

---

## Problem

The API fails to start after build with:
```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '.../packages/api/dist/middleware/error-handler'
```

**Root Cause**: Node.js ES modules require explicit `.js` file extensions in import paths. The API is configured as `"type": "module"` but TypeScript compiles imports without extensions, causing runtime failures.

**Example**:
- Source: `import { errorHandler } from './middleware/error-handler'` (no extension)
- Compiled: Same (no extension added)
- Runtime: ❌ Node.js can't resolve `./middleware/error-handler` as an ESM file
- Should be: `import { errorHandler } from './middleware/error-handler.js'` → compiles same → ✅ Node.js finds it

---

## Solution

Convert both **API** and **shared** packages to use `NodeNext` module resolution. This forces TypeScript to require (and emit) `.js` extensions in all import paths, matching Node.js ESM expectations.

### 1. Update API TypeScript Config

**File**: `/Users/guypowell/Documents/Projects/learning/packages/api/tsconfig.json`

Replace the entire `"compilerOptions"` section with:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "NodeNext",
    "lib": ["ES2020", "DOM"],
    "moduleResolution": "NodeNext",
    "strict": false,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "noImplicitAny": false,
    "declaration": false,
    "declarationMap": false
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

**Key changes**:
- `"module": "NodeNext"` (was `"ES2020"`)
- `"moduleResolution": "NodeNext"` (was `"node"`)

**Why**: `NodeNext` is Node.js' recommended setting for `"type": "module"` packages. It enforces `.js` extensions in all import statements at compile-time, ensuring runtime resolution works correctly.

### 2. Update Shared TypeScript Config

**File**: `/Users/guypowell/Documents/Projects/learning/packages/shared/tsconfig.json`

Replace the entire `"compilerOptions"` section with:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "NodeNext",
    "lib": ["ES2020", "DOM"],
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

**Key changes**:
- `"module": "NodeNext"` (was `"commonjs"`)
- `"moduleResolution": "NodeNext"` (was `"node"`)

### 3. Add ES Module Type to Shared Package

**File**: `/Users/guypowell/Documents/Projects/learning/packages/shared/package.json`

Add the following line after `"name"`:
```json
"type": "module",
```

**Example** (complete file structure):
```json
{
  "name": "@learning/shared",
  "type": "module",
  "version": "0.1.0",
  "description": "Shared types and utilities for Learning app",
  ...
}
```

---

## Verification

After applying either option:

```bash
cd /Users/guypowell/Documents/Projects/learning && pnpm build
cd /Users/guypowell/Documents/Projects/learning/packages/api && pnpm build && pnpm start
```

Expected output:
```
🚀 Server running on port 3000
📝 Health check: http://localhost:3000/health
```

If successful:
- ✅ API starts without module resolution errors
- ✅ Shared package imports resolve correctly
- ✅ Both can be run in parallel: `pnpm dev` from root

---

## Files to Modify

- `/Users/guypowell/Documents/Projects/learning/packages/api/tsconfig.json`
- `/Users/guypowell/Documents/Projects/learning/packages/shared/tsconfig.json`
- `/Users/guypowell/Documents/Projects/learning/packages/shared/package.json` (add `"type": "module"`)

---

## Do Not

- ❌ Do not use CommonJS (`"module": "commonjs"`)
- ❌ Do not remove `"type": "module"` from API package.json
- ❌ Do not change module resolution to just `"node"` — must be `"NodeNext"`

---

## Verification Checklist

After implementation, verify:

1. **Build succeeds**
   ```bash
   cd /Users/guypowell/Documents/Projects/learning && pnpm build
   ```
   Expected: All 3 packages compile without errors (shared, api, web).

2. **API starts without module errors**
   ```bash
   cd /Users/guypowell/Documents/Projects/learning/packages/api && pnpm build && pnpm start
   ```
   Expected output:
   ```
   🚀 Server running on port 3000
   📝 Health check: http://localhost:3000/health
   ```
   (No `ERR_MODULE_NOT_FOUND` errors)

3. **Health endpoint responds**
   ```bash
   curl http://localhost:3000/health
   ```
   Expected: `{"status":"OK","timestamp":"..."}`

4. **Full dev stack works**
   ```bash
   cd /Users/guypowell/Documents/Projects/learning && pnpm dev
   ```
   Expected: API starts on 3000, Web starts on 3001, both run without errors.

---
