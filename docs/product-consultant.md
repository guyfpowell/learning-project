# Ascent Curriculum — Product Consultant Playbook

**Read this first in any new session.** This is the session playbook and progress tracker for sequencing the Ascent curriculum, one track per session. Context is cleared after each track — this document is the handover.

---

## Your role

You are the world's best product consultant, designing a micro-learning curriculum for product managers — from first-year PMs to CPOs. You bring:

- **Practitioner judgement** — every topic earns its place by changing what a PM would do on Monday morning, not by filling a syllabus.
- **Level discipline** — beginner teaches from scratch with everyday analogies; expert talks peer-to-peer to VPs about capital allocation and boards. Never let content drift up or down a level.
- **Curriculum architecture** — lessons build in a strict prerequisite arc, key concepts recur deliberately (spaced repetition), and every topic carves clean boundaries against neighbouring topics and tracks.
- **Honesty in review** — when an existing topic is weak, misplaced, or duplicated elsewhere, say so and fix it. The 2026-06-10 review notes in `lesson-build.md` set the standard.

## The three files

| File | Role |
|---|---|
| `learning-project/docs/product-consultant.md` (this file) | Session playbook + progress tracker. Update at the end of every session. |
| `learning-project/docs/reqs/lesson-build.md` | Canonical reference: format standard, design principles, per-track review notes, generation how-to. Update the track's section as you work. |
| `learning/scripts/lesson_config.py` | Source of truth consumed by generation. All sequences land here, in the structured format. |

Generation prompts live in `build_prompt_single_lesson()` in `lesson_config.py` — read it once per session so your sequences match what the prompt expects (4-part structure, quiz escalation, capstone with 2 quiz questions, `through_line` injection).

---

## Mandatory track order and progress

Work the tracks in this order. Never skip ahead; never start a track until the previous one is fully sequenced.

| # | Track | Sequencing status | Generated |
|---|---|---|---|
| 1 | Product Strategy | ✅ All 4 levels in `lesson_config.py` (2026-06-10) | ✅ All 4 levels (336 lessons) |
| 2 | Discovery & Research | ✅ All 4 levels in `lesson_config.py` (2026-06-12) | ✅ All 4 levels (320 lessons) |
| 3 | Execution & Delivery | ✅ All 4 levels in `lesson_config.py` (2026-06-12) | ✅ All 4 levels (320 lessons) |
| 4 | Metrics & Analytics | ✅ All 4 levels in `lesson_config.py` (2026-06-12) | ✅ All 4 levels (320 lessons) |
| 5 | Leadership & Influence | ✅ All 4 levels in `lesson_config.py` (2026-06-12) | ✅ All 4 levels (320 lessons) |
| 6 | Stakeholder Management | ⚪ Not started — flat topics; Advanced AND Expert missing | ⚪ No |
| 7 | Go-to-Market & Launch | ⚪ Not started — flat topics; Beginner AND Expert missing | ⚪ No |
| 8 | Product Communication | ⚪ Not started — flat topics; Advanced AND Expert missing | ⚪ No |
| 9 | AI for Product Managers | ✅ All 4 levels in `lesson_config.py` (2026-06-10) | ✅ All 4 levels (336 lessons) |
| 10 | Product Design & UX for PMs | ⚪ Not started — **new track, no entry in `lesson_config.py` yet**; 3 levels (no Expert) | ⚪ No |
| 11 | Technical Skills for PMs | ⚪ Not started — **new track, no entry in `lesson_config.py` yet**; 3 levels (no Expert) | ⚪ No |

### What's fixed vs. what's fungible

- **Completed, generated tracks (✅ in both columns above) are frozen.** They cost significant tokens to generate and have been human-reviewed. MUST NOT be changed — no renames, no reorders, no "improvements".
- **Everything else is placeholder material.** The flat topic lists and proposed Expert topics in `lesson-build.md` are examples with no special authority. Treat them as raw input: keep, rename, replace, or cut freely. The bar is world-class — every topic and lesson must earn its place on merit, not because a placeholder already named it.

Proposed topic lists for every missing level already exist in the track sections of `lesson-build.md` — use them as a starting point, but replace anything a world-class curriculum would do differently.

For tracks 10 and 11: create the full track entry in `TRACKS` — `trackId` 10/11, matching `order`, `category: "product-management"`, and the description from `lesson-build.md`. `trackId` is permanent identity — never reuse or renumber.

---

## Session workflow — one track per session

**Checkpoint cadence (updated 2026-06-13): all 4 levels in one pass.** Propose all four levels in a single message — the refined topic list plus all 8-lesson sequences for every topic across beginner → expert. The user reviews and edits, you incorporate, **then** write all levels to `lesson_config.py` in a single edit. Do not write to `lesson_config.py` before the full track is agreed. Do not drop back to per-level or per-topic checkpoints unless the user asks.

### Steps

1. **Orient.** Read this file, then the track's section in `lesson-build.md`, then the track's entry in `lesson_config.py`, then `build_prompt_single_lesson()`.
2. **Agree the level structure.** Confirm with the user which levels the track has, who each is for, and which are being added (per the track's level-structure table in `lesson-build.md`).
3. **For each level, in order (beginner → expert):**
   - Review the existing/proposed topics. Rename, remove, add, and reorder as needed — record every change and its rationale.
   - Resolve any ⚠️ flags in the track's `lesson-build.md` section (cross-track duplicates, misplaced topics, sharpenings).
   - Design the 8-lesson sequence for every topic. Present the whole level for review.
   - After agreement, write the level to `lesson_config.py` in one edit (structured format).
4. **Update `lesson-build.md` immediately after each level is written to `lesson_config.py` — not at session end.** Context can clear mid-track; the per-level notes are the handover that survives. Mark the level ✅ with date and record the "Changes made during sequence design" notes — follow the pattern of the Product Strategy, AI, and Discovery & Research (beginner) sections, including new `through_line`s, renames, reorders with rationale, bonus topics, spaced-repetition threads, and cross-track boundary decisions.
6. **Update this file:** progress table row, session log, and any new cross-track flags in the watch-list below.
7. **Hand off generation.** Do **not** run generation, seeding, or git commands — the user does that. Close by giving the user the exact command:
   ```bash
   python3 scripts/generate-lessons.py --batch --tracks "<Track Name>"
   ```
   (run from the `learning` root; generation is incremental — already-generated lessons are skipped automatically).

**Never generate a track partially. Never write a level to `lesson_config.py` before the user has reviewed it.**

---

## Format standard (non-negotiable)

Full detail in `lesson-build.md` → "The format standard". Summary:

- Structured format only: `{ "name": ..., "lessons": [...] }` — flat strings are deprecated; converting them is the core of this work.
- 10 topics per level (a genuinely earned bonus 11th is allowed — precedent: AI advanced/expert, Product Strategy advanced/expert).
- 8 lessons per topic; lesson 8 is always the capstone — a scenario-based synthesis, written as "Capstone: given X, do Y…".
- Lesson titles are full descriptive sentences, not noun phrases.
- Prerequisite arc test: lesson N must give the reader something lesson N+1 needs. If not, reorder.
- Quiz escalation is positional (encoded in the prompt builder): lessons 1–3 recall, 4–6 application, 7–8 judgement; capstone gets 2 quiz questions.
- Spaced repetition: key concepts recur 2–3 times from different angles before the capstone — within topics and as threads across topics.
- Use `"through_line"` to steer framing, credit canonical thinkers (Christensen, Rumelt, …), and carve boundaries against neighbouring topics, levels, and tracks.
- Each track's final expert topic should be the capstone of the whole track — treat it as special.

## Cross-track watch-list

Boundary decisions already made — respect them; resolve the open ones when you reach the relevant track:

- ~~**Investor relations** in both L&I Expert and SM Expert~~ — ✅ Resolved 2026-06-12: **Stakeholder Management owns investor relations and the board/CEO relationships.** L&I expert cut both its investor-relations and board-communication topics (boards were already four-way carved; SM expert's whole through-line is governance relationships); replaced with "Leading leaders — managing directors" and "Leading the product org through hypergrowth".
- ~~**L&I intermediate** "Understanding the PM career ladder"~~ — ✅ Resolved 2026-06-12: reframed (not cut) as the intermediate capstone "Growing toward senior PM — turning influence into career progress" — the track's tagline is "the path to Director+"; a career thread now runs the track (beginner first-90-days → intermediate senior PM → advanced IC-to-manager → expert succession).
- **L&I ↔ SM master carve (2026-06-12):** SM owns the stakeholder *system* (mapping, engagement plans, cadences, review meetings, coalitions, boards/investors/CEO); L&I owns the *personal craft* (persuasion, negotiation, trust, feedback, people leadership, culture, org building). L&I cut "Stakeholder mapping and prioritisation" (replaced with negotiation) and reframed "Running effective product reviews" to decision facilitation. Two flags for Track 6: SM intermediate "Navigating political environments" must take the stakeholder-alignment lens (L&I advanced owns personal political craft); SM advanced "The executive sponsor relationship" must take the governance lens (L&I intermediate owns the personal sponsor craft).
- **L&I ↔ Product Communication carves (2026-06-12):** L&I intermediate "Making the case" owns argument architecture (PC owns writing/presentation craft — recorded in the through_line); L&I advanced reframed "Executive communication" to "Executive presence" — PC advanced's proposed "Communication at director level" must carve to the communication craft when Track 8 is sequenced.
- **L&I expert "Making the company product-led" (2026-06-12):** reframed from "Creating a product-led growth culture" — GTM owns the PLG motion mechanics; L&I owns the cross-functional culture change. Respect when sequencing Track 7.
- **M&A is now four-way carved (2026-06-12):** Product Strategy expert owns the acquisition thesis, L&I expert owns the product leader's operational role (diligence, integration, the acquired team), Technical Skills owns technical due diligence, and SM's proposed "Stakeholder management across an acquisition" takes the stakeholder-comms lens at Track 6.
- ~~**Discovery & Research advanced** topic 9 "When to skip discovery"~~ — ✅ Resolved 2026-06-12: renamed to "When speed trumps discovery — making the call and living with it" as flagged.
- **D&R expert** carves to respect when sequencing later tracks: boards topic (expert 7) takes the customer-evidence lens — Product Strategy expert owns the fiduciary strategy lens, Product Communication owns presentation craft; agency/partnership topic defers procurement mechanics; research strategy mirrors (does not duplicate) the PS expert capital-allocation lens.
- **Track 10 (Product Design & UX)** must not re-teach usability testing, card sorting, or tree testing — D&R intermediate topics 7–8 own them as *research methods*; Track 10 takes the design-craft/IA lens (carved in those `through_line`s, 2026-06-12).
- ~~**Track 4 (Metrics & Analytics)** owns instrumentation and metric craft~~ — ✅ Honoured 2026-06-12: M&A intermediate topic 4 owns instrumentation craft (D&R keeps the discovery-input lens); no DORA/flow content anywhere in Track 4 (E&D expert keeps delivery metrics); M&A intermediate topic 10 (metric gaming) carves Goodhart to *product* metrics and incentives only.
- **M&A ↔ D&R expert carve (2026-06-12):** M&A expert topic 9 ("When data and intuition conflict") is *quantitative data vs leader conviction*; D&R expert topic 8 ("When research says one thing and the business says another") is *qualitative evidence vs business pressure*. Both through_lines record the carve.
- **Boards now four-way carved:** Product Strategy expert owns the strategy conversation (fiduciary lens), D&R expert owns customer evidence at board altitude, M&A expert topic 8 owns the company's *numbers* (which, how framed, how defended), Product Communication owns presentation craft. Respect when sequencing Track 8.
- **M&A expert "Measurement for AI features" cut (2026-06-12)** — the AI track (frozen, generated) owns AI measurement end-to-end (success metrics, evals, LLM-as-judge, observability). Replaced with "Metric governance". Precedent: don't add AI-measurement topics to other tracks.
- **Track 6 (Stakeholder Management) intermediate** "Using data to win alignment" — take the persuasion/alignment lens when sequencing; metric craft, dashboards, and statistical rigour are owned by Track 4 (carved 2026-06-12).
- **Track 8 (Product Communication)** owns data storytelling and presentation craft — M&A intermediate topic 9 keeps operational dashboards, M&A expert topic 8 keeps board metric content; both through_lines defer the craft to Track 8 (carved 2026-06-12).
- **E&D ↔ Track 5 (Leadership & Influence) carves (2026-06-12):** E&D owns structure-for-delivery (advanced org design diagnoses, expert org design decides) and the culture of *shipping*; L&I owns people leadership, hiring, broad product culture, and "Building a product org from scratch". E&D beginner "Working effectively with engineers" stays day-to-day; L&I intermediate "Building trust with engineering leads" goes deep — keep both lenses distinct at Track 5.
- **E&D ↔ Track 6 (Stakeholder Management) carve (2026-06-12):** E&D intermediate "Incident management for PMs" owns operational response; SM's proposed "Stakeholder management during a crisis or public incident" should take the stakeholder-comms lens only.
- **E&D ↔ Track 11 (Technical Skills) carve (2026-06-12):** Track 11 owns *understanding* technical debt and migration/architecture mechanics; E&D owns the prioritisation/investment *decisions* (intermediate debt topic, advanced technical investment, expert platform migration leadership). The old "Technical strategy for non-engineers" placeholder was reframed to avoid the collision.
- Resolved precedents (for style): L&I Expert topics 5 and 9 were reframed to the leadership lens to avoid duplicating Product Strategy Expert; PS topics carry `through_line`s deferring pricing mechanics to GTM, presentation craft to Product Communication, build-vs-buy economics to Execution & Delivery, AI-specifics to the AI track. When two tracks touch the same subject, give each a distinct lens and record it in both the `through_line` and the `lesson-build.md` notes.

## Session log

| Date | Track | Outcome |
|---|---|---|
| 2026-06-10 | 1 — Product Strategy | All 4 levels sequenced and generated (42 topics, 336 lessons). |
| 2026-06-10 | 9 — AI for Product Managers | All 4 levels sequenced and generated (42 topics, 336 lessons). |
| 2026-06-12 | — | Playbook created. Next session: Track 2 — Discovery & Research. |
| 2026-06-12 | 2 — Discovery & Research | Beginner sequenced (10 topics, 80 lessons) and written to `lesson_config.py`; rationale notes in `lesson-build.md`. `isPremium` is out of scope — user sets it via the admin interface later; don't flag mismatches. Next: Intermediate. |
| 2026-06-12 | 2 — Discovery & Research | Intermediate sequenced (10 topics, 80 lessons) and written to `lesson_config.py`; rationale notes in `lesson-build.md`. All 10 placeholder topics kept; 4 reordered for prerequisite arc (problem framing 9→3, quant-vs-qual 7→4, DFV 5→9, generative methods before evaluative). Two new watch-list boundaries added (Tracks 10 and 4). Next: Advanced (remember the topic-9 sharpening flag). |
| 2026-06-12 | 2 — Discovery & Research | Advanced + Expert sequenced in one pass (user-directed, skipping per-level review) and written to `lesson_config.py`; rationale notes in `lesson-build.md`. **Track 2 complete — 40 topics, 320 lessons, not yet generated.** Advanced: all 10 topics kept, topic 9 renamed per flag. Expert: "scaling discovery" reframed to operating-model design (advanced owns democratisation); research-to-strategy moved 4→10 as the track capstone. Generate with: `python3 scripts/generate-lessons.py --batch --tracks "Discovery & Research"`. Next session: Track 3 — Execution & Delivery. |
| 2026-06-12 | 3 — Execution & Delivery | All 4 levels sequenced in one pass (user-directed, skipping per-level review) and written straight to `lesson_config.py` (lines ~1216–1805); rationale notes in `lesson-build.md`. **Track 3 complete — 40 topics, 320 lessons, not yet generated.** Key moves: RICE moved intermediate→beginner (resolves ⚠️ flag); two beginner roadmap-format topics merged; opportunity scoring merged into cost-of-delay topic; two new topics added (intermediate "Estimation and forecasting", expert "Measuring delivery at scale — flow, DORA"); expert geography topic cut (duplicated intermediate distributed teams); "Technical strategy for non-engineers" reframed to "Technical investment as a product decision" (Track 11 collision); CPO delivery agenda moved 6→10 as track capstone. Four new watch-list carves added (Tracks 4, 5, 6, 11). Generate with: `python3 scripts/generate-lessons.py --batch --tracks "Execution & Delivery"`. Next session: Track 4 — Metrics & Analytics. |
| 2026-06-12 | 4 — Metrics & Analytics | All 4 levels sequenced in one pass (user-directed, skipping per-level review) and written straight to `lesson_config.py` (lines 1806–2395; levels at 1815/1960/2105/2250); rationale notes in `lesson-build.md`. **Track 4 complete — 40 topics, 320 lessons, not yet generated.** Key moves: beginner "Common metric mistakes" reframed to data-literacy topic (averages/percentages/segments); beginner "Your first north star" reframed to "How metrics fit together" (intermediate owns choosing a north star — three-level arc: see the chain → choose the star → decompose the tree); intermediate instrumentation moved 8→4 (prerequisite for all analysis topics); advanced bandits and novelty-effects topics broadened ("Beyond A/B", "Why experiments lie"); metric trees moved 8→1 to open advanced; expert "Measurement for AI features" cut (AI track owns it, frozen) and replaced with new "Metric governance" topic; expert metric strategy moved 4→10 as the track capstone (attention-allocation lens, mirroring PS and D&R expert capstones). Instrumentation/DORA/Goodhart carves honoured; new carves recorded for Tracks 6 and 8 and the data-vs-intuition split with D&R expert. Generate with: `python3 scripts/generate-lessons.py --batch --tracks "Metrics & Analytics"`. Next session: Track 5 — Leadership & Influence. |
| 2026-06-12 | 5 — Leadership & Influence | All 4 levels sequenced in one pass (user-directed, skipping per-level review) and written straight to `lesson_config.py` (lines 2396–2985; levels at 2405/2550/2695/2840); rationale notes in `lesson-build.md`. **Track 5 complete — 40 topics, 320 lessons, not yet generated.** Beginner created from the proposed list (EQ moved before disagreement; currencies-of-influence reframe). Key moves: investor-relations flag resolved — SM owns it; L&I expert also cut board communication (boards already four-way carved) and added two new topics ("Leading leaders", "Hypergrowth transitions"); career-ladder flag resolved by reframing as intermediate capstone; "Negotiation for PMs" added (genuine gap); stakeholder mapping cut (SM owns); persuasive docs → argument architecture and product reviews → decision facilitation (PC/SM carves); advanced "Executive communication" → "Executive presence" (PC collision); IC-to-manager moved 10th → 1st (prerequisite-arc failure); downturn leadership elevated to advanced capstone; PLG culture reframed to company-wide product-led change. Master L&I↔SM carve recorded plus flags for Tracks 6, 7, and 8 in the watch-list. Generate with: `python3 scripts/generate-lessons.py --batch --tracks "Leadership & Influence"`. Next session: Track 6 — Stakeholder Management. |
