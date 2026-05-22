# Applied AI Engineering — A Tutored Course

A full, university-style course in Applied AI Engineering: the practical knowledge
required to build, evaluate, and ship production systems on top of large language
models. Designed to be taught one-to-one by an AI tutor.

## Bundled course vs. personal progress

This course ships as a Claude Code plugin. It cleanly separates the **course**
(read-only, bundled with the plugin) from a student's **progress** (written to your
home directory):

- **Bundled, read-only (in the plugin):** the syllabus, this README, every
  phase/topic `README.md`, every topic's `material.md`, every phase's
  `EXTRA-READINGS.md`, the flat `CHECKLIST.md`, the tutoring guide, the
  `/ai-course:learn` skill, the five phase-tutor agents, and the `templates/`
  starting files.
- **Personal, writable (in `~/applied-ai-course/`):** `PROGRESS.md`,
  `LEARNER-PROFILE.md`, `REVIEW.md`, and the per-topic `notes.md` / `quiz.md` /
  `exam.md` — all created locally as you learn, the first time `/ai-course:learn`
  runs.

## Start here

- Install the plugin (see the top-level `README.md`), then run `/ai-course:learn`.
- On the first run the tutor creates `~/applied-ai-course/` and seeds your
  `PROGRESS.md` from the bundled template.
- `SYLLABUS.md` — the static course definition: methodology, assessment, and the full
  outline of all 16 topics.
- `CHECKLIST.md` — a flat one-page checklist of every concept (quick reference).

## Structure

Five phases, sixteen topics. The bundled course content lives under `course/` in the
plugin; your progress lives under `~/applied-ai-course/`.

```
course/                              (bundled, read-only)
├── SYLLABUS.md  README.md  CHECKLIST.md  tutoring-agent-guide.md
├── templates/{PROGRESS,LEARNER-PROFILE}.template.md
└── phase-N-.../
    ├── README.md  EXTRA-READINGS.md
    └── topic-NN-.../{README.md, material.md}

~/applied-ai-course/                  (personal, writable)
├── PROGRESS.md  LEARNER-PROFILE.md  REVIEW.md
└── phase-N-.../topic-NN-.../{notes.md, quiz.md, exam.md}
```

## Using this course with an AI tutor

The course is taught by an AI tutor, all bundled in this plugin:

- `course/tutoring-agent-guide.md` — the authoritative teaching procedure.
- `agents/phase-*-tutor.md` — five per-phase tutor agents.
- `skills/learn/` — the `/ai-course:learn` skill, the student's entry point.

Start a session by running `/ai-course:learn`.
