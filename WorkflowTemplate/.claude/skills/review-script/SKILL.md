
---
name: review-script
description: Review the C# test/page-object changes generate-script and find-locators produced in the target repo against its own code-review checklist, before raise-pr. Produces findings only, never auto-fixes or pushes.
---

# review-script

Review the working-tree changes made by `/generate-script` and `/find-locators` in the target repo (`targetRepoPath` from `flow.config.json`), against that repo's own review standard. This is a read-only review step — it reports findings, it does not edit files, commit, or push.

## Inputs

- `targetRepoPath` from `flow.config.json`.
- `file_filter` (optional) — one or more specific file names/paths (or a distinctive part of one, e.g. `PinnedToolsTests.cs` or `ToolCardPage`), passed as an argument, e.g. `/review-script PinnedToolsTests.cs`. When given, review **only** the matching file(s), even if other uncommitted changes exist elsewhere in the repo.
- If no `file_filter` is given: the set of files changed by the earlier steps in this conversation, or `git status`/`git diff` in `targetRepoPath` to see current uncommitted changes.

## Steps

1. Read `flow.config.json` for `targetRepoPath`.
2. Re-read the target repo's own instructions to catch anything that's changed since this skill was written (source of truth, not memory):
   - `.github/instructions/code-review.instructions.md`
   - `.github/instructions/code-suggestions.instructions.md`
   - `.github/copilot-instructions.md`
3. Determine which files to review:
   - **`file_filter` given** — run `git status`/`git diff --name-only` in `targetRepoPath` and match it against the uncommitted files (exact name or substring match). Review only the match(es). If nothing uncommitted matches, say so explicitly and list what *is* uncommitted, rather than silently reviewing something else or the whole repo.
   - **No `file_filter`** — run `git status`/`git diff --name-only` in `targetRepoPath` as the source of truth for "what's uncommitted," not conversation memory (multiple `/generate-script` passes or manual edits since may have changed more than what's in this conversation). Review every file that shows up as modified/untracked there. If there are no uncommitted changes at all, say so and stop — there's nothing to review.
4. Review each changed file against the checklist below (ported from the target repo's own `code-review.instructions.md` — keep this in sync if that file changes):
   - **Raw driver calls inside a test method** (`driver.FindElement`, `WebDriver.FindElement`, `session.FindElementByXPath`, etc.) — all element interaction must go through a page object in `Pages/`.
   - **Missing or happy-path-only cleanup** — anything pinned/created/opened must be undone even if an earlier `Assert` throws; prefer `try/finally` or `[TearDown]` over trailing cleanup calls.
   - **Bare `Thread.Sleep` used as a wait** instead of an explicit wait (`BrowserUtil.WaitUntilPageLoad`, a `WaitFor...` page-object method).
   - **Re-reading transient UI state** (toasts/alerts that auto-dismiss) instead of snapshotting it once into a local variable.
   - **Missing/wrong test metadata**: `[TestCaseId("...")]`, appropriate `[Category(...)]`, method name matching `TC_<CaseId>_VerifyDescriptiveName`, class-level `[Category(...)]` and XML-doc summary naming the story/ACs.
   - **Assertions not grouped in `Assert.Multiple`** when checking multiple independent conditions from the same action.
   - **Silently swallowed exceptions** in test-body assertions or page-object action methods (as opposed to teardown/cleanup helpers, where it's fine).
   - **Any reintroduction of Playwright**, or the desktop app driven by anything other than `WindowsDriver<WindowsElement>`/Appium.
   - **Hardcoded secrets/credentials or environment-specific URLs** in test code or new `Data/`/`Configurations/` files.
   - **Enforced StyleCop rules**: SA1028 (trailing whitespace), SA1210 (using-directive order), SA1013/SA1508 (brace spacing), SA1413 (missing trailing commas in multi-line initializers), SA1000 (keyword spacing). Do NOT flag rules this repo intentionally relaxes: SA1600/SA1602 (missing XML doc on ordinary methods), SA1633 (missing file headers), SA1401 (public fields), SA1202 (member ordering), SA1101 (missing `this.` prefix).
   - Also don't flag: public/protected fields on base test classes, missing XML docs on ordinary (non-test-class) methods, test classes not sealed or lacking interfaces.
   - **Placeholder locators left unresolved** — if any `PLACEHOLDER_LOCATOR_...` value is still present, that's a blocking finding: `/find-locators` needs to run (or run again) before this can go to PR.
5. Present findings as a list, each with: file, line/method, what's wrong, why it matters (tie back to the specific rule above), and a suggested fix. If nothing is wrong, say so explicitly rather than inventing minor nitpicks.
6. Do not modify any files. If the user asks you to fix findings, that's a distinct, explicit follow-up action — apply fixes only after they say so, and re-show the diff before considering the review complete.

## Output

A findings list (or a clean bill of health) shown in the chat. This becomes the gate for `/raise-pr` — that skill should not be run while blocking findings (placeholder locators, Playwright reintroduction, hardcoded secrets) remain unresolved.
