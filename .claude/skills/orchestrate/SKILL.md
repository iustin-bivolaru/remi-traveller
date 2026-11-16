---
name: orchestrate
description: Run this repo's agentic workflow for a change request. The main session plans and splits the work, Sonnet `worker` subagents implement it in isolated worktrees, an Opus `code-reviewer` with no context reviews each diff, and the orchestrator merges, verifies, and pushes the feature branch. Use for every feature, fix, or refactor in this repo, i.e. any request that changes code, unless it's trivial.
argument-hint: <what you want built or changed>
---

# Orchestrate

You are the **orchestrator** (Opus). You plan, delegate, review, and merge. You do **not** write feature code yourself. Workers do.

Request: $ARGUMENTS

If the request is empty or too vague to split into tasks, ask the user one focused question before doing anything else.

## Hard rules (never break these)

- **Never commit or push to `main`.** Never run `git push` while `main` is checked out. `main` changes only through a PR that the user merges.
- **Push feature branches without asking.** Opening a PR or force-pushing still needs the user's explicit request.
- **No AI attribution.** No `Co-Authored-By: Claude ...` trailers and no "Generated with Claude Code" lines in commits or PRs.
- **Never touch `.env` files**, and never try to get around a sandbox or permission denial. Report it to the user instead.
- **At most 4 workers at once.**

## Step 0: Is this trivial?

If the change is a typo, a one-line config tweak, or a docs fix that needs no judgment, skip the workflow: do it yourself on a feature branch, run lint and format, commit, and push. **Tell the user you skipped the workflow and why.** Otherwise continue.

## Step 1: Branch

```
git switch main && git pull --ff-only origin main
git switch -c <type>/<short-kebab-name>     # type: feat | fix | chore | refactor | docs | test
```

If you're already on a feature branch that this request extends, stay on it. The working tree must be clean before you spawn workers, because worktrees branch from the current commit. Commit anything pending first.

## Step 2: Plan

Read enough of the code to understand the request. Then split it into **tasks** and group them into **worker assignments**:

- **Group related tasks into one worker** when they touch the same files, belong to the same feature, or one depends on another's output. Two workers editing the same file means merge conflicts, so avoid it.
- Give separate workers only to tasks that really are independent.
- Prefer fewer, larger assignments over many small ones. One worker is often the right answer.
- If an assignment depends on another's result, run it **after** that one is merged, not in parallel.

Show the user the plan briefly (assignments, what each one covers, and what runs in parallel versus in sequence), then go ahead without waiting for approval unless something really needs their decision.

## Step 3: Delegate

Start all independent workers **in a single message** so they run in parallel. Each call:

- `subagent_type: "worker"`
- `isolation: "worktree"`
- `description`: 3-5 words

Workers see **only** their prompt. Write each prompt so it stands on its own, using this template:

```
## Goal
<what to build or change, and why, in 1-3 sentences>

## Scope
Files to create/modify: <paths>
Do NOT touch: <paths or areas owned by other workers, if any>

## Requirements
- <concrete, checkable requirement>
- ...

## Context
<relevant existing code, conventions, signatures, or decisions the worker needs. Paste the key snippets rather than saying "see X">

## Done when
- <acceptance criteria>
- `uvx ruff check .` and `uvx ruff format --check .` pass
- Tests pass (if the project has tests), with new tests for new behavior
- Work is committed on your branch (no AI attribution in the message)
```

Wait for the workers to finish. Don't do their work in the meantime. Note each worker's **branch name and commit** from its report or the tool result.

If a worker reports it was blocked or that the task was ambiguous, decide yourself if you can, or ask the user, then re-delegate.

## Step 4: Review (no context)

For **each** worker branch, start a `code-reviewer`. Start them all in parallel, in a single message. The reviewer must get **only** the goal and the diff target, nothing else:

```
Review this change.

Goal: <the worker's Goal section, copied as-is>

Diff: git diff <feature-branch>...<worker-branch>
```

**Never** include the plan, the worker's report, your opinion of the code, or hints about what to look for. The reviewer is useful because it has no context.

## Step 5: Fix loop

- **Blockers and should-fix findings**: first check them yourself. Reviewers can be wrong. Send the confirmed ones back to a worker. Use SendMessage to the same worker if it's still reachable; otherwise start a new `worker` with a standalone prompt that includes the findings and the branch to continue on. Then review again with a **new** `code-reviewer` (again, only goal and diff).
- **Nits**: apply them only if they're cheap and clearly right. Otherwise list them in your report.
- At most **2 fix rounds** per assignment. If it still isn't approved, stop and ask the user, showing the outstanding findings.

## Step 6: Merge, verify, push

On the feature branch:

```
git merge --no-ff <worker-branch> -m "<summary of the assignment>"   # once per approved worker branch
uvx ruff check .
uvx ruff format --check .
<run tests if any>
git push -u origin <feature-branch>
```

- Merge conflict: resolve it yourself if it's mechanical. If it involves real logic, send it to a worker.
- If lint, format, or tests fail after the merge, delegate the fix to a worker. Don't patch feature code yourself.
- If `git push -u` can't set the upstream (the sandbox blocks writes to `.git/config`), the push itself still succeeded. Ignore the warning.

## Step 7: Report

Tell the user, concisely:

1. **What was done**: per assignment, one or two lines.
2. **Review outcome**: verdicts, what was fixed, and any nits left open.
3. **Verification**: lint, format, and test results after the merge.
4. **Branch**: name and head commit, plus that it's pushed.
5. **PR link**: `gh` usually can't reach GitHub from the sandbox, so give the compare URL to open the PR:
   `https://github.com/<owner>/<repo>/compare/main...<feature-branch>` (owner/repo come from `git remote get-url origin`).
   Open the PR yourself only if the user asks and `gh` works.
6. **Leftovers**: worker worktrees (paths come from the Agent tool results) that weren't cleaned up automatically. Give the user the `git worktree remove <path>` commands, since the sandbox may block running them.
