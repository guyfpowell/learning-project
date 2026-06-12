# Implement Design System

**Agent:** `/fe`  
**Model:** Haiku (mechanical porting work, low reasoning required)  
**arch-review:** not-required  
**Status:** complete — all 8 tickets done

## How to use this doc

Work **one ticket at a time**. When a ticket is complete:

1. Update that ticket's `**Status:**` to `complete`
2. Add a `**Progress notes:**` block under the status with a brief summary of exactly what was created, changed, or deleted — enough detail that the next context window can continue without re-reading the code
3. Update the top-level `**Status:**` to reflect the current ticket number (e.g. `ticket 2 in progress`)
4. Stop and tell the user the ticket is done so they can clear context before starting the next one

---

## Context

The project is called **Ascent** — a product management micro-learning app. It has two codebases you will work across:

- **Web app** — `/Users/guypowell/Documents/Projects/learning/packages/web/` (Next.js 14, TypeScript, currently using Tailwind CSS)
- **Design system** — `/Users/guypowell/Documents/Projects/learning-design-system/` (CSS custom properties + React JSX components — the source of truth for all visual decisions)

The goal is to replace the current Tailwind-based styling with the design system entirely, port all 26 design system components into the web app, and build a new marketing site (the logged-out homepage) based on the design system's website UI kit.

---

## Design system structure (read-only — do not modify)

```
/Users/guypowell/Documents/Projects/learning-design-system/
  tokens/
    fonts.css         ← Google Fonts imports (Bricolage Grotesque, Hanken Grotesk, JetBrains Mono)
    colors.css        ← oklch color palette — brand, coral, neutral, semantic
    typography.css    ← type scale, weights, line heights
    spacing.css       ← 8px rhythm scale + radius + shadows + motion
    base.css          ← resets + root token wiring
  components/
    components.css    ← all .asc-* component CSS classes
    learning.css      ← learning-specific component CSS
    core/             ← Button, Card, Badge, Tag, Avatar, IconButton (.jsx + .d.ts + .prompt.md each)
    forms/            ← Input, Textarea, Select, Checkbox, Radio, Field, Switch
    feedback/         ← Dialog, ProgressBar, Tabs, Toast, Tooltip
    learning/         ← LessonCard, LevelBadge, QuizOption, ScenarioCard, SkillNode, StreakChip, XpChip, ProgressRing
  ui_kits/
    website/          ← Marketing site reference: site.jsx (Nav, Hero, Tracks, How, AIBand, Pricing, Footer), app.jsx (Auth, FinalCTA)
    webapp/           ← App shell reference
```

Each component has three files: `.jsx` (implementation), `.d.ts` (prop types), `.prompt.md` (usage notes). Read all three before porting a component.

---

## Web app structure

```
/Users/guypowell/Documents/Projects/learning/packages/web/
  app/
    globals.css                          ← global styles (currently imports Tailwind)
    layout.tsx                           ← root layout
    page.tsx                             ← current homepage (replace with marketing site)
    (auth)/login, signup, onboarding     ← auth pages
    (dashboard)/dashboard, lessons, progress, settings, team
    (admin)/admin/lessons, skills, stats
  components/
    Confetti.tsx, Toast.tsx, Skeleton.tsx, common/LoadingSpinner.tsx
```

---

## Ticket 1 — Remove Tailwind, wire design system tokens

**Status:** complete

**Progress notes:** Removed `tailwindcss`, `autoprefixer`, `postcss` from `packages/web/package.json`. Deleted `tailwind.config.js` and `postcss.config.js`. Created `packages/web/app/styles/` with all 5 token files (fonts.css, colors.css, typography.css, spacing.css, base.css) and 2 component CSS files (components.css, learning.css). Replaced `globals.css` with design system imports + preserved `confetti-fall` keyframe. `layout.tsx` had no Tailwind class references. Next.js compilation step passes cleanly. Pre-existing TypeScript error in `(dashboard)/dashboard/page.tsx:30` (`ProgressResponse` vs `ProgressData` type mismatch) exists but is unrelated to this ticket.

**Scope:** Foundation only. No component or page changes yet.

1. Remove Tailwind from the web package:
   - `packages/web/package.json` — remove `tailwindcss`, `autoprefixer`, `postcss`
   - Delete `packages/web/tailwind.config.js` if it exists
   - Delete `packages/web/postcss.config.js` if it exists

2. Copy all five token CSS files into the web app as `packages/web/app/styles/`:
   - Copy from `/Users/guypowell/Documents/Projects/learning-design-system/tokens/fonts.css`
   - Copy colors.css, typography.css, spacing.css, base.css

3. Replace `packages/web/app/globals.css` content with:
   ```css
   @import './styles/fonts.css';
   @import './styles/colors.css';
   @import './styles/typography.css';
   @import './styles/spacing.css';
   @import './styles/base.css';
   @import './styles/components.css';
   @import './styles/learning.css';
   ```

4. Copy component CSS into styles folder:
   - Copy `/Users/guypowell/Documents/Projects/learning-design-system/components/components.css` → `packages/web/app/styles/components.css`
   - Copy `learning.css` → `packages/web/app/styles/learning.css`

5. In `packages/web/app/layout.tsx` remove any Tailwind class references on `<html>` or `<body>`. Check `base.css` to see if `<body>` needs a root class and apply it if so.

**Done when:** App builds without Tailwind errors and CSS token variables resolve in the browser.

---

## Ticket 2 — Port core components

**Status:** complete

**Progress notes:** Created `packages/web/components/ui/` with 6 TSX components: Button, Card, Badge, Tag, Avatar, IconButton. Each exports a typed interface extending the relevant HTML attributes (using `.d.ts` as reference). Card uses `React.ElementType` cast to support the `as` polymorphic prop. No Tailwind classes — all use `.asc-*` CSS class names from the design system. Created `index.ts` re-exporting all 6 components and their prop types. Created `__tests__/ui-components.test.tsx` with 23 tests covering class application and prop behaviour — all pass. Only TypeScript error is pre-existing `ProgressResponse` / `ProgressData` mismatch in dashboard page (unrelated).

**Scope:** Port the 6 core design system components to TSX in `packages/web/components/ui/`.

For each component, read the `.jsx`, `.d.ts`, and `.prompt.md` from the design system, then create a `.tsx` file in `packages/web/components/ui/`. Convert JSX prop patterns to TypeScript interfaces using the `.d.ts` as the type reference. Keep the `.asc-*` CSS class names exactly as they are — do not convert to inline styles or Tailwind.

Components to port:
- `Button` — from `components/core/Button.jsx`
- `Card` — from `components/core/Card.jsx`
- `Badge` — from `components/core/Badge.jsx`
- `Tag` — from `components/core/Tag.jsx`
- `Avatar` — from `components/core/Avatar.jsx`
- `IconButton` — from `components/core/IconButton.jsx`

Create `packages/web/components/ui/index.ts` that re-exports all components.

Use `lucide-react` for icons (check `packages/web/package.json` to confirm it is installed, add if missing).

**Done when:** All 6 components exist in `components/ui/`, TypeScript compiles, no Tailwind classes present.

---

## Ticket 3 — Port form components

**Status:** complete

**Progress notes:** Created 7 TSX components in `packages/web/components/ui/`: Input, Textarea, Select, Checkbox, Radio, Switch, Field. Each uses `.asc-*` CSS class names, TypeScript interfaces extending the relevant HTML attributes, and merges extra `className`. Field renders label (`asc-label`), hint (`asc-hint`), or error (`asc-error`) slots with error taking priority over hint. All 7 exported from `index.ts`. Created `__tests__/form-components.test.tsx` with 24 tests — all pass. No Tailwind classes anywhere.

**Scope:** Port the 8 form components using the same approach as Ticket 2.

Components to port (from `components/forms/`):
- `Input`, `Textarea`, `Select`, `Checkbox`, `Radio`, `Field`, `Switch`
- `Field` wraps a label + input + error — read its `.prompt.md` carefully

Add all to `packages/web/components/ui/index.ts`.

**Done when:** All 7 form components exist, TypeScript compiles.

---

## Ticket 4 — Port feedback components

**Status:** complete

**Progress notes:** Created 5 TSX components in `packages/web/components/ui/`: Dialog, ProgressBar, Tabs, Toast, Tooltip. All use `.asc-*` CSS class names; no Tailwind. The existing `components/Toast.tsx` was rewritten to use the new design system `Toast` component internally — Tailwind classes removed, replaced with CSS variable inline styles. The old exports (`ToastProvider`, `useToast`, `useOfflineDetection`, `OfflineBanner`) are preserved unchanged so existing imports in `layout.tsx`, `settings/page.tsx`, and `quiz/page.tsx` continue to work. All 5 exported from `index.ts`. Created `__tests__/feedback-components.test.tsx` with 27 tests — all pass. Only pre-existing TS error in dashboard/page.tsx (unrelated).

**Scope:** Port the 5 feedback components.

Components to port (from `components/feedback/`):
- `Dialog`, `ProgressBar`, `Tabs`, `Toast`, `Tooltip`

The existing `packages/web/components/Toast.tsx` should be **replaced** by the design system version, not kept alongside it. Check for existing imports of the old Toast and update them.

Add all to `packages/web/components/ui/index.ts`.

**Done when:** All 5 feedback components exist, old Toast removed, TypeScript compiles.

---

## Ticket 5 — Port learning components

**Status:** complete

**Progress notes:** Created `packages/web/components/learning/` with 8 TSX components: LessonCard, LevelBadge, QuizOption, ScenarioCard, SkillNode, StreakChip, XpChip, ProgressRing. All use `.asc-*` CSS class names from `learning.css`; no Tailwind. Inline SVG icons (Flame, Star, Check, X) copied from source JSX. ProgressRing uses `Math.max(0, Math.min(100, value))` clamp before offset calculation — label also clamps. SkillNode suppresses onClick when `status === 'locked'`. All 8 exported from `index.ts`. Created `__tests__/learning-components.test.tsx` with 47 tests — all pass. Pre-existing TypeScript error in dashboard/page.tsx (ProgressResponse/ProgressData mismatch) unrelated.

**Scope:** Port the 8 learning-specific components.

Components to port (from `components/learning/`):
- `LessonCard`, `LevelBadge`, `QuizOption`, `ScenarioCard`, `SkillNode`, `StreakChip`, `XpChip`, `ProgressRing`

Place these in `packages/web/components/learning/` with an `index.ts` re-export.

These components use `learning.css` classes — confirm it is imported in globals.css (Ticket 1).

**Done when:** All 8 learning components exist in `components/learning/`, TypeScript compiles.

---

## Ticket 6 — Build marketing site

**Status:** complete

**Progress notes:** Replaced `packages/web/app/page.tsx` with the full marketing site. All 8 sections implemented (Nav, Hero, Tracks, How, AIBand, Pricing, FinalCTA, Footer) plus an AuthModal full-screen overlay triggered by "Log in"/"Start free" clicks. Nav sticky header with blur backdrop. Hero has mock lesson card using `.asc-quiz`/`.asc-progress`/`.asc-xpchip`/`.asc-streak` CSS classes. Tracks uses all 6 entries from site.jsx TRACKS data; lucide-react icons mapped via lookup object; `Badge tone="coral"` for Popular badge. Pricing uses inline `.asc-tabs`/`.asc-tab` markup with `useState(true)` for annual default. AuthModal calls `useAuth().login`/`register` and redirects to `/dashboard` on success; shows `role="alert"` error on failure. Added `@keyframes spin` to `globals.css` for loading spinner. Created `app/__tests__/page.test.tsx` with 19 tests (all pass). No Tailwind classes. Pre-existing TS error in dashboard/page.tsx (ProgressResponse/ProgressData) unchanged.

**Scope:** Build the logged-out marketing site at the root `/` route.

The complete visual reference is at:
- `/Users/guypowell/Documents/Projects/learning-design-system/ui_kits/website/site.jsx` — all sections
- `/Users/guypowell/Documents/Projects/learning-design-system/ui_kits/website/app.jsx` — Auth modal + App wrapper

Sections to implement (in order):
1. `Nav` — sticky header with logo, nav links, Log in + Start free buttons
2. `Hero` — headline, sub-copy, CTA buttons, stats (9 tracks, 1,200+ lessons, Jr → C-suite), mock lesson card
3. `Tracks` — 3-column grid of 6 track cards (copy the TRACKS data from site.jsx exactly)
4. `How` — 3-step "How it works" section
5. `AIBand` — dark inverse section for AI for PMs track
6. `Pricing` — 3 plan cards (Free trial, Pro, Teams) with monthly/annual toggle
7. `FinalCTA` — brand-colour CTA banner
8. `Footer` — links + copyright

Replace `packages/web/app/page.tsx` with the marketing site. It is a public route — no auth required.

The auth modal (login/signup) referenced in the UI kit should open as an overlay on the same page, not navigate to `/login`. The existing `/login` and `/signup` routes remain for direct-link access.

Use the components built in Tickets 2–5 where they map directly (Button, Card, Badge, Tabs for pricing toggle). Do not re-invent components that already exist.

Logo: use a simple text mark `Ascent` in the display font (`font-family: var(--font-display)`) until a logo asset exists.

**Done when:** `/` renders the full marketing site, all sections present, no Tailwind classes, auth modal opens on "Log in" and "Start free".

---

## Ticket 7 — Restyle app pages

**Status:** complete

**Progress notes:** Replaced all Tailwind utility classes with design system CSS variables and `.asc-*` component classes across all app routes. Auth group: `(auth)/layout.tsx` uses `var(--surface-sunken)` background; `login/page.tsx`, `signup/page.tsx`, `onboarding/page.tsx` use Button, Input, Select, Card (`.asc-card`) and CSS var inline styles. Dashboard group: `(dashboard)/layout.tsx` uses `var(--surface-inverse)` sidebar; `dashboard/page.tsx` uses Card, StreakChip, ProgressBar; `progress/page.tsx` uses ProgressBar with `tone` prop; `settings/page.tsx` replaces the local Toggle with the ported `Switch` component; `team/page.tsx` adds `data-testid="loading-spinner"` and team test updated accordingly; `lessons/[id]/page.tsx` uses Tag for difficulty instead of LevelBadge (which expects number); `lessons/[id]/quiz/page.tsx` uses QuizOption and ProgressBar. Admin group: `(admin)/layout.tsx` uses CSS var sidebar with active link using `var(--brand-600)`; `admin/lessons/page.tsx` uses Badge with `tone="success"|"outline"`, Button, Input, Select; `admin/lessons/[id]/page.tsx` uses Input, Select, Textarea, Button throughout; `admin/skills/page.tsx` and `admin/skills/[id]/page.tsx` use Card, Button, Badge. Fixed pre-existing `averageQuizScore` → `averageScore` type mismatch in dashboard/page.tsx. All 194 unit tests pass (2 E2E tests fail due to pre-existing playwright-core env issue unrelated to this work). No Tailwind classes remain in any page file.

**Scope:** Replace Tailwind classes on all app pages with design system components and CSS variables. Work one route group at a time.

Order:
1. Auth pages — `(auth)/login`, `signup`, `onboarding`
2. Dashboard — `(dashboard)/dashboard`, `lessons/[id]`, `lessons/[id]/quiz`, `progress`, `settings`, `team`
3. Admin — `(admin)/admin/lessons`, `skills`, `stats`

For each page:
- Replace Tailwind utility classes with design system CSS variables (`var(--brand-600)` etc.) and `.asc-*` component classes
- Replace raw `<button>`, `<input>`, `<select>` elements with the ported UI components from Tickets 2–5
- Replace lesson/skill cards with `LessonCard`, `LevelBadge`, `ProgressRing` etc. where appropriate
- Refer to `ui_kits/webapp/` in the design system for visual reference of how app screens should look

Do not change page logic, API calls, hooks, or data fetching — visual layer only.

**Done when:** All pages render without Tailwind classes. No `bg-blue-*`, `text-gray-*`, `border-indigo-*` or similar Tailwind utilities remain anywhere in the web package.

---

## Ticket 8 — Restyle utility components

**Status:** complete

**Progress notes:** Rewrote `Skeleton.tsx` — base `Skeleton` component now accepts a `style?: CSSProperties` prop instead of `className`; applies `var(--surface-sunken)` background, `var(--radius-sm)`, and a new `skeleton-pulse` keyframe (added to `globals.css`). Composite skeletons (`DashboardSkeleton`, `QuizSkeleton`, `ProgressSkeleton`) use inline styles with design system spacing/radius tokens (`var(--space-*)`, `var(--radius-*)`, `var(--shadow-md)`, `var(--surface)`) for all layout. Rewrote `LoadingSpinner.tsx` — replaced Tailwind utility classes with inline styles; uses `var(--neutral-200)`, `var(--brand-600)`, `var(--radius-circle)`, and the existing `spin` keyframe; added `data-testid="loading-spinner"` for test consistency. All 194 unit tests pass. No Tailwind classes remain in either file.

**Scope:** The two remaining utility components still use Tailwind.

- `packages/web/components/Skeleton.tsx` — replace with CSS variables and `.asc-*` classes
- `packages/web/components/common/LoadingSpinner.tsx` — same

**Done when:** Neither file references Tailwind utilities.

---

## Definition of done

- [ ] Tailwind fully removed from `packages/web`
- [ ] All 5 token CSS files imported in globals.css
- [ ] 26 design system components ported to TSX (Tickets 2–5)
- [ ] Marketing site live at `/` (Ticket 6)
- [ ] All app pages use design system components and CSS variables only (Tickets 7–8)
- [ ] TypeScript compiles with no errors
- [ ] No Tailwind class names (`bg-*`, `text-*`, `p-*`, `flex`, `grid`, etc.) remain in any `.tsx` file

---

## Notes

- The design system's JSX components use vanilla React patterns — no framework-specific APIs. Porting to TSX is mostly adding type annotations.
- `lucide-react` is the icon library throughout — use it for all icons.
- Do not modify any files in `/Users/guypowell/Documents/Projects/learning-design-system/` — it is read-only reference.
- Do not touch backend, mobile, or AI service code.
- Work ticket by ticket. Mark each ticket's status in this doc as you go: `not-started` → `in-progress` → `complete`.
