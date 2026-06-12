# 005 — Robustify Lesson Generation

**Status**: draft  
**Arch review**: required - pending  
**Author**: Guy Powell  
**Date**: 2026-06-12

---

## Background & Root Cause

A single batch parse failure silently dropped one lesson. Nothing hard-failed. The lesson simply wasn't in the output, `lessonNumber` was assigned to whatever succeeded, and the "✅ Batch complete!" banner printed normally. Fixing it cost a full working session.

Full post-mortem: `docs/reqs/004-fixed-missed-lesson.md`.

The root cause is architectural: **the pipeline builds its output structure from what comes back, not from what it intended to send.** Numbers are assigned after generation based on results. A failure doesn't leave a gap — it disappears and the numbers close over it.

---

## Architectural Principle

**Declare the full curriculum upfront. Then fill the slots.**

Everything that will ever exist is already known from `lesson_config.py` before a single API call is made: every track, level, topic, lesson, its `lessonNumber`, its `topicId`, its `lessonIndex`. This is fully calculable from the config.

The pipeline must change from:
> "generate what you can, number what you got"

To:
> "build the skeleton from config once, then fill it — a failure holds its place, it does not disappear"

---

## Clean Separation of Concerns

Generation and seeding are **completely independent processes**. One generation run does not equal one seed run. The handoff between them is `generated-lessons.json` — a clean, complete file written explicitly by `--export`. The seed knows nothing about the generation pipeline, lesson status, or the skeleton.

- **Generation pipeline** (`generate-lessons.py`): config → skeleton → fill slots → export clean JSON
- **Seed** (`seed.ts`): reads `generated-lessons.json`, loads DB — knows nothing about how the file was produced

---

## The Sacred Artefact: `prisma/generated-lessons.json`

`prisma/generated-lessons.json` represents hundreds of person-hours of curriculum design and millions of tokens of paid AI generation. It is **irreplaceable**. Its contents must never be lost, damaged, or corrupted under any circumstances.

The following rules are absolute and must be enforced in code:

1. **No command in `generate-lessons.py` reads from or writes to `generated-lessons.json` except `--export` and `--migrate` (read-only).** Every other command — `--init`, `--generate`, `--one`, `--status`, `--validate` — must never touch this file.

2. **`--migrate` is read-only with respect to `generated-lessons.json`.** It reads the file to build the skeleton. It does not modify, truncate, or overwrite it under any circumstances.

3. **`--export` writes with full backup protection.** Before writing, it copies the existing `generated-lessons.json` to `prisma/backups/generated-lessons-<YYYY-MM-DD-HHMMSS>.json`. It then writes to a temporary file (`generated-lessons.tmp.json`), validates it (valid JSON, non-zero lesson count, no null content fields), and only then atomically replaces `generated-lessons.json`. If any step fails, the backup is intact and the original is untouched.

4. **`prisma/backups/` is gitignored** (backups are large and local) but the directory must be created automatically by `--export`. The current `generated-lessons.json` is committed to git and is the primary backup.

5. **The skeleton (`lesson-skeleton.json`) is the working file.** All generation activity — tracking status, recording failures, filling content — happens in the skeleton. `generated-lessons.json` is only ever written when the operator explicitly decides the curriculum is ready to seed.

---

## New File: `prisma/lesson-skeleton.json`

The single source of truth for what the curriculum is and what the generation status of each lesson is. **Committed to git** — it is an important record and the repo is private.

### How it is created

`--init` (new command) reads `lesson_config.py` and builds the complete skeleton. Every lesson across every track, level, topic gets a slot. `lessonNumber` is assigned at this point — once, from the sorted position of `(topicId asc, lessonIndex asc)` within the path — and **never touched again**.

### Schema per lesson slot

```json
{
  "track": "Discovery & Research",
  "level": "advanced",
  "category": "product-management",
  "description": "...",
  "order": 3,
  "trackId": 7,
  "levelId": 3,
  "isPremium": false,
  "topicName": "Democratising research across teams",
  "topicId": 6,
  "lessonIndex": 6,
  "totalLessons": 8,
  "isCapstone": false,
  "lessonNumber": 46,
  "status": "pending",
  "error": null,
  "generatedAt": null,
  "title": null,
  "summary": null,
  "content": null,
  "keyTakeaway": null,
  "durationMinutes": null,
  "difficulty": null,
  "quiz": null
}
```

**`status`** is the operating field:
- `"pending"` — not yet attempted
- `"complete"` — all content fields populated
- `"failed"` — last attempt failed; `error` field holds the reason; retried on next `--generate` run

`lessonNumber` is set once at `--init` time. Generation never writes to it.

### File structure

```json
{
  "schema_version": 1,
  "initialised_at": "2026-06-12T...",
  "paths": [
    {
      "track": "...",
      "level": "...",
      "trackId": 1,
      "levelId": 1,
      "category": "...",
      "description": "...",
      "order": 1,
      "isPremium": false,
      "lessons": [ ...slots... ]
    }
  ]
}
```

---

## `generated-lessons.json`

Written only by `--export`, with full backup protection as described above. Contains only `status: "complete"` lessons, in the same format the seed already reads. Format is unchanged — the seed requires no changes.

This file is the handoff artefact between generation and seeding. Once written, it is the seed's concern only. It has no `status` fields, no `error` fields, no skeleton metadata — just clean lesson data. It is committed to git.

---

## Commands

### `--init`

Builds `lesson-skeleton.json` from `lesson_config.py`.

- Computes `lessonNumber` for every lesson: sort each path by `(topicId asc, lessonIndex asc)`, assign 1..N. Written once. Never changed.
- If `lesson-skeleton.json` already exists: adds new slots for any lessons in config not already present (matched by `track|level|topicName|lessonIndex`). **Never modifies existing slots. Never changes an existing `lessonNumber`.**
- Prints a summary: N paths, N lessons total, N already complete, N pending.

**Note on mid-curriculum config additions**: if new lessons are inserted into the middle of an existing topic sequence (shifting surrounding `lessonNumber` values), `--init` will flag this as a warning and require a deliberate migration. Out of scope for this ticket.

### `--generate` (default)

Iterates all `status: "pending"` and `status: "failed"` slots and generates content.

- Batch and sync modes work as today but pull their work list from the skeleton.
- On success: populate all content fields, set `status: "complete"`, set `generatedAt`.
- On failure: set `status: "failed"`, set `error`. The slot stays. Its `lessonNumber` is untouched.
- Writes the skeleton incrementally — the skeleton is the checkpoint. `generate-checkpoint.json` and `batch-checkpoint.json` are retired.
- At end of run: if any `status: "failed"` slots exist, print a table (track | level | topic | lessonIndex | title | error) and exit non-zero. Does not auto-export.

### `--export`

An explicit, deliberate step. Writes `generated-lessons.json` from all `status: "complete"` lessons.

- Refuses to run if any `status: "pending"` or `status: "failed"` slots exist — prints the gap list and exits non-zero.
- On success: writes `generated-lessons.json` and prints a count summary.

Seeding is then a separate, independent action: `pnpm db:seed`.

### `--one --track "..." --level "..." --topic "..." --lesson-index N`

Targets one specific slot for repair or retry.

- Looks up the slot by `(track, level, topicName, lessonIndex)`.
- Confirms it is `pending` or `failed`. If already `complete`, prints a warning and requires `--force`.
- Generates via Claude API (sync, single request).
- On success: fills content fields, sets `status: "complete"`. No renumbering. No surgery on any other slot.
- On failure: updates `error`, leaves `status: "failed"`.

### `--status`

Prints a summary of the skeleton: complete / pending / failed counts per path, and a list of any failed or pending lessons with their errors.

### `--validate`

Checks the skeleton for integrity:
- `lessonNumber` sequence per path is contiguous 1..N.
- No duplicate `(trackId, levelId, lessonNumber)` combinations.
- All `complete` lessons have non-null content fields.
- Warns (does not error) on missing `quiz` or `keyTakeaway` in complete lessons.

---

## Batch Mode Changes

The batch manifest is now redundant as a separate file — the skeleton holds all metadata. The batch checkpoint shrinks to:

```json
{
  "batch_id": "msgbatch_...",
  "submitted_at": "...",
  "model": "claude-opus-4-8",
  "custom_id_map": {
    "req_00312": "Discovery & Research|advanced|Democratising research across teams|6"
  }
}
```

`custom_id_map` maps each submitted request to its stable slot key. On result processing, each result is looked up in the map and the slot updated in place in the skeleton.

Batch checkpoint is **archived** (not deleted) on completion to `prisma/batch-archive/batch-<batch_id>-<date>.json`.

---

## Migration: existing `generated-lessons.json`

A one-off `--migrate` command converts existing data to the new skeleton format:

1. Reads `generated-lessons.json`.
2. For each lesson, creates a skeleton slot with `status: "complete"`, preserving all existing field values including `lessonNumber` **exactly as stored — no recomputation**.
3. For any lesson in `lesson_config.py` not present in `generated-lessons.json`, creates a `status: "pending"` stub — surfacing any existing gaps.
4. Writes `lesson-skeleton.json`.
5. Does not modify `generated-lessons.json`.

---

## Seed Changes

None. The seed continues to read `generated-lessons.json` exactly as today. It has no knowledge of the skeleton, generation status, or this pipeline. `--export` is the gate that ensures only valid, complete lessons reach it.

---

## Implementation Order

1. **`--init` and skeleton format** — the foundation
2. **`--migrate`** — converts existing data, unblocks testing with real content
3. **`--generate`** — updated to read/write skeleton; replaces checkpoint system
4. **`--export`** — the gate before seeding
5. **`--one`** — repair command
6. **`--status` and `--validate`** — observability
7. **Retire `generate-checkpoint.json` and `batch-checkpoint.json`**

---

## Acceptance Criteria

- [ ] `--init` produces a skeleton with correct `lessonNumber` for every lesson; running it again adds new lessons without changing any existing slot
- [ ] A simulated batch failure leaves the lesson slot as `status: "failed"` with its `lessonNumber` intact; surrounding lessons are unaffected
- [ ] Re-running `--generate` retries only `failed` and `pending` slots
- [ ] `--generate` exits non-zero and prints a gap list if any slots remain `failed` after a run
- [ ] `--export` refuses to run with any pending/failed slots and prints the gap list
- [ ] `--export` succeeds and writes a valid `generated-lessons.json` when all slots are complete
- [ ] `--one` fills a specific failed/pending slot without touching any other slot
- [ ] `--migrate` produces a skeleton where all existing lessons are `status: "complete"` with original `lessonNumber` values preserved exactly
- [ ] The seed works correctly after migration with no changes to `seed.ts`
- [ ] `generated-lessons.json` written by `--export` is structurally identical to today's file
- [ ] `batch-checkpoint.json` and `generate-checkpoint.json` no longer written during normal runs

---

## Files Changed

| File | Change |
|------|--------|
| `scripts/generate-lessons.py` | New commands: `--init`, `--migrate`, `--export`, `--one`, `--validate`, `--status`; generate reads/writes skeleton; batch checkpoint simplified |
| `prisma/lesson-skeleton.json` | New — created by `--init` / `--migrate`; committed to git |
| `prisma/generated-lessons.json` | Written by `--export` only; format unchanged |
| `prisma/generate-checkpoint.json` | Retired |
| `prisma/batch-checkpoint.json` | Retired (replaced by slimmer file; archived on completion) |
| `prisma/batch-archive/` | New directory (gitignored) |
| `prisma/seed.ts` | No changes |
| `.gitignore` | Add `prisma/batch-archive/` only |
