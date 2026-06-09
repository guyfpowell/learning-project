# Lesson Generation — How-To Guide

Generates all Ascent micro-lessons and loads them into the database.

---

## Current Status — 2026-06-09

| Track | Level | Lessons | Status |
|---|---|---|---|
| AI for Product Managers | beginner | 80 | ✅ Generated — in `prisma/generated-lessons.json` |
| AI for Product Managers | intermediate | 80 | ⚪ Sequences not yet written |
| AI for Product Managers | advanced | 80 | ⚪ Sequences not yet written |
| AI for Product Managers | expert | 88 | ⚪ Sequences not yet written |
| All other tracks (8 tracks) | various | ~230 | ⚪ Not yet generated — deferred until AI track validated |

**Database:** Not yet seeded. Run `pnpm --filter api prisma:seed` to load the beginner lessons.

---

## Immediate Next Steps

1. **Seed the database** with the 80 beginner lessons already generated:
   ```bash
   pnpm --filter api prisma:seed
   ```

2. **Validate quality** — run the app, go through several beginner lessons, check content and quizzes feel right before investing time in the remaining levels.

3. **Write lesson sequences** for intermediate, advanced, and expert levels (see AI Track Curriculum Design section below), then add them to `scripts/lesson_config.py` under the AI for Product Managers track.

4. **Generate remaining AI track levels:**
   ```bash
   python3 scripts/generate-lessons.py
   ```
   (~5–8 minutes for ~248 more lessons across 3 levels)

5. **Re-seed** the database to add the new lessons.

6. **Decide on other tracks** — 8 non-AI tracks (~230 lessons total) are defined in `lesson_config.py` but not yet generated. Decision deferred until AI track is validated.

---

## Scripts

| File | Purpose |
|---|---|
| `scripts/lesson_config.py` | All curriculum data — TRACKS, LEVEL_GUIDANCE, prompt builders. **Edit this to add new levels.** |
| `scripts/generate-lessons.py` | Online generation via Claude API. Parallel (10 at once). **Primary script.** |
| `scripts/generate-lessons-local.py` | Offline generation via local Ollama. Sequential. Retained for cost-free/offline use. |
| `scripts/.env` | API key — gitignored. `ANTHROPIC_API_KEY=sk-ant-...` |
| `scripts/.env.example` | Template for the above — safe to commit. |

---

## Setup (one-time)

### Online script (Claude API)
- Python 3 installed (`python3 --version`)
- `ANTHROPIC_API_KEY` set in `scripts/.env`

### Local script (Ollama)
- Ollama running at `localhost:11434`
- Model pulled (see Local Model Options below)
- ~9–20 GB RAM free depending on model

---

## Running the Scripts

### Online (Claude API) — recommended

```bash
# Test first — generates 2 lessons, verifies API key and quality
python3 scripts/generate-lessons.py --test

# Full run — resumes from checkpoint if interrupted
python3 scripts/generate-lessons.py
```

### Local (Ollama) — offline / cost-free

```bash
python3 scripts/generate-lessons-local.py --test
python3 scripts/generate-lessons-local.py
```

Both scripts checkpoint after every lesson (`prisma/generate-checkpoint.json`). Re-run the same command to resume — already-completed lessons are skipped.

---

## Seeding the Database

```bash
pnpm --filter api prisma:seed
```

The seed detects `prisma/generated-lessons.json` and loads all lessons. If the DB already has lessons from a previous seed, run a reset first:

```bash
pnpm --filter api prisma:migrate reset   # ⚠️ wipes the DB
pnpm --filter api prisma:seed
```

---

## Controlling What Gets Generated

Edit `TRACKS_TO_RUN` near the top of the script you're running:

```python
TRACKS_TO_RUN = ["AI for Product Managers"]   # only this track
TRACKS_TO_RUN = None                           # all tracks
```

To regenerate from scratch: delete `prisma/generated-lessons.json` and `prisma/generate-checkpoint.json`, then re-run.

---

## Local Model Options (Ollama only)

| Model | RAM needed | Quality | Notes |
|---|---|---|---|
| `qwen2.5:32b-instruct-q4_K_M` | ~9 GB | Very good | Recommended — best quality-to-RAM ratio |
| `qwen2.5:14b` | ~10 GB | Good | Lighter alternative |
| `qwen2.5:7b` | ~5 GB | OK | Fast, lower quality |
| `qwen2.5:32b` | ~20 GB | Best | Full precision — needs a cleared machine |

Change the model by editing `MODEL` at the top of `generate-lessons-local.py`. Free up RAM before running — quit Docker, browser, VS Code. Flush cache with `sudo purge`.

---

## AI Track Curriculum Design

AI for Product Managers is the flagship beta track — the core reason the target audience would choose Ascent. All other tracks are secondary until this one is complete and validated.

### Level definitions

| Level | Who it's for | Outcome |
|---|---|---|
| **Beginner** | Any PM wanting to use AI in their day job | Use AI tools faster and more effectively for research, writing, analysis, reporting |
| **Intermediate** | PMs building or managing AI features | Write specs for AI features, prototype with AI, design for uncertainty, set success metrics, ship AI features |
| **Advanced** | PMs who need to go under the hood | Understand LLMs, RAG, vector embeddings, evals — hold engineering to account on AI architecture |
| **Expert** | VP/Director-level AI product leaders | Set AI product strategy, hire and structure AI teams, manage vendors, communicate to boards |

---

### Micro-lesson design principles

Encoded in `build_prompt_single_lesson()` in `lesson_config.py`. Do not change without updating the prompt.

- **150–200 words per lesson. No exceptions.**
- 3 minutes per lesson (`durationMinutes: 3`)
- 8 lessons per topic (lesson 8 is always the capstone)
- Quiz on every lesson — not just the capstone
- Plain text only — no markdown, no bold, no bullet points, no headers

**4-part structure (every lesson):**
1. Hook — 1 sentence that creates curiosity or makes the reader care
2. Concept — 2–3 short sentences explaining the core idea
3. Example — 1–2 sentences, specific realistic PM scenario with a named character
4. Takeaway — 1 sentence, one concrete thing to do starting today

**Quiz escalation across the 8 lessons:**
- Lessons 1–3: recall (did they get the concept?)
- Lessons 4–6: application (given this scenario, what would you do?)
- Lessons 7–8: judgement (synthesis, trade-offs)
- Capstone (lesson 8): 2 quiz questions instead of 1

**Tool references:** always suggest 2–3 options (e.g. "an AI assistant like Claude, ChatGPT, or Gemini"). Exception: tools that are genuinely unique (NotebookLM for multi-document synthesis, Perplexity for live web research).

**Spaced repetition is deliberate:** key concepts appear 2–3 times from different angles before the capstone. Do not remove this when writing new sequences.

---

### Beginner — ✅ Complete (in `lesson_config.py`, generated)

10 topics × 8 lessons = 80 lessons. In `prisma/generated-lessons.json`.

1. Your AI toolkit — the tools every PM should know
2. Summarising research and documents with AI
3. Competitor analysis with AI — faster, deeper, more frequent
4. Using AI for user interview synthesis
5. Data analysis and trend spotting without a data analyst
6. Generating usage reports and insight summaries
7. AI for writing — PRDs, briefs, and stakeholder updates
8. Building a personal AI workflow that sticks
9. Prompt basics for PMs — getting useful output first time
10. Where AI saves you time and where it wastes it

---

### Intermediate — ⚪ Topics agreed, sequences not yet written

10 topics × 8 lessons = 80 lessons to generate.

Through-line: *you are now responsible for shipping an AI feature or product, not just using one.*

1. Is this problem a good fit for AI? A decision framework
2. How to write a spec for an AI feature
3. AI prototyping — validating ideas before you build
4. Designing for uncertainty — when your product can be wrong
5. AI UX patterns — what good looks like for users
6. Human-in-the-loop — when to keep a human in the decision
7. Setting success metrics for an AI feature
8. Hallucinations and failure modes — designing around them
9. The latency, cost, and quality triangle
10. Shipping your first AI feature — what launch looks like differently

**To add:** design 8 lesson sequences per topic (same format as beginner). Add to `lesson_config.py` under `"level": "intermediate"` in the AI for Product Managers track.

---

### Advanced — ⚪ Topics agreed, sequences not yet written

10 topics × 8 lessons = 80 lessons to generate.

Through-line: *you can look under the hood, have an informed opinion on architecture decisions, and hold your engineering team to account.*

1. How LLMs work — what a PM actually needs to know
2. Tokens, context windows, and why they constrain your product
3. RAG explained — when and why you'd use it
4. Vector embeddings and semantic search — the PM mental model
5. Fine-tuning vs prompting vs RAG — choosing the right approach
6. Building an eval set — how to know if your AI is getting better or worse
7. LLM-as-judge — automating quality evaluation at scale
8. Observability for AI products — what to monitor in production
9. AI cost modelling — understanding your unit economics
10. When to swap models — how to make the call without breaking your product

**To add:** same as intermediate.

---

### Expert — ⚪ Topics agreed, sequences not yet written

11 topics × 8 lessons = 88 lessons to generate.

Through-line: *you set the vision, build the team, make the bets, and answer to the board.*

1. Defining an AI product strategy — bets, sequencing, and moats
2. Hiring for an AI-native team — what skills matter now
3. Augmenting your product team with AI agents — what to automate and what not to
4. Build vs buy vs fine-tune — the strategic decision framework
5. Managing AI vendor relationships and avoiding lock-in
6. Communicating AI strategy to a board or exec team
7. Regulatory and compliance strategy for AI products
8. Designing products that compound — systems that get smarter with use
9. AI product portfolio management — what to fund, what to kill
10. AI failure at scale — post-mortems, trust recovery, and what to do next
11. The player-coach VP — staying technically credible while leading strategically

Topic 11 is the capstone of the entire track — ties all four levels together. Treat it as special.

**To add:** same as intermediate.

---

## Troubleshooting

**`ModuleNotFoundError: No module named 'lesson_config'`**  
Run from the `learning` root: `python3 scripts/generate-lessons.py`. The script adds `scripts/` to its own path.

**`❌ ANTHROPIC_API_KEY is not set`**  
Check `scripts/.env` has `ANTHROPIC_API_KEY=sk-ant-...` with no spaces around `=`.

**`HTTP 429`**  
Rate limited — the script backs off and retries automatically.

**`❌ [model] not found in Ollama` (local script)**  
Pull the model first: `ollama pull qwen2.5:32b-instruct-q4_K_M`

**`HTTP Error 500 / signal: killed` (local script)**  
Out of memory — close more apps, switch to a smaller model, or use the online script.

**Seed fails with duplicate key error**  
Run `pnpm --filter api prisma:migrate reset` first (wipes the DB), then re-seed.
