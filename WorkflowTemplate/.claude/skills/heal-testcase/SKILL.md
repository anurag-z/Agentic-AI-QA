
---
name: heal-testcase
description: Diagnose an existing web test failure and, only if it's caused by a stale/broken locator (not a real assertion/functional failure), propose a fix using Playwright MCP inspection — never applied without explicit confirmation. Maintenance-time repair, independent of the create-test pipeline.
---

# heal-testcase

Repair a **web** test that used to pass and is now failing because the UI changed and a locator no longer matches anything. This is a maintenance action, invoked independently at any time — not part of the ADO->PR creation pipeline (`fetch-testcase` -> `generate-script` -> ... -> `raise-pr`).

**Scope for this version: web tests only** (Selenium, driven via a real browser). If the failing test belongs to the desktop `iDesign` app (Appium/WinAppDriver), say so explicitly and stop — Playwright MCP cannot inspect a WinForms app, so desktop healing must be done manually (e.g. Appium Inspector) and is out of scope here.

## Inputs (either is acceptable — ask which the user has if neither is provided)

- Pasted failure output (stack trace / exception message / console output) directly in the chat, **or**
- An explicit path to an ExtentReport (or other result file) the user points you at. Never search the repo for "the latest" report yourself — only use a report file the user explicitly names.
- The failing test's identifier (class + method name, or `[TestCaseId]`) if it isn't obvious from the failure evidence.
- `targetRepoPath` from `flow.config.json`, `appBaseUrl` for live inspection.

Refuse to proceed without actual failure evidence in one of the two forms above — never guess which locator broke from the test name alone.

## Step 1 — Classify the failure (do this before anything else)

Read the failure evidence and classify it:

- **Locator-shaped failure** — `NoSuchElementException`, `ElementNotInteractableException`, `StaleElementReferenceException`, or a timeout waiting for an element to appear/become interactable. -> proceed to Step 2.
- **Assertion/functional failure** — an `Assert.*` mismatch, wrong value/text found, unexpected application state. -> **stop immediately** and tell the user this looks like a real functional failure, not a broken locator, and healing it would risk hiding an actual bug. Do not touch any files.
- **Ambiguous / can't tell from the evidence given** -> stop and ask for more detail (fuller stack trace, or the actual report file) rather than guessing either way.

## Step 2 — Locate the implicated locator

From the stack trace / failing page-object method, identify the exact `private const string` locator field responsible for the failure in the relevant `Pages/**/*.cs` file. If it's not clear which locator/method is implicated, ask rather than guessing.

## Step 3 — Re-resolve the locator against the live app

1. Confirm `appBaseUrl` in `flow.config.json` (ask if empty).
2. Use the `playwright` MCP server to navigate to the same screen/state the failing test step targets.
3. Inspect the DOM/accessibility tree for an element that plausibly replaced the old one — same semantic role, nearby label/text, or position as what the old locator's name/intent suggests. Prefer `id`/`data-testid`/`name` over XPath, same as `find-locators`.
4. Verify the candidate resolves to exactly one element before treating it as a fix.
5. If nothing plausible is found, say so — don't force a low-confidence guess into the file.

## Step 4 — Confirm before applying (never skip this)

This modifies existing, possibly already-merged code — treat it as higher-stakes than `find-locators`' fresh placeholders. Before touching any file:

- Show the user: old locator value, proposed new value, the element found (role/text/attributes), and your confidence/reasoning for why it's the same logical element.
- Explicitly ask for confirmation to apply the change. Do not edit the file until the user confirms.
