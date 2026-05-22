# Applied AI Engineering — A Tutored Course

A full, university-style course in Applied AI Engineering: the practical knowledge
required to build, evaluate, and ship production systems on top of large language
models. Designed to be taught one-to-one by an AI tutor.

## Shared course vs. personal progress
This repository separates the **course** (shareable with anyone) from a student's
**progress** (local and git-ignored):

- **Shared (committed):** the syllabus, this README, every phase/topic `README.md`,
  every topic's `material.md` (the lessons themselves), every phase's
  `EXTRA-READINGS.md`, the flat `CHECKLIST.md`, the `PROGRESS.template.md` and
  `LEARNER-PROFILE.template.md` starting templates, and the tutoring machinery under
  `.claude/` (see "Using this course with an AI tutor"). This is the course itself.
- **Local (git-ignored):** the progress tracker (`PROGRESS.md`), the learner model
  (`LEARNER-PROFILE.md`), the review list (`REVIEW.md`), and the per-topic working
  files (`notes.md`, `quiz.md`, `exam.md`) — all generated locally as you learn.

## Start here
- `SYLLABUS.md` — the locked, static course definition: methodology, assessment, and
  the full outline of all 16 topics.
- `CHECKLIST.md` — a flat one-page checklist of every concept (quick reference).
- Run the `/learn` skill to begin. On your first session the tutor creates your
  personal `PROGRESS.md` from `PROGRESS.template.md` and tracks your position from
  there on.

## Structure
Five phases, sixteen topics. Each phase is a folder with a `README.md`; each topic is a
module folder with an overview plus your local notes/quiz/exam files.

```
knowledge/ai/
├── README.md                       <- you are here
├── SYLLABUS.md                     <- course definition (shared)
├── CHECKLIST.md                    <- flat concept checklist (shared)
├── PROGRESS.template.md            <- starting template for your tracker (shared)
├── LEARNER-PROFILE.template.md     <- starting template for the learner model (shared)
├── PROGRESS.md                     <- your progress (git-ignored; created on first run)
├── LEARNER-PROFILE.md              <- the tutor's model of you (git-ignored)
├── REVIEW.md                       <- your revisit list (git-ignored)
├── phase-1-core-llm-mechanics/     <- each phase folder: README.md + EXTRA-READINGS.md
│   ├── topic-01-llm-fundamentals/  <- each topic folder: README.md + material.md
│   ├── topic-02-tokenization/         (shared), plus notes/quiz/exam.md (git-ignored)
│   ├── topic-03-inference-and-sampling/
│   └── topic-04-context-window-and-the-llm-turn/
├── phase-2-prompting-and-context-control/
│   ├── topic-05-prompt-engineering/
│   └── topic-06-prompt-caching/
├── phase-3-agents-and-tool-use/
│   ├── topic-07-tool-use-and-function-calling/
│   └── topic-08-agents-and-harnesses/
├── phase-4-evaluation-retrieval-and-output/
│   ├── topic-09-evaluations/
│   ├── topic-10-rag/
│   └── topic-11-structured-outputs/
└── phase-5-production-safety-and-frontier/
    ├── topic-12-production-concerns/
    ├── topic-13-safety-guardrails-and-adversarial/
    ├── topic-14-model-training-landscape/
    ├── topic-15-model-and-ecosystem-landscape/
    └── topic-16-system-design-for-ai/
```

## Using this course with an AI tutor
The course is taught by an AI tutor. The teaching machinery lives in `.claude/` at the
repository root and is part of the shared course — it must travel with `knowledge/ai/`:

- `.claude/tutoring-agent-guide.md` — the authoritative teaching procedure.
- `.claude/agents/phase-*-tutor.md` — five per-phase tutor agents.
- `.claude/skills/learn/` — the `/learn` skill, the student's entry point.

Start a session by running the `/learn` skill.
