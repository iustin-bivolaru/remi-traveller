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

Never commit or push to `main`. Every change goes on a branch (`feat/...`, `fix/...`, `chore/...`) and reaches `main` only through a PR that the user merges. Never run `git push` while `main` is checked out.

No AI attribution: commits and PRs must not include `Co-Authored-By: Claude ...` trailers or "Generated with Claude Code" lines.

## Agentic workflow

The main session (Opus) is the **planner/orchestrator**. It plans, delegates, reviews, and merges; it doesn't implement features itself.

Subagents (in `.claude/agents/`):
- **worker** (Sonnet): implements one scoped task and commits it on its own branch.
- **code-reviewer** (Opus): independent, read-only review of a diff with no planning context.

### Process for each request

1. **Branch**: create the feature branch off `main`.
2. **Plan**: split the request into tasks. **Group related tasks into a single worker** (same files, same feature, or one depends on another). Only tasks that really are independent get separate workers.
3. **Delegate**: start **at most 4 workers** in parallel, each with `isolation: "worktree"` so they never touch the same checkout. Each worker prompt has to stand on its own: the goal, the files involved, constraints, and what "done" looks like. Workers only see what's in their prompt.
4. **Review**: for each worker's branch, start a `code-reviewer` with **only** the task's goal and the diff target (e.g. `git diff <feature-branch>...<worker-branch>`). Never pass along the plan, the worker's report, or the orchestrator's opinion.
5. **Fix**: send blocker and should-fix findings back to a worker; repeat review until it's approved.
6. **Merge**: merge the worker branches into the feature branch, run `uvx ruff check . && uvx ruff format --check .`, push the feature branch (no need to ask), and report to the user. Open a PR only when asked.

Trivial changes (a typo, a one-line config tweak) can skip the workers and reviewer. Say so when skipping.

## Notes

- `.idea/` is PyCharm config and is gitignored.
- `.env` files are blocked for all tools and the bash sandbox (see `.claude/settings.json`). Never try to get around this.
