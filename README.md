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

See `docs/superpowers/specs/2026-07-25-code-review-persona-design.md` for
the full design.
