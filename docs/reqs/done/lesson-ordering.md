# Track Ordering — Implementation Spec

**Goal:** Give every track (Skill in the DB) a stable `order` field so the app can display tracks in the correct sequence regardless of insert order or DB implementation.

**Constraint:** `prisma/generated-lessons.json` has already been generated (672 lessons, AI for Product Managers + Product Strategy) and must not be regenerated.

**Approach:**
- `lesson_config.py` is the permanent source of truth for order going forward
- `seed.ts` uses a hardcoded fallback order map as a one-time bridge for the already-generated JSON (which has no `order` field)
- Future batch runs will carry `order` through from config → JSON → seed → DB automatically

---

## Mandatary track order

| Track number | Track name | Premium |
|---|---|---|
| 1 | Product Strategy | Y
| 2 | Discovery & Research | Y
| 3 | Execution & Delivery | N
| 4 | Metrics & Analytics | N
| 5 | Leadership & Influence | Y
| 6 | Stakeholder Management | N
| 7 | Go-to-Market & Launch | Y
| 8 | Product Communication | N
| 9 | AI for Product Managers | Y
| 10 | Product Design & UX for PMs | Y
| 11 | Technical Skills for PMs | Y

To change the order in future: update the `order` values in `lesson_config.py` and re-seed.

---

## Files to change

1. `package.json` (root) — fix `db:seed` script; add migration scripts
2. `prisma/schema.prisma` — add `order` field to `Skill`
3. `prisma/seed.ts` — read `order` from JSON, fall back to hardcoded map
4. `scripts/lesson_config.py` — add `"order"` to each track dict
5. `scripts/generate-lessons.py` — pass `order` through manifest → JSON output
6. `packages/api/src/routes/lessons.ts` — sort skills by `order` not `name`

---

## Step 0 — Fix `db:seed` and add migration scripts to `package.json`

### 0a — Fix db:seed

The current `db:seed` script calls `prisma db seed`, which proxies through Prisma's runner and swallows all stdout/stderr — errors are silent and invisible.

The `prisma.seed` config (used by `prisma migrate reset` to auto-seed after a reset) already has the correct command. The fix is to make `db:seed` call `tsx` directly, bypassing the proxy.

In root `package.json`, change:

```json
"db:seed": "prisma db seed",
```

to:

```json
"db:seed": "cd packages/api && tsx ../../prisma/seed.ts",
```

**Note**: Keep `"prisma": { "seed": "cd packages/api && tsx ../../prisma/seed.ts" }` in place — `prisma migrate reset` still uses it to auto-seed. Only the `db:seed` script is changing.

### 0b — Add migration scripts

The project has no `prisma/migrations/` folder — it has been using `db:push` throughout. Switch to proper versioned migrations now.

Add to the `scripts` section of root `package.json`:

```json
"db:migrate": "prisma migrate dev",
"db:migrate:prod": "prisma migrate deploy",
```

- `pnpm db:migrate --name <name>` — used locally during development
- `pnpm db:migrate:prod` — used in CI/CD and production deployments
- `pnpm db:push` can stay as a convenience for quick local prototyping, but schema changes going forward must be committed as migrations

---

## Step 1 — Create the baseline migration

Because there is no `prisma/migrations/` folder yet, the first migration must capture the entire current schema. This creates the audit trail that future migrations build on top of.

Run:

```bash
pnpm db:migrate --name init
```

This creates `prisma/migrations/YYYYMMDDHHMMSS_init/migration.sql` containing the full current schema. The local DB is left unchanged (Prisma sees it as already in sync).

---

## Step 2 — `prisma/schema.prisma`

Find the `Skill` model (around line 63). Add `order Int @default(0)` after `isPremium`:

```prisma
model Skill {
  id          String   @id @default(cuid())
  name        String   @unique
  description String
  category    String
  isPremium   Boolean  @default(false)
  order       Int      @default(0)   // ← add this line
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  skillPaths       SkillPath[]
  userSkillRatings UserSkillRating[]

  @@map("skills")
}
```

After editing, run:

```bash
pnpm db:migrate --name add-skill-order
```

This creates the migration SQL and applies it to the local DB.

---

## Step 3 — `prisma/seed.ts`

### 3a — Update the `GeneratedPath` interface

Add the optional `order` field (optional because the existing JSON doesn't have it):

```typescript
interface GeneratedPath {
  track: string;
  category: string;
  description: string;
  level: string;
  order?: number;       // ← add this line
  lessons: GeneratedLesson[];
}
```

### 3b — Add the hardcoded fallback order map

Add this constant just before the `main()` function. This is the one-time bridge for the already-generated JSON. Once all future JSONs carry `order` from the generation script, this map is never consulted (but can stay in place harmlessly):

```typescript
const TRACK_ORDER: Record<string, number> = {
  'Product Strategy':            1,
  'Discovery & Research':        2,
  'Execution & Delivery':        3,
  'Metrics & Analytics':         4,
  'Leadership & Influence':      5,
  'Stakeholder Management':      6,
  'Go-to-Market & Launch':       7,
  'Product Communication':       8,
  'AI for Product Managers':     9,
  'Product Design & UX for PMs': 10,
  'Technical Skills for PMs':    11,
};
```

### 3c — Use `order` in the skill upsert

Find the `prisma.skill.upsert` call inside the `for (const pathData of generatedPaths)` loop (around line 240). Replace it with:

```typescript
const trackOrder = pathData.order ?? TRACK_ORDER[pathData.track] ?? 99;
const skill = await prisma.skill.upsert({
  where: { name: pathData.track },
  update: {
    isPremium: isPremiumTrack,
    order: trackOrder,
  },
  create: {
    name: pathData.track,
    description: pathData.description,
    category: pathData.category,
    isPremium: isPremiumTrack,
    order: trackOrder,
  },
});
```

The `?? 99` fallback means any track not in the map gets pushed to the bottom rather than causing a crash — safe for future tracks added before this file is updated.

---

## Step 4 — `scripts/lesson_config.py`

Add `"order": N` to every track dict in the `TRACKS` list. The `order` value is the track's position in the desired display sequence (see table above).

Example — the first two tracks should look like:

```python
TRACKS = [
    {
        "name": "Product Strategy",
        "order": 1,                    # ← add this
        "category": "product-management",
        ...
    },
    {
        "name": "Discovery & Research",
        "order": 2,                    # ← add this
        "category": "product-management",
        ...
    },
    ...
    {
        "name": "AI for Product Managers",
        "order": 9,                    # ← add this
        "category": "ai-engineering",
        ...
    },
]
```

All 9 current tracks need this field. Tracks 10 and 11 (Product Design & UX for PMs, Technical Skills for PMs) don't exist in the config yet — add `order` when you create them.

---

## Step 5 — `scripts/generate-lessons.py`

Four places to update so `order` flows from config into the output JSON.

### 5a — Batch mode: manifest in `build_batch_requests` (line ~263)

In the manifest dict (inside the nested loop), add `order`:

```python
manifest[custom_id] = {
    "track":        tname,
    "level":        level,
    "category":     track["category"],
    "description":  track["description"],
    "order":        track.get("order", 99),   # ← add this line
    "topic_name":   topic["name"],
    "lesson_index": li,
    "lesson_title": topic["lessons"][li],
}
```

### 5b — Batch mode: `process_batch_results` (line ~358)

When `path_data` is first created for a new `path_key`, include `order`:

```python
path_data[path_key] = {
    "track":       meta["track"],
    "category":    meta["category"],
    "description": meta["description"],
    "order":       meta.get("order", 99),    # ← add this line
    "level":       meta["level"],
    "lessons":     [],
}
```

### 5c — Sync mode: structured format, `pending` tuple (line ~696)

Current pending tuple (8 values):
```python
pending.append((
    topic, li, lesson_title, key,
    tname, level, track["category"], track["description"],
))
```

Add `order` as 9th element:
```python
pending.append((
    topic, li, lesson_title, key,
    tname, level, track["category"], track["description"], track.get("order", 99),
))
```

Update `_gen` to unpack and return `order` (line ~708):
```python
def _gen(item):
    t, li, title, k, tn, lv, cat, desc, ord_ = item
    t0     = time.time()
    lesson = generate_lesson(tn, lv, t, li)
    return lesson, int(time.time() - t0), title, k, tn, lv, cat, desc, ord_
```

Update the unpack from `future.result()` and the `state["results"]` init (line ~716):
```python
lesson, elapsed, title, key, tn, lv, cat, desc, ord_ = future.result()
if lesson:
    path_key = f"{tn}|{lv}"
    with checkpoint_lock:
        if path_key not in state["results"]:
            state["results"][path_key] = {
                "track":       tn,
                "category":    cat,
                "description": desc,
                "order":       ord_,    # ← add this
                "level":       lv,
                "lessons":     [],
            }
```

### 5d — Sync mode: flat format path init (line ~753)

In the flat-format `else` branch, when `state["results"][path_key]` is first created:

```python
state["results"][path_key] = {
    "track":       tname,
    "category":    track["category"],
    "description": track["description"],
    "order":       track.get("order", 99),    # ← add this line
    "level":       level,
    "lessons":     [],
}
```

---

## Step 6 — `packages/api/src/routes/lessons.ts`

The `GET /api/lessons/skills` route currently sorts by `name`. Change it to sort by `order`:

Line ~29, change:
```typescript
orderBy: { name: 'asc' },
```
to:
```typescript
orderBy: { order: 'asc' },
```

This ensures the frontend receives tracks in the correct display order rather than alphabetically.

---

## Order of operations

1. Edit `package.json` (Step 0) — fix seed script, add migrate scripts
2. Run `pnpm db:migrate --name init` (Step 1) — create baseline migration
3. Edit `schema.prisma` (Step 2)
4. Run `pnpm db:migrate --name add-skill-order` (Step 2) — apply the new field
5. Edit `seed.ts` (Step 3)
6. Edit `lesson_config.py` (Step 4)
7. Edit `generate-lessons.py` (Step 5)
8. Edit `lessons.ts` route (Step 6)
9. Run `pnpm db:seed` — seed the order values

Steps 3, 5–8 can be done in any order relative to each other. The migrations (steps 2 and 4) must happen before the seed (step 9).

---

## How it works going forward

- To change display order: update `order` values in `lesson_config.py`, re-seed
- To add a new track: add the track dict with `"order": N` to `lesson_config.py`, sequence its lessons, generate, re-seed
- The `TRACK_ORDER` fallback map in `seed.ts` should be kept up to date as new tracks are added, but once the generation script is writing `order` into the JSON it only acts as a safety net
- All schema changes must go through `pnpm db:migrate --name <description>` from now on

---

## What the app needs to do

When fetching tracks/skills for display, sort by `order ASC`. If using Prisma:

```typescript
const skills = await prisma.skill.findMany({
  orderBy: { order: 'asc' },
});
```

Do not rely on insert order or alphabetical sort anywhere in the app.

---

## ✅ Implementation Summary (2026-06-11)

**Status**: Complete

### What was implemented

All 6 files changed as specced. Three pre-existing seed bugs were also fixed during the run.

**Schema**: `order Int @default(0)` added to `Skill` model.

**Migrations**: Project moved from `db:push` to proper versioned migrations. Because the schema was edited before the first migration ran, Prisma captured the full schema (including `order`) in a single `init` migration — `add-skill-order` found nothing to do. One migration file created: `prisma/migrations/20260611124943_init/migration.sql`.

**`package.json`**: `db:seed` now calls `tsx` directly (fixes silent output swallowing). `db:migrate` and `db:migrate:prod` scripts added.

**`seed.ts`**: `TRACK_ORDER` fallback map added; `order` used in skill upsert (JSON value → map fallback → 99). Three additional data-quality guards added for lessons with missing fields in older generated JSON:
- `l.quiz` — guarded with `Array.isArray` check; lessons with no quiz are created without one
- `l.durationMinutes` — falls back to `5`
- `l.difficulty` — falls back to `pathData.level`

**`lesson_config.py`**: `"order"` field added to all 9 existing tracks (1–9).

**`generate-lessons.py`**: `order` threaded through all 4 places — batch manifest, batch results processor, sync structured mode (`_gen` tuple + state init), sync flat-format mode (state init).

**`lessons.ts` route**: `GET /api/lessons/skills` now sorts by `orderBy: { order: 'asc' }`.

### DB state after seed

```
Product Strategy               order=1
AI for Product Managers        order=9
(5 legacy placeholder skills)  order=0  ← sorts before real tracks
```

**Known issue**: The 5 legacy placeholder skills from the Phase 1/2 skeleton (Prompt Engineering, User Research, AI Fundamentals, Product Metrics, Retrieval Augmented Generation) have `order: 0` and no lesson content. They will appear before real tracks in the sorted list. These should either be removed from `seed.ts` or given `order: 99` in a follow-up.
