**Claude code local only, do not make any API calls**

## Your role

You are the world's best product consultant, designing a micro-learning curriculum for product managers — from first-year PMs to CPOs. You bring:

## Docuemnts or files to review 
/Users/guypowell/Documents/Projects/learning/scripts/generate-lessons.py
/Users/guypowell/Documents/Projects/learning/scripts/lesson_config.py
/Users/guypowell/Documents/Projects/learning/prisma/generated-lessons.json line numbers around 29861 check a few lessons

## Tasks
1. Switch to claude code opus
2. Give it all the instructions to generate the full lesson for "Quality without policing — spot checks, shared debriefs, and catching drift early"
3. Insert it in into /Users/guypowell/Documents/Projects/learning/prisma/generated-lessons.json in the correct place
4. Check everything work and do any tidy up

## Additional considerations for task 2:
- Extract the full topic object from lesson_config.py for "Democratising research across teams"
- The lesson is index 5 (0-based), so it's lesson 6 of 8 in the topic sequence
- Use the exact prompt format from build_prompt_single_lesson() function
- The generated JSON must match the schema: title, summary, content, durationMinutes, difficulty, topicName, lessonIndex (6), totalLessons (8), isCapstone (false), keyTakeaway, quiz (1 question)

## Additional considerations for task 4:
- Verify the JSON is syntactically valid after insertion
- Check that lessonIndex is 6 and totalLessons is 8 (not counting this lesson)
- Verify all lessons in "Democratising research across teams" are in order (lessons 1-8 present)
- Verify no duplicate lessonNumbers in the output
- After fixing, run: `pnpm --filter api prisma:seed` to validate against database schema

## Lesson needing completing

 "Quality without policing — spot checks, shared debriefs, and catching drift early",

## Where to put it

Between 
"Coaching the product trio — turning PMs and designers into competent everyday researchers"
currently generated-lessons.json:29861

and 

"When democratisation fails — the bad-research flood and how to recover trust in evidence"
currently generated-lessons.json:29888

## Correct order once complete

"Coaching the product trio — turning PMs and designers into competent everyday researchers"

"Quality without policing — spot checks, shared debriefs, and catching drift early"

"When democratisation fails — the bad-research flood and how to recover trust in evidence"

---

# Post-mortem — what went wrong, the mess it caused, and the long-winded fix

*Written 2026-06-12 after completing the manual repair above. Purpose: hand this to whoever writes the "robustify the generation pipeline" ticket. This is deliberately exhaustive.*

## 1. The original problem: a silently-dropped lesson

The advanced level of the **Discovery & Research** track was generated via the batch pipeline (`scripts/generate-lessons.py --batch`, Opus 4.8, async Batch API). Each lesson is an independent request in the batch (`build_batch_requests()` emits one `custom_id` per lesson). The topic **"Democratising research across teams"** (topicId 6) is an 8-lesson sequence. Seven of its eight lessons landed in `prisma/generated-lessons.json`. **Lesson 6 of 8 — "Quality without policing — spot checks, shared debriefs, and catching drift early" — did not.**

The single most important fact: **the gap was silent.** Nothing in the pipeline hard-failed. The likely cause is one of:

- That individual batch request `errored`/`expired` while the others `succeeded`. `process_batch_results()` prints a `❌` line per failed request and increments a `failed` counter, but it does **not** raise, does **not** write a manifest of what's missing, and does **not** block the subsequent seed. The failure scrolls past in the console and is gone.
- The lesson was generated but its JSON failed to parse (`raw_text.find("{")` / `json.loads`), hitting the same swallow-and-continue path.

Either way the output file was written as if complete, the run printed its celebratory "✅ Batch complete!" box, and the topic was left with **7 lessons where the schema and the curriculum design both expect 8**. Because generation is incremental (lessons already present are skipped on re-run), a naive re-run would *not* have noticed or back-filled the hole either — it skips by `track|level|topic|lessonIndex` key, and a missing key just looks like "not done yet" only if you re-run that exact track, which nobody did because nothing flagged it.

**How it was even noticed:** a human happened to spot that the topic had 7 lessons, not 8. There is no automated check that every topic has its full `totalLessons` count. That is the root cause to fix.

## 2. Why fixing it was a disproportionate mess

What should be a one-line "insert a row" turned into a multi-stage manual operation. The reasons compound:

### 2a. `generated-lessons.json` is a single giant artefact that cannot be casually regenerated or even opened

- The file is **~350 KB / ~31,000 lines** and represents **millions of tokens of paid Opus generation** across all generated tracks. It is the single source of truth fed to the seed. It is too large to read in one pass (tools cap at 256 KB) and far too valuable to "just regenerate" to fill one hole — regenerating means API spend and risks perturbing the other ~700 lessons.
- So the fix had to be **surgical**: hand-author the one missing lesson to the exact schema and house style, splice it into the right place by hand, and touch nothing else.

### 2b. Re-generating the single lesson "properly" wasn't trivial either

To produce a lesson indistinguishable from the batch output, the repair had to:
- Locate and read the topic object for "Democratising research across teams" out of `lesson_config.py` (itself **3,830 lines / >256 KB**, also too big to read whole — had to be grepped).
- Reproduce the exact prompt contract from `build_prompt_single_lesson()` (4-part structure, 150–200 words, plain text, AI tools named as a 2–3 option trio, quiz-escalation tier — lesson 6 is an **application**-tier quiz, not recall or judgement).
- Match the surrounding lessons' voice, the `—` em-dash encoding, the field order, and the metadata stamping that the pipeline normally applies (`topicName`, `topicId`, `lessonIndex` = 6, `totalLessons` = 8, `isCapstone` = false).

None of this is hard, but all of it is manual reconstruction of logic that already exists in code — because there is no supported "generate/insert exactly one lesson into an existing file at the correct position" entry point. The pipeline only knows how to generate *missing* lessons in bulk and merge.

### 2c. The killer: `lessonNumber` is a path-wide counter, but its values are NOT unique across the file

This is what turned the fix from "annoying" into "very long-winded", and it is the most important thing for the ticket.

- Every lesson carries a `lessonNumber` — a **contiguous 1..N counter scoped to the skill-path** (one track+level), stamped by `_assign_lesson_numbers()` at generation time.
- The seed (`prisma/seed.ts:147`) upserts each lesson on the composite key **`(skillPathId, lessonNumber)`**. So within a path `lessonNumber` must be **unique and contiguous**, or two lessons collide on the same key and **one silently overwrites the other at seed time** — a second silent data-loss bug stacked on top of the first.
- Inserting the missing lesson at position 6 of the topic means it must take `lessonNumber 46`, and **every later lesson in that entire path must shift +1** (46→47, 47→48, … 79→80) — **34 lessons**, spanning topics 6, 7, 8 and beyond, across ~950 lines.
- **The trap:** `lessonNumber` resets to 1 in every path, so the *same integer values repeat across the file* — `"lessonNumber": 46` appears **25 times**, `47` 24 times, `60` 24 times, and so on. A naive global find-and-replace of `"lessonNumber": 46` → `47` would have corrupted 24 other unrelated tracks. There is no field on the `lessonNumber` line that scopes it to a path.

So the renumber could not be done by value. Each of the 34 increments had to be performed as an **individually anchored edit**, disambiguated by pinning it to the **adjacent unique lesson title** (the only nearby globally-unique string), e.g. "change the `lessonNumber` that sits immediately before the lesson titled *'Capstone: given a major study landing two weeks before annual planning…'* from 78 to 79." Thirty-four such anchored edits, plus the insertion, plus two JSON-validity checks and a duplicate-scan to confirm — to fix one missing lesson.

### 2d. Verification was also manual

There is no validator. Confirming the repair meant: re-running `python3 -m json.tool` for syntactic validity, then an `awk`/`sort`/`uniq` pass to prove the path's `lessonNumber` sequence was contiguous (…45, 46, 47…80) and duplicate-free, and that the next path still restarted at 1.

## 3. Net cost

One dropped lesson → reconstruct a topic object and a prompt contract from two oversized config files → hand-author a schema- and style-matched lesson → one insertion edit → **34 individually-anchored renumber edits** → multiple validation passes. A whole working session, with real risk at every step of corrupting an expensive, hard-to-regenerate artefact.

## 4. What to robustify (suggested ticket scope)

In rough priority order:

1. **Fail loudly on incomplete generation.** After a batch/sync run, assert that every topic has exactly `totalLessons` lessons and that lesson indices are the complete set `1..totalLessons`. Any gap should be a non-zero exit and a printed list of missing `track|level|topic|lessonIndex`, not a `❌` that scrolls away under a "✅ complete" banner.
2. **Add a standalone validator** (e.g. `scripts/validate-lessons.py`) runnable in CI and pre-seed, checking: JSON validity; per-topic completeness (1..N present, no gaps, no dupes); per-path `lessonNumber` contiguity from 1 and uniqueness; metadata coherence (`lessonIndex`/`totalLessons`/`isCapstone`/`topicId`). The seed should refuse to run if it fails.
3. **Make `lessonNumber` derivable, not hand-maintained.** Either have the seed compute ordering from `(topicId, lessonIndex)` at load time and stop trusting a stored counter, or always re-run `_assign_lesson_numbers()` after any edit. A path-wide counter whose values repeat file-wide and which gates a unique DB key is a foot-gun — at minimum the renumber must be a scripted, idempotent reindex, never a manual edit.
4. **Provide a supported "regenerate one lesson into the existing file" command** — `generate-lessons.py --one "<track>" "<level>" "<topic>" <lessonIndex>` — that runs the real prompt, inserts at the correct position, and reindexes `lessonNumber` automatically. That single command would have replaced this entire session.
5. **Detect dropped lessons at the source.** Cross-check the batch results against the submitted manifest and report any `custom_id` that was submitted but never came back `succeeded`, so an `errored`/`expired` request can't vanish unnoticed.
