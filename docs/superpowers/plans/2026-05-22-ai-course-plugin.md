# AI Course Plugin — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Repackage the Applied AI Engineering course (content + `/learn` skill + 5 phase-tutor agents + tutoring guide) as an installable Claude Code plugin, with student progress written to a fixed home directory instead of into the repo.

**Architecture:** The repo root becomes the plugin root. Bundled, read-only course content sits under `course/`; the `/learn` skill and agents are auto-discovered from `skills/` and `agents/`; manifests live in `.claude-plugin/`. All writable student state moves to `~/applied-ai-course/`, created on first run. Every path reference is rewritten to read from `${CLAUDE_PLUGIN_ROOT}` and write to `~/applied-ai-course/`.

**Tech Stack:** Claude Code plugin system (`plugin.json`, `marketplace.json`), Markdown, JSON. No build step, no test framework — verification is JSON validation, path-reference grep, and a manual install test.

**Source spec:** `docs/superpowers/specs/2026-05-22-ai-course-plugin-design.md`

---

## Resolved verification items (from spec §7)

Confirmed against Claude Code plugin docs before this plan was written:

- **`${CLAUDE_PLUGIN_ROOT}` works in agent definition files** — not just skills. No orchestrator fallback is needed; the 5 phase-tutor agents are kept and reference the plugin root directly.
- **Plugin skills are always namespaced.** A plugin named `ai-course` with a skill `learn` is invoked as **`/ai-course:learn`** — never bare `/learn`. Agents are addressed as **`ai-course:phase-1-tutor`** … `ai-course:phase-5-tutor`.
- **`.claude-plugin/`** holds only `plugin.json` and `marketplace.json`; component dirs (`skills/`, `agents/`) sit at the plugin root and are auto-discovered.

## Fixed names (change only here if desired)

- **Plugin name:** `ai-course` → command is `/ai-course:learn`. (To shorten/lengthen the command, change this single value in Task 6 and re-apply Tasks 8–13.)
- **Marketplace name:** `kurrek-courses`.
- **Home progress dir:** `~/applied-ai-course/`.
- **Personal-files destination:** `~/Documents/resume/`.

## Path Substitution Table (Table A)

Applied by Tasks 8–12. `OLD` strings are the current references; `NEW` are the replacements.

| OLD | NEW |
|-----|-----|
| `knowledge/ai/SYLLABUS.md` | `${CLAUDE_PLUGIN_ROOT}/course/SYLLABUS.md` |
| `knowledge/ai/CHECKLIST.md` | `${CLAUDE_PLUGIN_ROOT}/course/CHECKLIST.md` |
| `knowledge/ai/README.md` | `${CLAUDE_PLUGIN_ROOT}/course/README.md` |
| `.claude/tutoring-agent-guide.md` | `${CLAUDE_PLUGIN_ROOT}/course/tutoring-agent-guide.md` |
| `knowledge/ai/PROGRESS.template.md` | `${CLAUDE_PLUGIN_ROOT}/course/templates/PROGRESS.template.md` |
| `knowledge/ai/LEARNER-PROFILE.template.md` | `${CLAUDE_PLUGIN_ROOT}/course/templates/LEARNER-PROFILE.template.md` |
| `knowledge/ai/phase-` (in material/topic read paths) | `${CLAUDE_PLUGIN_ROOT}/course/phase-` |
| `knowledge/ai/PROGRESS.md` | `~/applied-ai-course/PROGRESS.md` |
| `knowledge/ai/LEARNER-PROFILE.md` | `~/applied-ai-course/LEARNER-PROFILE.md` |
| `knowledge/ai/REVIEW.md` | `~/applied-ai-course/REVIEW.md` |
| per-topic `notes.md` / `quiz.md` / `exam.md` write paths | `~/applied-ai-course/phase-*/topic-*/{notes,quiz,exam}.md` |
| `.claude/agents/` | `agents/` |
| `/learn` (the skill invocation) | `/ai-course:learn` |
| `phase-N-tutor` (agent reference) | `ai-course:phase-N-tutor` |

Rule of thumb: **reads** of bundled content → `${CLAUDE_PLUGIN_ROOT}/course/...`; **writes** of student progress → `~/applied-ai-course/...`.

---

## Sequencing constraint

The restructure (Tasks 3–5) moves and renames the `material.md` files. If anything
else is concurrently editing course content (e.g. applying the `dev/REVIEW-FIXES.md`
backlog), pause it first. Run Tasks 1–15 as one uninterrupted pass; resume any content
editing only after Task 15 passes. Content fixes themselves are out of scope here —
this plan moves content verbatim.

---

## Task 1: Move personal files out of the repo

**Files:**
- Move out: `ben-kurrek-resume-ai.tex`, `ben-kurrek-resume-crypto.tex`, `venice-ai-interview-prep.md`, `knowledge/ai/INTERVIEW-PREP.md`
- Destination: `~/Documents/resume/`

These files are currently untracked, so this is a plain filesystem move — no git operation, no commit.

- [ ] **Step 1: Create the destination directory**

Run: `mkdir -p ~/Documents/resume`

- [ ] **Step 2: Move the four personal files**

```bash
cd /Users/ben/Desktop/resume
mv ben-kurrek-resume-ai.tex ben-kurrek-resume-crypto.tex venice-ai-interview-prep.md ~/Documents/resume/
mv knowledge/ai/INTERVIEW-PREP.md ~/Documents/resume/
```

- [ ] **Step 3: Verify they are gone from the repo and present in the destination**

Run: `ls ~/Documents/resume/ && ls /Users/ben/Desktop/resume/*.tex 2>/dev/null; echo "exit=$?"`
Expected: the four files listed under `~/Documents/resume/`; the `.tex` glob prints nothing with `exit=2` (no matches).

---

## Task 2: Migrate existing progress into `~/applied-ai-course/`

**Files:**
- Copy from: `knowledge/ai/PROGRESS.md`, `knowledge/ai/LEARNER-PROFILE.md`, `knowledge/ai/REVIEW.md`, and every existing `knowledge/ai/phase-*/topic-*/{notes,quiz,exam}.md`
- Copy to: `~/applied-ai-course/` (mirrored phase/topic structure)

Preserves the author's place in the course. Home dir is outside the repo — no commit.

- [ ] **Step 1: Create the home progress directory**

Run: `mkdir -p ~/applied-ai-course`

- [ ] **Step 2: Copy the three top-level progress files**

```bash
cd /Users/ben/Desktop/resume/knowledge/ai
cp PROGRESS.md LEARNER-PROFILE.md REVIEW.md ~/applied-ai-course/
```

- [ ] **Step 3: Copy every per-topic progress file into a mirrored structure**

```bash
cd /Users/ben/Desktop/resume/knowledge/ai
for f in $(find . -type f \( -name notes.md -o -name quiz.md -o -name exam.md \)); do
  dest=~/applied-ai-course/"${f#./}"
  mkdir -p "$(dirname "$dest")"
  cp "$f" "$dest"
done
```

- [ ] **Step 4: Verify the migration**

Run: `find ~/applied-ai-course -type f | sort`
Expected: `PROGRESS.md`, `LEARNER-PROFILE.md`, `REVIEW.md` at the top level, plus the per-topic `notes/quiz/exam.md` files under `phase-*/topic-*/`. Confirm `~/applied-ai-course/PROGRESS.md` shows "Topic 9 — Evaluations" / "Topic 7 ... 95/100".

Run: `grep -c "Topic 7" ~/applied-ai-course/PROGRESS.md`
Expected: ≥ 1.

---

## Task 3: Create the plugin skeleton; move the skill, agents, and guide

**Files:**
- Create dirs: `.claude-plugin/`, `skills/`, `agents/`, `course/`, `course/templates/`, `dev/`
- Move: `.claude/skills/learn/` → `skills/learn/`; `.claude/agents/*.md` → `agents/`; `.claude/tutoring-agent-guide.md` → `course/tutoring-agent-guide.md`

All these files are untracked — plain `mv`, no `git mv`.

- [ ] **Step 1: Create the plugin directory skeleton**

```bash
cd /Users/ben/Desktop/resume
mkdir -p .claude-plugin skills agents course/templates dev
```

- [ ] **Step 2: Move the `/learn` skill**

Run: `mv /Users/ben/Desktop/resume/.claude/skills/learn /Users/ben/Desktop/resume/skills/learn`

- [ ] **Step 3: Move the five phase-tutor agents**

Run: `mv /Users/ben/Desktop/resume/.claude/agents/phase-1-tutor.md /Users/ben/Desktop/resume/.claude/agents/phase-2-tutor.md /Users/ben/Desktop/resume/.claude/agents/phase-3-tutor.md /Users/ben/Desktop/resume/.claude/agents/phase-4-tutor.md /Users/ben/Desktop/resume/.claude/agents/phase-5-tutor.md /Users/ben/Desktop/resume/agents/`

- [ ] **Step 4: Move the tutoring guide into the course bundle**

Run: `mv /Users/ben/Desktop/resume/.claude/tutoring-agent-guide.md /Users/ben/Desktop/resume/course/tutoring-agent-guide.md`

- [ ] **Step 5: Verify the skeleton**

Run: `ls -1 skills/learn agents course && echo "---" && find .claude -type f 2>/dev/null | wc -l`
Expected: `skills/learn/SKILL.md`; five `phase-*-tutor.md` in `agents/`; `tutoring-agent-guide.md` in `course/`; the `.claude` file count is `0` (now empty).

---

## Task 4: Move course content into `course/`; strip writable files from the bundle

**Files:**
- Move: `knowledge/ai/SYLLABUS.md`, `README.md`, `CHECKLIST.md`, `phase-*/` → `course/`
- Move: `knowledge/ai/PROGRESS.template.md`, `LEARNER-PROFILE.template.md` → `course/templates/`
- Delete from the bundle: every `notes.md` / `quiz.md` / `exam.md` (already migrated in Task 2)

- [ ] **Step 1: Move the top-level course files**

```bash
cd /Users/ben/Desktop/resume
mv knowledge/ai/SYLLABUS.md knowledge/ai/README.md knowledge/ai/CHECKLIST.md course/
```

- [ ] **Step 2: Move the templates into `course/templates/`**

```bash
cd /Users/ben/Desktop/resume
mv knowledge/ai/PROGRESS.template.md knowledge/ai/LEARNER-PROFILE.template.md course/templates/
```

- [ ] **Step 3: Move all five phase folders**

```bash
cd /Users/ben/Desktop/resume
mv knowledge/ai/phase-1-core-llm-mechanics knowledge/ai/phase-2-prompting-and-context-control knowledge/ai/phase-3-agents-and-tool-use knowledge/ai/phase-4-evaluation-retrieval-and-output knowledge/ai/phase-5-production-safety-and-frontier course/
```

- [ ] **Step 4: Remove the writable progress files from the bundle**

The `notes/quiz/exam.md` files were migrated to `~/applied-ai-course/` in Task 2; they must not ship inside the read-only plugin.

```bash
cd /Users/ben/Desktop/resume
find course -type f \( -name notes.md -o -name quiz.md -o -name exam.md \) -delete
```

- [ ] **Step 5: Verify the bundle contains only read-only content**

Run: `find course -type f \( -name notes.md -o -name quiz.md -o -name exam.md \) | wc -l`
Expected: `0`.

Run: `ls course && find course -name material.md | wc -l`
Expected: `SYLLABUS.md README.md CHECKLIST.md tutoring-agent-guide.md templates/` and the five `phase-*` dirs; `16` material.md files.

---

## Task 5: Relocate historical artifacts to `dev/`; remove emptied directories

**Files:**
- Move: `knowledge/ai/REVIEW-FIXES.md` → `dev/REVIEW-FIXES.md`
- Move: each `course/phase-*/CORRECTIONS.md` → `dev/CORRECTIONS-phase-N.md`
- Move: each `course/phase-*/ASSESSMENT-REVIEW.md` → `dev/ASSESSMENT-REVIEW-phase-N.md`
- Delete: the emptied `knowledge/` and `.claude/` directories

`REVIEW-FIXES.md` still sits at `knowledge/ai/REVIEW-FIXES.md` — Task 4 did not move it (it moved only `SYLLABUS.md`, `README.md`, `CHECKLIST.md`, the templates, and the five `phase-*` folders). The per-phase `CORRECTIONS.md` / `ASSESSMENT-REVIEW.md` reports rode into `course/phase-*/` with their phase folders in Task 4.

- [ ] **Step 1: Move `REVIEW-FIXES.md`**

```bash
cd /Users/ben/Desktop/resume
mv knowledge/ai/REVIEW-FIXES.md dev/REVIEW-FIXES.md
```

- [ ] **Step 2: Move the per-phase work reports, suffixing with the phase number**

```bash
cd /Users/ben/Desktop/resume/course
for n in 1 2 3 4 5; do
  d=$(ls -d phase-${n}-* )
  [ -f "$d/CORRECTIONS.md" ] && mv "$d/CORRECTIONS.md" ../dev/CORRECTIONS-phase-${n}.md
  [ -f "$d/ASSESSMENT-REVIEW.md" ] && mv "$d/ASSESSMENT-REVIEW.md" ../dev/ASSESSMENT-REVIEW-phase-${n}.md
done
```

- [ ] **Step 3: Remove the emptied source directories**

`knowledge/ai/` now holds only files already migrated to `~/applied-ai-course/` in
Task 2 (`PROGRESS.md`, `LEARNER-PROFILE.md`, `REVIEW.md`) plus possible OS cruft
(`.DS_Store`); `.claude/` is empty. Confirm the migrated copies exist, confirm no
unexpected `.md` files remain, then remove both trees.

```bash
cd /Users/ben/Desktop/resume
test -f ~/applied-ai-course/PROGRESS.md && test -f ~/applied-ai-course/LEARNER-PROFILE.md && test -f ~/applied-ai-course/REVIEW.md && echo "migration confirmed"
find knowledge -name '*.md' ! -name 'PROGRESS.md' ! -name 'LEARNER-PROFILE.md' ! -name 'REVIEW.md'   # expect no output
find .claude -type f   # expect no output
rm -rf knowledge .claude
```

- [ ] **Step 4: Verify**

Run: `ls dev && echo "---" && ls knowledge .claude 2>&1`
Expected: `dev/` holds `REVIEW-FIXES.md` + five `CORRECTIONS-phase-N.md` + five `ASSESSMENT-REVIEW-phase-N.md`; `knowledge` and `.claude` report "No such file or directory".

Run: `find course -name CORRECTIONS.md -o -name ASSESSMENT-REVIEW.md -o -name REVIEW-FIXES.md | wc -l`
Expected: `0`.

---

## Task 6: Write the plugin manifest

**Files:**
- Create: `.claude-plugin/plugin.json`

- [ ] **Step 1: Write `plugin.json`**

```json
{
  "name": "ai-course",
  "displayName": "Applied AI Engineering — Tutored Course",
  "version": "1.0.0",
  "description": "A mastery-based, tutored course in Applied AI Engineering: build, evaluate, and ship production systems on top of large language models. Sixteen topics, five phases, gated exams.",
  "author": {
    "name": "Ben Kurrek"
  },
  "keywords": ["course", "education", "ai-engineering", "llm", "tutor"]
}
```

(`skills/` and `agents/` are auto-discovered at the plugin root, so they need no manifest entry.)

- [ ] **Step 2: Verify the JSON is valid**

Run: `python3 -m json.tool .claude-plugin/plugin.json > /dev/null && echo VALID`
Expected: `VALID`.

---

## Task 7: Write the marketplace manifest

**Files:**
- Create: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Write `marketplace.json`**

```json
{
  "name": "kurrek-courses",
  "owner": {
    "name": "Ben Kurrek"
  },
  "description": "Tutored, mastery-based engineering courses.",
  "plugins": [
    {
      "name": "ai-course",
      "source": "./",
      "description": "Applied AI Engineering — a tutored course on building production LLM systems.",
      "version": "1.0.0"
    }
  ]
}
```

The `"source": "./"` makes the repo root itself the plugin.

- [ ] **Step 2: Verify the JSON is valid**

Run: `python3 -m json.tool .claude-plugin/marketplace.json > /dev/null && echo VALID`
Expected: `VALID`.

- [ ] **Step 3: Commit the restructure and manifests**

```bash
cd /Users/ben/Desktop/resume
git add .claude-plugin course skills agents dev
git commit -m "Restructure course into Claude Code plugin layout

Move knowledge/ai/ -> course/, .claude/skills + agents -> plugin root,
historical work-reports -> dev/. Add plugin.json and marketplace.json.
Path references and the home-directory progress model are rewritten in
the following commits.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

Expected: commit succeeds. If `git commit` hangs or fails on GPG signing, STOP and ask the user to refresh the gpg-agent cache.

---

## Task 8: Rewrite `skills/learn/SKILL.md`

**Files:**
- Modify: `skills/learn/SKILL.md`

The skill is the entry point. Apply Table A, and rewrite Step 1 so a brand-new student's home directory is created and seeded.

- [ ] **Step 1: Apply the read-path substitutions**

In `skills/learn/SKILL.md`, replace every occurrence:
- `knowledge/ai/SYLLABUS.md` → `${CLAUDE_PLUGIN_ROOT}/course/SYLLABUS.md`
- `.claude/tutoring-agent-guide.md` → `${CLAUDE_PLUGIN_ROOT}/course/tutoring-agent-guide.md`
- `knowledge/ai/PROGRESS.template.md` → `${CLAUDE_PLUGIN_ROOT}/course/templates/PROGRESS.template.md`
- `knowledge/ai/LEARNER-PROFILE.template.md` → `${CLAUDE_PLUGIN_ROOT}/course/templates/LEARNER-PROFILE.template.md`
- `.claude/agents/phase-*-tutor.md` → `agents/phase-*-tutor.md`

- [ ] **Step 2: Apply the write-path substitutions**

Replace every occurrence:
- `knowledge/ai/PROGRESS.md` → `~/applied-ai-course/PROGRESS.md`
- `knowledge/ai/LEARNER-PROFILE.md` → `~/applied-ai-course/LEARNER-PROFILE.md`
- `knowledge/ai/REVIEW.md` → `~/applied-ai-course/REVIEW.md`
- per-topic `notes.md` / `quiz.md` / `exam.md` → `~/applied-ai-course/phase-*/topic-*/{notes,quiz,exam}.md`

- [ ] **Step 3: Replace the Step 1 "Load state" block with the plugin/home-dir version**

Replace the existing `## Step 1 — Load state` section (the "Read, in order:" list) with:

```markdown
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
```

- [ ] **Step 4: Fix the skill's self-reference to its own command name**

Search `skills/learn/SKILL.md` for any bare `/learn` references in prose and replace with `/ai-course:learn`. (The frontmatter `name: learn` stays as-is — the namespace is applied by Claude Code from the plugin name.)

- [ ] **Step 5: Update the closing "Notes" path list**

In the final `## Notes` section, the line listing git-ignored progress files no longer applies. Replace the bullet that begins "All teaching state goes only in the git-ignored files" with:

```markdown
- All teaching state is written under `~/applied-ai-course/` — `PROGRESS.md`,
  `LEARNER-PROFILE.md`, `REVIEW.md`, and per-topic `notes.md` / `quiz.md` /
  `exam.md`. Never write into the plugin directory (`${CLAUDE_PLUGIN_ROOT}`) — it is
  read-only and replaced on plugin update.
```

- [ ] **Step 6: Verify no stale paths remain in the skill**

Run: `grep -nE 'knowledge/ai|\.claude/(agents|skills|tutoring)' skills/learn/SKILL.md; echo "exit=$?"`
Expected: no matches, `exit=1`.

- [ ] **Step 7: Commit**

```bash
git add skills/learn/SKILL.md
git commit -m "Rewrite /learn skill for plugin paths and home-dir progress

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 9: Rewrite the five phase-tutor agents

**Files:**
- Modify: `agents/phase-1-tutor.md`, `agents/phase-2-tutor.md`, `agents/phase-3-tutor.md`, `agents/phase-4-tutor.md`, `agents/phase-5-tutor.md`

Each agent has the same path references. Apply the same edits to all five.

- [ ] **Step 1: Apply read-path substitutions in all five files**

In each `agents/phase-N-tutor.md`, replace:
- `knowledge/ai/SYLLABUS.md` → `${CLAUDE_PLUGIN_ROOT}/course/SYLLABUS.md`
- `.claude/tutoring-agent-guide.md` → `${CLAUDE_PLUGIN_ROOT}/course/tutoring-agent-guide.md`
- `knowledge/ai/PROGRESS.template.md` → `${CLAUDE_PLUGIN_ROOT}/course/templates/PROGRESS.template.md`
- `knowledge/ai/LEARNER-PROFILE.template.md` → `${CLAUDE_PLUGIN_ROOT}/course/templates/LEARNER-PROFILE.template.md`
- the `material.md` read path `knowledge/ai/phase-<phase>-.../` → `${CLAUDE_PLUGIN_ROOT}/course/phase-<phase>-.../` (each agent names its own phase folder)

- [ ] **Step 2: Apply write-path substitutions in all five files**

In each file, replace:
- `knowledge/ai/PROGRESS.md` → `~/applied-ai-course/PROGRESS.md`
- `knowledge/ai/LEARNER-PROFILE.md` → `~/applied-ai-course/LEARNER-PROFILE.md`
- the topic's `notes.md` read path → `~/applied-ai-course/<phase>/<topic>/notes.md`

- [ ] **Step 3: Verify no stale paths remain**

Run: `grep -nE 'knowledge/ai|\.claude/' agents/*.md; echo "exit=$?"`
Expected: no matches, `exit=1`.

- [ ] **Step 4: Commit**

```bash
git add agents/
git commit -m "Rewrite phase-tutor agents for plugin paths and home-dir progress

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 10: Rewrite `course/tutoring-agent-guide.md`

**Files:**
- Modify: `course/tutoring-agent-guide.md`

The guide is the authoritative procedure. Two sections are rewritten in full; the rest gets Table A substitutions.

- [ ] **Step 1: Replace the "Files and what they are" section**

Replace the entire `## Files and what they are` section with:

```markdown
## Files and what they are

The course splits into **read-only bundled content** (inside the installed plugin,
addressed via `${CLAUDE_PLUGIN_ROOT}`) and **writable student progress** (under
`~/applied-ai-course/`). Never write into the plugin directory — it is replaced on
plugin update.

Read-only, bundled:

- `${CLAUDE_PLUGIN_ROOT}/course/SYLLABUS.md` — the static course definition.
- `${CLAUDE_PLUGIN_ROOT}/course/phase-*/topic-*/material.md` — the authored material
  for each topic. **This is the source you teach from.** Do not improvise beyond it.
- `${CLAUDE_PLUGIN_ROOT}/course/phase-*/topic-*/README.md` — static topic overview.
- `${CLAUDE_PLUGIN_ROOT}/course/templates/PROGRESS.template.md` and
  `LEARNER-PROFILE.template.md` — starting templates. A new student has neither
  progress file; on the first session you create each by copying its template into
  `~/applied-ai-course/` (see "Starting or resuming a session"). Never teach from, or
  write progress into, the templates themselves.

Writable, per-student, under `~/applied-ai-course/`:

- `~/applied-ai-course/PROGRESS.md` — the student's progress tracker. Its status
  vocabulary is `NOT STARTED` → `TEACHING` → `QUIZZING` → `EXAM READY` →
  `EXAM PASSED`.
- `~/applied-ai-course/REVIEW.md` — a running list of cross-cutting items the student
  has flagged to revisit. Append an entry whenever the student asks to note something.
- `~/applied-ai-course/LEARNER-PROFILE.md` — a living model of *this* student: their
  strengths, recurring mistakes, the misconceptions they have actually fallen for,
  their performance by question type, and how they learn best. You read it at the
  start of every session and update it continuously as you teach. It is distinct from
  `REVIEW.md`: `REVIEW.md` holds *content to revisit*; `LEARNER-PROFILE.md` holds
  *what you have learned about the student*.
- `~/applied-ai-course/phase-*/topic-*/notes.md`, `quiz.md`, `exam.md` — the
  student's working files, created when first needed.
```

- [ ] **Step 2: Replace the "Specialized phase tutors" section**

Replace the entire `## Specialized phase tutors` section with:

```markdown
## Specialized phase tutors

The course is taught by five specialized agents, one per phase, bundled in the
plugin's `agents/` directory and addressed as subagent types:

- `ai-course:phase-1-tutor` — Phase 1, Core LLM Mechanics (Topics 1–4)
- `ai-course:phase-2-tutor` — Phase 2, Prompting and Context Control (Topics 5–6)
- `ai-course:phase-3-tutor` — Phase 3, Agents and Tool Use (Topics 7–8)
- `ai-course:phase-4-tutor` — Phase 4, Evaluation, Retrieval and Output Control (Topics 9–11)
- `ai-course:phase-5-tutor` — Phase 5, Production, Safety and Frontier (Topics 12–16)

Use the tutor whose phase contains the student's current topic (per
`~/applied-ai-course/PROGRESS.md`).
```

- [ ] **Step 3: Apply Table A substitutions across the rest of the file**

In the remaining sections ("Starting or resuming a session", "The tutoring loop",
"Principles"), replace every occurrence:
- `knowledge/ai/PROGRESS.template.md` → `${CLAUDE_PLUGIN_ROOT}/course/templates/PROGRESS.template.md`
- `knowledge/ai/LEARNER-PROFILE.template.md` → `${CLAUDE_PLUGIN_ROOT}/course/templates/LEARNER-PROFILE.template.md`
- `PROGRESS.md` (bare references) → `~/applied-ai-course/PROGRESS.md`
- `LEARNER-PROFILE.md` (bare references) → `~/applied-ai-course/LEARNER-PROFILE.md`
- `REVIEW.md` (bare references) → `~/applied-ai-course/REVIEW.md`
- `SYLLABUS.md` (bare references) → `${CLAUDE_PLUGIN_ROOT}/course/SYLLABUS.md`
- `material.md` (bare references) → keep as `material.md` where generic, or
  `${CLAUDE_PLUGIN_ROOT}/course/phase-*/topic-*/material.md` where a path is meant
- `notes.md` / `quiz.md` / `exam.md` (bare references) → leave the bare filename
  where generic; the writable location is `~/applied-ai-course/phase-*/topic-*/`

- [ ] **Step 4: Update the "Keep ... untouched" principle**

In the Principles section, the bullet beginning "Keep `SYLLABUS.md`, every
`material.md`, and every `README.md` untouched while tutoring" — replace it with:

```markdown
- The bundled course content under `${CLAUDE_PLUGIN_ROOT}` is read-only — never
  write to it. All mutable state goes only under `~/applied-ai-course/`.
```

- [ ] **Step 5: Verify no stale paths remain**

Run: `grep -nE 'knowledge/ai|\.claude/' course/tutoring-agent-guide.md; echo "exit=$?"`
Expected: no matches, `exit=1`.

- [ ] **Step 6: Commit**

```bash
git add course/tutoring-agent-guide.md
git commit -m "Rewrite tutoring guide for plugin paths and home-dir progress

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 11: Rewrite the template headers

**Files:**
- Modify: `course/templates/PROGRESS.template.md`
- Modify: `course/templates/LEARNER-PROFILE.template.md`

The templates' header text describes being copied into a git-ignored sibling. Update it to describe the home directory.

- [ ] **Step 1: Update `PROGRESS.template.md` header**

Replace the top block (from the title down to the "Status values:" line, exclusive) — currently:

```markdown
# Course Progress — Template

Starting template for a student's progress tracker. **This file is shared/committed.**

On a student's first `/learn` session the tutor copies this file to `PROGRESS.md`
(which is git-ignored and personal) and fills it in from there. Do not teach from,
or write progress into, this template — only into the `PROGRESS.md` copy.
```

with:

```markdown
# Course Progress — Template

Starting template for a student's progress tracker. This file is bundled, read-only,
inside the plugin.

On a student's first `/ai-course:learn` session the tutor copies this file to
`~/applied-ai-course/PROGRESS.md` and fills it in from there. Do not teach from, or
write progress into, this template — only into the `~/applied-ai-course/PROGRESS.md`
copy.
```

- [ ] **Step 2: Update `LEARNER-PROFILE.template.md` header**

Replace the equivalent top block — currently:

```markdown
# Learner Profile — Template

Starting template for the tutor's living model of a student. **This file is
shared/committed.**

On a student's first `/learn` session the tutor copies this file to
`LEARNER-PROFILE.md` (which is git-ignored and personal) and populates it from there.
Do not write observations into this template — only into the `LEARNER-PROFILE.md` copy.
```

with:

```markdown
# Learner Profile — Template

Starting template for the tutor's living model of a student. This file is bundled,
read-only, inside the plugin.

On a student's first `/ai-course:learn` session the tutor copies this file to
`~/applied-ai-course/LEARNER-PROFILE.md` and populates it from there. Do not write
observations into this template — only into the `~/applied-ai-course/LEARNER-PROFILE.md`
copy.
```

- [ ] **Step 3: Apply remaining substitutions**

In both template files, replace any other `.claude/tutoring-agent-guide.md` reference
with `${CLAUDE_PLUGIN_ROOT}/course/tutoring-agent-guide.md`.

- [ ] **Step 4: Verify**

Run: `grep -nE 'git-ignored|knowledge/ai|/learn[^.]' course/templates/*.md; echo "exit=$?"`
Expected: no matches, `exit=1`.

- [ ] **Step 5: Commit**

```bash
git add course/templates/
git commit -m "Update progress templates for plugin/home-dir model

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 12: Rewrite `course/README.md`

**Files:**
- Modify: `course/README.md`

The course README's "Shared course vs. personal progress", "Start here", "Structure",
and "Using this course with an AI tutor" sections all assume the git-ignore model and
the old paths. Rewrite them for the plugin model.

- [ ] **Step 1: Replace the "Shared course vs. personal progress" section**

Replace that entire section with:

```markdown
## Bundled course vs. personal progress

This course ships as a Claude Code plugin. It cleanly separates the **course**
(read-only, bundled with the plugin) from a student's **progress** (written to your
home directory):

- **Bundled, read-only (in the plugin):** the syllabus, this README, every
  phase/topic `README.md`, every topic's `material.md`, every phase's
  `EXTRA-READINGS.md`, the flat `CHECKLIST.md`, the tutoring guide, the `/ai-course:learn`
  skill, the five phase-tutor agents, and the `templates/` starting files.
- **Personal, writable (in `~/applied-ai-course/`):** `PROGRESS.md`,
  `LEARNER-PROFILE.md`, `REVIEW.md`, and the per-topic `notes.md` / `quiz.md` /
  `exam.md` — all created locally as you learn, the first time `/ai-course:learn` runs.
```

- [ ] **Step 2: Replace the "Start here" section**

```markdown
## Start here

- Install the plugin (see the top-level `README.md`), then run `/ai-course:learn`.
- On the first run the tutor creates `~/applied-ai-course/` and seeds your
  `PROGRESS.md` from the bundled template.
- `SYLLABUS.md` — the static course definition: methodology, assessment, and the full
  outline of all 16 topics.
- `CHECKLIST.md` — a flat one-page checklist of every concept (quick reference).
```

- [ ] **Step 3: Replace the "Structure" section**

Replace the directory-tree code block and its intro with the following. The outer
fence below uses **four** backticks so the inner three-backtick fence is literal
content — write the inner fence into `course/README.md` as a normal triple-backtick
code block.

````markdown
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
````

- [ ] **Step 4: Replace the "Using this course with an AI tutor" section**

```markdown
## Using this course with an AI tutor

The course is taught by an AI tutor, all bundled in this plugin:

- `course/tutoring-agent-guide.md` — the authoritative teaching procedure.
- `agents/phase-*-tutor.md` — five per-phase tutor agents.
- `skills/learn/` — the `/ai-course:learn` skill, the student's entry point.

Start a session by running `/ai-course:learn`.
```

- [ ] **Step 5: Verify no stale references remain**

Run: `grep -nE 'knowledge/ai|git-ignored|\.claude/' course/README.md; echo "exit=$?"`
Expected: no matches, `exit=1`.

- [ ] **Step 6: Commit**

```bash
git add course/README.md
git commit -m "Rewrite course README for the plugin model

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 13: Write the top-level plugin README

**Files:**
- Create: `README.md` (repo root)

- [ ] **Step 1: Write `README.md`**

The outer fence below uses **four** backticks so the inner three-backtick fences are
literal content — write each inner fence into `README.md` as a normal triple-backtick
code block.

````markdown
# Applied AI Engineering — Tutored Course (Claude Code plugin)

A mastery-based, tutored course in Applied AI Engineering: the practical knowledge to
build, evaluate, and ship production systems on top of large language models. Sixteen
topics across five phases, each gated by a closed-book exam (85 to pass).

The course is delivered as a Claude Code plugin. An AI tutor teaches interactively —
explain, check, quiz, exam — and tracks your progress.

## Install

```
/plugin marketplace add <path-or-git-url-to-this-repo>
/plugin install ai-course@kurrek-courses
/reload-plugins
```

Or, for a single session without installing:

```
claude --plugin-dir <path-to-this-repo>
```

## Use

Run `/ai-course:learn` and pick where to start. On the first run the tutor creates
`~/applied-ai-course/` and seeds your progress files; after that it resumes wherever
you left off, from anywhere.

- **Course content** (read-only) is bundled under `course/`.
- **Your progress** (writable) lives in `~/applied-ai-course/` — outside the plugin,
  so a plugin update never touches it.

See `course/README.md` for the course structure and `course/SYLLABUS.md` for the full
outline.

## Layout

- `.claude-plugin/` — plugin and marketplace manifests
- `skills/learn/` — the `/ai-course:learn` entry-point skill
- `agents/` — five per-phase tutor agents
- `course/` — bundled, read-only course content
- `dev/` — internal authoring notes (not part of the taught course)
````

- [ ] **Step 2: Verify the JSON-block-free README reads correctly**

Run: `head -5 README.md`
Expected: the title line `# Applied AI Engineering — Tutored Course (Claude Code plugin)`.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "Add top-level plugin README with install instructions

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 14: Rewrite `.gitignore`

**Files:**
- Modify: `.gitignore`

The old course/progress ignore rules are obsolete (progress now lives outside the
repo). Replace the file's contents with rules for the plugin repo.

- [ ] **Step 1: Replace `.gitignore` with**

```gitignore
# Internal authoring artifacts — not part of the published course
dev/

# OS / editor cruft
.DS_Store
```

Rationale: `dev/` holds the historical work-reports (`REVIEW-FIXES.md`,
`CORRECTIONS-*`, `ASSESSMENT-REVIEW-*`) — kept locally, never published. Student
progress is in `~/applied-ai-course/`, outside the repo, so it needs no rule.

- [ ] **Step 2: Verify `dev/` is ignored and `course/` is not**

Run: `git check-ignore dev/REVIEW-FIXES.md && ! git check-ignore course/SYLLABUS.md && echo OK`
Expected: `OK`.

- [ ] **Step 3: Commit**

```bash
git add .gitignore
git commit -m "Replace .gitignore with plugin-repo rules

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 15: Full verification

**Files:** none modified — this task only inspects.

- [ ] **Step 1: Confirm no stale path references anywhere in the plugin**

Run: `grep -rnE 'knowledge/ai|\.claude/(agents|skills|tutoring-agent-guide)' skills agents course .claude-plugin README.md; echo "exit=$?"`
Expected: no matches, `exit=1`.

- [ ] **Step 2: Confirm both manifests are valid JSON**

Run: `for f in .claude-plugin/plugin.json .claude-plugin/marketplace.json; do python3 -m json.tool "$f" > /dev/null && echo "$f VALID"; done`
Expected: both lines print `VALID`.

- [ ] **Step 3: Confirm the bundle has no writable files and the structure is right**

Run: `find course -type f \( -name notes.md -o -name quiz.md -o -name exam.md -o -name CORRECTIONS.md -o -name ASSESSMENT-REVIEW.md \) | wc -l`
Expected: `0`.

Run: `find course -name material.md | wc -l && ls skills/learn/SKILL.md && ls agents/*.md | wc -l`
Expected: `16`, the `SKILL.md` path, and `5`.

- [ ] **Step 4: Confirm `dev/` content is excluded from git but present on disk**

Run: `git status --porcelain dev/ | wc -l && ls dev/ | wc -l`
Expected: `0` (git ignores it) and `11` (one `REVIEW-FIXES.md` + five `CORRECTIONS-phase-N.md` + five `ASSESSMENT-REVIEW-phase-N.md`).

- [ ] **Step 5: Manual install test — run by the user in Claude Code**

This step cannot be run by an agent; it uses interactive slash commands. Ask the user
to run, in a Claude Code session:

​```
/plugin marketplace add /Users/ben/Desktop/resume
/plugin install ai-course@kurrek-courses
/reload-plugins
/ai-course:learn
​```

Expected: `/ai-course:learn` starts; on a machine with no `~/applied-ai-course/`, it
creates the directory and seeds `PROGRESS.md` + `LEARNER-PROFILE.md` from the
templates, then shows the phase menu. On this machine (progress already migrated in
Task 2), it resumes at "Topic 9 — Evaluations".

Confirm nothing was written inside the plugin directory:
Run (after the test): `git status --porcelain course skills agents`
Expected: no output — the plugin tree is unchanged by teaching.

- [ ] **Step 6: Final confirmation**

Report to the user: the plugin installs, `/ai-course:learn` runs, progress reads and
writes to `~/applied-ai-course/`, and the plugin directory stays read-only. The
branch `ai-course-plugin` holds the full conversion.
```
