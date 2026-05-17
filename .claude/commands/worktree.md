---
description: Create a git worktree on the given branch and spawn a Claude Code subagent to work in it. Use for reviewing a feature branch in isolation without disturbing your main checkout, running an `ios-pr-reviewer` against a PR's branch, parallelizing analysis across branches, or kicking off a long-running planner without blocking your current session.
argument-hint: <branch> <agent-name> [task description...]
disable-model-invocation: true
allowed-tools: Bash(git *) Bash(mkdir *) Bash(ls *) Bash(realpath *) Bash(basename *) Read Grep Glob Agent
---

# /worktree — create a worktree and dispatch a subagent into it

## Inputs

- `$0` — branch name (required). Existing local branch, or remote branch like `origin/feature-x` (a local tracking branch will be created).
- `$1` — subagent name (required). One of: `ios-pr-reviewer`, `concurrency-migration-planner`, `ios-architect`, or any other agent under `.claude/agents/` / `~/.claude/agents/`.
- `$ARGUMENTS` from position 2 onward — the task prompt to send the agent. Optional but strongly recommended; without it the agent has nothing to do.

If `$0` or `$1` is empty, print usage and stop:

```
Usage: /worktree <branch> <agent-name> [task description]
Example: /worktree feature/checkout-async ios-pr-reviewer "review the latest commit for concurrency issues"
```

## Repo state (resolved before this prompt reaches Claude)

- Repo root: !`git rev-parse --show-toplevel 2>/dev/null || echo "(not a git repo)"`
- Repo basename: !`basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)"`
- Current branch: !`git branch --show-current 2>/dev/null || echo "(detached or no repo)"`
- Local branches: !`git for-each-ref --format='%(refname:short)' refs/heads/ 2>/dev/null | head -20`
- Remote branches: !`git for-each-ref --format='%(refname:short)' refs/remotes/ 2>/dev/null | head -20`
- Existing worktrees: !`git worktree list 2>/dev/null`

If the repo state above shows "not a git repo", stop and tell the user this command only works inside a git repository.

## Procedure

1. **Resolve the branch.**
   - If `$0` matches an existing local branch, use it directly.
   - If `$0` matches a remote branch (e.g., `origin/feature-x`), run `git fetch origin <branch>` and let `git worktree add` create the local tracking branch (`-b $0 origin/$0`).
   - If `$0` doesn't exist anywhere, stop and report what branches *do* exist with the closest matches.

2. **Compute the worktree path.** Use `<repo-root>/../<repo-basename>-<sanitized-branch>`. Sanitize by replacing `/` and any non-alphanumeric characters (except `-` and `.`) with `-`. Example: `feature/checkout-async` → `myrepo-feature-checkout-async`.

3. **Refuse to clobber.** If the computed path already exists (file or directory), stop. Show the path and instruct the user to `git worktree remove <path>` or pick a different branch — do not overwrite.

4. **Create the worktree.**
   ```sh
   git worktree add <path> <branch>
   ```
   If creation fails, surface the exact `git` error and stop.

5. **Dispatch the agent.** Use the Agent tool to spawn `$1` (set `subagent_type` to the agent name). The prompt you pass MUST include:
   - The task text (the joined arguments from position 2 onward).
   - An explicit instruction that **the agent's working directory is the main session's CWD, not the worktree.** The agent must use absolute paths rooted at the worktree path for every `Read`, `Grep`, `Glob`, and `Bash` call. For `git` invocations inside the worktree, prefix with `-C <worktree-path>` (e.g., `git -C /abs/path/worktree diff main...HEAD`).
   - The absolute worktree path, computed in step 2.

6. **Report to the user.** Once the agent returns, output (in this order):
   - **Worktree:** the absolute path.
   - **Branch:** the branch name and where it tracks (if applicable).
   - **Agent:** which agent ran.
   - **Output:** the agent's findings, passed through unchanged.
   - **Cleanup:** `git worktree remove <path>` when done. (Do not run this yourself — the user reviews first.)

## Notes for Claude

- Do not delete the worktree after the agent finishes. The user keeps it until they explicitly clean up.
- Do not `cd` into the worktree from this command — `cd` doesn't persist across Bash calls and the agent runs in the parent CWD anyway. Use absolute paths.
- If the user gave no task text (only branch and agent), spawn the agent with a default task derived from its purpose: `ios-pr-reviewer` → "review the diff of this branch against its merge base"; `concurrency-migration-planner` → "plan the concurrency migration for this target"; `ios-architect` → stop and ask the user for the architecture question (the architect needs a real question, not a generic prompt).
- If the agent itself errors, leave the worktree in place — the user may want to inspect it manually.
- Do not interpret the task text. Pass it through verbatim to the agent.
