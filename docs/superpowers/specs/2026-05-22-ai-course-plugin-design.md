# Design Spec — Package the Applied AI Engineering course as a Claude Code plugin

**Date:** 2026-05-22
**Status:** Approved design — ready for implementation planning
**Topic:** Convert the tutored course in `knowledge/ai/` (+ the `.claude/` teaching
machinery) into an installable Claude Code plugin.

---

## 1. Goal

The Applied AI Engineering course currently lives as loose content in a personal
repo: course material under `knowledge/ai/`, and the teaching machinery (the `/learn`
skill, five phase-tutor agents, the tutoring guide) under `.claude/`. The two halves
must travel together and have hardcoded relative paths.

Package the whole thing as a single Claude Code **plugin** so it is cleanly bundled,
versioned, and installable — via a marketplace — by the author and anyone else.

## 2. Locked decisions

These were settled during brainstorming and are not open for re-litigation in the
implementation plan:

1. **Writable progress location:** a fixed home-directory path,
   `~/applied-ai-course/` (visible, not hidden). One canonical store so `/learn`
   resumes from anywhere.
2. **Plugin source:** restructure in place — the current repo root becomes the
   plugin root. No separate repo.
3. **Distribution:** marketplace + installable — the repo carries both a plugin
   manifest and a single-plugin marketplace manifest.
4. **Personal files:** the author's personal files (`ben-kurrek-resume-ai.tex`,
   `ben-kurrek-resume-crypto.tex`, `venice-ai-interview-prep.md`, and the
   git-ignored `knowledge/ai/INTERVIEW-PREP.md`) move *out* of the repo to a personal
   folder so the published plugin contains none of them.

## 3. Core principle: read-only plugin, writable home dir

A plugin installs into a versioned, managed cache
(`~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/...`) that is **replaced on
every update**. Nothing writable may live inside it.

Therefore the course splits cleanly in two:

- **Bundled, read-only (ships in the plugin):** syllabus, all READMEs, all
  `material.md`, `CHECKLIST.md`, `EXTRA-READINGS.md`, the tutoring guide, the `/learn`
  skill, the five phase-tutor agents, and the two starting templates.
- **Writable, per-student (created in `~/applied-ai-course/`):** `PROGRESS.md`,
  `LEARNER-PROFILE.md`, `REVIEW.md`, and the per-topic `notes.md` / `quiz.md` /
  `exam.md`.

The previous "course vs. progress" separation enforced by `.gitignore` is **replaced**
by this physical boundary (plugin directory vs. home directory).

## 4. Target architecture

### 4.1 Plugin layout (repo root = plugin root)

```
<plugin-root>/
├── .claude-plugin/
│   ├── plugin.json                 plugin manifest
│   └── marketplace.json            single-plugin marketplace, lists this plugin
├── skills/
│   └── learn/SKILL.md              moved from .claude/skills/learn/
├── agents/
│   ├── phase-1-tutor.md            moved from .claude/agents/
│   ├── phase-2-tutor.md
│   ├── phase-3-tutor.md
│   ├── phase-4-tutor.md
│   └── phase-5-tutor.md
├── course/                         moved from knowledge/ai/  (read-only content)
│   ├── SYLLABUS.md
│   ├── README.md                   the course overview (rewritten for plugin model)
│   ├── CHECKLIST.md
│   ├── tutoring-agent-guide.md      moved from .claude/tutoring-agent-guide.md
│   ├── templates/
│   │   ├── PROGRESS.template.md
│   │   └── LEARNER-PROFILE.template.md
│   └── phase-N-.../
│       ├── README.md
│       ├── EXTRA-READINGS.md
│       └── topic-NN-.../
│           ├── README.md
│           └── material.md
├── README.md                       NEW — plugin install + usage doc (repo root)
├── dev/                             git-ignored — historical work artifacts
│   ├── CORRECTIONS-phase-N.md       (relocated from each phase folder)
│   ├── ASSESSMENT-REVIEW-phase-N.md
│   └── REVIEW-FIXES.md
└── .gitignore
```

Notes:
- Per-topic folders under `course/` hold **only** `README.md` + `material.md`. The
  writable `notes/quiz/exam.md` no longer live here.
- `course/` wraps all bundled content so it is cleanly separated from the plugin
  machinery (`skills/`, `agents/`, `.claude-plugin/`).
- The historical work-reports (`CORRECTIONS.md`, `ASSESSMENT-REVIEW.md`,
  `REVIEW-FIXES.md`) move into a git-ignored `dev/` folder — neither published nor
  intermixed with `course/`. They are flattened with a phase suffix since they all
  collapse into one folder.

### 4.2 Home progress store

All writable student state, created on demand:

```
~/applied-ai-course/
├── PROGRESS.md
├── LEARNER-PROFILE.md
├── REVIEW.md
└── phase-N-.../
    └── topic-NN-.../
        ├── notes.md
        ├── quiz.md
        └── exam.md
```

First-run behavior: on the first `/learn`, the skill creates `~/applied-ai-course/`,
then creates `PROGRESS.md` and `LEARNER-PROFILE.md` by copying them from
`course/templates/`. Per-topic files are created lazily when first needed (unchanged
from the current guide). The phase/topic directory names mirror those under
`course/` exactly.

### 4.3 Path-resolution model

Every hardcoded `knowledge/ai/...` and `.claude/...` reference is replaced by one of
two bases:

- **Reads** — course content, the guide, the templates →
  `${CLAUDE_PLUGIN_ROOT}/course/...`
- **Writes** — all student progress → `~/applied-ai-course/...` (implementations
  expand `$HOME`).

`${CLAUDE_PLUGIN_ROOT}` is the environment variable Claude Code sets to the installed
plugin's directory.

## 5. Files that change

| File | Change |
|------|--------|
| `.claude-plugin/plugin.json` | NEW — manifest: name, version, description, author. |
| `.claude-plugin/marketplace.json` | NEW — single-plugin marketplace listing this plugin. |
| `README.md` (repo root) | NEW — what the plugin is, how to add the marketplace and install it, how to start with `/learn`. |
| `skills/learn/SKILL.md` | Moved; every path rewritten to the read/write model; first-run bootstrap creates `~/applied-ai-course/` and copies the templates. |
| `agents/phase-1..5-tutor.md` | Moved; every path rewritten to the read/write model. |
| `course/tutoring-agent-guide.md` | Moved; largest rewrite — the "Files and what they are" section and every step reframed around plugin-reads / home-dir-writes; "Specialized phase tutors" now points at `agents/`. |
| `course/templates/PROGRESS.template.md` | Moved; header text updated to describe being copied into `~/applied-ai-course/`. |
| `course/templates/LEARNER-PROFILE.template.md` | Moved; same header update. |
| `course/README.md` | Moved; "Shared/Local", "Start here", "Structure", "Using this course" sections rewritten for the plugin model (the git-ignore split is gone). |
| `course/**` (syllabus, material, etc.) | Moved from `knowledge/ai/`; content unchanged. |
| `.gitignore` | Rewritten — drops the obsolete course/progress rules; ignores `dev/` and any remaining local cruft. |

## 6. Migration steps (one-time, mechanical)

1. **Move personal files out of the repo** to `~/Documents/resume/` (or another
   path the author picks): `ben-kurrek-resume-ai.tex`,
   `ben-kurrek-resume-crypto.tex`, `venice-ai-interview-prep.md`, and
   `knowledge/ai/INTERVIEW-PREP.md`.
2. **Migrate existing progress** into `~/applied-ai-course/`: the author's current
   `knowledge/ai/PROGRESS.md`, `LEARNER-PROFILE.md`, `REVIEW.md`, and every existing
   per-topic `notes.md` / `quiz.md` / `exam.md`, into the mirrored phase/topic
   structure — so the author's place in the course (Topic 9 in progress, Topic 7
   passed 95/100) is preserved.
3. **Relocate historical artifacts** (`CORRECTIONS.md`, `ASSESSMENT-REVIEW.md`,
   `REVIEW-FIXES.md`) into the git-ignored `dev/` folder.
4. **Restructure** the remaining content into the §4.1 layout, then remove the
   emptied `knowledge/` and `.claude/` directories (preserving any unrelated Claude
   Code project settings, if `.claude/` contains them).
5. **Rewrite paths** in all files per §5.
6. **Author the new manifests and READMEs** (`plugin.json`, `marketplace.json`,
   root `README.md`).
7. **Optional:** rename the working directory (`~/Desktop/resume`) to something
   accurate, e.g. `applied-ai-engineering-plugin`.

## 7. Sequencing and risks

- **Verification item — `${CLAUDE_PLUGIN_ROOT}` in agents.** The variable is
  documented for skills. The implementation plan must first confirm whether it also
  resolves inside **agent** definition files. If it does not, the fallback is to make
  the `/learn` skill the sole orchestrator (it already teaches in the main
  conversation) and treat the five phase-tutor agents as optional, skill-resolved
  helpers. This must be resolved before the agent files are rewritten.
- **Plugin command namespacing.** As a plugin skill, `/learn` may be invoked as a
  namespaced command (e.g. `/applied-ai-engineering:learn`). The implementation plan
  must confirm the final invocation name and make every doc reference match it.
- **Concurrent content edits.** Something is concurrently editing the `material.md`
  files (the `REVIEW-FIXES.md` backlog is being applied). The restructure moves and
  renames those files. The restructure must run as one discrete pass with no
  concurrent content editing; content fixes resume afterward.
- **Content backlog is out of scope.** The `REVIEW-FIXES.md` P0 accuracy bugs are
  *not* addressed by this work. This spec covers packaging only; the course content
  is moved verbatim.

## 8. Out of scope

- Fixing the `REVIEW-FIXES.md` P0/P1/P2/P3 curriculum backlog.
- Any change to course *content* (`material.md`, exams, quizzes).
- Hands-on labs (a separate `REVIEW-FIXES.md` P3 item).
- Publishing the marketplace to a public/remote location — the marketplace manifest
  is created, but where it is hosted is a later decision.

## 9. Success criteria

- The repo is a valid Claude Code plugin: `plugin.json` + `marketplace.json` present
  and well-formed; `skills/` and `agents/` discovered.
- Adding the marketplace and installing the plugin makes `/learn` available.
- A first `/learn` run on a clean machine creates `~/applied-ai-course/`, seeds
  `PROGRESS.md` + `LEARNER-PROFILE.md` from the templates, and begins teaching from
  bundled `course/` content — with no writes into the plugin directory.
- A returning `/learn` run reads progress from `~/applied-ai-course/` and resumes
  correctly.
- The published plugin contains no personal files.
- No file in the plugin references an obsolete `knowledge/ai/...` or `.claude/...`
  path.
