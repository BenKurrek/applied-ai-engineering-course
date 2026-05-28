---
name: learn
description: Entry point for the Applied AI Engineering tutored course. Use when the user types /ai-course:learn or asks to start, resume, or continue the course, or to be taught a course topic. Presents the phases and topics to choose from, then teaches continuously — topic after topic — until the student stops.
---

# /ai-course:learn — Applied AI Engineering course

This skill starts an interactive tutoring session for the Applied AI Engineering course.
It runs in the main conversation: once the student picks where to begin, **you become
the tutor and teach directly here**, turn by turn.

## Step 1 — Load state

The course content is read-only and bundled with the plugin; the student's progress
is written to `~/applied-ai-course/`. On the very first run that directory does not
exist yet.

1. **Bootstrap if new.** If `~/applied-ai-course/` does not exist, this is a new
   student: create it, then create `~/applied-ai-course/PROGRESS.md` by copying
   `${CLAUDE_PLUGIN_ROOT}/course/templates/PROGRESS.template.md` and
   `~/applied-ai-course/LEARNER-PROFILE.md` by copying
   `${CLAUDE_PLUGIN_ROOT}/course/templates/LEARNER-PROFILE.template.md`.
2. Read `~/applied-ai-course/PROGRESS.md` — the topic tracker and "Current position".
3. Read `${CLAUDE_PLUGIN_ROOT}/course/SYLLABUS.md` — the five phases and sixteen
   topics (use it for exact phase/topic names in the menus below).
4. Read `${CLAUDE_PLUGIN_ROOT}/course/tutoring-agent-guide.md` — the authoritative
   teaching procedure; its "Starting or resuming a session" steps cover new-student
   bootstrapping in full.
5. Read `~/applied-ai-course/LEARNER-PROFILE.md` — the model of this student.

## Step 2 — Show where they are, then ask how to begin

In one or two lines, tell the student where they stand. For a returning student, use
their `PROGRESS.md` "Current position" — e.g. *"You're currently on Topic 9 —
Evaluations. Topic 7 is passed (95/100)."* For a **brand-new student** (freshly
created `PROGRESS.md`, nothing passed), just say the course is ready.

Then ask how they want to begin — **using `AskUserQuestion`**, never a free-text
prompt. Two options always fits cleanly in `AskUserQuestion`'s 4-option cap and is
the correct UX for a binary entry choice. Branch on whether they have prior progress:

**Brand-new student** — call `AskUserQuestion` with these two options:
1. **"Start from the beginning"** — *"Topic 1 — LLM Fundamentals, beginning with the
   §1.0 history context."* Set the current position to Topic 1 §1.0 in `PROGRESS.md`
   and **skip straight to Step 5** (teaching). Do not run Steps 3 or 4.
2. **"Pick a specific topic to start from"** — Continue to Step 3 (phase pick) → Step
   4 (topic pick).

**Returning student** — call `AskUserQuestion` with these two options:
1. **"Resume where I left off"** — describe the position concretely in the option's
   description, e.g. *"Topic 9 §9.3 (TEACHING)"*. Skip to Step 5 and start teaching
   from that sub-chapter.
2. **"Pick a different topic"** — Continue to Step 3.

Only when `AskUserQuestion` is unavailable (e.g. a tutor subagent without that tool)
do you fall back to asking the question as plain text with a numbered list.

## Step 3 — Pick a phase

There are **five phases**, which exceeds `AskUserQuestion`'s 4-option cap, so present
them as a **numbered list** in chat and ask the student to reply with a number. Do
not try to split the five across two chip questions — a single numbered list is
clearer for the student than a forced two-step dance.

```
1. Phase 1 — Core LLM Mechanics (Topics 1–4)
2. Phase 2 — Prompting and Context Control (Topics 5–6)
3. Phase 3 — Agents and Tool Use (Topics 7–8)
4. Phase 4 — Evaluation, Retrieval and Output (Topics 9–11)
5. Phase 5 — Production, Safety and Frontier (Topics 12–16)
```

Wait for the student's choice before continuing.

## Step 4 — Pick a starting topic

The number of topics in the chosen phase decides the tool:

- **Phases 1–4 (≤ 4 topics each):** call `AskUserQuestion`, one option per topic.
  Label each option `Topic <n> — <name>` and put the status from `PROGRESS.md`
  (`NOT STARTED` / `TEACHING` / `QUIZZING` / `EXAM READY` / `EXAM PASSED`, with the
  score if passed) in the option's description.

- **Phase 5 (5 topics, exceeds the 4-option cap):** fall back to a numbered list in
  chat, with the same per-topic status annotation. Example:

  ```
  1. Topic 12 — Production Concerns                  (NOT STARTED)
  2. Topic 13 — Safety, Guardrails & Adversarial     (NOT STARTED)
  3. Topic 14 — Model Training Landscape             (NOT STARTED)
  4. Topic 15 — Model & Ecosystem Landscape          (NOT STARTED)
  5. Topic 16 — System Design for AI                 (NOT STARTED)
  ```

Wait for the choice before continuing.

## Step 5 — Teach, continuously

From the chosen topic onward, teach **topic by topic, in syllabus order**, following
`${CLAUDE_PLUGIN_ROOT}/course/tutoring-agent-guide.md` exactly for every topic. That guide is authoritative:
follow its "Starting or resuming a session" steps and its full per-topic tutoring loop
(teach in small chunks → invite questions → check → notes → formative quiz with fresh
questions → closed-book exam from the Exam Question Bank → grade out of 100 → gate at
85 → update `~/applied-ai-course/PROGRESS.md` and `~/applied-ai-course/LEARNER-PROFILE.md`).

Then continue:

- When a topic's exam is **passed** (≥ 85), **automatically continue to the next topic**
  in syllabus order — including across phase boundaries.
- **Never advance past a topic whose exam is not yet passed.** Review the weak areas
  and retake (with different exam-bank items) until it passes.
- If a topic you reach is already `EXAM PASSED`, ask the student whether to re-teach it
  or skip to the next one.
- **Keep going until the student tells you to stop.** They may stop at any point — a
  session can end mid-topic; progress is saved under `~/applied-ai-course/`.

## Notes

- `${CLAUDE_PLUGIN_ROOT}/course/SYLLABUS.md` defines the course; `${CLAUDE_PLUGIN_ROOT}/course/tutoring-agent-guide.md`
  defines how to teach it. Follow both exactly — do not improvise curriculum.
- The per-phase tutor personas in `agents/phase-*-tutor.md` carry the same
  procedure scoped to a single phase; the guide is authoritative and phase-agnostic,
  so following the guide is sufficient when teaching across phases.
- All teaching state is written under `~/applied-ai-course/` — `PROGRESS.md`,
  `LEARNER-PROFILE.md`, `REVIEW.md`, and per-topic `notes.md` / `quiz.md` /
  `exam.md`. Never write into the plugin directory (`${CLAUDE_PLUGIN_ROOT}`) — it is
  read-only and replaced on plugin update.
- The default progression is syllabus order. If `PROGRESS.md` shows a **non-linear
  track** (topics taken out of order, some only partially taught), follow the guide's
  handling of that — match the student's actual position; do not force linear order.
