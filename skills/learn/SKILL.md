---
name: learn
description: Entry point for the Applied AI Engineering tutored course in knowledge/ai/. Use when the user types /learn or asks to start, resume, or continue the course, or to be taught a course topic. Presents the phases and topics to choose from, then teaches continuously — topic after topic — until the student stops.
---

# /learn — Applied AI Engineering course

This skill starts an interactive tutoring session for the course in `knowledge/ai/`.
It runs in the main conversation: once the student picks where to begin, **you become
the tutor and teach directly here**, turn by turn.

## Step 1 — Load state

Read, in order:

1. `knowledge/ai/PROGRESS.md` — the topic tracker and the student's "Current position".
   **If this file does not exist, this is a new student:** create it by copying
   `knowledge/ai/PROGRESS.template.md`, then carry on.
2. `knowledge/ai/SYLLABUS.md` — the five phases and sixteen topics (use it for the
   exact phase/topic names in the menus below).
3. `.claude/tutoring-agent-guide.md` — the authoritative teaching procedure; its
   "Starting or resuming a session" steps cover new-student bootstrapping in full.
4. `knowledge/ai/LEARNER-PROFILE.md` — the model of this student. If this file does
   not exist, create it by copying `knowledge/ai/LEARNER-PROFILE.template.md`.

## Step 2 — Show where they are

In one or two lines, tell the student their current position from `PROGRESS.md` — e.g.
"You're currently on Topic 9 — Evaluations. Topic 7 is passed (95/100)."

A **brand-new student** (freshly created `PROGRESS.md`, nothing passed) has no prior
position — just say the course is ready and go straight to Step 3 to pick a starting
point.

If a returning student just wants to pick up where they left off, they can say so
(e.g. "resume" / "continue"): skip the menus and start teaching from the "Current
position" sub-chapter. Otherwise continue to Step 3.

## Step 3 — Pick a phase

Present the five phases as a **numbered list** and ask the student to reply with a
number. Use a numbered list, not the multiple-choice chip tool — there are five phases
and that tool caps at four options.

```
1. Phase 1 — Core LLM Mechanics (Topics 1–4)
2. Phase 2 — Prompting and Context Control (Topics 5–6)
3. Phase 3 — Agents and Tool Use (Topics 7–8)
4. Phase 4 — Evaluation, Retrieval and Output (Topics 9–11)
5. Phase 5 — Production, Safety and Frontier (Topics 12–16)
```

Wait for the student's choice before continuing.

## Step 4 — Pick a starting topic

Present the chosen phase's topics as a **numbered list**, each annotated with its
status from `PROGRESS.md` (`NOT STARTED` / `TEACHING` / `QUIZZING` / `EXAM READY` /
`EXAM PASSED`, with the score if passed). Ask which topic to start from, and wait for
the choice. Example:

```
1. Topic 9  — Evaluations          (TEACHING — current position: 9.1)
2. Topic 10 — RAG                  (NOT STARTED)
3. Topic 11 — Structured Outputs   (NOT STARTED)
```

## Step 5 — Teach, continuously

From the chosen topic onward, teach **topic by topic, in syllabus order**, following
`.claude/tutoring-agent-guide.md` exactly for every topic. That guide is authoritative:
follow its "Starting or resuming a session" steps and its full per-topic tutoring loop
(teach in small chunks → invite questions → check → notes → formative quiz with fresh
questions → closed-book exam from the Exam Question Bank → grade out of 100 → gate at
85 → update `PROGRESS.md` and `LEARNER-PROFILE.md`).

Then continue:

- When a topic's exam is **passed** (≥ 85), **automatically continue to the next topic**
  in syllabus order — including across phase boundaries.
- **Never advance past a topic whose exam is not yet passed.** Review the weak areas
  and retake (with different exam-bank items) until it passes.
- If a topic you reach is already `EXAM PASSED`, ask the student whether to re-teach it
  or skip to the next one.
- **Keep going until the student tells you to stop.** They may stop at any point — a
  session can end mid-topic; progress is saved in the git-ignored files.

## Notes

- `knowledge/ai/SYLLABUS.md` defines the course; `.claude/tutoring-agent-guide.md`
  defines how to teach it. Follow both exactly — do not improvise curriculum.
- The per-phase tutor personas in `.claude/agents/phase-*-tutor.md` carry the same
  procedure scoped to a single phase; the guide is authoritative and phase-agnostic,
  so following the guide is sufficient when teaching across phases.
- All teaching state goes only in the git-ignored files: `PROGRESS.md`,
  `LEARNER-PROFILE.md`, `REVIEW.md`, and the per-topic `notes.md` / `quiz.md` /
  `exam.md`. Never write progress into `SYLLABUS.md`, `material.md`, `README.md`, or
  the `*.template.md` files.
- The default progression is syllabus order. If `PROGRESS.md` shows a **non-linear
  track** (topics taken out of order, some only partially taught), follow the guide's
  handling of that — match the student's actual position; do not force linear order.
