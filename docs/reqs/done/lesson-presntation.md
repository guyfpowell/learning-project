---
title: Lesson Presentation
status: dev-complete
arch-review: not-required
---

## Problem

The lesson page is missing key fields from the generated lesson data. Currently it only shows title, metadata (day/duration/difficulty), optional image, and content. The `summary`, `keyTakeaway`, `topicName`, `isCapstone`, `lessonIndex`, and `totalLessons` fields exist in `generated-lessons.json` but are discarded at seed time — they are not in the Prisma schema or shared types.

## Desired Presentation Flow

1. **Title** — already shown
2. **Summary** — short description of what the lesson covers (below title, above content)
3. **Content** — the lesson body
4. **Complete Lesson** button
5. **After completion**: show **keyTakeaway** (the single sentence the user should take away), then the Take Quiz button

## Data Fields to Add

All 6 fields exist in `generated-lessons.json` and must be threaded through the full stack:

| Field | Type | DB column | Notes |
|---|---|---|---|
| `summary` | `String?` `@db.Text` | summary | Short lesson description |
| `keyTakeaway` | `String?` `@db.Text` | key_takeaway | Shown after lesson completion, before quiz |
| `topicName` | `String?` | topic_name | Groups lessons within a track |
| `isCapstone` | `Boolean?` | is_capstone | Marks last lesson in a topic group |
| `lessonIndex` | `Int?` | lesson_index | Position within the topic group |
| `totalLessons` | `Int?` | total_lessons | Total lessons in the topic group |

## Scope of Changes

### 1. Prisma schema (`prisma/schema.prisma`)
Add 6 optional fields to the `Lesson` model.

### 2. Migration
Run `prisma migrate dev --name add-lesson-presentation-fields` in a real terminal (requires TTY).

### 3. Shared types (`packages/shared/src/types/lesson.ts`)
Add 6 optional fields to the `Lesson` interface.

### 4. Seed (`prisma/seed.ts`)
- Delete the 3 hand-crafted placeholder lessons (day 1–3, "What is Product Strategy?", "Understanding Your Users", "Setting Strategic Goals")
- Thread all 6 new fields into the generated lesson `create` call

### 5. Lesson page (`packages/web/app/(dashboard)/lessons/[id]/page.tsx`)
- Show `summary` between the title/metadata block and the content
- After completion, show `keyTakeaway` above the Take Quiz button

### 6. No API changes needed
The Prisma client returns all model fields by default; the controller passes the lesson object through unchanged.

## Acceptance Criteria

- [x] `pnpm build` passes with no TypeScript errors
- [x] Lesson page shows summary below title
- [x] After clicking "Complete Lesson", keyTakeaway is shown above "Take Quiz"
- [x] Placeholder lessons (days 1–3) no longer appear after re-seed
- [x] Generated lessons include all 6 new fields after re-seed

---

## ✅ Implementation Summary

**Migration**: `20260611133025_add_lesson_presentation_fields` — applied successfully.

**Files changed**:
- `prisma/schema.prisma` — 6 optional fields added to `Lesson` model (`summary`, `keyTakeaway`, `topicName`, `isCapstone`, `lessonIndex`, `totalLessons`)
- `packages/shared/src/types/lesson.ts` — 6 optional fields added to `Lesson` interface
- `prisma/seed.ts` — removed 3 hand-crafted placeholder lessons + empty skill path; `GeneratedLesson` type and lesson create call updated to populate all 6 new fields
- `packages/web/app/(dashboard)/lessons/[id]/page.tsx` — `summary` displayed in italic below title; `keyTakeaway` shown in a branded left-border callout after completion, above Take Quiz

**Seed result**: 672 AI-generated lessons across 8 skill paths, all with `summary` and `keyTakeaway` populated.
