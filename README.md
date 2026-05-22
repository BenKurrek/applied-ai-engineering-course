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
