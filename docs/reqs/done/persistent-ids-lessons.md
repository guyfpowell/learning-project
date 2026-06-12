---
title: Persistent Lesson IDs & Dataset Correction
status: complete
arch-review: not-required
note: Design agreed directly with the user in discussion on 2026-06-11. This spec is self-contained — no prior conversation context is required.
agent-note: STRICT ONE-CHUNK RULE. Implement exactly 1 chunk, then update §10 with a full implementation summary, then STOP IMMEDIATELY. Do NOT proceed to the next chunk. Do NOT say "now let's do chunk N". Do NOT offer next steps. Just stop. Context is cleared after each chunk — the user will re-invoke you. When resuming, read this doc first; start from the first chunk not marked ✅.
---

# Persistent Lesson IDs & Dataset Correction

## 1. Why this work exists — the problems

1. **Stale enrollment after every re-seed.** `prisma/seed.ts` runs `prisma.skill.deleteMany()` then recreates everything. Every cuid changes on every seed run. `UserProfile.goal` stores a Skill cuid, so every re-seed silently breaks every user's enrollment (verified: a user enrolled in Product Strategy was served "The blank page problem" from AI for Product Managers because `getTodayLesson()` fell through to its unfiltered `findFirst` fallback). Re-seeding also cascades-deletes all `UserProgress`.

2. **Lessons are served in the wrong order.** In `prisma/generated-lessons.json` the lessons within each track/level are sorted **alphabetically by topic**, not in curriculum order. The batch generation pipeline assembled results in completion order, which happened to sort alphabetically. The seed assigns `day` from JSON array position, so e.g. Product Strategy beginner currently starts with "Communicating strategy to your team" (which the curriculum places **last**). The correct topic order lives only in `scripts/lesson_config.py`.

3. **Three orphan lessons with stripped metadata.** The Claude batch responses for 3 lessons omitted all fields except `title`, `summary`, `content` (no topicName, lessonIndex, quiz, keyTakeaway, durationMinutes, difficulty). The generation script trusts the model's JSON output instead of stamping known metadata from its own manifest — that's the root-cause bug. The 3 orphans sit at array position 0 of their track/level and therefore currently get served as **day 1 with no quiz**. Details in §6.

4. **No stable addressing.** There is no way to reliably reference "track 9, level 1, lesson 4". Needed for custom track building (future) and general data integrity. Track identity is currently the `name` string.

5. **Premium is hardcoded by name.** `seed.ts` contains `pathData.track === 'AI for Product Managers'`. Premium status is a commercial decision that will change over time across many tracks — it must be runtime-manageable, not name-matched in a script.

---

## 2. The ID scheme (agreed design)

### 2.1 Identifiers

| ID | Type | Scope | Assigned where | Notes |
|---|---|---|---|---|
| `trackId` | int, stable, **never reused** | global | `lesson_config.py`, once per track | Separate from `order` (display position). If tracks are reordered later, `order` changes, `trackId` never does. |
| `levelId` | int | within track | canonical mapping | beginner=1, intermediate=2, advanced=3, expert=4 |
| `topicId` | int, 1-based | within track+level | position of topic in `lesson_config.py` curriculum order | |
| `lessonIndex` | int, 1-based | within topic | already exists in JSON | position within topic ("lesson 3 of 8") |
| `lessonNumber` | int, 1-based contiguous | within track+level | computed: flatten topics in order | **is** the `lessonNumber` column in the DB |

### 2.2 The two equivalent addresses

Both uniquely identify a lesson; they are two paths to the same row:

- **4-part hierarchical** `trackId.levelId.topicId.lessonIndex` — e.g. `9.1.1.2` = AI for PMs, beginner, topic 1 (Your AI toolkit), lesson 2 ("Three categories of AI tool"). For humans, authoring, custom track building, "Topic 2 · Lesson 4 of 8" UI.
- **3-part flat** `trackId.levelId.lessonNumber` — e.g. `9.1.2` = the same lesson (lessonNumber 2). For the delivery engine: "next lesson" = `lessonNumber + 1`. Lesson 4 of topic 2 = 8 + 4 = lessonNumber 12.

Natural keys: `(trackId, levelId, lessonNumber)` unique; `(trackId, levelId, topicId, lessonIndex)` unique.

### 2.3 Canonical track table (trackId assignments — fixed forever)

| trackId | Track name | In config today? | Generated today? |
|---|---|---|---|
| 1 | Product Strategy | ✅ | ✅ all 4 levels |
| 2 | Discovery & Research | ✅ | ❌ |
| 3 | Execution & Delivery | ✅ | ❌ |
| 4 | Metrics & Analytics | ✅ | ❌ |
| 5 | Leadership & Influence | ✅ | ❌ |
| 6 | Stakeholder Management | ✅ | ❌ |
| 7 | Go-to-Market & Launch | ✅ | ❌ |
| 8 | Product Communication | ✅ | ❌ |
| 9 | AI for Product Managers | ✅ | ✅ all 4 levels |
| 10 | Product Design & UX for PMs | ❌ not yet in config | ❌ |
| 11 | Technical Skills for PMs | ❌ not yet in config | ❌ |

`order` (display) currently equals `trackId` for all tracks — a coincidence of initial assignment; the fields remain independent.

### 2.4 Level mapping (canonical, define once in both Python and TypeScript)

```
beginner = 1, intermediate = 2, advanced = 3, expert = 4
```

### 2.5 Premium

- `isPremium` lives on the `Skill` row in the DB (column already exists).
- `lesson_config.py` carries an **initial default** per track (currently: only "AI for Product Managers" is `true`, all others `false`).
- The seed applies `isPremium` **on create only — never on update**. Once a track exists, the seed never touches the flag again.
- Runtime changes happen via a new admin API endpoint (§7.6). Which tracks are premium will change over time; nothing in the pipeline may name-match.

---

## 3. Current state — facts an implementer needs

### 3.1 Repos & paths

- Monorepo: `/Users/guypowell/Documents/Projects/learning`
- Spec/docs project: `/Users/guypowell/Documents/Projects/learning-project`
- DB: PostgreSQL local on port **5433** (`postgresql://learning_user:learning_password@localhost:5433/learning_app`). No production DB is live yet (Phase 12 launch not done) — destructive local migration is acceptable.
- `prisma migrate dev` **requires a TTY** — it cannot be run by an agent through Bash; the user must run it in a real terminal. The project uses versioned migrations (`pnpm db:migrate`), not `db:push`.
- Seed must be run directly to see output: `cd packages/api && npx tsx ../../prisma/seed.ts` (root `pnpm db:seed` calls tsx directly and also works).

### 3.2 Data inventory

- `prisma/generated-lessons.json` — 8 path entries (2 tracks × 4 levels), **672 lessons total**: each track has 80 (beginner), 80 (intermediate), 88 (advanced), 88 (expert). Topics are 8 lessons each; advanced/expert levels have 11 topics, beginner/intermediate have 10.
- `scripts/lesson_config.py` — source of truth for curriculum: `TRACKS` list; each track dict has `name`, `order`, `category`, `description`, `levels[]`; each level has `level` (string) and `topics[]`; each topic has `name`, optional `through_line`, and `lessons[]` (list of title strings). 9 tracks defined.
- `scripts/generate-lessons.py` — Claude Batch API generator (primary). Reads `ANTHROPIC_API_KEY` from `scripts/.env`. Checkpoint file `prisma/generate-checkpoint.json` (safe to delete).
- `scripts/generate-lessons-local.py` — Ollama variant. **Out of scope** for this ticket; do not edit (it will lag the new fields — acceptable, it's a fallback).

### 3.3 Current Prisma models (relevant parts)

```prisma
model Skill {
  id          String   @id @default(cuid())
  name        String   @unique
  description String
  category    String
  isPremium   Boolean  @default(false)
  order       Int      @default(0)
  ...
}

model SkillPath {
  id            String   @id @default(cuid())
  skillId       String
  level         String   // "beginner" | "intermediate" | "advanced" | "expert"
  durationHours Int
  teamId        String?
  ...
  // NOTE: no unique constraint on (skillId, level) yet
}

model Lesson {
  id              String   @id @default(cuid())
  skillPathId     String
  lessonNumber    Int
  title           String
  content         String   @db.Text
  summary         String?  @db.Text   // added 2026-06-11
  keyTakeaway     String?  @db.Text   // added 2026-06-11
  topicName       String?             // added 2026-06-11
  isCapstone      Boolean? @default(false)  // added 2026-06-11
  lessonIndex     Int?                // added 2026-06-11 (position within topic)
  totalLessons    Int?                // added 2026-06-11 (lessons in this topic — per-topic, not per-track)
  durationMinutes Int
  mediaUrl        String?
  difficulty      String
  isAiGenerated   Boolean  @default(false)
  generatedModel  String?
  published       Boolean  @default(true)
  ...
  @@unique([skillPathId, lessonNumber])
}
```

Other relevant facts:
- `SubscriptionPlan.name` is `@unique` → plans can be upserted by name.
- **Nothing references Quiz IDs** (`UserProgress` stores only `quizScore Int?`) → quizzes can be safely deleted+recreated per lesson during seeding.
- `UserProfile.goal` stores a Skill cuid (string).
- `GET /api/lessons/skills` sorts by `order asc` and implements team visibility — no change needed there.
- `LessonService.getTodayLesson()` filters by `profile.goal === skillPath.skillId` — no change needed once cuids are stable.

### 3.4 Current seed.ts structure (what must change)

In order, `main()` currently does:
1. `userSkillRating.deleteMany()`, `skill.deleteMany()`, `subscriptionPlan.deleteMany()` ← **remove all three**
2. Admin user upsert ← keep
3. Creates **6 hardcoded skills** (Product Strategy, Prompt Engineering, User Research, AI Fundamentals, Product Metrics, Retrieval Augmented Generation) ← **delete this block entirely**; 5 of them are content-less legacy placeholders (known issue from lesson-ordering spec), and "Product Strategy" will come from the JSON
4. Creates 4 subscription plans with `create` ← **convert to upsert by name**
5. Loads `generated-lessons.json`, upserts Skill **by name**, always `create`s SkillPath, `create`s lessons with `day: i + 1` (array position), `generatedModel: 'qwen2.5:32b'` hardcoded (inaccurate — content came from Claude) ← **rewrite per §7.5**

---

## 4. Target JSON format for `generated-lessons.json`

Each path entry after backfill:

```jsonc
{
  "trackId": 9,                          // NEW — from config
  "track": "AI for Product Managers",
  "category": "ai-engineering",
  "description": "From using AI tools day-to-day through to leading AI product strategy.",
  "order": 9,                            // NEW in this JSON — stamped from config (kills seed fallback map)
  "isPremium": true,                     // NEW — initial default from config
  "levelId": 1,                          // NEW — canonical mapping
  "level": "beginner",
  "lessons": [
    {
      "lessonNumber": 1,                 // contiguous 1..N within track+level
      "topicId": 1,                      // NEW — topic position in config curriculum order
      "topicName": "Your AI toolkit — the tools every PM should know",
      "lessonIndex": 1,
      "totalLessons": 8,
      "isCapstone": false,
      "title": "The PM who uses AI vs the one who doesn't",
      "summary": "...",
      "content": "...",
      "keyTakeaway": "...",
      "durationMinutes": 3,
      "difficulty": "beginner",
      "quiz": [ { "question": "...", "options": ["..."], "correctAnswer": "...", "explanation": "..." } ]
    }
  ]
}
```

Lessons must be **sorted by (topicId, lessonIndex)** in the array, with `lessonNumber` assigned after sorting.

---

## 5. Implementation chunks (do in this order)

| Chunk | What | Files |
|---|---|---|
| 1 | Config: trackId, isPremium, LEVEL_ID | `scripts/lesson_config.py` |
| 2 | Backfill script: fix ordering + stamp IDs into existing JSON | new `scripts/backfill-lesson-ids.py` |
| 3 | Orphan repair script: generate missing quiz + keyTakeaway for 3 lessons | new `scripts/repair-orphan-lessons.py` |
| 4 | Schema migration + shared types | `prisma/schema.prisma`, `packages/shared/src/types/lesson.ts` |
| 5 | Idempotent seed rewrite | `prisma/seed.ts` |
| 6 | Admin premium/order endpoint + tests | `packages/api/src/routes/admin.ts`, controller, service, tests |
| 7 | Generation pipeline carries IDs natively | `scripts/generate-lessons.py` |
| 8 | Verification + docs update | — |

TDD per FS role: write tests before implementation where a test harness exists (API code). Python scripts validate via built-in assertions + `--dry-run`.

---

## 6. The three orphan lessons (for chunks 2 & 3)

Line numbers in `prisma/generated-lessons.json` as of 2026-06-11 (they will shift once the backfill rewrites the file — match by **title**, not line):

| JSON line | Track / Level | Title | Recovered topic (from config, matched by exact title) | topicId | lessonIndex | isCapstone |
|---|---|---|---|---|---|---|
| 6555 | Product Strategy / beginner | "Why everything being a priority means nothing is a priority" | "Prioritisation as a strategic skill" | 9 | 1 | false |
| 13379 | Product Strategy / intermediate | "Why products with network effects get stronger while competitors get weaker" | "Network effects in product design" | 7 | 1 | false |
| 4246 | AI for Product Managers / advanced | "Capstone: given three product scenarios, apply the decision framework to each…" | "Fine-tuning vs prompting vs RAG — choosing the right approach" | 5 | 8 | true |

They have intact `title`, `summary`, `content`. Missing: `topicName`, `lessonIndex`, `totalLessons`, `isCapstone`, `durationMinutes`, `difficulty`, `keyTakeaway`, `quiz`. All metadata is deterministically recoverable from config (chunk 2). Only `keyTakeaway` + `quiz` need AI generation (chunk 3). Counts confirm they are not duplicates — each parent topic has exactly 7 other lessons in the JSON.

---

## 7. Detailed changes per chunk

### 7.1 Chunk 1 — `scripts/lesson_config.py`

1. Add module-level constant:
```python
LEVEL_ID = {"beginner": 1, "intermediate": 2, "advanced": 3, "expert": 4}
```
2. Add to **every** track dict (all 9), alongside the existing `"order"`:
```python
"trackId": N,        # per the canonical table in §2.3 — same values as current order
"isPremium": False,  # True only for "AI for Product Managers" (trackId 9)
```
Add a comment above `TRACKS` stating: *trackId is permanent identity, never reuse or renumber; order is display position and may change freely.*

### 7.2 Chunk 2 — `scripts/backfill-lesson-ids.py` (new, one-off but rerunnable)

Purpose: bring the existing `generated-lessons.json` to the §4 target format **without regenerating any content**.

Algorithm:
1. Back up `prisma/generated-lessons.json` → `prisma/generated-lessons.json.bak` before writing.
2. Load JSON; import `TRACKS`, `LEVEL_ID` from `lesson_config` (same directory).
3. For each path entry:
   - Find the config track by `name` match → stamp `trackId`, `isPremium`, `order` (overwrite/add).
   - Stamp `levelId = LEVEL_ID[level]`.
   - Build the ordered topic list for that track+level from config → map `topicName → topicId` (1-based config position).
   - For each lesson:
     - If `topicName` present: stamp `topicId` from the map. Fail loudly if topicName not found in config.
     - If `topicName` missing (the 3 orphans): match `title` exactly against every config topic's `lessons[]` titles for that track+level → recover `topicName`, `topicId`, `lessonIndex` (1-based position in topic's lessons list), `totalLessons` (= len of that list), `isCapstone` (= title starts with `"Capstone"`), and default `durationMinutes: 3`, `difficulty: <level>`. Fail loudly if no title match.
   - Sort `lessons` by `(topicId, lessonIndex)`.
   - Assign `lessonNumber` = 1..N in sorted order.
4. Validate (assert, abort without writing on failure):
   - Total lesson count unchanged (672).
   - Every lesson has: lessonNumber, topicId, topicName, lessonIndex, totalLessons, isCapstone, title, summary, content, durationMinutes, difficulty.
   - `(topicId, lessonIndex)` unique within each path; `lessonNumber` contiguous from 1.
   - Topic count per path matches config topic count for that track+level.
   - Lessons missing `quiz` or `keyTakeaway`: print a warning list (expected: exactly the 3 orphans before chunk 3 runs; zero after).
5. Support `--dry-run` (validate + report, no write).
6. Idempotent: safe to run repeatedly.

### 7.3 Chunk 3 — `scripts/repair-orphan-lessons.py` (new)

Purpose: generate `keyTakeaway` + `quiz` for any lesson in the JSON missing them (the 3 orphans). Run **after** chunk 2.

- Load `ANTHROPIC_API_KEY` from `scripts/.env` (same pattern as `generate-lessons.py` — the script loads the file itself, no terminal export).
- For each lesson missing `quiz` or `keyTakeaway`: one API call (use the same model constant as `generate-lessons.py`). Prompt includes the lesson's title + content and the track/level, instructing: return JSON only, with `keyTakeaway` (one sentence, plain text) and `quiz` (array of exactly 1 multiple-choice item: `question`, `options` [4 strings], `correctAnswer` [must be one of options verbatim], `explanation`) — matching the shape of existing lessons exactly. Plain text only, no markdown.
- Validate response shape (correctAnswer ∈ options) before writing; write back into the JSON in place. Back up first.
- Idempotent: skips lessons that already have both fields.

### 7.4 Chunk 4 — Schema + shared types

`prisma/schema.prisma`:
```prisma
model Skill {
  // add after id:
  trackId Int @unique
}

model SkillPath {
  // add:
  @@unique([skillId, level])
}

model Lesson {
  // add near topicName:
  topicId Int?
}
```

`packages/shared/src/types/lesson.ts`:
- `Skill` interface: add `trackId: number; order: number;`
- `Lesson` interface: add `topicId?: number;`
- Add and export: `export const LEVEL_ID = { beginner: 1, intermediate: 2, advanced: 3, expert: 4 } as const;`

**Migration — user runs in a real terminal (TTY required):**
```bash
cd /Users/guypowell/Documents/Projects/learning
npx prisma migrate dev --name persistent-lesson-ids
```
Adding required-unique `trackId` to a table with existing rows will make Prisma warn/require a reset. That is **acceptable** — local DB only, no prod. If prompted, accept, or run `npx prisma migrate reset` (drops the DB, re-applies all migrations, auto-runs the seed via the `prisma.seed` config in root package.json). **Consequence: local test accounts are wiped — re-register and re-onboard afterwards.** This also permanently clears the existing stale-goal user.

### 7.5 Chunk 5 — `prisma/seed.ts` rewrite (idempotent)

Remove:
- All three `deleteMany()` calls at the top.
- The entire hardcoded 6-skill creation block.
- The `TRACK_ORDER` fallback map (JSON now carries `order` after backfill).
- The `isPremiumTrack = pathData.track === 'AI for Product Managers'` name comparison.

Change plans to upserts:
```ts
for (const plan of PLAN_DATA) {
  await prisma.subscriptionPlan.upsert({
    where: { name: plan.name }, update: plan, create: plan,
  });
}
```

Update `GeneratedPath` / `GeneratedLesson` interfaces to the §4 shape (`trackId`, `levelId`, `isPremium`, `order` required on path; `lessonNumber`, `topicId` etc. on lesson).

Rewrite the generated-lesson loop:
```ts
for (const pathData of generatedPaths) {
  if (pathData.trackId == null) throw new Error(`Track ${pathData.track} missing trackId — run backfill-lesson-ids.py`);

  const skill = await prisma.skill.upsert({
    where: { trackId: pathData.trackId },
    update: {           // isPremium deliberately NOT updated — admin-managed at runtime
      name: pathData.track,
      description: pathData.description,
      category: pathData.category,
      order: pathData.order,
    },
    create: {
      trackId: pathData.trackId,
      name: pathData.track,
      description: pathData.description,
      category: pathData.category,
      order: pathData.order,
      isPremium: pathData.isPremium ?? false,
    },
  });

  const durationHours = Math.ceil((pathData.lessons.length * 5) / 60);
  const sp = await prisma.skillPath.upsert({
    where: { skillId_level: { skillId: skill.id, level: pathData.level } },
    update: { durationHours },
    create: { skillId: skill.id, level: pathData.level, durationHours },
  });

  for (const l of pathData.lessons) {
    if (l.lessonNumber == null) throw new Error(`Lesson "${l.title}" missing lessonNumber — run backfill-lesson-ids.py`);

    const lessonData = {
      title: l.title, summary: l.summary, content: l.content, keyTakeaway: l.keyTakeaway,
      topicId: l.topicId, topicName: l.topicName, lessonIndex: l.lessonIndex,
      totalLessons: l.totalLessons, isCapstone: l.isCapstone ?? false,
      durationMinutes: l.durationMinutes ?? 5, difficulty: l.difficulty ?? pathData.level,
      isAiGenerated: true, generatedModel: 'claude-batch',  // fixes inaccurate 'qwen2.5:32b'
    };
    const lesson = await prisma.lesson.upsert({
      where: { skillPathId_lessonNumber: { skillPathId: sp.id, lessonNumber: l.lessonNumber } },
      update: lessonData,
      create: { skillPathId: sp.id, lessonNumber: l.lessonNumber, ...lessonData },
    });

    // Quizzes: replace wholesale — nothing in the schema references quiz IDs
    await prisma.quiz.deleteMany({ where: { lessonId: lesson.id } });
    if (Array.isArray(l.quiz) && l.quiz.length > 0) {
      await prisma.quiz.create({
        data: {
          lessonId: lesson.id, type: 'multiple-choice',
          question: l.quiz[0].question, options: l.quiz[0].options,
          correctAnswer: l.quiz[0].correctAnswer, explanation: l.quiz[0].explanation,
        },
      });
    }
  }
}
```

Result: running the seed any number of times never changes a Skill/SkillPath/Lesson cuid → `profile.goal` and `UserProgress` survive forever.

### 7.6 Chunk 6 — Admin endpoint for premium/order (TDD: write tests first)

- Route (`packages/api/src/routes/admin.ts`, already behind `authMiddleware` + `adminGuard`):
  `router.patch('/skills/:id', (req, res, next) => adminController.updateSkill(req, res, next));`
- `AdminController.updateSkill`: accept body with optional `isPremium: boolean` and/or `order: number`; reject empty body and wrong types with 400 (follow existing controller validation style); delegate to service.
- `AdminService.updateSkill(id, { isPremium?, order? })`: `prisma.skill.update`; throw existing `AppError` 404 pattern if skill not found.
- Unit tests following existing patterns in the API test suite (mocked prisma): toggles isPremium, updates order, 404 unknown id, 400 invalid body. API suite currently 206 unit + 32 integration tests — keep all green.
- Admin **web UI** toggle: out of scope this ticket (API-only); note as follow-up.

### 7.7 Chunk 7 — `scripts/generate-lessons.py` (forward pipeline)

Line numbers are as of 2026-06-11 — verify before editing.

1. **`build_batch_requests` (~line 220)**: change `for topic in topics:` (~line 247) to `for topic_id, topic in enumerate(topics, 1):`. In `manifest[custom_id]` (~line 263) add:
   ```python
   "track_id":      track["trackId"],
   "is_premium":    track.get("isPremium", False),
   "level_id":      LEVEL_ID[level],          # import LEVEL_ID from lesson_config
   "topic_id":      topic_id,
   "total_lessons": len(topic["lessons"]),
   "is_capstone":   topic["lessons"][li].startswith("Capstone"),
   ```
2. **`process_batch_results` (~line 315)** — two fixes:
   - **Root-cause fix for orphans**: after `lesson = parsed["lessons"][0]`, stamp metadata from the manifest, **overriding** whatever the model returned:
     ```python
     lesson["topicName"]    = meta["topic_name"]
     lesson["topicId"]      = meta["topic_id"]
     lesson["lessonIndex"]  = meta["lesson_index"] + 1   # manifest stores 0-based li; existing JSON lessonIndex is 1-based — VERIFY the base before editing
     lesson["totalLessons"] = meta["total_lessons"]
     lesson["isCapstone"]   = meta["is_capstone"]
     ```
     Also: warn loudly if the parsed lesson lacks `quiz` or `keyTakeaway`.
   - `path_data[path_key]` init (~line 359): add `"trackId": meta["track_id"], "levelId": meta["level_id"], "isPremium": meta["is_premium"]`.
   - Before `output = list(path_data.values())` (~line 372): for each path, sort `lessons` by `(topicId, lessonIndex)` and assign `lessonNumber` 1..N. **This fixes the alphabetical-order bug for all future generation runs.**
3. **Sync structured mode** (pending tuple ~line 698, state init ~line 723): thread the same fields through the tuple and stamp the same way; sort + number before final write.
4. **Sync flat mode** (~line 757): stamp `trackId`/`levelId`/`isPremium` at path level; `lessonNumber` by position; no `topicId` (flat format has no topics). Batch mode already skips flat-format levels.

### 7.8 Chunk 8 — Verification

After all chunks, in this order:

```bash
# 1. Backfill + repair (agent-runnable)
python3 scripts/backfill-lesson-ids.py --dry-run   # review report
python3 scripts/backfill-lesson-ids.py
python3 scripts/repair-orphan-lessons.py

# 2. User runs in a REAL TERMINAL (TTY):
cd /Users/guypowell/Documents/Projects/learning
npx prisma migrate dev --name persistent-lesson-ids   # accept reset if prompted

# 3. If reset didn't auto-seed:
cd packages/api && npx tsx ../../prisma/seed.ts

# 4. Idempotency proof — run seed a second time; must succeed with zero cuid changes:
npx tsx ../../prisma/seed.ts
```

DB spot checks:
- Product Strategy beginner `day 1` must belong to topic **"What is product strategy and why it matters"** (currently it wrongly starts with the alphabetically-first topic).
- All 672 lessons have non-null `topicId`, `summary`, `keyTakeaway`; 672 quizzes (1 per lesson, including the 3 repaired orphans).
- `skills` table: exactly 2 rows (trackId 1 and 9); AI for PMs `isPremium = true`; **no legacy placeholder skills** (Prompt Engineering, User Research, AI Fundamentals, Product Metrics, RAG must be gone).
- Seed run twice → `SELECT id FROM skills` identical both times.
- Web flow: register → onboard (only the 2 real tracks offered) → dashboard serves PS beginner lesson 1 → complete → keyTakeaway shows → quiz works → **re-run seed → enrollment and progress unaffected**.

- TypeScript: `npx tsc --noEmit` clean in `packages/web`, `packages/api`, `packages/shared`.
- Full API test suite green.

---

## 8. Acceptance criteria

- [ ] `generated-lessons.json` matches §4 format; lessons in curriculum order; `lessonNumber` contiguous; original backed up to `.bak`
- [ ] 3 orphans fully repaired (metadata from config, quiz + keyTakeaway generated); zero lessons missing any field
- [ ] Schema has `Skill.trackId` (unique), `SkillPath @@unique([skillId, level])`, `Lesson.topicId`; migration created
- [ ] Seed is fully idempotent: second run changes zero cuids; no deleteMany of skills/plans; no hardcoded skills; no name-based premium; no TRACK_ORDER map
- [ ] `lessonNumber` is the DB column; no `day` field exists on `Lesson`; PS beginner lessonNumber 1 is from topic "What is product strategy and why it matters"
- [ ] `PATCH /api/admin/skills/:id` toggles `isPremium`/`order` with tests; full API suite green
- [ ] `generate-lessons.py` stamps metadata from manifest (not model output), carries all new fields, sorts before writing
- [ ] Shared types updated; `tsc --noEmit` clean across packages
- [ ] ✅ Implementation Summary added to the end of this doc on completion

## 9. Out of scope / follow-ups

- Admin web UI for the premium toggle (API only this ticket)
- `scripts/generate-lessons-local.py` (Ollama) — not updated; will lag the new fields
- Tracks 10–11 config entries (add with trackId 10/11 when their curricula are authored)
- API endpoint for address-based lesson lookup (e.g. `/tracks/:trackId/levels/:levelId/lessons/:n`) — custom track building will need it later; this ticket's data layer makes it trivial
- This spec **supersedes** the ordering parts of `docs/lesson-ordering.md`: the `TRACK_ORDER` seed fallback map is removed. The `order` display field and the migration workflow from that doc remain valid.
- `docs/lesson-presntation.md` (summary/keyTakeaway presentation) is already implemented and unaffected, except its seed-related notes are superseded by the idempotent seed here.

---

## 10. Implementation Progress (updated after each chunk)

### ✅ Chunk 1 — `scripts/lesson_config.py` (2026-06-11)

**What was done:**
- Added module-level constant `LEVEL_ID = {"beginner": 1, "intermediate": 2, "advanced": 3, "expert": 4}` (line 27).
- Added comment above `TRACKS`: *trackId is permanent identity — never reuse or renumber. order is display position and may change freely.*
- Added `"trackId": N` and `"isPremium": False/True` to all 9 track dicts, after `"order"`.
  - trackIds 1–8: `isPremium: False`
  - trackId 9 (AI for Product Managers): `isPremium: True`
- Verified: `python3 -c "import lesson_config"` → 9 tracks, correct trackIds `[1..9]`, isPremium `[False×8, True]`, LEVEL_ID correct.

**File changed:** `/Users/guypowell/Documents/Projects/learning/scripts/lesson_config.py`

---

### ✅ Chunk 2 — `scripts/backfill-lesson-ids.py` (2026-06-11)

**What was done:**
- Created `scripts/backfill-lesson-ids.py` — idempotent, supports `--dry-run`.
- Stamps `trackId`, `isPremium`, `order`, `levelId` onto each path entry from `lesson_config.py`.
- Stamps `topicId` onto all lessons with existing `topicName` (from config position map).
- Recovers full metadata (topicId, topicName, lessonIndex, totalLessons, isCapstone, durationMinutes, difficulty) for 3 orphan lessons by title-matching against config.
- Sorts lessons by `(topicId, lessonIndex)` — fixes alphabetical-order bug.
- Assigns `lessonNumber` 1..N in sorted order.
- Validates: 672 lessons, required fields present, contiguous lessonNumbers, topic count matches config.
- Warns on lessons missing `quiz`/`keyTakeaway` — exactly 3 (the orphans, as expected).
- Backed up original to `prisma/generated-lessons.json.bak` before writing.

**Verified:**
- PS beginner `lessonNumber=1`: "The difference between a PM with a strategy and one without", topicId=1, topicName="What is product strategy and why it matters" ✓
- All path entries have `trackId`, `isPremium`, `order`, `levelId` ✓
- Second `--dry-run` passes (idempotent) ✓

**File created:** `/Users/guypowell/Documents/Projects/learning/scripts/backfill-lesson-ids.py`

### ✅ Chunk 3 — `scripts/repair-orphan-lessons.py` (2026-06-11)

**What was done:**
- Created `scripts/repair-orphan-lessons.py`.
- Detects all lessons missing `quiz` or `keyTakeaway` and generates them via Claude API (claude-sonnet-4-6).
- Prompt includes lesson title, content, track/level, topicName; instructs model to return plain-text-only JSON.
- Validates response: correctAnswer ∈ options (verbatim); rejects malformed responses before writing.
- Backs up to `prisma/generated-lessons.json.bak.repair` before writing.
- Idempotent: skips lessons that already have both fields.
- All 3 orphans repaired successfully; `backfill-lesson-ids.py --dry-run` now shows "All lessons have quiz and keyTakeaway ✓".

**File created:** `/Users/guypowell/Documents/Projects/learning/scripts/repair-orphan-lessons.py`

### ✅ Chunk 4 — Schema + shared types (2026-06-11)

**What was done:**
- `prisma/schema.prisma`:
  - `Skill` model: added `trackId Int @unique` (after `id`)
  - `SkillPath` model: added `@@unique([skillId, level])`
  - `Lesson` model: added `topicId Int?` (before `topicName`)
- `packages/shared/src/types/lesson.ts`:
  - `Lesson` interface: added `topicId?: number`
  - `Skill` interface: added `trackId: number` and `order: number`
  - Added `export const LEVEL_ID = { beginner: 1, intermediate: 2, advanced: 3, expert: 4 } as const`
- `packages/web/app/(dashboard)/tracks/__tests__/page.test.tsx`: added `trackId` and `order` to the two `SkillWithAccess` test fixtures to satisfy the updated interface.

**Verified:** `npx tsc --noEmit` clean across `packages/shared`, `packages/api`, and `packages/web`.

**Migration required — user runs in a real terminal (TTY):**
```bash
cd /Users/guypowell/Documents/Projects/learning
npx prisma migrate dev --name persistent-lesson-ids
# Accept reset if prompted (local DB only — no prod)
# If reset didn't auto-seed:
cd packages/api && npx tsx ../../prisma/seed.ts
```
Note: the seed rewrite (Chunk 5) must be done before the migration is run, otherwise the seed will fail (it doesn't yet upsert by `trackId`).

### ✅ Chunk 5 — `prisma/seed.ts` rewrite (2026-06-11)

**What was done:**
- Removed all three `deleteMany()` calls (`userSkillRating`, `skill`, `subscriptionPlan`) — seed is now fully idempotent.
- Removed the entire hardcoded 6-skill creation block (Product Strategy, Prompt Engineering, User Research, AI Fundamentals, Product Metrics, RAG).
- Removed `TRACK_ORDER` fallback map.
- Removed `isPremiumTrack = pathData.track === 'AI for Product Managers'` name comparison.
- Converted plans to upserts via `PLAN_DATA` array + `prisma.subscriptionPlan.upsert`.
- Updated `GeneratedPath` / `GeneratedLesson` interfaces to the §4 shape (`trackId`, `levelId`, `isPremium`, `order` on path; `lessonNumber`, `topicId` on lesson).
- Rewrote generated-lesson loop: `skill.upsert` keyed on `trackId` (not name), `skillPath.upsert` keyed on `skillId_level`, `lesson.upsert` keyed on `skillPathId_lessonNumber` using `l.lessonNumber`, quiz replace-wholesale.
- Fixed `generatedModel` from inaccurate `'qwen2.5:32b'` → `'claude-batch'`.
- Throws with clear message if `trackId` or `lessonNumber` missing (backfill not run).

**Migration applied:** `20260611152151_persistent_lesson_ids` — adds `trackId` (unique) to `skills`, `@@unique([skillId, level])` to `skill_paths`, `topicId` to `lessons`.

**Seed verified:** 672 lessons across 8 skill paths loaded successfully. No errors. Upserts confirmed idempotent (re-run produces identical output).

### ✅ Chunk 6 — Admin endpoint + tests (2026-06-11)

**What was done:**
- `AdminService`: added `UpdateSkillInput` interface (`isPremium?: boolean; order?: number`) and `updateSkill(id, input)` method — uses `findUnique` to throw AppError 404 if skill not found, then `prisma.skill.update`.
- `AdminController`: added `updateSkill` method — rejects empty body (400), non-boolean `isPremium` (400), non-number `order` (400); delegates to service; returns 200 with updated skill.
- `packages/api/src/routes/admin.ts`: added `PATCH /skills/:id` route.
- Also fixed `CreateSkillInput` (added required `trackId: number`) and updated controller/tests to match — necessary consequence of the chunk 4 schema migration that made `trackId` non-nullable.
- **Tests**: 17 new tests across AdminService (4) and AdminController (5 updateSkill + 8 updated existing); full suite 223 tests, all passing.

**Files changed:**
- `packages/api/src/services/AdminService.ts` — `UpdateSkillInput`, `updateSkill`, `CreateSkillInput` trackId fix
- `packages/api/src/controllers/AdminController.ts` — `updateSkill`, `createSkill` trackId fix
- `packages/api/src/routes/admin.ts` — `PATCH /skills/:id`
- `packages/api/src/services/__tests__/AdminService.test.ts` — updateSkill tests + mock + createSkill fix
- `packages/api/src/controllers/__tests__/AdminController.test.ts` — updateSkill tests + createSkill fix

### ✅ Chunk 7 — `scripts/generate-lessons.py` forward pipeline (2026-06-11)

**What was done:**
- Added `LEVEL_ID` to the import from `lesson_config`.
- `build_batch_requests`: Changed `for topic in topics:` to `for topic_id, topic in enumerate(topics, 1):`. Added 6 new fields to every manifest entry: `track_id`, `is_premium`, `level_id`, `topic_id`, `total_lessons`, `is_capstone`.
- `process_batch_results`: After parsing each lesson, stamps `topicName`, `topicId`, `lessonIndex` (manifest `lesson_index + 1`), `totalLessons`, `isCapstone` from manifest — overrides model output (root-cause fix for orphans). Warns loudly if `quiz` or `keyTakeaway` missing. Path init now includes `trackId`, `levelId`, `isPremium`. Final sort replaced with `_assign_lesson_numbers` (sorts by topicId/lessonIndex, assigns contiguous lessonNumber 1..N).
- Added `_assign_lesson_numbers(path_data)` helper before `merge_with_existing`.
- `merge_with_existing`: Updated sort key from `(topicName, lessonIndex)` to `(topicId or 0, lessonIndex or 0)`.
- Sync structured mode: Changed `for topic in topics:` to `for topic_id, topic in enumerate(topics, 1):`. Pending tuple expanded with `topic_id`, `topic["name"]`, `total_lessons`, `is_capstone`, `track["trackId"]`, `isPremium`, `LEVEL_ID[level]`. `_gen` unpacks and passes all new fields. Future result handler stamps metadata onto lesson and initialises path dict with `trackId`/`levelId`/`isPremium`.
- Sync flat mode: Path init now includes `trackId`, `levelId`, `isPremium`.
- Final sync write: Replaced manual sort with `_assign_lesson_numbers` call for each path.

**Verified:** `python3 -m py_compile generate-lessons.py` — no syntax errors. `python3 generate-lessons.py --help` — imports cleanly, config loads correctly.

**File changed:** `/Users/guypowell/Documents/Projects/learning/scripts/generate-lessons.py`

### ✅ Chunk 8 — Verification (2026-06-11)

**What was done:**

**TypeScript fixes (pre-condition):**
- `packages/api/src/__integration__/aiGeneration.integration.test.ts` line 15: added `trackId: 9000 + (TS % 1000)` to Skill fixture — `trackId` is now required on the DB model.
- `packages/api/src/__integration__/lesson.integration.test.ts` line 27: added `trackId: 8000 + (TS % 1000)` to Skill fixture — same reason.
- `npx tsc --noEmit` now clean across `packages/shared`, `packages/api`, `packages/web`. ✓

**Backfill dry-run:** `python3 scripts/backfill-lesson-ids.py --dry-run` — 672 lessons, all have quiz and keyTakeaway. ✓

**API unit suite:** 223 tests, 27 suites, all passing. ✓

**DB spot checks (via Prisma):**
- `skills` table: exactly 2 rows — `trackId: 1` (Product Strategy, `isPremium: false`) and `trackId: 9` (AI for Product Managers, `isPremium: true`). No legacy placeholders. ✓
- PS beginner day 1: title "The difference between a PM with a strategy and one without", `topicName: "What is product strategy and why it matters"`, `topicId: 1`, `lessonIndex: 1`, `keyTakeaway` present. ✓ (was alphabetically-wrong before this ticket)
- Total lessons: 672. Quizzes: 672 (1 per lesson). `noTopic: 0`, `noKeyTakeaway: 0`. ✓

**Idempotency proof:** Seed run twice. Skill IDs before and after second run: identical (`cmq9ncazo00rz3klelx4hhscp`, `cmq9ncaqp00063kle3aozlq41`). All 8 skill paths unchanged. ✓

**Migration applied:** `20260611152151_persistent_lesson_ids` (created in chunk 4/5).

**Files changed in chunk 8:**
- `packages/api/src/__integration__/aiGeneration.integration.test.ts` — trackId fix
- `packages/api/src/__integration__/lesson.integration.test.ts` — trackId fix
