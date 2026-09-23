
---
name: optimize-script
description: Suggest performance/maintainability improvements to a generated test's execution time (redundant waits, repeated lookups, duplicated locator logic), scoped to the target repo's own optimization guidance. Read-only findings, distinct from review-script's correctness checklist.
---

# optimize-script

Review a generated/modified test (and its page-object changes) for things that make it run slower than necessary, using the target repo's own optimization guidance as the standard — not generic "make it faster" instincts, which this repo explicitly warns against for a UI automation suite (raw speed tricks tend to make tests flakier, not just faster).

This is a **separate concern from `/review-script`**: `review-script` checks correctness/convention (does it follow the rules), `optimize-script` checks execution-time cost of code that's already otherwise correct. Run this after `/review-script` finds no blocking issues, or alongside it.

## Inputs

- `targetRepoPath` from `flow.config.json`.
- `file_filter` (optional) — same as `review-script`: a specific file name/substring to scope to; otherwise uses `git status`/`git diff --name-only` in `targetRepoPath` for the uncommitted change set.

## Steps

1. Read `flow.config.json` for `targetRepoPath`.
2. Re-read `.github/instructions/code-suggestions.instructions.md` in the target repo directly (source of truth, not memory — it may have changed).
3. Determine scope exactly like `review-script` does: `file_filter` match against uncommitted files if given, otherwise all uncommitted `.cs` files.
4. For each file, look specifically for:
   - **Redundant element/page lookups** — the same locator queried repeatedly inside a loop or across nearby lines instead of caching the result in a local variable. Appium/WinAppDriver round-trips are genuinely expensive here, so this is a real perf cost, not a micro-optimization.
   - **Duplicated locator/action logic** copy-pasted across the changed test rather than reusing (or newly consolidating into) an existing page-object method or `Helpers/` utility.
   - **`Thread.Sleep` used where an explicit wait would both be faster and more reliable** (`BrowserUtil.WaitUntilPageLoad`, a page-object `WaitFor...` method) — flag it here even if `review-script` also flagged it as a correctness issue; here the angle is "this sleep is also slower than necessary," not just "flaky."
   - **Repeated string literals** (tool names, URLs, timeouts) that already exist elsewhere in `Data/`/`Enums/`/`Configurations/` and should reuse that source instead of a fresh literal.
   - **Unnecessary re-navigation** — reloading a page/screen the test was already on, when the existing state could be reused.
5. Do **not** suggest any of the following, even if they'd technically execute faster — the target repo explicitly rules these out:
   - Migrating anything to Playwright or another driver.
   - Removing or "simplifying away" cleanup/teardown calls, even ones that look redundant.
   - Parallelizing or reordering tests that share desktop/browser session state, without confirming the base test class actually supports it.
   - Introducing new third-party assertion/mocking libraries.
   - Broad rewrites "for cleaner code" that touch StyleCop-relaxed areas as a side effect.
6. Present findings as a list: file, location, what's slow, why (tie to the guidance above), and the suggested fix consistent with the repo's existing patterns. If nothing meaningful is found, say so rather than inventing marginal nitpicks.
7. Do not modify any files. Apply a suggested fix only if the user explicitly asks, and re-show the diff afterward.

## Output

A findings list (or a clean bill of health) shown in the chat — informational, not a blocker for `/raise-pr` the way `review-script`'s findings are, unless the user decides otherwise.
