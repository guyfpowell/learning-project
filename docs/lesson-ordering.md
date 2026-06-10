# Track Ordering — Implementation Spec

**Goal:** Give every track (Skill in the DB) a stable `order` field so the app can display tracks in the correct sequence regardless of insert order or DB implementation.

**Constraint:** `prisma/generated-lessons.json` has already been generated (672 lessons, AI for Product Managers + Product Strategy) and must not be regenerated.

**Approach:**
- `lesson_config.py` is the permanent source of truth for order going forward
- `seed.ts` uses a hardcoded fallback order map as a one-time bridge for the already-generated JSON (which has no `order` field)
- Future batch runs will carry `order` through from config → JSON → seed → DB automatically

---

## Desired track order

| order | Track name |
|---|---|
| 1 | Product Strategy |
| 2 | Discovery & Research |
| 3 | Execution & Delivery |
| 4 | Metrics & Analytics |
| 5 | Leadership & Influence |
| 6 | Stakeholder Management |
| 7 | Go-to-Market & Launch |
| 8 | Product Communication |
| 9 | AI for Product Managers |
| 10 | Product Design & UX for PMs |
| 11 | Technical Skills for PMs |

To change the order in future: update the `order` values in `lesson_config.py` and re-seed.

---

## Files to change

1. `prisma/schema.prisma` — add `order` field to `Skill`
2. `prisma/seed.ts` — read `order` from JSON, fall back to hardcoded map
3. `scripts/lesson_config.py` — add `"order"` to each track dict
4. `scripts/generate-lessons.py` — pass `order` through manifest → JSON output

---

## Step 1 — `prisma/schema.prisma`

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

After editing, run the migration:

```bash
pnpm --filter api prisma:migrate dev --name add-skill-order
```

---

## Step 2 — `prisma/seed.ts`

### 2a — Update the `GeneratedPath` interface

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

### 2b — Add the hardcoded fallback order map

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

### 2c — Use `order` in the skill upsert

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

## Step 3 — `scripts/lesson_config.py`

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

## Step 4 — `scripts/generate-lessons.py`

Three places to update so `order` flows from config into the output JSON.

### 4a — Batch mode: manifest in `build_batch_requests`

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

### 4b — Batch mode: `process_batch_results`

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

### 4c — Sync mode: structured topic path

In the sync mode inner loop where `state["results"][path_key]` is first created, add `order`. You'll also need to capture `track.get("order", 99)` in the `pending` tuple alongside `cat` and `desc`.

Current pending tuple (5 values after key):
```python
pending.append((
    topic, li, lesson_title, key,
    tname, level, track["category"], track["description"],
))
```

Updated (add order as 9th element):
```python
pending.append((
    topic, li, lesson_title, key,
    tname, level, track["category"], track["description"], track.get("order", 99),
))
```

Update `_gen` to unpack and return `order`, and when creating `state["results"][path_key]`:

```python
state["results"][path_key] = {
    "track":       tn,
    "category":    cat,
    "description": desc,
    "order":       ord_,    # ← add this (use a variable name that doesn't shadow builtins)
    "level":       lv,
    "lessons":     [],
}
```

---

## Order of operations

1. Edit `schema.prisma` (Step 1)
2. Run `pnpm --filter api prisma:migrate dev --name add-skill-order`
3. Edit `seed.ts` (Step 2)
4. Edit `lesson_config.py` (Step 3)
5. Edit `generate-lessons.py` (Step 4)
6. Run `pnpm --filter api prisma:seed`

Steps 3–5 can be done in any order. The migration (step 2) must happen before the seed (step 6).

---

## How it works going forward

- To change display order: update `order` values in `lesson_config.py`, re-seed
- To add a new track: add the track dict with `"order": N` to `lesson_config.py`, sequence its lessons, generate, re-seed
- The `TRACK_ORDER` fallback map in `seed.ts` should be kept up to date as new tracks are added, but once the generation script is writing `order` into the JSON it only acts as a safety net

---

## What the app needs to do

When fetching tracks/skills for display, sort by `order ASC`. If using Prisma:

```typescript
const skills = await prisma.skill.findMany({
  orderBy: { order: 'asc' },
});
```

Do not rely on insert order or alphabetical sort anywhere in the app.
