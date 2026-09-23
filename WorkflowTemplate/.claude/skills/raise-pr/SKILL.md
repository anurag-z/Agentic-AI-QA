
---
name: raise-pr
description: Commit the reviewed test/page-object changes in the target repo to a new branch and open a PR via the github MCP server. Always pauses for explicit confirmation before pushing or opening the PR — never auto-pushes.
---

# raise-pr

Final step of the flow. Commits the uncommitted changes in `targetRepoPath` to a new branch and opens a PR against `baseBranch`, using the `github` MCP server for the PR itself. **Never pushes or opens a PR without explicit confirmation from the user, every single run — no exceptions, even if a previous run was already approved.**

## Preconditions (check before doing anything)

1. `git status` in `targetRepoPath` shows uncommitted changes. If there's nothing to commit, say so and stop.
2. `/review-script` has been run against these changes in this conversation with **no blocking findings remaining** (raw driver calls outside page objects, missing cleanup, Playwright reintroduction, hardcoded secrets, or unresolved `PLACEHOLDER_LOCATOR_...` values). If it hasn't been run yet, or it reported blocking findings that haven't been addressed, stop and tell the user to run/resolve `/review-script` first — do not proceed on your own judgment that the code "looks fine."
3. `/optimize-script` findings, if any exist in the conversation, are informational only and do not block this step.

## Steps

1. Read `flow.config.json` for `targetRepoPath` and `baseBranch`.
2. Confirm the current branch in `targetRepoPath` is `baseBranch` (or at least not already a stale automation branch from a prior run) and it's up to date — run `git fetch` and check if `baseBranch` is behind `origin/<baseBranch>`; if so, tell the user and ask whether to pull first rather than branching from stale state.
3. Derive a branch name: `automation/TC-<case-id>-<short-slug-of-title>` (e.g. `automation/TC-4521-verify-invalid-login`), using the test case ID/title from earlier in the conversation. If no case ID is available, ask the user for a branch name instead of inventing one.
4. Create the branch locally (`git checkout -b <branch-name>`) and stage + commit exactly the files touched by this flow's earlier steps (`git add <specific files>` — never `git add -A`/`git add .` blindly, to avoid sweeping in unrelated local changes). Write a commit message summarizing what was added (test name, ADO test case ID, brief description) — show it to the user as part of the confirmation in step 5, not silently.
5. **Stop here and show the user a full summary before doing anything remote:**
   - Branch name and base branch.
   - List of files committed.
   - The commit message.
   - Draft PR title and description (reference the ADO test case/story ID, summarize what the test covers, note it was generated via this flow).
   Explicitly ask: proceed with push + PR, edit something first, or abort (leaving the local branch/commit intact but unpushed). Do not continue without an explicit yes.
6. Only after explicit confirmation: push the branch (`git push -u origin <branch-name>`), then use the `github` MCP server to open the PR against `baseBranch` with the confirmed title/description. If you're unsure of the exact tool name for creating a PR, list the `github` MCP server's available tools first and pick the appropriate one (commonly something like a "create pull request" tool) rather than guessing at parameters.
7. Show the user the resulting PR URL.

## Constraints

- Never push or call the PR-creation tool without the explicit confirmation in step 5, on every run.
- Never bypass the `review-script` precondition based on your own assessment that the code looks fine.
- Never stage/commit files outside what this flow touched — no blind `git add -A`.
- If pushing or PR creation fails partway (e.g. branch already exists remotely, auth error), report the exact error and stop — do not force-push or retry with destructive flags on your own judgment.

## Output

The PR URL (on success), or a clear stop-and-report of exactly which precondition/step blocked progress.
