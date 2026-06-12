# Lesson Generation — How-To Guide

Generates all Ascent micro-lessons and loads them into the database.

---

## Current Status — 2026-06-12

| Track | Level | Lessons | Status |
|---|---|---|---|
| AI for Product Managers | all 4 levels | 336 | ✅ Generated — in `prisma/generated-lessons.json` |
| Product Strategy | all 4 levels | 336 | ✅ Generated — in `prisma/generated-lessons.json` |
| Discovery & Research | all 4 levels | 320 | 🟡 Sequenced in `lesson_config.py` (2026-06-12), not generated |
| Execution & Delivery | all 4 levels | 320 | 🟡 Sequenced in `lesson_config.py` (2026-06-12), not generated |
| Metrics & Analytics | all 4 levels | 320 | 🟡 Sequenced in `lesson_config.py` (2026-06-12), not generated |
| Leadership & Influence | all 4 levels | 320 | 🟡 Sequenced in `lesson_config.py` (2026-06-12), not generated |
| All other tracks (5 tracks) | various | — | ⚪ Not sequenced or generated — see `docs/product-consultant.md` for order and progress |

**Database:** Not yet seeded. Run `pnpm --filter api prisma:seed` to load the beginner lessons. **This may be out of date**

## Mandatary track order

| Track number | Track name  |
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

---

## Immediate Next Steps

1. **Seed the database** with the 80 beginner lessons already generated:
   ```bash
   pnpm --filter api prisma:seed
   ```

2. **Validate quality** — run the app, go through several beginner lessons, check content and quizzes feel right before investing time in the remaining levels.

3. ~~**Write lesson sequences** for intermediate, advanced, and expert levels~~ ✅ Done — all sequences written and in `lesson_config.py`

4. **Generate remaining AI track levels** (use batch mode — Opus quality, laptop-safe):
   ```bash
   python3 scripts/generate-lessons.py --batch
   ```
   (~1 hour for ~256 lessons across 3 levels — intermediate 80, advanced 88, expert 88)
   Close your laptop any time. Run `--status` to check, `--batch` again to collect results.

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

### Batch mode — recommended for full runs

Uses Anthropic Batch API with Opus 4.8. Better quality than Sonnet, 50% cost discount, laptop-safe.
Submit and close your laptop — results are held by Anthropic for 29 days.

```bash
# Test first — submits 2 lessons as a batch, polls to completion, prints output
python3 scripts/generate-lessons.py --batch --test

# Full batch run — submits all lessons, saves batch_id, polls until done
python3 scripts/generate-lessons.py --batch

# Check status of an in-flight batch (after closing laptop and returning)
python3 scripts/generate-lessons.py --status

# Resume polling / collect results (run any time after submitting)
python3 scripts/generate-lessons.py --batch
```

Batch checkpoint is saved to `prisma/batch-checkpoint.json`. Close your laptop any time — re-running
`--batch` will resume from the saved batch_id automatically.

### Sync mode — for quick tests and small runs

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

Use the `--tracks` flag (no source editing needed):

```bash
python3 scripts/generate-lessons.py --batch --tracks "AI for Product Managers" "Product Strategy"
```

If `--tracks` is omitted, the `TRACKS_TO_RUN` default at the top of the script is used
(currently AI for Product Managers + Product Strategy; set to `None` for all tracks).
Unknown track names fail fast with the list of valid names.

**Generation is incremental (2026-06-10):** any lesson already present in
`prisma/generated-lessons.json` (keyed by track | level | topic | lessonIndex) is skipped
automatically, and results merge into the existing file without duplicates. The workflow for
future batches is simply: add new tracks/levels/topics to `lesson_config.py`, then re-run —
only the delta is generated.

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

### Intermediate — ✅ Sequences written and added to `lesson_config.py` (2026-06-10)

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

**To add sequences:**
1. Discuss each topic's 8-lesson arc with the user — agree the lesson titles and the capstone scenario together before writing
2. Once agreed, add to `lesson_config.py` under `"level": "intermediate"` in the AI for Product Managers track (structured format, same as beginner)
3. Mark topic as ✅ here once its sequence is agreed and added to `lesson_config.py`

**Process:** Do one topic at a time. Propose the 8 lessons, user reviews and edits, then commit to `lesson_config.py`. Do not generate all topics in one go.

---

### Advanced — ✅ Sequences written and added to `lesson_config.py` (2026-06-10)

11 topics × 8 lessons = 88 lessons to generate. (Topic 11 is a bonus topic added during sequence design.)

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
11. Multimodal AI — what it is, what it unlocks, and how to build for it *(bonus topic)*

---

### Expert — ✅ Sequences written and added to `lesson_config.py` (2026-06-10)

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
11. The player-coach VP — staying technically credible while leading strategically *(bonus topic)*

Topic 11 is the capstone of the entire track — ties all four levels together. Treat it as special.

**To add sequences:** Same process as intermediate — one topic at a time, discuss and agree with user before writing to `lesson_config.py`.

#### Potential future additions to Expert level

The following topics were identified as gaps during sequence design. Add as bonus topics once the core 11 are generated and validated:

- **Org design for AI product teams** — Topic 2 covers hiring profiles but not team structure. Where does eval/quality sit? Does ML engineering report into product? How do you avoid the "AI team as internal consultancy" failure mode? Genuine VP-level decision not covered anywhere in the track.
- **Responsible AI and ethics beyond compliance** — Topic 7 covers regulatory obligations (the floor). This topic covers what you choose not to build even when it's legal: dual-use risks, when to slow down a capability, how to build an ethical decision-making framework into your product culture.
- **Pricing AI features strategically** *(borderline)* — Advanced Topic 9 covers cost modelling and Expert Topic 9 touches portfolio funding, but the strategic pricing question (usage-based vs flat, protecting margin as model costs fall) may warrant its own expert topic.

---

## Non-AI Track Review — Issues & Improvement Plan

Reviewed 2026-06-10. **Do not generate any non-AI track in its current state.** All tracks need to be upgraded to the structured 8-lesson format and reviewed for missing levels before generation.

---

### How to use this section (read first in any new session)

#### The format standard

All tracks must use the **structured 8-lesson format** — the same format we designed for the AI track. This is the only format going forward. The legacy flat-string format (one string = one lesson) is deprecated.

**Structured format rules:**
- 10 topics per level
- 8 lessons per topic (80 lessons per level)
- Lesson 8 is always the capstone — a scenario-based synthesis of the whole topic
- Quiz escalation: lessons 1–3 test recall, 4–6 test application, 7–8 test judgement
- Capstone (lesson 8) gets 2 quiz questions instead of 1
- 150–200 words per lesson, plain text, no markdown
- Spaced repetition is deliberate — key concepts recur 2–3 times before the capstone
- Each lesson title is a full, descriptive sentence (not just a noun phrase)

**Lesson arc principle:** each lesson must be a prerequisite for the next. Test this by asking: does lesson N give the reader something they need for lesson N+1? If not, reorder.

#### The process for each session

When starting a new track:
1. Read this document (the track's section below) to understand what exists and what's needed
2. Read the current topics from `lesson_config.py` (the track section) to see the raw material
3. Agree the level structure with the user (which levels, who they're for, any level to add)
4. Agree the topic list for each level — review existing topics, rename/remove/add as needed
5. Design 8-lesson sequences together, one topic at a time — propose, discuss, confirm, move on
6. After all topics in a level are agreed, write them all to `lesson_config.py` in one edit
7. Update this document to mark the level as complete
8. Generate only when all levels for the track are fully sequenced

**Never generate a track partially.** Complete all levels first.

#### Improvement order

| # | Track | Status |
|---|---|---|
| 1 | Product Strategy | ✅ All four levels sequenced — ready to generate |
| 2 | Discovery & Research | ✅ All four levels sequenced (2026-06-12) — ready to generate |
| 3 | Execution & Delivery | ✅ All four levels sequenced (2026-06-12) — ready to generate |
| 4 | Metrics & Analytics | ✅ All four levels sequenced (2026-06-12) — ready to generate |
| 5 | Leadership & Influence | ✅ All four levels sequenced (2026-06-12) — ready to generate |
| 6 | Stakeholder Management | ⚪ Not started |
| 7 | Go-to-Market & Launch | ⚪ Not started |
| 8 | Product Communication | ⚪ Not started |
| 9 | AI for Product Managers | ✅ All four levels sequenced — ready to generate |
| 10 | Product Design & UX for PMs | ⚪ Not started — new track |
| 11 | Technical Skills for PMs | ⚪ Not started — new track |

Track order above matches the `TRACKS` list order in `lesson_config.py` (reordered 2026-06-10 — AI for Product Managers moved to position 9, after Product Communication).

---

## Track 1: Product Strategy

**Status:** ✅ All four levels sequenced and in `lesson_config.py` (2026-06-10) — 42 topics, 336 lessons. Not yet generated.

**Category:** `product-management`

**Description:** Vision, positioning, and the bets that matter.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs who are new to strategy thinking | ✅ Sequences written — in `lesson_config.py` |
| Intermediate | PMs who can set strategy and need sharper tools | ✅ Sequences written — in `lesson_config.py` |
| Advanced | Senior PMs and leads owning strategic direction | ✅ Sequences written — in `lesson_config.py` |
| Expert | Directors and VPs setting company-level product strategy | ✅ Sequences written — in `lesson_config.py` |

### Beginner — ✅ Complete

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

**Changes made during sequence design:**
- Topic 7 renamed: "Setting SMART product goals" → "Setting goals that actually guide decisions"
- Topic 9 renamed: "Prioritisation fundamentals" → "Prioritisation as a strategic skill" — `through_line` added to keep it strategy-flavoured; execution frameworks (RICE etc.) belong in Execution & Delivery track. Minor overlap with that track is intentional.

### Intermediate — ✅ Complete

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

**Changes made during sequence design:**
- Topic 6 renamed: "Strategic bets under uncertainty" → "Making a bet you can defend" — `through_line` added: structured decision-making under uncertainty; a bet has explicit assumptions, evidence thresholds, and conditions for changing course.
- Topic 1 ("Jobs to be done as a strategy lens") — `through_line` added: use JTBD framework and terminology, credit Clayton Christensen, cover hire/fire metaphor and all three job dimensions (functional, social, emotional).

### Advanced — ✅ Complete (2026-06-10)

11 topics × 8 lessons = 88 lessons. In `lesson_config.py`. (Topic 11 is a bonus topic added during sequence design, same pattern as the AI track.)

1. Category creation vs market disruption
2. Ecosystem strategy and partner leverage
3. Pricing as a strategic weapon
4. Horizontal vs vertical product expansion
5. Competitive response playbooks
6. Make vs buy vs partner decisions at scale
7. Strategic pivots — when and how
8. Two-sided markets and platform design
9. Building a strategic narrative for executives
10. Planning beyond 18 months — how to think about strategy when you can't predict the market
11. Strategy for the mature product — finding your second act before the first one fades *(bonus topic)*

**Changes made during sequence design:**
- Topic 10 renamed: "Long-range product strategy under ambiguity" → "Planning beyond 18 months — how to think about strategy when you can't predict the market" (applying the sharpening flagged in the earlier review).
- Topic 11 added as a bonus: nothing covered diagnosing a flattening growth curve and sequencing renewal — topic 4 covers expansion *options* and Expert topic 7 covers *killing* a product line, but not the diagnosis in between. Uses the S-curve as the organising mental model.
- `through_line` added to six topics to carve boundaries against neighbouring content:
  - Topic 1 — credit Christensen, use "disruption" precisely (not a synonym for innovation); category design as the alternative playbook; head-on competition as the legitimate third option.
  - Topic 2 — leveraging ecosystems you *don't* own; owning a platform is topic 8, platform-at-company-scale is expert level.
  - Topic 3 — strategy-level pricing only; launch pricing/packaging mechanics stay in the Go-to-Market track.
  - Topic 6 — strategy lens (differentiation, control, optionality); delivery-level build-vs-buy economics stay in Execution & Delivery; AI-specific version stays in the AI track.
  - Topic 8 — builds on intermediate network effects topic; assumes the four types and cold start are known, goes deeper into market design and governance.
  - Topic 9 — credit Rumelt (diagnosis–guiding policy–coherent action); the strategic argument itself, not presentation craft (that's Product Communication).
- Deliberate spaced repetition across topics: bets/assumptions thread (topics 6, 7, 10 — echoing intermediate "Making a bet you can defend"), pricing thread (3 → 5 → 8), partner/ecosystem thread (2 → 6 → 8), S-curve/expansion thread (4 → 11).

### Expert — ✅ Complete (2026-06-10)

11 topics × 8 lessons = 88 lessons. In `lesson_config.py`. (Topic 3 is a bonus topic added during sequence design.)

Through-line: *you are accountable for product strategy at the company level — where product bets and business strategy are the same thing.*

1. When product strategy is company strategy
2. Portfolio strategy — managing multiple bets across a product portfolio
3. Business model transformation — changing how your product makes money *(bonus topic)*
4. Strategic acquisitions — what to buy, what to avoid, and why
5. Platform strategy at scale — building an ecosystem others build on
6. Competitor intelligence at scale — building a strategic radar
7. Strategy in regulated markets — constraints as a strategic lens
8. International product strategy — sequencing and localisation
9. When to kill a product line — the signals and the decision
10. Communicating strategy to a board — what they're actually asking
11. The 10-year vision — what it is, what it isn't, and how to set one

Topic 11 is the capstone of the entire track — it ties vision (beginner), bets under uncertainty (intermediate), and long-range planning (advanced) together at company scale. Treat it as special.

**Changes made during sequence design:**
- Bonus topic added — "Business model transformation": the level's own through-line says product bets and business strategy are the same thing, yet nothing covered changing how the product makes money (perpetual→subscription, seats→usage). Advanced topic 3 covers repricing *within* a model; this covers changing the machine itself, including the revenue trough and customer migration.
- Topics reordered from the original proposal so the level reads as an arc: frame-setter first (topic 1), money and structural moves together (2–5), outward-facing strategy (6–8), then the endings — kill decision, board, vision (9–11). The 10-year vision moved to last as the track capstone.
- `through_line` added to nine topics to carve boundaries against neighbouring content:
  - Topic 1 — level frame-setter; capital allocation and P&L lens, not roadmaps.
  - Topic 2 — allocation across bets; the deep kill decision is topic 9; AI portfolio management stays in the AI track.
  - Topic 3 — builds on advanced pricing: repricing changes the number, transformation changes the machine.
  - Topic 4 — acquisition as strategy instrument (the thesis); M&A execution role stays in Leadership & Influence, technical due diligence in Technical Skills.
  - Topic 5 — the owner's side of advanced ecosystem + two-sided market topics; assumes both.
  - Topic 7 — regulation as strategy variable; AI-specific compliance stays in the AI track.
  - Topic 8 — market selection, sequencing, operating model; launch execution stays in Go-to-Market.
  - Topic 9 — builds on advanced mature-product topic; company-level line decision with divestiture as the live alternative.
  - Topic 10 — what boards are actually asking (fiduciary lens); presentation craft stays in Product Communication; builds on the advanced strategic narrative topic.
- ⚠️ **Cross-track duplicate to resolve:** Leadership & Influence Expert (flat list) contains "10-year product vision setting", which duplicates topic 11 here. Remove or reframe it (e.g. toward the leadership/alignment side of vision) when that track is reviewed.

---

## Track 2: Discovery & Research

**Status:** ✅ All four levels sequenced and in `lesson_config.py` (2026-06-12) — not yet generated

**Category:** `product-management`

**Description:** Find real problems before you build.

**`isPremium`:** out of scope for sequencing — the user sets premium status via the admin interface later. Don't flag config/doc mismatches.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs running their first research or interviews | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Intermediate | PMs building a continuous discovery practice | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Advanced | Senior PMs and research leads running complex studies | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Expert | Directors and VPs building a research practice at org scale | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |

### Beginner — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. Why discovery matters before building
2. Mapping assumptions before building
3. Surveys vs interviews — when to use each
4. How to recruit research participants
5. Writing a good discussion guide
6. Running your first user interview
7. Listening for the job to be done
8. Spotting and avoiding confirmation bias
9. Affinity mapping your research notes
10. Synthesising findings into insights

Level arc: why discovery → name your guesses → pick the method → find the people → write the guide → run the interview → hear the job → guard against bias → organise the notes → turn it into insight.

**Changes made during sequence design:**
- Reordered: "Mapping assumptions before building" moved from 9th to 2nd — it was sequenced after synthesis, but assumption mapping is how you decide *what to research*; it must precede method choice, recruiting, and interviewing. Original order failed the prerequisite-arc test.
- Reordered: "Surveys vs interviews" moved from 5th to 3rd — method choice follows directly from the research question (topic 2's output) and must precede recruiting.
- Renamed and repositioned: "What jobs to be done interviews reveal" → "Listening for the job to be done", placed 7th (resolves the ⚠️ "slightly advanced" flag). Framed as a beginner *listening lens* — Christensen's "progress people are trying to make", milkshake story, everyday analogies — immediately after "Running your first user interview" so it deepens a skill just learned. The `through_line` defers the full JTBD/outcomes framework upward; the strategy lens lives in Product Strategy intermediate ("Jobs to be done as a strategy lens").
- "Spotting and avoiding confirmation bias" placed 8th as the deliberate bridge between the asking topics (5–7) and the sense-making topics (9–10) — bias enters both.
- No topics cut, none added — all ten earn their place; held at 10 per the format standard.
- `through_line` added to all 10 topics. Key boundary decisions:
  - Topic 1 — credit Marty Cagan (value risk); finding problems is this track, strategic direction is Product Strategy.
  - Topic 2 — output is a research question; experiment design beyond interviews/surveys stays in intermediate (DFV checks).
  - Topic 3 — survey craft kept basic; statistical rigour is advanced, usability methods are intermediate.
  - Topic 5 — credit Rob Fitzpatrick's The Mom Test ("would you use this?" trap; past behaviour beats hypothetical opinion).
  - Topic 6 — stays in the room; contextual inquiry is intermediate.
  - Topic 9 — repositories/scaling stay in intermediate.
  - Topic 10 — level capstone; presentation craft defers to Product Communication, org-scale insight activation defers to advanced ("Insight activation — making research stick").
- Deliberate spaced repetition across topics: assumptions thread (topic 2 → topic 10 lesson 4 "Back to your assumption map" → capstone), bias thread (topic 4 convenience trap → topic 8 → topic 9 "instead of forcing them"), past-behaviour/Mom Test thread (topic 5 lesson 3 → topic 7 lesson 5 "Asking about the last time").

### Intermediate — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. Continuous discovery habits
2. Opportunity solution trees
3. Problem framing and reframing techniques
4. Quantitative vs qualitative research trade-offs
5. Contextual inquiry in the field
6. Diary studies for longitudinal insight
7. Usability testing methods compared
8. Card sorting and tree testing
9. Desirability, feasibility, and viability checks
10. Building a lightweight research repository

Level arc: build the habit → structure the learning (tree) → frame the problem → choose your evidence → watch the work → follow it over time → test usability → test findability → test the whole bet → make it compound. Level through-line: *beginner taught you to run one good study; intermediate turns research from a project into a weekly operating habit.*

**Changes made during sequence design:**
- All 10 placeholder topics kept — none cut, added, or renamed — but four were reordered (prerequisite-arc failures):
  - "Problem framing and reframing techniques" moved 9th → 3rd — framing must precede method work and solution testing; at 9th it arrived after everything it should shape (same failure mode as beginner's assumption-mapping reorder).
  - "Quantitative vs qualitative research trade-offs" moved 7th → 4th — it's the evidence-choosing judgement topics 5–9 depend on, so it precedes the individual methods.
  - Methods grouped generative-before-evaluative: contextual inquiry (5) and diary studies (6) before usability testing (7) and card sorting/tree testing (8).
  - "Desirability, feasibility, and viability checks" moved 5th → 9th — it tests *solutions*, so it needs the tree (topic 2) and the evaluative methods behind it; at 5th it taught solution testing before any evaluative method existed.
- Card sorting/tree testing considered for cutting (narrowest topic) but kept: findability problems are common, the methods are cheap and remote-friendly, and the `through_line` keeps it a research method (IA craft deferred to Track 10).
- `through_line` added to all 10 topics. Key boundary decisions:
  - Topic 1 — credit Teresa Torres; scaling habits across teams defers to advanced (democratising research).
  - Topic 2 — Torres's opportunity solution tree; roadmap prioritisation defers to Execution & Delivery, outcome-metric setting to Metrics & Analytics.
  - Topic 3 — customer-problem level only; strategic diagnosis (Rumelt) stays in Product Strategy.
  - Topic 4 — behavioural data as discovery input in scope; instrumentation/metric craft defers to Metrics & Analytics, statistical rigour and formal mixed-methods to advanced.
  - Topic 5 — credit Beyer & Holtzblatt (master-apprentice stance); full ethnography defers to advanced.
  - Topic 6 — lightweight team-level studies; org-scale longitudinal programmes defer to expert.
  - Topic 7 — credit Jakob Nielsen (five users, think-aloud); design craft/heuristics defer to Track 10 (Product Design & UX), research ops to advanced.
  - Topic 8 — research method only; information architecture craft defers to Track 10.
  - Topic 9 — credit Marty Cagan's risk framing (echoes beginner topic 1's "four risks"); deep feasibility defers to Execution & Delivery, full business cases to Product Strategy. Resumes the beginner assumption-mapping thread — solutions are stacks of guesses too.
  - Topic 10 — level capstone; research ops, democratisation, and insight activation defer to advanced.
- Spaced-repetition threads: opportunity tree (topic 2 → topic 3 lesson 7 → topic 9 lessons 1/8 → topic 10 lesson 6); assumption mapping from beginner (topic 9 lessons 1/3); bias from beginner (topic 4 cherry-picking, topic 5 say-vs-do, topic 7 leading tasks); weekly cadence (topic 1 → topic 7 small rounds → topic 10 capture habit); Mom Test/past behaviour (topic 1 lesson 6 story-based interviews).

### Advanced — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. Mixed-methods research design
2. Survey validity and statistical rigour
3. Ethnographic research for product teams
4. Customer journey mapping at scale
5. Research operations and tooling
6. Democratising research across teams
7. Measuring the ROI of discovery
8. Rapid synthesis under time pressure
9. When speed trumps discovery — making the call and living with it
10. Insight activation — making research stick

Level arc: design rigour (1–4: formal study design, quant rigour, deep qual, org-scale synthesis) → scale machinery (5–6: ops, democratisation) → judgement and impact (7–10: ROI, speed, skipping, activation). Level through-line: *intermediate made discovery a weekly habit; advanced makes it rigorous, scalable, and consequential.*

**Changes made during sequence design:**
- All 10 placeholder topics kept; order already passed the prerequisite-arc test, so no reorders.
- Renamed: "When to skip discovery" → "When speed trumps discovery — making the call and living with it" (resolves the ⚠️ watch-list flag). Framed around reversibility (one-way vs two-way doors), cost of delay, and instrumenting what you skipped so the market becomes the study — the honest counterweight to the track, not a licence to skip.
- "Insight activation" confirmed as the level capstone — pays off the promise made in beginner topic 10 and intermediate topic 10; org-strategy wiring deferred to expert.
- `through_line` added to all 10 topics. Key boundary decisions:
  - Topic 1 — owns study architecture; statistical detail deferred to topic 2.
  - Topic 2 — survey rigour only; experiment statistics (A/B significance) belong to Metrics & Analytics.
  - Topic 3 — pragmatic product ethnography, not academic anthropology; the promised step beyond intermediate contextual inquiry.
  - Topic 4 — evidence-led synthesis; workshop facilitation and visual craft defer to Track 10, journey metric instrumentation to Metrics & Analytics.
  - Topic 5 — credit Kate Towsey (ResearchOps); ops serves researchers, democratisation (topic 6) serves non-researchers.
  - Topic 7 — value = risk avoided + decisions changed; full financial modelling defers to Product Strategy; argument returns at expert scale.
  - Topic 10 — presentation craft defers to Product Communication.
- Spaced-repetition threads: confirmation bias from beginner returns in topic 8 ("bias at speed") and topic 2 (reading results honestly); the four-risks/risk-reduction frame from beginner topic 1 returns in topic 9 lesson 2; ops governance (topic 5) sets up expert ethics-at-scale; topic 6 builds directly on topic 5.

### Expert — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. Building a research practice from scratch
2. Designing the research operating model — centralised, embedded, or hybrid
3. Research strategy — deciding what the organisation learns
4. Research partnerships and external agencies
5. Longitudinal research programmes — what they are and when to invest
6. Building organisational research literacy
7. Communicating research to boards and executives
8. When research says one thing and the business says another
9. Research ethics at scale — responsibilities as your reach grows
10. The research-to-strategy engine — making evidence move the company's biggest bets

Level arc: build it → structure it → aim it → extend it → invest long → raise the org → the top of the org → the hard moments → the responsibility → the payoff. Level through-line: *you own the capability, not the studies — peer-to-peer with VPs and CPOs about headcount, budgets, and boards.*

**Changes made during sequence design (vs proposed topic list):**
- Reframed: "Scaling discovery across multiple product teams" → "Designing the research operating model — centralised, embedded, or hybrid". Advanced topic 6 already owns democratisation mechanics; the genuinely VP-level question is org design — reporting lines, intake, researcher careers, migrating models as the company changes.
- Moved and elevated: "The research-to-strategy link" 4th → 10th, renamed "The research-to-strategy engine — making evidence move the company's biggest bets" — the capstone of the whole track per the format standard, tying beginner's first interview through to evidence shaping bets, portfolio reviews, and the three-year view.
- Sharpened: "Research strategy — what to study and what to skip" → "Research strategy — deciding what the organisation learns", framed as capital allocation of learning capacity — deliberately mirroring the Product Strategy expert lens (strategy allocates money, research strategy allocates learning).
- Remaining seven topics kept with reordering into the build → aim → extend → raise → defend → payoff arc above.
- `through_line` added to all 10 topics. Key boundary decisions:
  - Topic 4 — make-vs-buy logic from Product Strategy applied to learning capacity; core continuous discovery is never outsourced; procurement mechanics out of scope.
  - Topic 5 — pays off the promise from intermediate diary studies (org-scale longitudinal programmes).
  - Topic 6 — distinct from advanced democratisation: that taught teams to *do* research, this teaches the org to *consume* evidence well.
  - Topic 7 — carved against Product Strategy expert's board topic (fiduciary lens shared, but this is customer evidence at board altitude); presentation craft defers to Product Communication.
  - Topic 8 — the business sometimes should win; the craft is separating motivated reasoning from legitimate judgement on both sides.
  - Topic 9 — institutional responsibility beyond per-study consent (covered in advanced ops): power asymmetries, AI on research data, refusing dark-pattern requests.
- Spaced-repetition threads: capital-allocation lens (topic 3 → topic 10 portfolio review); longitudinal asset (topic 5 → topic 10 lesson 6 "the long view"); advanced ROI argument returns at company scale (topic 1 sceptical CFO, topic 3); credibility/trust thread (topic 1 credibility account → topic 7 standing slot → topic 8 losing well).

---

## Track 3: Execution & Delivery

**Status:** ✅ All four levels sequenced and in `lesson_config.py` (2026-06-12) — 40 topics, 320 lessons. Not yet generated.

**Category:** `product-management`

**Description:** Roadmaps, prioritisation, shipping under pressure.

*Sequenced in a single pass (all four levels, user-directed, skipping per-level review) on 2026-06-12.*

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs learning to run a team and ship features | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Intermediate | PMs managing complexity, debt, and cross-functional delivery | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Advanced | Senior PMs and leads managing delivery at programme scale | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Expert | Directors and VPs accountable for delivery culture and org-level output | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |

### Beginner — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. What makes a good product roadmap
2. Choosing a roadmap format — timelines, now-next-later, and when each fits
3. Your first prioritisation framework — RICE, value vs effort, and what the scores hide
4. Writing clear user stories
5. Acceptance criteria that engineers trust
6. Working effectively with engineers
7. Sprint planning for PMs
8. Definition of done — making 'done' mean done
9. Managing dependencies between teams
10. Running a useful retrospective

Level arc: plan (1–3: roadmap, format, prioritisation) → break it down (4–5: stories, criteria) → run the team (6–8: engineers, sprints, done) → beyond the team (9: dependencies) → improve (10: retros, the level capstone).

**Changes made during sequence design:**
- Merged: "Roadmap formats and when to use them" + "Now-next-later planning" → "Choosing a roadmap format — timelines, now-next-later, and when each fits". Two beginner topics on roadmap formats was redundant — now-next-later *is* a format (credited to Janna Bastow), taught as the beginner default within one format-choice topic.
- Moved down from intermediate: RICE → new topic 3 "Your first prioritisation framework — RICE, value vs effort, and what the scores hide" (resolves the ⚠️ flag — RICE is entry-level craft, and Product Strategy beginner's `through_line` explicitly defers execution frameworks to this track). Framed as: frameworks structure the argument, they don't make the decision.
- Reordered: "Definition of done" moved 9th → 8th (it's within-team sprint craft, paired with acceptance criteria via an explicit cross-reference lesson); "Managing dependencies between teams" moved 8th → 9th (the step beyond a single team, setting up the advanced dependency topic).
- Renamed: "Definition of done" → "Definition of done — making 'done' mean done".
- `through_line` added to all 10 topics. Key boundary decisions:
  - Topic 1 — roadmap building only; the strategy-vs-roadmap distinction stays in Product Strategy beginner; stakeholder negotiation defers to Stakeholder Management.
  - Topic 3 — the strategic *why* of prioritisation stays in Product Strategy beginner ("Prioritisation as a strategic skill"); sharper tools (cost of delay) are intermediate.
  - Topic 4 — credit the Connextra format and INVEST; PRD craft defers to Product Communication.
  - Topic 6 — day-to-day collaboration only; deep trust-building with engineering leads defers to Leadership & Influence; technical fluency to Technical Skills for PMs.
  - Topic 10 — credit Norm Kerth's prime directive; the level capstone; org-scale post-mortem culture defers to expert.
- Spaced-repetition threads: roadmap-as-intent (topic 1 → topic 2 format switching → topic 7 sprint goal); acceptance criteria ↔ definition of done (topic 5 lesson 7 and topic 8 lesson 1 cross-reference each other); the estimate conversation (topic 6 lesson 5) seeds the intermediate estimation topic; dependencies (topic 9) seeds the advanced scale topic.

### Intermediate — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. Cost of delay and opportunity scoring — prioritising with time in the equation
2. Ruthless prioritisation under constraints
3. Estimation and forecasting — giving dates without lying
4. Scope creep and how to handle it
5. Technical debt — when to say yes
6. Feature flags and phased rollouts
7. Incident management for PMs
8. Cross-functional team dynamics
9. Working in a platform or infra team
10. Delivering with distributed teams

Level arc: sharper prioritisation (1–2: tools, judgement) → commitments (3–4: dates, scope) → the recurring claims on capacity (5: debt) → shipping safely (6–7: rollouts, incidents) → the team system (8) → the special contexts (9–10). Level through-line: *beginner taught you to run one team's delivery in calm conditions; intermediate keeps you shipping when reality pushes back.*

**Changes made during sequence design:**
- RICE moved down to beginner (see above), freeing a slot.
- Merged: "Opportunity scoring frameworks" absorbed into topic 1 "Cost of delay and opportunity scoring — prioritising with time in the equation" — opportunity scoring alone was too thin for 8 lessons; combined with cost of delay and WSJF (credit Don Reinertsen) it forms the natural "beyond RICE" toolkit, adding the dimension RICE ignores: time.
- New topic added: "Estimation and forecasting — giving dates without lying" — nothing in the placeholder list (or the whole curriculum) covered the question PMs get asked most: *when will it be done?* Story points, velocity-based forecasting, ranges over dates, and re-forecasting without drama.
- Topic 10 ("Delivering with distributed teams") confirmed as the level capstone — every practice in the level stretched across time zones.
- `through_line` added to all 10 topics. Key boundary decisions:
  - Topic 1 — outcome-metric definition defers to Metrics & Analytics; choosing discovery opportunities defers to D&R (opportunity solution trees).
  - Topic 2 — the craft of delivering the no defers to Stakeholder Management; the strategy that makes the no defensible echoes Product Strategy beginner.
  - Topic 5 — credit Martin Fowler's deliberate-vs-reckless quadrant. Carved against Track 11: Technical Skills owns *understanding* what debt is; this track owns the *prioritisation call*. Scales up at advanced (technical investment).
  - Topic 6 — delivery mechanics only; launch comms and GTM tiers defer to Go-to-Market; AI rollouts (shadow mode) stay in the AI track.
  - Topic 7 — the PM supports response, doesn't command it; incident command belongs to engineering, crisis stakeholder comms to Stakeholder Management, org-scale post-mortem culture to expert.
  - Topic 8 — the operating system of the trio; deep interpersonal craft defers to Leadership & Influence.
  - Topic 9 — internal-platform delivery reality; platform *business strategy* (two-sided markets) stays in Product Strategy.
  - Topic 10 — day-to-day distributed delivery; org-level site strategy belongs to expert org design.
- Spaced-repetition threads: prioritisation (beginner RICE → topics 1–2 → scope trades in topic 4); estimation (beginner "estimate conversation" → topic 3 → re-forecasting in topic 4 lesson 7); debt (topic 5 → flag debt in topic 6 lesson 7 → incident debt in topic 7 lesson 6); blameless review (topic 7 lesson 7 echoes beginner retros, seeds expert post-mortem culture).

### Advanced — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. OKRs for product teams
2. Quarterly planning at a growing company
3. Portfolio-level prioritisation
4. Programme management across squads
5. Dependency management at scale
6. Scaling agile beyond a single scrum team
7. Org design and its impact on delivery
8. Technical investment as a product decision — platforms, debt, and capacity
9. Build vs buy at the portfolio level
10. Delivery risk identification and mitigation

Level arc: alignment machinery (1–3: goals, cadence, allocation) → running big things (4–6: programmes, dependencies, scaled process) → the structures underneath (7–9: org, technical investment, build-vs-buy economics) → risk across all of it (10, the level capstone). Level through-line: *intermediate kept one team shipping under pressure; advanced makes many teams ship in the same direction.*

**Changes made during sequence design:**
- Reframed: "Technical strategy for non-engineers" → "Technical investment as a product decision — platforms, debt, and capacity". The placeholder collided head-on with Track 11 (Technical Skills for PMs), whose entire purpose is technical fluency. What this track genuinely owns is the *investment decision* — how much delivery capacity goes to platforms, debt, and engineering health, and how to defend the allocation. Scales the intermediate debt topic from one sprint to a portfolio stance.
- Reordered four topics for the prerequisite arc: "Quarterly planning" moved 5th → 2nd (the cadence that operationalises OKRs, needed before portfolio allocation); "Scaling agile" moved 9th → 6th (process machinery belongs with programmes/dependencies, before the org-design topic it previews); "Technical investment" 6th → 8th (after org design, before build-vs-buy which extends it); "Delivery risk" moved 8th → 10th as the level capstone — pre-mortems (credit Gary Klein) synthesise programmes, dependencies, planning, and technical investment.
- `through_line` added to all 10 topics. Key boundary decisions:
  - Topic 1 — credit Grove and Doerr; builds on Product Strategy beginner goal-craft; metric definition and instrumentation defer to Metrics & Analytics; failure modes (theatre, sandbagging) get equal billing.
  - Topic 3 — delivery allocation only; company-level portfolio strategy (core/adjacent/transformational) stays in Product Strategy expert.
  - Topic 6 — sceptical of framework maximalism; SAFe/LeSS as options to adapt, not religions to install.
  - Topic 7 — credit Skelton & Pais (team topologies) and Conway's law. **The advanced/expert carve:** advanced *diagnoses* structure and influences it; expert *decides* and executes reorgs. People-leadership stays in Leadership & Influence.
  - Topic 9 — Product Strategy advanced owns the strategic make-vs-buy lens and explicitly defers the delivery economics here: TCO, integration tax, capability drain, portfolio audit.
- Spaced-repetition threads: dependencies (beginner topic 9 → topic 5 → structural answer previews topic 7); allocation (topic 3 → topic 8 → topic 9); the bottleneck platform team (topic 5 lesson 5 echoes intermediate topic 9); pre-mortem (topic 10) mirrors the expert post-mortem topic from the other side.

### Expert — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

Through-line: *you are accountable for how the whole organisation ships — culture, structure, pace, and the system that produces delivery, not the deliveries themselves.*

1. Delivery at company scale — what changes when you have 50 teams
2. Building a delivery culture — what it is and how to create it
3. Org design choices and their delivery consequences
4. Measuring delivery at scale — flow, DORA, and the Goodhart trap
5. Shipping when you can't fail — high-stakes delivery leadership
6. Leading through a major platform migration
7. When to slow down to go faster — the counter-intuitive call
8. Post-mortem culture at scale — learning from failure as an org
9. Delivery in a public company — the constraints you haven't faced yet
10. The CPO delivery agenda — owning how the company ships

Level arc: the scale frame (1) → culture (2) → structure (3) → measurement (4) → the hard moments (5–7: can't-fail, migration, the deliberate pause) → learning (8) → the top-floor constraints (9) → the agenda (10).

Topic 10 is the capstone of the entire track — from a beginner's first roadmap to owning the system that turns strategy into shipped product. Treat it as special.

**Changes made during sequence design (vs proposed topic list):**
- Cut: "Managing delivery across geographies and time zones" — duplicated intermediate topic 10 ("Delivering with distributed teams"); the genuinely expert-level remainder (site strategy, what to co-locate) was folded into topic 3 (org design, lesson 4).
- New topic added in its place: "Measuring delivery at scale — flow, DORA, and the Goodhart trap" — the proposed level owned culture and structure but had no measurement layer. Credit DORA (Nicole Forsgren, *Accelerate*) and flow metrics; the org dynamics of measuring engineers (Goodhart's law, never weaponising) are the expert content. Carved against Metrics & Analytics: that track owns product metrics; this owns measuring the machine that ships.
- Moved and elevated: "The CPO delivery agenda" 6th → 10th as the track capstone — operating rhythm, allocation stance, delivery brand, personal standard, succession test.
- Reordered: org design moved 4th → 3rd (structure directly follows culture, before measurement); remaining topics sequenced into the arc above.
- `through_line` added to all 10 topics. Key boundary decisions:
  - Topic 2 — specifically the culture of *shipping*; broad product-culture building stays in Leadership & Influence.
  - Topic 3 — the deciding altitude (advanced diagnoses, expert decides — the carve recorded in the advanced notes); absorbs site strategy; hiring mechanics stay in Leadership & Influence.
  - Topic 5 — assurance and delivery leadership; launch communications stay in Go-to-Market.
  - Topic 6 — the leadership problem of re-platforming; technical mechanics stay in Technical Skills for PMs; builds on advanced technical investment and build-vs-buy.
  - Topic 8 — credit John Allspaw and the blameless SRE tradition; scales beginner retros and intermediate incident reviews; AI-failure post-mortems stay in the AI track.
  - Topic 9 — delivering inside public-company constraints; investor communication itself stays in Leadership & Influence / Stakeholder Management.
  - Topic 10 — building the product org (hiring, careers) stays in Leadership & Influence ("Building a product org from scratch").
- Spaced-repetition threads: finishing-over-starting (topic 2 → WIP/smaller portfolio in topic 7 lesson 5); debt and technical investment from lower levels return in topics 6 and 7; the learning loop (beginner retros → intermediate blameless review → topic 8); the delivery-brand/credibility thread (topic 4 honest measurement → topic 9 disclosure discipline → topic 10 lesson 5).

---

## Track 4: Metrics & Analytics

**Status:** ✅ All four levels sequenced and in `lesson_config.py` (2026-06-12) — 40 topics, 320 lessons. Not yet generated.

**Category:** `product-management`

**Description:** Instrument, measure, and read the signal.

*Sequenced in a single pass (all four levels, user-directed, skipping per-level review) on 2026-06-12.*

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs with limited data experience who need a foundation | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Intermediate | PMs who can read data and need to define, instrument, and experiment | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Advanced | Senior PMs who own measurement architecture, economics, and causality | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Expert | Directors and VPs building measurement culture and data strategy | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |

### Beginner — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. Why metrics matter — the PM's relationship with data
2. What makes a good metric
3. Vanity metrics vs actionable metrics
4. Leading vs lagging indicators
5. Understanding your funnel
6. Reading a dashboard without a data analyst
7. Averages, percentages, and segments — reading numbers without being fooled
8. How metrics fit together — from your feature to the company's goals
9. When the numbers are wrong — data quality basics
10. Asking better questions of your data

Level arc: why measure → what good looks like → spot the fakes → the time dimension → the first structure (funnel) → read the dashboard → read the numbers honestly → see the connections → trust but verify → turn reading into asking. Level through-line: *you can read, question, and act on product data without a data analyst at your elbow.*

**Changes made during sequence design:**
- Reframed: "Common metric mistakes PMs make" → "Averages, percentages, and segments — reading numbers without being fooled". The placeholder was a listicle, not a skill; the genuine day-one gap is basic data literacy — averages hide extremes, percentages need denominators, totals hide segments. The useful "mistakes" survive inside it (small-sample swings, mix shifts).
- Reframed: "Your first north star metric" → "How metrics fit together — from your feature to the company's goals". Choosing a north star duplicated intermediate topic 1; what a beginner actually needs is the metric *chain* — feature → product → company. Introduces the north star idea (lesson 5) and defers choosing one to intermediate; also plants the seed for the advanced metric-tree topic. Deliberate three-level arc: beginner sees the connections, intermediate chooses the north star, advanced decomposes it.
- Reordered: "Vanity metrics vs actionable metrics" moved 6th → 3rd — it's the direct payoff of "What makes a good metric" (the behaviour test with teeth) and must precede the funnel/dashboard topics it inoculates.
- `through_line` added to all 10 topics. Key boundary decisions: topic 1 carves the track against D&R (research finds problems; this track measures what happens once you ship); topic 3 credits Eric Ries; topic 5 credits Dave McClure (AARRR) and defers funnel diagnosis to intermediate; topic 9 defers instrumentation craft to intermediate; topic 10 (level capstone) defers experimentation to intermediate and presentation to Product Communication.
- Spaced-repetition threads: denominators (topic 5 lesson 6 ↔ topic 7 lesson 3); rates-over-counts (topic 2 lesson 3 → topic 3 lesson 6); data-informs-judgement-decides (topic 1 lesson 5 → topic 10, and onward to the advanced/expert culture topics); seasonality (topic 6 lesson 5 → topic 9 sniff tests).

### Intermediate — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. North star metrics — choosing one that matters
2. Defining activation and retention metrics
3. HEART framework for UX measurement
4. Instrumentation and event tracking
5. Funnel analysis and where users drop off
6. Cohort analysis for retention
7. A/B testing fundamentals
8. Statistical significance explained for PMs
9. Building dashboards teams actually use
10. Avoiding metric gaming

Level arc: define what matters (1–3: north star, activation/retention, HEART) → instrument it (4) → analyse it (5–6: funnels, cohorts) → experiment on it (7–8) → operationalise it (9) → defend it (10, the level capstone). Level through-line: *beginner taught you to read data; intermediate makes you the team's measurement owner — define, instrument, experiment.*

**Changes made during sequence design:**
- All 10 placeholder topics kept — none cut or added — but the order failed the prerequisite-arc test in two places:
  - "Instrumentation and event tracking" moved 8th → 4th — you can't run funnels, cohorts, or A/B tests on events you never tracked; instrumentation must precede every analysis topic. (Same failure mode as D&R beginner's assumption-mapping reorder.)
  - "Defining activation and retention metrics" moved 3rd → 2nd and HEART 2nd → 3rd — activation/retention extend the north star directly; HEART is the broader definition toolkit.
- `through_line` added to all 10 topics. Key boundary decisions:
  - Topic 1 — credit Sean Ellis and John Cutler/Amplitude; a north star measures value delivered, not activity; OKR machinery stays in E&D advanced. Builds on beginner topic 8.
  - Topic 3 — credit Google's Kerry Rodden (HEART, goals-signals-metrics); usability methods stay in D&R, design craft in Track 10.
  - Topic 4 — **the instrumentation carve honoured:** this track owns tracking craft; D&R intermediate topic 4 owns behavioural data as a discovery input (carved 2026-06-12). Stack tooling decisions defer to expert.
  - Topic 7 — **the experiment-statistics carve honoured:** D&R advanced topic 2 explicitly defers A/B statistics here. Guardrail metrics seeded in the capstone (paid off at advanced topic 5).
  - Topic 8 — survey statistics stay in D&R advanced; deeper causal inference is this track's advanced level.
  - Topic 9 — operational dashboards only; data storytelling and presentation craft defer to Product Communication.
  - Topic 10 — level capstone; Goodhart's law at product altitude. Carved against E&D expert topic 4: that track owns Goodhart for *delivery* metrics (measuring the machine that ships); this owns *product* metrics and team incentives. Seeds the expert measurement-culture topic.
- Spaced-repetition threads: aha moment (topic 2 → topic 6 behavioural cohorts → topic 7 hypotheses); peeking (topic 7 lesson 6 → topic 8 lesson 6); beginner funnel returns as diagnosis (topic 5); beginner dashboard-reading returns as dashboard-making (topic 9); honest numbers (topic 8 "significant but tiny" → topic 10 capstone).

### Advanced — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. Metric trees and decomposition
2. Engagement quality vs raw engagement
3. LTV, CAC, and payback period
4. Causal inference vs correlation in product data
5. Guardrail metrics — protecting what you're not trying to move
6. Why experiments lie — novelty, seasonality, and results that don't last
7. Beyond A/B — measuring impact when you can't run a clean test
8. Measuring network effects
9. Ecosystem-level measurement — portfolios, platforms, and cross-product value
10. Data-informed vs data-driven culture

Level arc: architecture (1: the tree) → what the numbers mean (2–3: quality, economics) → causality and experiment rigour (4–7: inference, guardrails, result-scepticism, the un-A/B-testable) → measurement at the frontier (8–9: networks, ecosystems) → the judgement layer (10, the level capstone). Level through-line: *intermediate made you the team's measurement owner; advanced makes you the architect — economics, causality, and measurement where experiments can't reach.*

**Changes made during sequence design:**
- Renamed and broadened: "Multi-armed bandit experiments" → "Beyond A/B — measuring impact when you can't run a clean test". Bandits alone are too narrow for 8 lessons; the genuinely senior skill is the full toolkit for the un-A/B-testable — bandits, switchbacks, difference-in-differences, synthetic controls — and matching tool to constraint.
- Renamed and broadened: "Novelty effects and how they skew results" → "Why experiments lie — novelty, seasonality, and results that don't last". Novelty is one of several result-skewers; the topic now covers change aversion, seasonality, sample pollution, winner's curse, and long-running holdouts.
- Renamed: "Guardrail metrics and sensitive reaction groups" → "Guardrail metrics — protecting what you're not trying to move" ("sensitive reaction groups" was opaque; the segment lens survives as lesson 5).
- Renamed for scope: "Ecosystem-level measurement" → "Ecosystem-level measurement — portfolios, platforms, and cross-product value", and carved against topic 8: network effects measures the *effects*, ecosystem measures the *multi-product portfolio* (cannibalisation, bundles, platform health).
- Reordered into the arc above: "Metric trees and decomposition" moved 8th → 1st (the structural tool the level hangs off; pays off beginner topic 8 and intermediate north star inputs); engagement quality and LTV/CAC moved up together (6th → 2nd, 5th → 3rd); experiment-rigour topics grouped 4–7 in design→reading→workaround order (guardrails before result-reading, since guardrails are designed in advance); "Data-informed vs data-driven culture" held at 10 as the level capstone.
- `through_line` added to all 10 topics. Key boundary decisions: topic 1 defers OKR cascading to E&D advanced; topic 3 credits David Skok, defers pricing to Product Strategy and channels to GTM; topic 8 carves against Product Strategy intermediate (strategy owns the claim, this owns the instruments); topic 9 defers platform business strategy to Product Strategy; topic 10 defers qualitative evidence craft to D&R and seeds the expert level.
- Spaced-repetition threads: cohort curves (intermediate topic 6 → LTV in topic 3); vanity-metrics instinct from beginner returns at altitude (topic 2); guardrails seeded in intermediate topic 7 capstone get their full topic (5); power-user trap (topic 4) echoes the cohort comparisons; the overrule discipline (topic 10) seeds expert topics 1 and 9.

### Expert — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. Building a measurement culture — what it takes and why most companies fail
2. The data team relationship — how to get what you need as a product leader
3. The analytics stack — what a VP of Product actually needs to understand
4. Building self-serve analytics — the org capability you need
5. Metric governance — one source of truth in a company full of numbers
6. Experimentation at scale — when to run fewer, better tests
7. Privacy-preserving measurement — the constraints ahead
8. Communicating data to boards and executives
9. When data and intuition conflict — making the call
10. The metric strategy — deciding what the company measures and why

Level arc: culture (1) → partnership (2) → machinery (3) → capability (4) → governance (5) → programme (6) → constraints (7) → the top of the org (8) → the hard call (9) → the strategy (10). Level through-line: *you own how the company measures — the culture, the capability, and the strategy of what gets measured at all; peer-to-peer with CPOs, CFOs, and boards.*

Topic 10 is the capstone of the entire track — from a beginner reading a dashboard to deciding what the company pays attention to at all. Treat it as special.

**Changes made during sequence design (vs proposed topic list):**
- Cut: "Measurement for AI features — the expert view" — collides with the AI track (frozen and generated), which owns AI measurement end-to-end: success metrics (intermediate 7), eval sets (advanced 6), LLM-as-judge (advanced 7), observability (advanced 8). An expert AI-measurement topic here would duplicate, not deepen.
- New topic in its place: "Metric governance — one source of truth in a company full of numbers" — the proposed level had culture and tooling but nothing on the 'two dashboards, two revenue numbers' problem: definition ownership, certified metrics, change control, deprecation. A real VP-level capability nothing else in the curriculum covers, and the necessary partner to self-serve.
- Moved and elevated: "Metric strategy — what your company chooses to measure and why" 4th → 10th, renamed "The metric strategy — deciding what the company measures and why" — the track capstone per the format standard. Framed as attention allocation, deliberately mirroring Product Strategy expert (capital allocation) and D&R expert (learning allocation).
- Reordered the rest into the arc above: analytics stack 8th → 3rd and self-serve 9th → 4th (machinery before governance and programme); data-vs-intuition 6th → 9th (the judgement topic belongs late, after the machinery it overrules).
- `through_line` added to all 10 topics. Key boundary decisions:
  - Topic 1 — scales the advanced data-informed capstone from team to org; broad product culture stays in Leadership & Influence, shipping culture in E&D.
  - Topic 2 — the data function specifically; the research-team relationship belongs to D&R expert.
  - Topic 3 — fluency, not engineering; deep technical mechanics stay in Technical Skills for PMs.
  - Topic 4 — democratising data the way D&R advanced democratised research; pairs with topic 5 (freedom needs rules).
  - Topic 6 — programme/portfolio altitude only; test mechanics live at intermediate/advanced; AI evals stay in the AI track.
  - Topic 7 — institutional responsibility, echoing D&R expert research ethics; legal mechanics belong to counsel, AI-regulation specifics to the AI track.
  - Topic 8 — carved against three neighbours: Product Strategy expert owns the board *strategy* conversation, D&R expert owns *customer evidence* at board altitude, Product Communication owns presentation craft — this owns the company's *numbers*: which, how framed, how defended.
  - Topic 9 — carved against D&R expert topic 8 ("research says one thing, business says another" — qualitative evidence vs business pressure); this is quantitative data vs leader conviction. Decision quality vs outcome quality; the overrule on the record.
- Spaced-repetition threads: honest-numbers thread completes (intermediate gaming → expert topic 1 "celebrating honest misses" → topic 8 "presenting bad numbers"); the overrule discipline (advanced topic 10 → topic 9 protocol and track record); governance (topic 5) underpins boards (topic 8 "consistency compounds"); the metric chain that opened the track at beginner closes as company strategy (topic 10).

---

## Track 5: Leadership & Influence

**Status:** ✅ All four levels sequenced and in `lesson_config.py` (2026-06-12) — 40 topics, 320 lessons. Not yet generated.

**Category:** `product-management`

**Description:** Stakeholders, narrative, and the path to Director+.

*Sequenced in a single pass (all four levels, user-directed, skipping per-level review) on 2026-06-12.*

**The track's defining carve (decided 2026-06-12): L&I vs Stakeholder Management.** SM owns the stakeholder *system* — identifying, mapping, engagement plans, cadences, review meetings, coalitions, and the relationships with external governance (board, investors, CEO). L&I owns the *personal craft* of influence and leadership — persuasion, negotiation, trust, feedback, leading teams, growing people, culture, org building, and the career path itself. Every cut and reframe below follows from this carve.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs in their first 1–2 years learning to influence without authority | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Intermediate | Mid-level PMs moving decisions, managing up, and holding teams together | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Advanced | Senior PMs moving into lead or director roles — the shift from doing to leading | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |
| Expert | VPs and CPOs leading the product organisation itself | ✅ Sequences written — in `lesson_config.py` (2026-06-12) |

### Beginner — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. Why PMs need influence without authority
2. Understanding what motivates different people at work
3. Reading the room — emotional intelligence basics for PMs
4. Building trusted relationships in your first year
5. The currencies of influence — getting things done without a title
6. Disagreeing without damaging the relationship
7. When to push back and when to let it go
8. Getting credit for your work without making enemies
9. The PM's relationship with their manager
10. Your first 90 days — building influence from zero

Level arc: the frame (why influence) → see others (motivation) → see the moment (EQ) → the foundation (trust) → the tactics (currencies) → the hard moments (disagreement, battle-picking) → visibility (credit) → the key relationship (your manager) → the synthesis window (first 90 days). Level through-line: *you can get things done, build trust, and start to influence the people and decisions around you.*

**Changes made during sequence design (vs proposed topic list):**
- Reordered: "Reading the room — EQ basics" moved 6th → 3rd — EQ is a prerequisite for handling disagreement (proposed 5th), not a follow-up to it. Seeing people (topic 2) and seeing the moment (topic 3) must both precede the trust, tactics, and conflict topics.
- Renamed: "How to get things done without a title" → "The currencies of influence — getting things done without a title", and positioned 5th after trust — the exchange toolkit (credit Cohen & Bradford) only works once you know what people value (topic 2) and they trust you (topic 4). Topic 1 keeps the why/mindset; this owns the tactics.
- "Your first 90 days" confirmed as the level capstone (credit Michael Watkins) — applies every foundation to the highest-leverage window a PM gets.
- `through_line` added to all 10 topics. Key boundary decisions: topic 1 carves the track against Stakeholder Management (SM owns stakeholder identification mechanics; this track owns the personal skill of moving people); topic 4 credits Maister's trust equation and defers eng-lead depth to intermediate; topic 7 defers the formal stakeholder "no" to SM; topic 9 keeps the beginner altitude (your own boss) and defers exec sponsors to intermediate.
- Spaced-repetition threads: the scoreboard (topic 2 → topic 5 lesson 5 → topic 9 lesson 6); disagree-and-commit (topic 6 lesson 7 → topic 7 lesson 4); the trust account (topic 4 → topic 5 lesson 7 → topic 10 lesson 5); pushing back upward (topic 7 lesson 5 → topic 9 lesson 7).

### Intermediate — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. Influencing decisions you don't own
2. Making the case — building arguments that change minds
3. Negotiation for PMs — getting to yes without burning the relationship
4. Facilitating decisions — running the room so things actually get decided
5. Giving and receiving feedback well
6. Building trust with engineering leads
7. Managing up to your exec sponsors
8. Handling conflict in cross-functional teams
9. Leading through ambiguity
10. Growing toward senior PM — turning influence into career progress

Level arc: the persuasion core (1–4: decisions, arguments, negotiation, facilitation) → relationship maintenance (5: feedback) → the two key relationships (6–7: eng lead, exec sponsors) → the hard moments (8–9: conflict, ambiguity) → the payoff (10, the level capstone). Level through-line: *beginner gave you the personal foundations; intermediate makes you the PM who moves decisions, manages upward, and holds a team together under tension.*

**Changes made during sequence design (vs flat topic list):**
- Cut: "Stakeholder mapping and prioritisation" — core Stakeholder Management territory (SM beginner "Who are your stakeholders", SM intermediate "Building a stakeholder engagement plan"). Replaced with a genuine curriculum gap: **"Negotiation for PMs — getting to yes without burning the relationship"** (credit Fisher & Ury) — PMs negotiate scope, dates, and resources weekly and nothing in the curriculum taught it.
- Reframed: "Influencing without authority" → "Influencing decisions you don't own" — the flat title duplicated beginner topic 1; the intermediate step up is moving decisions made above and around you (real decision-makers, option framing, pre-wiring).
- Reframed: "Writing persuasive product documents" → "Making the case — building arguments that change minds" — Product Communication owns writing and presentation craft; L&I owns the argument architecture itself, whatever the medium.
- Reframed: "Running effective product reviews" → "Facilitating decisions — running the room so things actually get decided" — SM beginner already owns "Running a product review meeting"; the influence skill L&I genuinely owns is group decision facilitation (decision modes, drawing out quiet voices, landing the commit).
- Resolved the ⚠️ career-ladder flag by reframing, not cutting: "Understanding the PM career ladder" → "Growing toward senior PM — turning influence into career progress", elevated to level capstone. The track's tagline is literally "the path to Director+" — career progression is the track's promise, reframed from HR-ladder tour to influence campaign (sponsors vs mentors, visible stretch, the promotion case as a case to be made). The IC-vs-manager fork is advanced; succession is expert — a deliberate career thread runs the whole track.
- "Managing up to your exec sponsors" kept with a carve: L&I owns the personal relationship craft; SM advanced's proposed "The executive sponsor relationship" must take the governance/engagement lens or be folded when Track 6 is sequenced (flagged in the watch-list).
- "Building trust with engineering leads" kept per the 2026-06-12 carve — E&D beginner stays day-to-day, this goes deep; technical fluency defers to Track 11.
- "Handling disagreement in cross-functional teams" sharpened to "Handling conflict…" — E&D intermediate's team-dynamics topic explicitly defers the interpersonal craft here.
- Spaced-repetition threads: negotiation interests reappear in conflict mediation (topic 8 lesson 4); the beginner scoreboard thread continues (topic 6 lesson 6, topic 7 lesson 6); borrowed authority from beginner returns as sponsor-capital judgement (topic 7 lesson 4); sponsors-not-mentors (topic 10 lesson 4) seeds the advanced sponsorship lesson.

### Advanced — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. Moving from IC to manager — the hardest transition in product
2. Managing a team of PMs
3. Creating psychological safety in product teams
4. Mentoring and growing junior PMs
5. Hiring and structuring PM interviews
6. Building a product culture
7. Executive presence — influence in the room at director level
8. Navigating organisational politics
9. Storytelling as a leadership tool — aligning teams through narrative
10. Product leadership during a company downturn

Level arc: the threshold (1) → the core job (2) → the environment (3) → the individuals (4) → adding people (5) → the culture they join (6) → the upward and outward turn (7–9: presence, politics, narrative) → the exam (10, the level capstone). Level through-line: *intermediate made you the PM who moves decisions; advanced is the shift from doing to leading — people, culture, and the organisation around you.*

**Changes made during sequence design (vs flat topic list):**
- Reordered: "Moving from IC to manager" moved 10th → 1st — a prerequisite-arc failure in the flat list: mentoring, hiring, and managing a team all assume you've already crossed the threshold the list put last. It is the level's frame-setter, including whether to cross at all (the IC path as a path, not a consolation).
- Elevated: "Product leadership during a company downturn" 9th → 10th as the level capstone — leading through layoffs, frozen budgets, and fear tests every skill in the level at once (safety, culture, storytelling, presence, managing the team).
- Reframed: "Executive communication — what changes at director level" → "Executive presence — influence in the room at director level" — Product Communication advanced's proposed topic 1 is "Communication at director level"; a head-on collision. PC owns the communication craft; L&I owns presence — composure as signal, conviction with calibration, gravitas without persona (flagged for Track 8 in the watch-list).
- Reframed: "Strategic storytelling for product leaders" → "Storytelling as a leadership tool — aligning teams through narrative" — Product Strategy advanced owns the strategic argument (Rumelt) and Product Communication owns narrative craft; what L&I owns is story as a leadership instrument: meaning, repetition, and the counter-story test.
- "Navigating organisational politics" kept with a carve: this is the leader's personal political craft (power maps, agendas, integrity); SM intermediate's "Navigating political environments" must take the stakeholder-alignment lens or be cut when Track 6 is sequenced (flagged in the watch-list).
- "Creating psychological safety" credits Amy Edmondson; the blameless post-mortem machinery stays in E&D — this is the leader's everyday behaviour. "Building a product culture" honours the 2026-06-12 carve: E&D expert owns the culture of *shipping*; this owns broad product culture; company-wide transformation is expert.
- Spaced-repetition threads: the feedback skills from intermediate return as developmental feedback (topic 4) and performance honesty (topic 2); sponsorship (topic 4 lesson 7) pays off the intermediate career capstone; the three culture levers (topic 6) seed the expert culture topics; storytelling-in-hard-moments (topic 9 lesson 5) is rehearsal for the downturn capstone.

### Expert — ✅ Complete (2026-06-12)

10 topics × 8 lessons = 80 lessons. In `lesson_config.py`.

1. CPO and VP Product responsibilities compared
2. Building a product org from scratch
3. Leading leaders — managing directors and heads of product
4. Leading the product org through hypergrowth and scale transitions
5. Building strategic capability in your product org — making strategy everyone's job
6. Making the company product-led — culture change beyond the product org
7. Product considerations in M&A — the product leader's role
8. Succession planning in product leadership
9. Leading with a 10-year vision — keeping the organisation aligned and believing
10. Operating as the senior-most product person

Level arc: what the job is (1) → build the org (2) → lead its leaders (3) → carry it through scale (4) → raise its capability (5) → change the company around it (6) → the hard external event (7: M&A) → secure its future (8: succession) → sustain belief over years (9: vision) → the chair itself (10). Level through-line: *you lead the product organisation itself — peer-to-peer with CPOs and VPs about orgs, succession, culture change, and the chair where the buck stops.*

Topic 10 is the capstone of the entire track — from a beginner with no authority to the chair where the buck stops, only to discover authority still isn't enough. Treat it as special.

**Changes made during sequence design (vs flat topic list):**
- Cut: "Investor relations for product leaders" — **resolves the ⚠️ watch-list flag: Stakeholder Management Expert owns investor relations.** SM expert's entire through-line is boards, investors, and the most senior stakeholders as *relationships*; a second investor topic here would duplicate, not deepen.
- Cut: "Board-level product communication" — boards are already four-way carved (Product Strategy expert owns the strategy conversation, D&R expert customer evidence, M&A expert the numbers, Product Communication presentation craft), and SM expert owns the board *relationship* ("Managing your board of directors", "Managing activist or difficult board members"). A fifth board topic was curriculum bloat, not coverage.
- New topic in a freed slot: **"Leading leaders — managing directors and heads of product"** — advanced owns managing PMs; nothing anywhere covered managing *managers*: judging through the layer, autonomy contracts, skip-levels, and the director who struggles. The genuine gap between managing a team and succession planning.
- New topic in the other freed slot: **"Leading the product org through hypergrowth and scale transitions"** — culture dilution, the layer you must add, the displaced early guard, and your own transformation as the org doubles. Carved against E&D expert: that track owns delivery-at-scale and org-structure mechanics; this owns the people and leadership side of scale.
- Reframed: "Creating a product-led growth culture" → "Making the company product-led — culture change beyond the product org" — Go-to-Market owns the PLG motion mechanics; the CPO-level leadership content is cross-functional culture change in functions that don't report to you.
- "Product considerations in M&A" sharpened to "…— the product leader's role", honouring the carve recorded in Product Strategy expert (PS owns the acquisition thesis; Technical Skills owns technical due diligence; SM takes the stakeholder-comms lens at Track 6): this is product due diligence, the integration decision, and the acquired team's first year.
- The 2026-06-10 reframes of topics 5 and 9 (strategic capability, leading with the vision) held as recorded — `through_line`s now encode both carves against Product Strategy expert.
- Reordered into the arc above: org building (2) directly after the frame-setter; M&A and succession swapped so the level ends on the long game (succession → vision → the chair).
- Spaced-repetition threads: the first-team idea (topic 1 lesson 5 → topic 3 lesson 7); delegation from advanced returns as autonomy contracts (topic 3) and giving away your own job (topic 8 lesson 4); the culture levers from advanced return at founding scale (topic 2 lesson 6), scale transitions (topic 4 lesson 2), and M&A integration (topic 7 lesson 6); storytelling from advanced returns as communicating through layers (topic 4 lesson 5) and sustaining vision belief (topic 9); the leader's-shadow idea closes the track (topic 10 lesson 2).

---

## Track 6: Stakeholder Management

**Status:** ⚪ Not started — advanced and expert levels missing, existing levels in flat format

**Category:** `product-management`

**Description:** Align, communicate, and build trust across the org.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs learning to manage their first stakeholders | ⚪ Topics exist, needs structured sequences |
| Intermediate | PMs navigating complex stakeholder environments | ⚪ Topics exist, needs structured sequences |
| Advanced | Senior PMs managing executives, conflict, and political environments | ⚪ Missing — needs topics and sequences |
| Expert | VPs and CPOs managing boards, investors, and org-wide alignment | ⚪ Missing — needs topics and sequences |

### Beginner — current topics (flat format, needs structured sequences)

1. Who are your stakeholders and why it matters
2. Stakeholder communication cadences
3. Writing clear product update emails
4. Running a product review meeting
5. Saying no without burning bridges
6. Managing sales and product tension
7. Working effectively with customer success
8. Aligning with marketing on roadmap
9. Keeping design partners engaged
10. Documentation your stakeholders will actually read

### Intermediate — current topics (flat format, needs structured sequences)

1. RACI models for product teams
2. Managing conflicting priorities between execs
3. Building a stakeholder engagement plan
4. Using data to win alignment
5. Navigating political environments ⚠️ *L&I advanced now owns the leader's personal political craft (power maps, agendas, integrity — sequenced 2026-06-12); take the stakeholder-alignment lens here or cut*
6. Communicating delays and bad news well
7. Running product steering committees
8. Cross-org dependency management
9. Managing external stakeholders and partners
10. Stakeholder retrospectives

### Advanced — missing, proposed topics

Through-line: *you manage upward, sideways, and across organisational boundaries with confidence.*

1. Stakeholder management at scale — many stakeholders, many agendas
2. Managing the hostile stakeholder — when alignment keeps failing
3. Building coalitions for unpopular decisions
4. Stakeholder management in a matrix organisation
5. Managing stakeholders through major change or restructure
6. The executive sponsor relationship — how to make it work ⚠️ *L&I intermediate now owns the personal sponsor-relationship craft ("Managing up to your exec sponsors", sequenced 2026-06-12); take the governance/engagement-machinery lens here or fold into another topic*
7. When stakeholders are wrong — making the case with data and narrative
8. Global stakeholder management across geographies and cultures
9. Managing external stakeholders — customers, partners, analysts
10. Stakeholder management during a crisis or public incident

### Expert — missing, proposed topics

Through-line: *you manage boards, investors, and the most senior stakeholders in the business.*

1. Managing your board of directors as a product leader
2. The CEO relationship — how to make it work and what breaks it
3. Investor relations for product leaders ✅ *Resolved 2026-06-12: SM owns it — L&I expert cut its investor-relations topic (and its board-communication topic) when sequenced; SM expert owns the board/investor/CEO relationships outright*
4. Building exec alignment on a multi-year product vision
5. Stakeholder management across an acquisition or merger
6. Managing activist or difficult board members
7. Communicating product strategy across a global executive team
8. When the organisation is your hardest stakeholder
9. Legacy stakeholders — managing people who predate you
10. Stakeholder management when things go publicly wrong

---

## Track 7: Go-to-Market & Launch

**Status:** ⚪ Not started — beginner and expert levels missing, existing levels in flat format

**Category:** `product-management`

**Description:** Ship features that land, not just features that ship.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs running their first feature launches | ⚪ Missing — needs topics and sequences |
| Intermediate | PMs owning GTM strategy for significant launches | ⚪ Topics exist, needs structured sequences |
| Advanced | Senior PMs leading complex, multi-market launches | ⚪ Topics exist, needs structured sequences |
| Expert | VPs and CPOs responsible for launch strategy and GTM capability | ⚪ Missing — needs topics and sequences |

### Beginner — missing, proposed topics

Through-line: *you can plan and execute a feature launch confidently and learn from it.*

1. What go-to-market actually means for a PM
2. The PM's role in a launch — and what's not yours
3. Launch tiers — when to go big and when to go quiet
4. Writing your first launch brief
5. Working with marketing for a feature launch
6. The soft launch — why you don't ship to everyone at once
7. What to track in the first week after launch
8. When a launch goes wrong — your immediate response
9. Post-launch retrospectives — learning from what just happened
10. Building launch confidence — the habits that make launches less scary

### Intermediate — current topics (flat format, needs structured sequences)

1. Go-to-market strategy basics for PMs
2. Launch tiers and how to decide which applies
3. Writing a launch brief
4. Beta programmes and early access design
5. Product-led vs sales-led go-to-market
6. Pricing and packaging for a new launch
7. Defining launch success metrics
8. Working with PR and comms
9. Internal launch communications
10. Post-launch retrospectives

### Advanced — current topics (flat format, needs structured sequences)

1. Platform and ecosystem launch readiness
2. Enterprise GTM complexity
3. International launch planning
4. Sequencing launches across markets
5. Managing launch risk
6. Competitive launch responses
7. Analyst and influencer relations for PMs
8. Category-defining launch narratives
9. Measuring long-term launch impact
10. Post-launch strategy evolution

### Expert — missing, proposed topics

Through-line: *you own GTM capability and launch strategy at the organisation level.*

1. GTM strategy at company scale — when launches are business events
2. Platform launches — coordinating an ecosystem
3. Launching into a new market category — creating demand that doesn't exist yet
4. Launch strategy under regulatory scrutiny
5. Managing a recall or major rollback — the hardest launch conversation
6. The global launch — coordinating across geographies at speed
7. When to delay a launch — the executive call and how to make it
8. Building a launch-capable organisation — processes, people, and culture
9. Competitive launch responses at the market level
10. Measuring the long-term business impact of launches

---

## Track 8: Product Communication

**Status:** ⚪ Not started — advanced and expert levels missing, existing levels in flat format

**Category:** `product-management`

**Description:** Write, present, and persuade with clarity.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs learning to communicate clearly and concisely | ⚪ Topics exist, needs structured sequences |
| Intermediate | PMs developing a strategic communication style | ⚪ Topics exist, needs structured sequences |
| Advanced | Senior PMs and leads communicating at director level | ⚪ Missing — needs topics and sequences |
| Expert | VPs and CPOs communicating as industry and company voices | ⚪ Missing — needs topics and sequences |

### Beginner — current topics (flat format, needs structured sequences)

1. Why PMs need to write well
2. Writing for busy readers
3. Structuring a product update
4. Async communication as a PM
5. Asking good questions in reviews
6. Using data to tell a story
7. The Minto Pyramid for product docs ⚠️ *Slightly advanced for beginner but important — keep, sequence carefully*
8. Getting feedback on your writing
9. Common PM writing mistakes
10. Building a writing habit

### Intermediate — current topics (flat format, needs structured sequences)

1. Crafting a product narrative
2. Presenting strategy to leadership
3. Writing PRDs engineers trust
4. Presenting trade-offs clearly
5. Demo best practices
6. All-hands product updates that stick
7. Writing strategy memos
8. Handling hard questions in reviews
9. Building credibility through communication
10. Visual communication for PMs

### Advanced — missing, proposed topics

Through-line: *you communicate at director level — across the organisation, upward to executives, and outward to the market.*

1. Communication at director level — what changes and what doesn't
2. Writing for external audiences — press, analysts, and partners
3. Crisis communication for product leaders
4. Building a communication strategy, not just individual documents
5. Presenting to boards — the PM's guide
6. The long-term product narrative — building it over years
7. Communicating across cultures and geographies
8. When your communication fails — diagnosis and recovery
9. Ghostwriting for your CPO or CEO
10. Communication as a leadership tool — how great leaders use it

### Expert — missing, proposed topics

Through-line: *you communicate as a company voice and an industry voice — internally and externally.*

1. The CPO communication agenda — what you own and what you don't
2. Building a communication culture in your product organisation
3. Investor and analyst communication
4. Managing communications in a public company
5. The product story as company narrative — when they become the same thing
6. Crisis communications at scale — when the company is in the news
7. Communication across an acquisition or merger
8. Speaking as an industry voice — conferences, media, and thought leadership
9. Internal vs external communication strategy — managing both at once
10. The long game — communicating a multi-year vision to multiple audiences simultaneously

---

## Track 10: Product Design & UX for PMs

**Status:** ⚪ Not started — new track, does not exist yet

**Category:** `product-management`

**Description:** Design thinking, UX principles, and working with designers to build products people love.

**Rationale:** Absent from the entire curriculum despite being a core PM skill. This is a clear gap.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs who are new to design collaboration and UX thinking | ⚪ Needs topics and sequences |
| Intermediate | PMs who work closely with designers and want deeper fluency | ⚪ Needs topics and sequences |
| Advanced | Senior PMs who shape design direction and work at design system level | ⚪ Needs topics and sequences |

### Beginner — proposed topics

Through-line: *you understand what good design is, how to work with designers, and how to contribute usefully to design decisions.*

1. Why design matters to a PM — not just the designer's job
2. Design thinking — what it is and isn't
3. How to read a wireframe
4. Working effectively with a designer
5. What makes a good user experience
6. Accessibility basics every PM should know
7. Information architecture — organising what you build
8. The design review — how to give useful feedback
9. Design systems and why they matter
10. When to push back on design — and when to trust the process

### Intermediate — proposed topics

Through-line: *you are a genuine design partner — fluent in the language, tools, and trade-offs.*

1. Jobs to be done as a design lens
2. Usability testing — running your own without a researcher
3. Interaction design patterns every PM should know
4. Designing for mobile vs desktop — the real differences
5. Prototyping tools for PMs — when and how to use them
6. User flows and task analysis
7. Inclusive design — beyond accessibility compliance
8. Design debt — what it is, how it accumulates, when to pay it
9. Measuring the quality of design decisions
10. The PM-designer relationship at its best — and how to get there

### Advanced — proposed topics

Through-line: *you shape design strategy and influence how design works across your organisation.*

1. Design strategy — shaping product direction through design decisions
2. Design systems at scale — the PM's stake
3. Research-driven design at pace — when you can't slow down for a study
4. Design leadership for PMs — influencing without owning design
5. The ethics of design — dark patterns and the case for designing against them
6. Designing for behaviour change
7. Cross-platform design coherence — when users expect consistency
8. Designing for trust — the invisible layer
9. Design operations — how design works at scale
10. Design and brand — where product and marketing strategy meet

---

## Track 11: Technical Skills for PMs

**Status:** ⚪ Not started — new track, does not exist yet

**Category:** `product-management`

**Description:** APIs, system design, databases, and the technical fluency that makes you a better partner to engineering.

**Rationale:** Increasingly expected at most companies. Not covered anywhere in the curriculum. Technical fluency is a meaningful differentiator for mid-level and senior PMs.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs with no technical background who want to build foundational fluency | ⚪ Needs topics and sequences |
| Intermediate | PMs who can hold technical conversations and want to go deeper | ⚪ Needs topics and sequences |
| Advanced | Senior PMs who contribute to architecture and technical strategy discussions | ⚪ Needs topics and sequences |

### Beginner — proposed topics

Through-line: *you understand how software is built well enough to be a credible partner to engineering from day one.*

1. Why technical knowledge makes you a better PM
2. How software is actually built — the basics every PM should know
3. What APIs are and why they matter to your product
4. Databases in plain language
5. Frontend vs backend — what lives where and why it matters
6. How to read a basic technical diagram
7. Cloud services — what PMs need to know and what they don't
8. Security basics every PM should understand
9. Understanding version control — why engineers care about git
10. Working with engineers — the language that builds trust

### Intermediate — proposed topics

Through-line: *you can hold a technical conversation, scope work accurately, and flag technical risk before it's a problem.*

1. System design concepts for PMs
2. Microservices vs monoliths — the PM implications
3. Caching — what it is and when it matters to your product
4. Queues and async processing — why it affects user experience
5. Third-party integrations — scoping, risk, and dependency management
6. Data pipelines — what PMs need to understand
7. Authentication and authorisation — the PM's guide
8. Performance and scalability — what to ask for and what it costs
9. Technical debt — understanding what you're accumulating
10. Reading a basic engineering spec or design doc

### Advanced — proposed topics

Through-line: *you contribute to technical strategy conversations, understand architecture trade-offs, and hold engineering accountable at the right level.*

1. Architecture decisions and their product consequences
2. Platform engineering — what it is and what product gets from it
3. Data architecture for product decisions
4. Real-time systems — when you need them and what they actually cost
5. Privacy by design — building compliance into the architecture
6. Technical due diligence — evaluating a codebase or acquisition
7. The build vs buy decision — the deep technical factors that change the answer
8. AI and ML infrastructure — what PMs need to understand
9. Observability and monitoring — the PM's stake in production health
10. Contributing to the engineering roadmap — how to do it without overstepping

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
