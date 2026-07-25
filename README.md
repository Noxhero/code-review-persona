# code-review-persona

Claude Code skill that simulates 4 reviewer personas (security, perf,
readability, and a beginner maintaining this code in 2 years) over your
local git diff, before a real human review.

## Install

Clone or symlink this repo into `.claude/skills/code-review-persona/`
(project-level) or `~/.claude/skills/code-review-persona/` (personal).

## Use

Invoke `/code-review-persona` in Claude Code with uncommitted or
branch-ahead-of-main changes present. It writes a report to
`docs/reviews/<timestamp>-persona-review.md` and posts a condensed summary
in chat.

## Notes

- The report and all findings are written in French.
- Each run spawns 4 subagents (one per persona) in parallel — be aware of
  the added time and token cost compared to a single-pass review.
- The report file is written into `<your project>/docs/reviews/` — i.e.
  into the repo being reviewed, not into this skill's own repo.
- Prerequisites: must be run inside a git repository. Having a local
  `main` branch gives full branch-ahead-of-main coverage; without one, it
  falls back to reviewing only uncommitted changes.

See `docs/superpowers/specs/2026-07-25-code-review-persona-design.md` for
the full design.
