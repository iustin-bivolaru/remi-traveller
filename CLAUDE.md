# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

Early-stage Python project. `main.py` is still the PyCharm sample script. No dependencies or tests yet.

## Commands

```
python main.py              # run
uvx ruff check .            # lint
uvx ruff format .           # format
uvx ruff format --check .   # format check (what CI runs)
```

Ruff config lives in `pyproject.toml`. CI (`.github/workflows/ci.yml`) runs `ruff check` and `ruff format --check` on pushes to `main` and on every PR.

## Branches

Never commit or push to `main`. Every change goes on a branch (`feat/`, `fix/`, `chore/`, `refactor/`, `docs/`, `test/` + short kebab name) and reaches `main` only through a PR that the user merges. Never run `git push` while `main` is checked out.

No AI attribution: commits and PRs must not include `Co-Authored-By: Claude ...` trailers or "Generated with Claude Code" lines.

## Agentic workflow

The main session (Opus) is the **planner/orchestrator**. It plans, delegates, reviews, and merges; it doesn't implement features itself.

Subagents (in `.claude/agents/`):
- **worker** (Sonnet): implements one scoped task and commits it on its own branch.
- **code-reviewer** (Opus): independent, read-only review of a diff with no planning context.

### Process

**For every code change, follow the `orchestrate` skill** (`.claude/skills/orchestrate/SKILL.md`; the user can also run it as `/orchestrate <request>`). It covers the full process: branch, plan, delegate, review, fix, merge, push, report. In short:

1. Create a feature branch off `main`.
2. Split the request into tasks and group related tasks into one worker.
3. Run at most 4 workers in parallel, each with `isolation: "worktree"`.
4. Have a `code-reviewer` check each worker branch, given only the goal and the diff.
5. Send confirmed findings back to a worker, up to 2 rounds.
6. Merge the worker branches, run lint and format, push the feature branch, and report with a compare link.

Trivial changes (a typo, a one-line config tweak) can skip the workers and reviewer. Say so when skipping.

Workflow state lives in `PLAN.md` (gitignored). It's overwritten at each milestone and loaded automatically after `/clear` by a `SessionStart` hook. The format is in the skill.

## Notes

- `.idea/` is PyCharm config and is gitignored.
- `.env` files are blocked for all tools and the bash sandbox (see `.claude/settings.json`). Never try to get around this.
