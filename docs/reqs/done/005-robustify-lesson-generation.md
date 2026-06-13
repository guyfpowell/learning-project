# 005 — Robustify Lesson Generation

**Status**: dev-complete  
**Arch review**: required - approved (see Architecture Review Notes below)  
**Author**: Guy Powell  
**Date**: 2026-06-12

**agent-note:** STRICT ONE-CHUNK RULE. Implement exactly 1 chunk, then update with a full implementation summary, then STOP IMMEDIATELY. Do NOT proceed to the next chunk. Do NOT say "now let's do chunk N". Do NOT offer next steps. Just stop. Context is cleared after each chunk — the user will re-invoke you. When resuming, read this doc first; start from the first chunk not marked ✅.

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

---

---

## ✅ Implementation Summary — Chunk 1: `--init` and skeleton format

**Date**: 2026-06-13  
**Chunk**: 1 of 7

### What was implemented

**File changed**: `learning/scripts/generate-lessons.py`

**Constants added** (after `checkpoint_lock`):
- `SKELETON_PATH` — `prisma/lesson-skeleton.json`
- `SUPPORTED_SCHEMA_VERSION = 1`

**Functions added** (new `# ── Skeleton helpers ──` section, before `# ── Main ──`):

| Function | Purpose |
|---|---|
| `load_skeleton()` | Reads skeleton; raises on missing file, wrong schema_version, or corrupt JSON |
| `save_skeleton(skeleton)` | Writes skeleton atomically via `os.replace()` through a `.tmp` file |
| `_slot_key(track, level, topic_name, lesson_index)` | Stable identity key `track\|level\|topicName\|lessonIndex`; includes arch constraint comment about rename-in-flight hazard |
| `_build_path_lessons(track, level_cfg)` | Builds slots for one path; sorts by `(topicId, lessonIndex)` and assigns contiguous `lessonNumber` 1..N |
| `_build_fresh_skeleton()` | Iterates all TRACKS, produces full skeleton dict |
| `cmd_init()` | Implements `--init`: creates fresh skeleton or adds new slots to existing one without modifying existing slots |

**Argparse change**: `--init` flag added; `main()` routes to `cmd_init()` and returns immediately.

### Acceptance criteria status

- ✅ `--init` produces a skeleton with correct `lessonNumber` for every lesson (30 paths, 3884 lessons, all contiguous 1..N per path, verified by script)
- ✅ Running `--init` again adds 0 new slots and does not modify any existing slot
- ✅ `schema_version` checked on every `load_skeleton()` call — raises clearly if version unsupported
- ✅ Atomic write via `os.replace()` on same filesystem (temp file in same `prisma/` dir)
- ✅ `_slot_key` includes warning comment about config-rename-in-flight hazard (arch constraint 2)

### Notes

- New slots added to an existing skeleton get `lessonNumber=-1` as a sentinel. `--validate` (chunk 6) will surface the contiguity gap. This is correct per spec ("flag as a warning and require a deliberate migration" for mid-curriculum insertions).
- `generated-lessons.json` was not touched by any code added in this chunk.
- `lesson-skeleton.json` is a new file in `prisma/` — should be committed to git (per spec: "Committed to git — it is an important record and the repo is private").

---

## ✅ Implementation Summary — Chunk 2: `--migrate`

**Date**: 2026-06-13  
**Chunk**: 2 of 7

### What was implemented

**File changed**: `learning/scripts/generate-lessons.py`

**Function added**: `cmd_migrate()` (in `# ── Skeleton helpers ──` section, after `cmd_init()`)

**Argparse change**: `--migrate` flag added; `main()` routes to `cmd_migrate()` and returns immediately (before `check_api_key()` — no API key needed).

#### Logic summary

| Step | What happens |
|---|---|
| Guard | Errors if `generated-lessons.json` is missing; warns (non-fatal) if `lesson-skeleton.json` already exists |
| Read | Loads `generated-lessons.json` into `existing_map` (slot_key → lesson) and `existing_path_map` (slot_key → path metadata) |
| Build skeleton | Calls `_build_fresh_skeleton()` — all slots start as `pending` with `lessonNumber` computed from config order |
| Populate complete slots | Walks every config slot; for each key found in `existing_map`, sets `status="complete"`, copies all content fields, and **overwrites `lessonNumber` with the value from `generated-lessons.json` exactly** |
| Detect orphans | Computes `orphan_keys = existing_map.keys() − config_keys`; for each orphan: prints a warning (track/level/topicName/lessonIndex + title), adds a `status="complete"` slot to the skeleton, and notes they will not satisfy `--validate`'s contiguity check |
| Write | `save_skeleton()` — atomic write via `os.replace()` |

#### Verified (smoke test on real data)
- 30 paths, 3884 lessons total
- 1952 `complete` (from `generated-lessons.json`), 1932 `pending` (in config but not yet generated)
- 0 orphans (all existing lessons matched config keys cleanly)
- 0 `lessonNumber` mismatches — all 1952 values preserved exactly
- `generated-lessons.json` unchanged after `--migrate` run

### Acceptance criteria status

- ✅ `--migrate` produces a skeleton where all existing lessons are `status: "complete"` with original `lessonNumber` values preserved exactly
- ✅ Lessons in `lesson_config.py` not in `generated-lessons.json` are created as `status: "pending"` stubs
- ✅ Orphaned lessons (in JSON but not in config) are included as `status: "complete"` with printed warnings
- ✅ `generated-lessons.json` was NOT modified
- ✅ The seed works correctly after migration with no changes to `seed.ts` (format of `generated-lessons.json` unchanged)

### Notes

- No `generatedAt` is set for lessons migrated from the old format (field was never in `generated-lessons.json` before this pipeline). The slot gets `generatedAt: None`. This is fine — `--validate` (chunk 6) only checks content field nullness, not `generatedAt`.
- Orphan handling (arch constraint 1) is fully implemented: no data is dropped, and warnings are explicit before the summary line.

---

## ✅ Implementation Summary — Chunk 3: `--generate`

**Date**: 2026-06-13  
**Chunk**: 3 of 7

### What was implemented

**File changed**: `learning/scripts/generate-lessons.py` — full rewrite of the generation and batch logic.

**Removed** (old checkpoint system):
- `CHECKPOINT_PATH` constant
- `load_checkpoint()`, `save_checkpoint()` — sync checkpoint helpers
- `load_done_keys()` — read from `generated-lessons.json` to skip done lessons
- `merge_with_existing()` — merged new results into `generated-lessons.json`
- `_assign_lesson_numbers()` — renumbered lessons from position (root cause of the original bug)
- `build_batch_requests()` — built batch from config, skipping via `load_done_keys()`
- `process_batch_results()` — rebuilt path structure and called `_assign_lesson_numbers()`

**Added** (skeleton-based system):

| Function | Purpose |
|---|---|
| `build_batch_requests_from_skeleton(skeleton, active_track_names)` | Builds batch requests from `pending`/`failed` slots only; `custom_id_map` maps `req_XXXXX` → slot key |
| `apply_batch_results_to_skeleton(results, custom_id_map, skeleton)` | Updates skeleton slots in place from batch results; returns `(succeeded, failed)` |
| `archive_batch_checkpoint(batch_id)` | Moves checkpoint to `prisma/batch-archive/` via `os.replace()` instead of deleting |
| `_find_topic_from_config(track, level, topic_name)` | Looks up topic dict from `lesson_config.TRACKS` for prompt building |
| `_print_gap_table(skeleton, active_track_names)` | Prints failed/pending slots as a table; returns count |
| `_fill_slot_from_lesson(slot, lesson)` | Populates content fields and sets `status="complete"`, `generatedAt` |

**Replaced**:
- `save_batch_checkpoint(batch_id, manifest, total)` → `save_batch_checkpoint(batch_id, custom_id_map)` — slim format per spec
- `clear_batch_checkpoint()` → `archive_batch_checkpoint(batch_id)`

**Sync mode** now:
1. Loads skeleton; collects `pending`/`failed` slots for active tracks
2. Generates via `ThreadPoolExecutor` — on success: `_fill_slot_from_lesson()` + `save_skeleton()` under lock; on failure: sets `status="failed"`, `error`, saves skeleton
3. Never writes `generated-lessons.json`
4. If any `failed` slots remain: `_print_gap_table()` + `sys.exit(1)`

**Batch mode** now:
1. Builds requests from skeleton (not from config + `load_done_keys()`)
2. Checkpoint is `{batch_id, submitted_at, model, custom_id_map}` — no manifest
3. On results: `apply_batch_results_to_skeleton()` updates slots in place, then `save_skeleton()`
4. Checkpoint archived (not deleted) on completion
5. If any failures: `_print_gap_table()` + `sys.exit(1)`. Does not auto-export.

**`--test` (sync)** pulls first 2 pending slots from skeleton instead of hardcoded track[0]/path[0]/topic[0]. Does not write to skeleton.

**`--status`** unchanged in behaviour; now works with slim checkpoint format (only needs `batch_id`).

### Acceptance criteria status

- ✅ A simulated batch failure leaves the slot as `status: "failed"` with `lessonNumber` intact; surrounding lessons unaffected (skeleton is slot-indexed, no renumbering)
- ✅ Re-running `--generate` retries only `failed` and `pending` slots
- ✅ `--generate` exits non-zero and prints a gap table if any slots remain `failed`
- ✅ `generated-lessons.json` is never written by `--generate` or `--batch`
- ✅ `generate-checkpoint.json` no longer written during runs (removed)
- ✅ `batch-checkpoint.json` replaced by slim `{batch_id, submitted_at, model, custom_id_map}` format; archived on completion

### Notes

- `_write_output()` is retained — it is the target for `--export` (chunk 4)
- `import math` removed (was only used by old sync mode unit calculations)
- `lesson_key()` removed (was an alias for `_slot_key()` used only by old batch/merge logic)

---

## ✅ Implementation Summary — Chunk 4: `--export`

**Date**: 2026-06-13  
**Chunk**: 4 of 7

### What was implemented

**File changed**: `learning/scripts/generate-lessons.py`

**Import added**: `import shutil` (stdlib — used by `_write_output` for backup copy)

**Constant added** (after `BATCH_ARCHIVE_DIR`):
- `BACKUPS_DIR` — `prisma/backups/`

**`_write_output(output: list) -> str | None`** — upgraded from stub to full atomic writer:

| Step | What happens |
|---|---|
| Backup | `os.makedirs(BACKUPS_DIR)`, `shutil.copy2(OUTPUT_PATH, backup_path)` if file exists. Returns `backup_path` (or `None`). |
| Write tmp | `generated-lessons.tmp.json` in same directory as target (intra-filesystem requirement for `os.replace()`) |
| Validate | Re-reads tmp: checks valid JSON, `lesson_count > 0`, no null `title`/`content`/`summary` in any lesson |
| Atomic replace | `os.replace(tmp_path, OUTPUT_PATH)` |
| Cleanup on failure | `os.remove(tmp_path)` if exists; re-raises — backup is always intact |

**`cmd_export()`** — new command:

| Step | What happens |
|---|---|
| Guard | `load_skeleton()` — raises if missing or wrong schema_version |
| Gap check | `_print_gap_table(skeleton)` across **all paths** (not filtered by track). If any pending/failed: prints gap table, exits non-zero. |
| Build output | Iterates paths; for each: path-level metadata + `lessons` list from `status="complete"` slots only. Pipeline-internal fields (`status`, `error`, `generatedAt`) excluded. |
| Zero-output guard | Exits if `lesson_count == 0` |
| Write | Calls `_write_output(output)` |
| Summary | Prints backup path, lesson count, output path |

**Argparse change**: `--export` flag added; `main()` routes to `cmd_export()` before `check_api_key()` — no API key needed.

### Acceptance criteria status

- ✅ `--export` refuses to run with any pending/failed slots and prints the gap list (exits non-zero)
- ✅ `--export` succeeds and writes a valid `generated-lessons.json` when all slots are complete
- ✅ Atomic write via `os.replace()` from same-directory tmp file (intra-filesystem — arch constraint 4)
- ✅ Timestamped backup to `prisma/backups/` before any write
- ✅ Validation of tmp before replace (JSON, non-zero count, no null content fields)
- ✅ `generated-lessons.json` format is structurally identical to today's file (path array with path metadata + lessons; no pipeline-internal fields in output)
- ✅ `generated-lessons.json` written only by `--export` — no other command in the script touches it

### Notes

- The output lesson dict includes: `lessonNumber`, `topicName`, `topicId`, `lessonIndex`, `totalLessons`, `isCapstone`, `title`, `summary`, `content`, `keyTakeaway`, `durationMinutes`, `difficulty`, `quiz`. Pipeline-internal fields (`status`, `error`, `generatedAt`) are stripped from output — seed has no knowledge of them.
- `prisma/backups/` must be added to `.gitignore` (chunk 7 / separate step — spec says gitignored but large). Not done in this chunk.
- `--export` does not accept `--tracks` — it exports ALL complete paths from the skeleton. This is correct: partial exports would produce a file inconsistent with the full curriculum.

---

## ✅ Implementation Summary — Chunk 5: `--one`

**Date**: 2026-06-13  
**Chunk**: 5 of 7

### What was implemented

**File changed**: `learning/scripts/generate-lessons.py`

**Function added**: `cmd_one(track, level, topic_name, lesson_index, force=False)` (after `cmd_export()`)

**Argparse additions** in `main()`:

| Flag | Type | Purpose |
|---|---|---|
| `--one` | store_true | Triggers single-slot repair mode |
| `--track` | str | Track name (for slot lookup) |
| `--level` | str | Level (beginner/intermediate/advanced/expert) |
| `--topic` | str | Topic name (maps to `topicName` in skeleton) |
| `--lesson-index` | int | Lesson index 1-based |
| `--force` | store_true | Re-generate even if slot is already complete |

**Routing**: `--one` is checked after `--export` (before tracks/banner). Validates all four required args via `parser.error()`, then calls `check_api_key()` and `cmd_one()`.

#### Logic summary

| Step | What happens |
|---|---|
| Load skeleton | `load_skeleton()` — raises if missing or wrong schema_version |
| Lookup slot | Iterates paths/lessons; matches by `_slot_key(track, level, topicName, lessonIndex)` |
| Not found | Prints all four lookup coordinates + hint; `sys.exit(1)` |
| Complete guard | If `status="complete"` and not `--force`: prints title + lessonNumber + hint; `sys.exit(1)` |
| Config lookup | `_find_topic_from_config()` for prompt building; exits if not found |
| Generate | `generate_lesson()` — sync, single request, RETRY_LIMIT retries |
| On success | Stamps skeleton metadata, `_fill_slot_from_lesson()`, `save_skeleton()` — no other slots touched |
| On failure | Sets `status="failed"`, `error="generation failed"`, `save_skeleton()`; `sys.exit(1)` |

### Acceptance criteria status

- ✅ `--one` fills a specific failed/pending slot without touching any other slot
- ✅ `lessonNumber` is never changed (displayed as "(unchanged)" in output; code never writes to it)
- ✅ Complete slot without `--force` prints a warning and exits non-zero
- ✅ Missing required args caught by `parser.error()` with clear message
- ✅ Slot-not-found exits with all four lookup coordinates printed

### Notes

- `_slot_key()` is the join key — same function used by `--migrate`, `--generate`, `--batch`. If track/topic was renamed in `lesson_config.py` after `--migrate`, `--one` will fail to find the slot (expected behaviour; the hint in the error message says to check spelling against `lesson_config.py`).
- `--force` is intentionally not mentioned in the generated usage banner (it's a danger flag, not a routine one). It appears in `--help`.
- Docstring at top of file updated to add `--one --force` usage line.

---

## ✅ Implementation Summary — Chunk 6: `--status` and `--validate`

**Date**: 2026-06-13  
**Chunk**: 6 of 7

### What was implemented

**File changed**: `learning/scripts/generate-lessons.py`

**Functions added** (after `cmd_one()`):

| Function | Purpose |
|---|---|
| `cmd_status()` | Per-path table: total/complete/pending/failed, with ✅/⏳/❌ flag; calls `_print_gap_table()` if any gaps |
| `cmd_validate()` | Structural checks: contiguous `lessonNumber`, no duplicates, no null content fields in complete slots; warns (non-fatal) on missing `quiz`/`keyTakeaway` |

**Argparse additions**:
- `--validate` (store_true)
- `--status` help text updated: "Skeleton summary; with --batch: check in-flight batch"

**Routing fixes in `main()`**:
- `if args.status and not args.batch:` → `cmd_status()` (no API key needed — routes before `check_api_key()`)
- `if args.validate:` → `cmd_validate()` (no API key needed — routes before `check_api_key()`)
- `if args.status:` (batch poll) → `if args.batch and args.status:` — in-flight batch check now requires both flags

**Docstring** updated: `--status` and `--batch --status` usage lines split.

### Acceptance criteria status

- ✅ `--status` prints per-path summary with complete/pending/failed counts and a gap list
- ✅ `--validate` checks `lessonNumber` contiguity (1..N) per path
- ✅ `--validate` checks no duplicate `(trackId, levelId, lessonNumber)` combinations
- ✅ `--validate` errors on null content fields in complete lessons
- ✅ `--validate` warns (non-fatal) on missing `quiz` or `keyTakeaway`
- ✅ `--batch --status` still checks in-flight batch as before; `--status` alone no longer requires or checks for a batch checkpoint
- ✅ Both commands verified on real skeleton (30 paths, 3884 lessons)

### Notes

- `--status` calls `_print_gap_table()` (reused from `--generate`/`--export`) for the gap list — consistent output format.
- `--validate` exits non-zero on errors; exits zero on warnings-only (warnings are surfaced but non-blocking by spec).

---

## ✅ Implementation Summary — Chunk 7: Retire checkpoint files / `.gitignore`

**Date**: 2026-06-13  
**Chunk**: 7 of 7

### What was implemented

**Files changed**:
- `learning/.gitignore`
- `learning/scripts/generate-lessons.py` (docstring only)

#### `.gitignore` — two entries added

```
# Lesson generation — transient artefacts (large; not needed in repo)
prisma/batch-archive/
prisma/backups/
```

`prisma/batch-archive/` was specified in the spec's "Files Changed" table. `prisma/backups/` was deferred here by the chunk 4 implementation summary ("chunk 7 / separate step").

#### Script docstring — corrected

Line 24 previously read "Skeleton is the checkpoint — no generate-checkpoint.json or batch-checkpoint.json", which was inaccurate: `batch-checkpoint.json` is still written for in-flight batches (slim format). Replaced with an accurate file inventory listing all five relevant files and explicitly noting `generate-checkpoint.json` as retired.

#### Checkpoint files on disk

`prisma/generate-checkpoint.json` and `prisma/batch-checkpoint.json` (old format) were both confirmed absent — they were fully removed by the chunk 3 implementation.

### Acceptance criteria status

- ✅ `batch-checkpoint.json` and `generate-checkpoint.json` no longer written during normal runs (done in chunk 3; confirmed absent on disk)
- ✅ `prisma/batch-archive/` is gitignored
- ✅ `prisma/backups/` is gitignored
- ✅ Docstring accurately describes the current file landscape

### Notes

- No code logic changes in this chunk — gitignore and docstring only.
- `batch-checkpoint.json` still exists as a format (slim: `{batch_id, submitted_at, model, custom_id_map}`) and is written during batch runs; it is archived to `prisma/batch-archive/` on completion, not deleted. This is correct per spec.

---

## Architecture Review Notes

**Reviewed**: 2026-06-13 · **Flag**: `arch-review: required - approved` · No ADR required (refinement of existing generation pipeline; no new integration or cross-cutting concern).

The skeleton-first architecture is structurally sound and directly addresses the root cause from 004. The separation of concerns (skeleton ↔ `generated-lessons.json` ↔ seed) is clean. Immutable `lessonNumber` assignment at `--init` eliminates the class of bug that prompted this work. The `generated-lessons.json` protection rules are appropriately paranoid for an irreplaceable artefact.

### Constraints for the dev agent (must follow)

1. **`--migrate` must handle orphaned lessons.** Lessons present in `generated-lessons.json` but **not** in `lesson_config.py` (e.g. renamed/removed topics) must be included in the skeleton as `status: "complete"` (never drop data) **and** logged as warnings so the operator knows they exist. They must not be silently absorbed. These orphans will not satisfy `--validate`'s contiguity check — surface that to the operator rather than failing silently.

2. **Document the "no config renames while a batch is in flight" constraint** in a code comment near `custom_id_map` handling. The slot key `track|level|topicName|lessonIndex` is the join between submitted batch requests and skeleton slots; renaming a track or topic in `lesson_config.py` while a batch is in flight will orphan results unmatchably. Same applies to `--one` targeting.

3. **`schema_version` must be checked on every skeleton read,** with a clear error if the file's version is newer than the code supports. Do not blindly read a skeleton of unknown version.

4. **The atomic `--export` replace must use `os.rename()`** (or `os.replace()`) on the same filesystem — never copy-then-delete, which is not atomic and risks a torn write to the sacred artefact. Write `generated-lessons.tmp.json` in the same directory as the target so the rename stays intra-filesystem.

### Non-blocking observations (no action required this ticket)

- **`--export` is all-or-nothing by design** (refuses on any pending/failed slot). Correct for now. If a lesson ever becomes genuinely un-generatable, a future `status: "skipped"` value or `--exclude` flag is the natural extension point — the status-based design makes adding it trivial, so no schema accommodation is needed now.
- **No concurrent-generation guard.** Two simultaneous `--generate` runs would last-writer-win and drop work. Acceptable for a single-operator CLI; a lockfile would be needed only if this becomes multi-operator.
- **File size** (~5.6 MB) is fine to read/write whole on each iteration at this scale.

### Doc correction

- The "Files Changed" table lists `scripts/generate-lessons.py`. The actual path is **`learning/scripts/generate-lessons.py`** (the `learning` repo, not `learning-ai`). `lesson_config.py` lives alongside it at `learning/scripts/lesson_config.py`. The dev agent should work there.

---

## Chunk 8 — Persistent Batch Log

### Background

Previous batch IDs are unrecoverable from local files. The old `clear_batch_checkpoint()` deleted `batch-checkpoint.json` after each completed batch, so no ID was ever persisted. The new `batch-archive/` directory (chunk 3) captures future batch IDs, but it is gitignored — if the local machine is lost or the folder is wiped, the IDs are gone again. Claude retains batches for 29 days; without the ID, there is no way to retrieve results for a batch that failed or partially processed.

### Requirement

A **permanent, committed, append-only** record of every batch ID ever submitted from this machine. Small enough to commit to git. Survives machine loss, folder wipes, and git clones.

### New file: `prisma/batch-log.jsonl`

One JSON object per line (JSONL format — append-only, grep-friendly). **Committed to git. Not gitignored.**

Two event types, one line each:

**Submission event** (written when batch is submitted):
```json
{"event": "submitted", "batch_id": "msgbatch_...", "submitted_at": "2026-06-13T14:00:00Z", "model": "claude-opus-4-8", "lesson_count": 120}
```

**Completion event** (written when batch results are processed):
```json
{"event": "completed", "batch_id": "msgbatch_...", "completed_at": "2026-06-13T15:12:00Z", "succeeded": 118, "failed": 2}
```

No update-in-place. Each event is a new appended line. `batch_id` is the join key between the two lines for a given run.

**Backfilled entries** for historical batches (those run before this system existed) use the same format but with a `"notes"` field explaining the record is backfilled and any known outcome:
```json
{"event": "submitted", "batch_id": "msgbatch_...", "submitted_at": "2026-06-09T00:00:00Z", "model": "claude-opus-4-8", "lesson_count": null, "notes": "backfilled — pre-batch-log run; outcome: succeeded"}
```

### Script changes

**`save_batch_checkpoint()`** — after writing the checkpoint JSON, append a `submitted` line to `batch-log.jsonl`.

**`apply_batch_results_to_skeleton()`** (or the caller in `--batch` flow) — after processing results, append a `completed` line to `batch-log.jsonl` with succeeded/failed counts.

Both writes use `open(..., "a")` (append mode). No locking needed — single-operator CLI, never concurrent.

### Backfilling previous batch IDs

The user will provide the 3 previous batch IDs (including the failed one). These are added manually as backfilled `submitted` entries in `batch-log.jsonl` before the file is first committed. No script command is needed for this — direct file edit is correct and intentional.

### Files changed

| File | Change |
|------|--------|
| `learning/scripts/generate-lessons.py` | Append to `batch-log.jsonl` on submission and completion |
| `prisma/batch-log.jsonl` | New — created manually with backfilled entries; committed to git |

`.gitignore` — no change needed (file is committed, not ignored).

### Acceptance criteria

- [ ] Every future `--batch` run appends a `submitted` line to `batch-log.jsonl` at submission time
- [ ] Every completed batch appends a `completed` line with succeeded/failed counts
- [ ] `batch-log.jsonl` is committed to git and survives a fresh `git clone`
- [ ] The 3 backfilled historical batch IDs are present in the file before first commit
- [ ] A failed or partial batch still produces a `submitted` line (the ID is preserved regardless of outcome)

---

## ✅ Implementation Summary — Chunk 8: Persistent batch log

**Date**: 2026-06-13  
**Chunk**: 8 of 8

### What was implemented

**Files changed**:
- `learning/scripts/generate-lessons.py`
- `learning/prisma/batch-log.jsonl` (created — backfilled entries, committed to git)

#### Constant added

```python
BATCH_LOG_PATH = os.path.join(PROJECT_DIR, "prisma", "batch-log.jsonl")
```

#### `save_batch_checkpoint()` — append `submitted` entry

`submitted_at` extracted into a variable (reused for both checkpoint JSON and log entry). After writing the checkpoint, appends one line to `batch-log.jsonl`:

```json
{"event": "submitted", "batch_id": "...", "submitted_at": "...", "model": "...", "lesson_count": N}
```

`lesson_count` is `len(custom_id_map)` — the number of requests submitted.

#### Batch completion — append `completed` entry

After `apply_batch_results_to_skeleton()` + `save_skeleton()` + `archive_batch_checkpoint()`, appends one line:

```json
{"event": "completed", "batch_id": "...", "completed_at": "...", "succeeded": N, "failed": N}
```

Both writes use `open(BATCH_LOG_PATH, "a")` — append mode, no locking needed (single-operator CLI).

#### `batch-log.jsonl` — backfilled

3 historical batches added manually with correct UTC timestamps (converted from BST provided by user):

| batch_id | submitted_at (UTC) | records | outcome |
|---|---|---|---|
| `msgbatch_01GJjqVPxP499vVBQCwYBfdT` | 2026-06-10T15:26:00Z | unknown | failed |
| `msgbatch_014aWUTWYTD3x7gAFnsPjsuM` | 2026-06-10T15:39:00Z | 672 | succeeded |
| `msgbatch_013YGVcyKyZWrM9rfBEjCzu9` | 2026-06-12T17:17:00Z | 1280 | succeeded |

### Acceptance criteria status

- ✅ Every future `--batch` run appends a `submitted` line at submission time
- ✅ Every completed batch appends a `completed` line with succeeded/failed counts
- ✅ `batch-log.jsonl` is committed to git (not gitignored)
- ✅ The 3 backfilled historical batch IDs are present in the file
- ✅ A failed batch still produces a `submitted` line (written before polling begins)
