# Learning App — Startup & Test Instructions

## Prerequisites

- Node.js 18+
- pnpm 8+
- Docker & Docker Compose (for database)

---

## Essential Commands

Start:
cd /Users/guypowell/Documents/Projects/learning && pnpm dev

Build:
cd /Users/guypowell/Documents/Projects/learning && pnpm build

API:
cd /Users/guypowell/Documents/Projects/learning/packages/api && pnpm build && pnpm start

Web:
cd /Users/guypowell/Documents/Projects/learning/packages/web && pnpm start

Setup & run dev:
Step 1 — First-time setup:
cd /Users/guypowell/Documents/Projects/learning && pnpm install && docker-compose up -d && pnpm db:migrate

Step 2 — Seed the database:
cd /Users/guypowell/Documents/Projects/learning/packages/api && npx tsx ../../prisma/seed.ts

Step 3 — Start dev servers:
cd /Users/guypowell/Documents/Projects/learning && pnpm dev

Run tests:
```bash
cd /Users/guypowell/Documents/Projects/learning && pnpm test
```

Build & run production:
```bash
```

---

## Quick Start

### 1. Install Dependencies

```bash
cd /Users/guypowell/Documents/Projects/learning

# Install all workspace dependencies
pnpm install
```

### 2. Start Database & Cache

```bash
# Start PostgreSQL (5433) and Redis (6380) in Docker
docker-compose up -d

# Verify containers running
docker-compose ps
```

### 3. Set Up Database

```bash
# Sync Prisma schema to database
pnpm db:push

# Seed initial data (skills, plans, lessons)
pnpm db:seed
```

### 4. Start Development Servers

```bash
# Terminal 1: Start both API and Web in parallel
pnpm dev
```

This will start:
- **API**: http://localhost:3000 (with `/health` endpoint)
- **Web**: http://localhost:3001

### 5. Access the Application

Open browser to: **http://localhost:3001**

---

## Mobile App (learning-app)

App repo: `/Users/guypowell/Documents/Projects/learning-app`

### Local — Expo Go

```bash
cd /Users/guypowell/Documents/Projects/learning-app
npm install
npx expo start
```

Scan the QR code with **Expo Go** (iOS/Android). The API must be running locally at `http://localhost:3000`.

### Tests

```bash
cd /Users/guypowell/Documents/Projects/learning-app
npx jest --ci
```

### EAS — Internal test build (preview)

Requires `eas-cli` and an Expo account.

```bash
npm install -g eas-cli
eas login

# Build for internal distribution (both platforms)
cd /Users/guypowell/Documents/Projects/learning-app
eas build --profile preview --platform all

# iOS only
eas build --profile preview --platform ios

# Android only
eas build --profile preview --platform android
```

Share the install link from the EAS dashboard with testers. Android installs via APK; iOS requires TestFlight or device registration.

---

## Detailed Setup by Package

### Backend (Express API)

```bash
cd packages/api

# Install dependencies
pnpm install

# Start development server (auto-reloads on file changes)
pnpm dev

# Run tests
pnpm test

# Run tests in watch mode
pnpm test:watch

# Generate test coverage report
pnpm test:coverage

# Build for production
pnpm build

# Start production server
pnpm start
```

**API runs on**: http://localhost:3000
**Health check**: http://localhost:3000/health
**API endpoints**: http://localhost:3000/api/*

---

### Frontend (Next.js)

```bash
cd packages/web

# Install dependencies
pnpm install

# Start development server (with hot reload)
pnpm dev

# Run tests (Jest + React Testing Library)
pnpm test

# Run tests in watch mode
pnpm test:watch

# Generate test coverage report
pnpm test:coverage

# Build for production
pnpm build

# Start production server
pnpm start

# Lint code
pnpm lint
```

**Web runs on**: http://localhost:3001
**Pages**: 
- Landing: http://localhost:3001
- Login: http://localhost:3001/login
- Signup: http://localhost:3001/signup
- Onboarding: http://localhost:3001/onboarding
- Dashboard: http://localhost:3001/dashboard
- Progress: http://localhost:3001/progress
- Settings: http://localhost:3001/settings
- Lesson: http://localhost:3001/lessons/[id]
- Quiz: http://localhost:3001/lessons/[id]/quiz

---

## Database Commands

```bash
# From root directory (learning/)

# Sync Prisma schema to database
pnpm db:push

# Seed database with initial data
pnpm db:seed

# Reset database (WARNING: deletes all data)
pnpm db:reset

# Open Prisma Studio (interactive database viewer)
pnpm prisma studio
```

---

## Docker Commands

```bash
# Start services
docker-compose up -d

# Stop services
docker-compose down

# View logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f postgres
docker-compose logs -f redis

# Rebuild containers
docker-compose up -d --build

# Remove containers and volumes (WARNING: deletes data)
docker-compose down -v
```

**Services**:
- PostgreSQL: localhost:5433
- Redis: localhost:6380
- Ollama (optional): localhost:11435

**Database Credentials**:
- User: `learning_user`
- Password: `learning_password`
- Database: `learning_app`

---

## Testing

### Run All Tests

```bash
# From root directory
pnpm test
```

### Backend Tests

```bash
cd packages/api

# Run tests once
pnpm test

# Run in watch mode (re-run on file change)
pnpm test:watch

# Generate coverage report
pnpm test:coverage
```

### Frontend Tests

```bash
cd packages/web

# Run tests once
pnpm test

# Run in watch mode
pnpm test:watch

# Generate coverage report
pnpm test:coverage
```

### Test Coverage Target

- **Backend**: 80%+ on critical paths (services, auth)
- **Frontend**: 80%+ on critical paths (hooks, api-client, auth)

---

## Build & Production

### Build All Packages

```bash
# From root directory
pnpm build
```

### Frontend Production Build

```bash
cd packages/web

# Generate optimized production build
pnpm build

# Test production build locally
pnpm start
```

Production build output in `packages/web/.next/`

### Backend Production Build

```bash
cd packages/api

# Compile TypeScript to JavaScript
pnpm build

# Start production server
pnpm start
```

Production build output in `packages/api/dist/`

---

## Code Formatting & Linting

```bash
# From root directory

# Format all code (Prettier)
pnpm format

# Frontend linting
cd packages/web && pnpm lint

# Frontend linting with auto-fix
cd packages/web && pnpm lint --fix
```

---

## Environment Variables

### Backend (`packages/api/.env.local`)

```
DATABASE_URL=postgresql://learning_user:learning_password@localhost:5433/learning_app
JWT_SECRET=your-secret-key-change-in-production
PORT=3000
NODE_ENV=development
```

### Frontend (`packages/web/.env.local`)

```
NEXT_PUBLIC_API_URL=http://localhost:3000/api
```

---

## Troubleshooting

### Port Already in Use

```bash
# Kill process on port 3000 (API)
lsof -ti:3000 | xargs kill -9

# Kill process on port 3001 (Web)
lsof -ti:3001 | xargs kill -9
```

### Database Connection Failed

```bash
# Verify Docker is running
docker ps

# Check database logs
docker-compose logs postgres

# Restart database
docker-compose restart postgres
```

### Clear Dependencies & Cache

```bash
# Remove node_modules and lock files
rm -rf node_modules pnpm-lock.yaml

# Reinstall everything
pnpm install
```

### TypeScript Errors

```bash
# Check for TypeScript errors
cd packages/web
pnpm tsc --noEmit

cd ../api
pnpm tsc --noEmit
```

---

## Full Development Workflow

### Terminal 1: Start Databases

```bash
docker-compose up -d
pnpm db:push
pnpm db:seed
```

### Terminal 2: Start API

```bash
cd packages/api
pnpm dev
```

### Terminal 3: Start Web

```bash
cd packages/web
pnpm dev
```

### Terminal 4: Run Tests (optional)

```bash
# Watch mode for either package
cd packages/web && pnpm test:watch
# or
cd packages/api && pnpm test:watch
```

---

## Checklist Before Commit

- [ ] `pnpm test` passes in all packages
- [ ] `pnpm build` succeeds in all packages
- [ ] `pnpm lint` passes (web only)
- [ ] No TypeScript errors: `tsc --noEmit`
- [ ] Code formatted: `pnpm format`
- [ ] Database migrations applied: `pnpm db:push`

---

## Useful Commands Summary

| Command | What It Does |
|---------|-------------|
| `pnpm install` | Install all dependencies |
| `docker-compose up -d` | Start database + Redis |
| `pnpm db:push` | Sync database schema |
| `pnpm db:seed` | Add sample data |
| `pnpm dev` | Start API + Web in parallel |
| `pnpm test` | Run all tests |
| `pnpm build` | Build all packages for production |
| `pnpm format` | Format all code |
| `docker-compose down` | Stop all services |
| `pnpm db:reset` | ⚠️ Delete all data and reset |

---

## Next Steps

1. ✅ Install dependencies: `pnpm install`
2. ✅ Start database: `docker-compose up -d`
3. ✅ Seed data: `pnpm db:push && pnpm db:seed`
4. ✅ Start servers: `pnpm dev`
5. ✅ Open http://localhost:3001
6. ✅ Test user flow: register → onboarding → lesson → quiz
7. ✅ Run tests: `pnpm test`
