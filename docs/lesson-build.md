# Lesson Generation — How-To Guide

Generates all Ascent micro-lessons and loads them into the database.

---

## Current Status — 2026-06-09

| Track | Level | Lessons | Status |
|---|---|---|---|
| AI for Product Managers | beginner | 80 | ✅ Generated — in `prisma/generated-lessons.json` |
| AI for Product Managers | intermediate | 80 | ✅ Sequences written — ready to generate |
| AI for Product Managers | advanced | 88 | ✅ Sequences written — ready to generate (11 topics, includes multimodal bonus) |
| AI for Product Managers | expert | 88 | ✅ Sequences written — ready to generate |
| All other tracks (8 tracks) | various | ~230 | ⚪ Not yet generated — deferred until AI track validated |

**Database:** Not yet seeded. Run `pnpm --filter api prisma:seed` to load the beginner lessons.

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
| 1 | Product Strategy | ⚪ Not started |
| 2 | Discovery & Research | ⚪ Not started |
| 3 | Execution & Delivery | ⚪ Not started |
| 4 | Metrics & Analytics | ⚪ Not started |
| 5 | Leadership & Influence | ⚪ Not started |
| 6 | Stakeholder Management | ⚪ Not started |
| 7 | Go-to-Market & Launch | ⚪ Not started |
| 8 | Product Communication | ⚪ Not started |
| 9 | Product Design & UX for PMs | ⚪ Not started — new track |
| 10 | Technical Skills for PMs | ⚪ Not started — new track |

---

## Track 1: Product Strategy

**Status:** ⚪ Not started — topics exist in flat format, sequences not designed

**Category:** `product-management`

**Description:** Vision, positioning, and the bets that matter.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs who are new to strategy thinking | ⚪ Topics exist, needs structured sequences |
| Intermediate | PMs who can set strategy and need sharper tools | ⚪ Topics exist, needs structured sequences |
| Advanced | Senior PMs and leads owning strategic direction | ⚪ Topics exist, needs structured sequences |
| Expert | Directors and VPs setting company-level product strategy | ⚪ Missing — needs topics and sequences |

### Beginner — current topics (flat format, needs structured sequences)

Review notes inline. Sharpen any flagged titles before designing sequences.

1. What is product strategy and why it matters
2. Understanding your users before setting strategy
3. Defining your target market segment
4. Writing a compelling product vision statement
5. Competitive analysis basics for PMs
6. Value proposition design
7. Setting SMART product goals ⚠️ *"SMART goals" is dated and generic — consider "Setting goals that actually guide decisions"*
8. The difference between strategy and roadmap
9. Prioritisation fundamentals ⚠️ *Overlaps with Execution & Delivery track — discuss whether to keep here or remove*
10. Communicating strategy to your team

### Intermediate — current topics (flat format, needs structured sequences)

1. Jobs to be done as a strategy lens
2. Market sizing: TAM, SAM, and SOM
3. Identifying and strengthening product moats
4. Product-market fit — signals and how to measure them
5. Platform vs point solution strategy
6. Strategic bets under uncertainty ⚠️ *Too vague — sharpen to something like "Making a bet you can defend — structured decision-making under uncertainty"*
7. Network effects in product design
8. Growth loops and compounding advantages
9. Freemium as a strategic choice
10. Sequencing your market entry

### Advanced — current topics (flat format, needs structured sequences)

1. Category creation vs market disruption
2. Ecosystem strategy and partner leverage
3. Pricing as a strategic weapon
4. Horizontal vs vertical product expansion
5. Competitive response playbooks
6. Make vs buy vs partner decisions at scale
7. Strategic pivots — when and how
8. Two-sided markets and platform design
9. Building a strategic narrative for executives
10. Long-range product strategy under ambiguity ⚠️ *Too vague — sharpen to "Planning beyond 18 months — how to think about strategy when you can't predict the market"*

### Expert — missing, proposed topics

Through-line: *you are accountable for product strategy at the company level — where product bets and business strategy are the same thing.*

1. When product strategy is company strategy
2. Portfolio strategy — managing multiple bets across a product portfolio
3. Strategic acquisitions — what to buy, what to avoid, and why
4. Platform strategy at scale — building an ecosystem others build on
5. Communicating strategy to a board — what they're actually asking
6. Competitor intelligence at scale — building a strategic radar
7. When to kill a product line — the signals and the decision
8. Strategy in regulated markets — constraints as a strategic lens
9. International product strategy — sequencing and localisation
10. The 10-year vision — what it is, what it isn't, and how to set one

---

## Track 2: Discovery & Research

**Status:** ⚪ Not started — topics exist in flat format, sequences not designed

**Category:** `product-management`

**Description:** Find real problems before you build.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs running their first research or interviews | ⚪ Topics exist, needs structured sequences |
| Intermediate | PMs building a continuous discovery practice | ⚪ Topics exist, needs structured sequences |
| Advanced | Senior PMs and research leads running complex studies | ⚪ Topics exist, needs structured sequences |
| Expert | Directors and VPs building a research practice at org scale | ⚪ Missing — needs topics and sequences |

### Beginner — current topics (flat format, needs structured sequences)

1. Why discovery matters before building
2. How to recruit research participants
3. Writing a good discussion guide
4. Running your first user interview
5. Surveys vs interviews — when to use each
6. Affinity mapping your research notes
7. Spotting and avoiding confirmation bias
8. What jobs to be done interviews reveal ⚠️ *Slightly advanced for beginner — fine to keep but sequence carefully*
9. Mapping assumptions before building
10. Synthesising findings into insights

### Intermediate — current topics (flat format, needs structured sequences)

1. Continuous discovery habits
2. Opportunity solution trees
3. Contextual inquiry in the field
4. Usability testing methods compared
5. Desirability, feasibility, and viability checks
6. Card sorting and tree testing
7. Quantitative vs qualitative research trade-offs
8. Diary studies for longitudinal insight
9. Problem framing and reframing techniques
10. Building a lightweight research repository

### Advanced — current topics (flat format, needs structured sequences)

1. Mixed-methods research design
2. Survey validity and statistical rigour
3. Ethnographic research for product teams
4. Customer journey mapping at scale
5. Research operations and tooling
6. Democratising research across teams
7. Measuring the ROI of discovery
8. Rapid synthesis under time pressure
9. When to skip discovery ⚠️ *Sharpen to "When speed trumps discovery — making the call and living with it"*
10. Insight activation — making research stick

### Expert — missing, proposed topics

Through-line: *you are building a research capability across an organisation, not just running studies.*

1. Building a research practice from scratch
2. Research strategy — what to study and what to skip
3. Scaling discovery across multiple product teams
4. The research-to-strategy link — making research move the business
5. Research partnerships and external agencies
6. Building organisational research literacy
7. When research says one thing and business says another
8. Longitudinal research programmes — what they are and when to invest
9. Communicating research to boards and executives
10. Research ethics at scale — responsibilities as your reach grows

---

## Track 3: Execution & Delivery

**Status:** ⚪ Not started — topics exist in flat format, sequences not designed

**Category:** `product-management`

**Description:** Roadmaps, prioritisation, shipping under pressure.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs learning to run a team and ship features | ⚪ Topics exist, needs structured sequences |
| Intermediate | PMs managing complexity, debt, and cross-functional delivery | ⚪ Topics exist, needs structured sequences |
| Advanced | Senior PMs and leads managing delivery at programme scale | ⚪ Topics exist, needs structured sequences |
| Expert | Directors and VPs accountable for delivery culture and org-level output | ⚪ Missing — needs topics and sequences |

### Beginner — current topics (flat format, needs structured sequences)

1. What makes a good product roadmap
2. Roadmap formats and when to use them
3. Now-next-later planning
4. Writing clear user stories
5. Acceptance criteria that engineers trust
6. Working effectively with engineers
7. Sprint planning for PMs
8. Managing dependencies between teams
9. Definition of done
10. Running a useful retrospective

### Intermediate — current topics (flat format, needs structured sequences)

1. RICE prioritisation scoring ⚠️ *Consider whether this is beginner-level — RICE is widely taught at entry level*
2. Opportunity scoring frameworks
3. Ruthless prioritisation under constraints
4. Technical debt — when to say yes
5. Feature flags and phased rollouts
6. Incident management for PMs
7. Scope creep and how to handle it
8. Cross-functional team dynamics
9. Working in a platform or infra team
10. Delivering with distributed teams

### Advanced — current topics (flat format, needs structured sequences)

1. OKRs for product teams
2. Portfolio-level prioritisation
3. Programme management across squads
4. Dependency management at scale
5. Quarterly planning at a growing company
6. Technical strategy for non-engineers
7. Org design and its impact on delivery
8. Delivery risk identification and mitigation
9. Scaling agile beyond a single scrum team
10. Build vs buy at the portfolio level

### Expert — missing, proposed topics

Through-line: *you are accountable for how the whole organisation ships — culture, structure, and pace.*

1. Delivery at company scale — what changes when you have 50 teams
2. Building a delivery culture — what it is and how to create it
3. Shipping when you can't fail — high-stakes delivery leadership
4. Org design choices and their delivery consequences
5. Managing delivery across geographies and time zones
6. The CPO delivery agenda — what you own at the top
7. When to slow down to go faster — the counter-intuitive call
8. Leading through a major platform migration
9. Post-mortem culture at scale — learning from failure as an org
10. Delivery in a public company — the constraints you haven't faced yet

---

## Track 4: Metrics & Analytics

**Status:** ⚪ Not started — beginner level missing entirely, existing levels in flat format

**Category:** `product-management`

**Description:** Instrument, measure, and read the signal.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs with limited data experience who need a foundation | ⚪ Missing — needs topics and sequences |
| Intermediate | PMs who can read data and need to run experiments | ⚪ Topics exist, needs structured sequences |
| Advanced | Senior PMs who own measurement strategy | ⚪ Topics exist, needs structured sequences |
| Expert | Directors and VPs building data culture and measurement strategy | ⚪ Missing — needs topics and sequences |

**Note:** Beginner is the most urgent gap in the entire curriculum — understanding what a metric is, reading a dashboard, avoiding vanity metrics — these are day-one PM skills with no entry point in the current content.

### Beginner — missing, proposed topics

Through-line: *you understand what metrics are, why they matter, and how to use them to make better decisions.*

1. Why metrics matter — the PM's relationship with data
2. What makes a good metric
3. Leading vs lagging indicators
4. Understanding your funnel
5. Reading a dashboard without a data analyst
6. Vanity metrics vs actionable metrics
7. Common metric mistakes PMs make
8. Your first north star metric
9. When the numbers are wrong — data quality basics
10. Asking better questions of your data

### Intermediate — current topics (flat format, needs structured sequences)

1. North star metrics — choosing one that matters
2. HEART framework for UX measurement
3. Defining activation and retention metrics
4. Funnel analysis and where users drop off
5. Cohort analysis for retention
6. A/B testing fundamentals
7. Statistical significance explained for PMs
8. Instrumentation and event tracking
9. Building dashboards teams actually use
10. Avoiding metric gaming

### Advanced — current topics (flat format, needs structured sequences)

1. Causal inference vs correlation in product data
2. Multi-armed bandit experiments
3. Novelty effects and how they skew results
4. Guardrail metrics and sensitive reaction groups
5. LTV, CAC, and payback period
6. Engagement quality vs raw engagement
7. Measuring network effects
8. Metric trees and decomposition
9. Ecosystem-level measurement
10. Data-informed vs data-driven culture

### Expert — missing, proposed topics

Through-line: *you are building a measurement culture and a data strategy across the organisation.*

1. Building a measurement culture — what it takes and why most companies fail
2. The data team relationship — how to get what you need as a product leader
3. Experimentation at scale — when to run fewer, better tests
4. Metric strategy — what your company chooses to measure and why
5. Communicating data to boards and executives
6. When data and intuition conflict — making the call
7. Privacy-preserving measurement — the constraints ahead
8. The analytics stack — what a VP of Product needs to understand
9. Building self-serve analytics — the org capability you need
10. Measurement for AI features — the expert view

---

## Track 5: Leadership & Influence

**Status:** ⚪ Not started — beginner level missing, existing levels in flat format

**Category:** `product-management`

**Description:** Stakeholders, narrative, and the path to Director+.

### Level structure

| Level | Who it's for | Status |
|---|---|---|
| Beginner | PMs in their first 1–2 years learning to influence without authority | ⚪ Missing — needs topics and sequences |
| Intermediate | Mid-level PMs building credibility and managing up | ⚪ Topics exist, needs structured sequences |
| Advanced | Senior PMs moving into lead or director roles | ⚪ Topics exist, needs structured sequences |
| Expert | VPs and CPOs leading product organisations | ⚪ Topics exist, needs structured sequences |

### Beginner — missing, proposed topics

Through-line: *you can get things done, build trust, and start to influence the people and decisions around you.*

1. Why PMs need influence without authority
2. Understanding what motivates different people at work
3. How to get things done without a title
4. Building trusted relationships in your first year
5. Disagreeing without damaging the relationship
6. Reading the room — emotional intelligence basics for PMs
7. When to push back and when to let it go
8. Getting credit for your work without making enemies
9. The PM's relationship with their manager
10. Your first 90 days — building influence from zero

### Intermediate — current topics (flat format, needs structured sequences)

1. Influencing without authority
2. Stakeholder mapping and prioritisation
3. Writing persuasive product documents
4. Running effective product reviews
5. Giving and receiving feedback well
6. Managing up to your exec sponsors
7. Leading through ambiguity
8. Building trust with engineering leads
9. Handling disagreement in cross-functional teams
10. Understanding the PM career ladder ⚠️ *Slightly out of place here — consider moving to a career track if one is added*

### Advanced — current topics (flat format, needs structured sequences)

1. Executive communication — what changes at director level
2. Mentoring and growing junior PMs
3. Creating psychological safety in product teams
4. Navigating organisational politics
5. Strategic storytelling for product leaders
6. Building a product culture
7. Hiring and structuring PM interviews
8. Managing a team of PMs
9. Product leadership during a company downturn
10. Moving from IC to manager

### Expert — current topics (flat format, needs structured sequences)

1. CPO and VP Product responsibilities compared
2. Board-level product communication
3. Product considerations in M&A
4. Building a product org from scratch
5. Product strategy at company scale
6. Investor relations for product leaders
7. Creating a product-led growth culture
8. Succession planning in product leadership
9. 10-year product vision setting
10. Operating as the senior-most product person

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
5. Navigating political environments
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
6. The executive sponsor relationship — how to make it work
7. When stakeholders are wrong — making the case with data and narrative
8. Global stakeholder management across geographies and cultures
9. Managing external stakeholders — customers, partners, analysts
10. Stakeholder management during a crisis or public incident

### Expert — missing, proposed topics

Through-line: *you manage boards, investors, and the most senior stakeholders in the business.*

1. Managing your board of directors as a product leader
2. The CEO relationship — how to make it work and what breaks it
3. Investor relations for product leaders
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

## Track 9: Product Design & UX for PMs

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

## Track 10: Technical Skills for PMs

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
