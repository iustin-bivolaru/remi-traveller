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
- **At most 4 workers at once.** If there are more independent assignments, run the rest in later batches.
- **Network:** git commands that reach GitHub (`pull`, `fetch`, `push`, `ls-remote`) must declare `github.com` in the Bash call's `allowed_domains`. If one is still refused, don't look for a way around it: tell the user and give them the exact command to run.

## Plan file (`PLAN.md`)

`PLAN.md` in the repo root holds the workflow's state, so the user can `/clear` at any time without losing track. It's gitignored and never committed. Workers don't use it. A `SessionStart` hook loads it into context automatically after `/clear` or a compaction.

**On start:** if `PLAN.md` exists, read it.
- Its branch matches the current branch and its status isn't `done`: this is a continuation. Resume from **Next** instead of re-planning.
- Its branch differs, or its status is `done`: it's a finished or other plan. If it isn't `done`, ask the user whether to abandon it. Then overwrite it with the new plan.

**When to write:** overwrite the whole file (never append) at each milestone:
- after Step 2 (plan made)
- after the workers report (Step 3)
- after each review verdict and fix round (Steps 4-5)
- after the merge and push (Step 6). Set `Status: done`.
- whenever you're waiting on the user. Put the question under **Open**.

Don't write it on turns where nothing changed.

**Keep it condensed:** it shows the current state, not a history (git has the history). Aim for under ~40 lines. Drop details once they stop mattering. Use exactly this shape:

```markdown
# PLAN: <feature-branch>
Request: <one line>
Feature head at planning: <sha>
Status: step <n>/7 (<name>) | done

## Assignments
| # | Scope | Worker branch | Commit | Review |
|---|-------|---------------|--------|--------|
| 1 | <scope> | <branch or -> | <sha or -> | pending / approve / fix round N: <k> findings |

## Decisions
- <decision> (<why>)

## Open
- <question for the user or blocker, or "none">

## Next
- <the exact next action, specific enough to resume cold>
```

## Step 0: Is this trivial?

If the change is a typo, a one-line config tweak, or a docs fix that needs no judgment, skip the workflow: do it yourself on a feature branch, run lint and format, commit, and push. **Tell the user you skipped the workflow and why.** Otherwise continue.

## Step 1: Branch

```
git switch main && git pull --ff-only origin main
git switch -c <type>/<short-kebab-name>     # type: feat | fix | chore | refactor | docs | test
```

If the pull fails, continue from local `main` and tell the user it may be out of date.

If you're already on a feature branch that this request extends, stay on it. The working tree must be clean before you spawn workers, because worker worktrees are expected to start from the current commit (each worker checks this; see the template). If there are pending changes that belong to this request, commit them. If they don't belong to it, ask the user before you stash or commit them. Never sweep unrelated work into the branch.

Record the feature head: `git rev-parse HEAD`. Every worker prompt needs it.

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
## First step (required)
Run: git merge-base --is-ancestor <feature-head-sha> HEAD
If it fails, your worktree isn't based on the feature branch: stop and report that, and change nothing.
<For a fix round only: first run `git reset --hard <previous-worker-branch>` to pick up the earlier work, then do the check above.>

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
The changed code isn't in your checkout. Read files as they are on that branch with `git show <worker-branch>:<path>`.
```

The last line is a technical pointer, not context. **Never** include the plan, the worker's report, your opinion of the code, or hints about what to look for. The reviewer is useful because it has no context.

## Step 5: Fix loop

- **Blockers and should-fix findings**: first check them yourself. Reviewers can be wrong. Send the confirmed ones back to a worker. Use SendMessage to the same worker if it's still reachable; it keeps working on its own branch. Otherwise, start a new `worker` (again with `isolation: "worktree"`) using the template's fix-round first step: `git reset --hard <previous-worker-branch>`. It can't check out the old branch directly, because that branch is still checked out in the old worktree. Add the findings to its Requirements. The new worker's branch then becomes the branch you review and merge. Then review again with a **new** `code-reviewer` (again, only goal and diff).
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
- `git push -u` exits non-zero if the sandbox blocks writing the upstream to `.git/config`, even when the push went through. Don't go by the exit code. Check the output for `-> <branch>`, or run `git ls-remote origin <branch>` and compare it with `git rev-parse HEAD`.

## Step 7: Report

Tell the user, concisely:

1. **What was done**: per assignment, one or two lines.
2. **Review outcome**: verdicts, what was fixed, and any nits left open.
3. **Verification**: lint, format, and test results after the merge.
4. **Branch**: name and head commit, plus that it's pushed.
5. **PR link**: `gh` usually fails in the sandbox with a TLS certificate error, even though plain `git` over HTTPS works. Give the compare URL to open the PR:
   `https://github.com/<owner>/<repo>/compare/main...<feature-branch>` (owner/repo come from `git remote get-url origin`).
   Open the PR yourself only if the user asks and `gh` works.
6. **Leftovers**: worker worktrees (paths come from the Agent tool results) that weren't cleaned up automatically. Give the user the `git worktree remove <path>` commands, since the sandbox may block running them.
