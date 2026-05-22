# Applied AI Engineering — Tutored Course (Claude Code plugin)

A mastery-based, tutored course in Applied AI Engineering: the practical knowledge to
build, evaluate, and ship production systems on top of large language models. Sixteen
topics across five phases, each gated by a closed-book exam (85 to pass).

The course is delivered as a Claude Code plugin. An AI tutor teaches interactively —
explain, check, quiz, exam — and tracks your progress.

## Install

This course is a Claude Code plugin, distributed through this repository. Install it
from inside Claude Code with two commands:

```
/plugin marketplace add BenKurrek/applied-ai-engineering-course
/plugin install ai-course@kurrek-courses
```

The first command registers this repo as a plugin **marketplace**; the second installs
the **`ai-course`** plugin from it. Registering the marketplace is a one-time step —
after that, updates and re-installs are just `/plugin install`. If `/ai-course:learn`
is not available immediately after installing, run `/reload-plugins` or restart
Claude Code.

> Plugins run in **Claude Code** (the CLI, desktop app, and IDE extensions) — not in
> the claude.ai web chat.

### One-command install (optional)

To skip the `marketplace add` step, pre-register the marketplace in a project's
`.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "kurrek-courses": {
      "source": { "source": "github", "repo": "BenKurrek/applied-ai-engineering-course" }
    }
  }
}
```

Installation is then a single `/plugin install ai-course@kurrek-courses`. Add
`"enabledPlugins": { "ai-course@kurrek-courses": true }` alongside it and the plugin
loads automatically with no commands at all — handy for a classroom or team.

### Run without installing

For a one-off session, point Claude Code at a local clone:

```
claude --plugin-dir <path-to-your-clone>
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
