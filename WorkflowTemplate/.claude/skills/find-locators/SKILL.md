---
name: find-locators
description: Use the Playwright MCP server to inspect the running app and resolve placeholder locators left by generate-script into real selectors, for the target repo's page objects. Inspection only — never generates Playwright test code.
---

# find-locators

Resolve the placeholder locators left by `/generate-script` (e.g. `"PLACEHOLDER_LOCATOR_<Name>"`) into real, stable selectors, by driving the actual running app through the `playwright` MCP server.

**Hard guardrail:** the `playwright` MCP server exists here purely as a browser/DOM inspection tool. Nothing it returns may be translated into Playwright API calls or Playwright test syntax anywhere in the target repo — the target repo's test framework is Selenium (web) / Appium+WinAppDriver (desktop), per `.github/copilot-instructions.md`, and that must not change. Output of this skill is locator **strings** only (XPath/CSS/id), written into the existing `private const string` page-object fields.

## Inputs

- The placeholder locators reported by the most recent `/generate-script` run in this conversation (which files/fields need real values).
- `appBaseUrl` from `flow.config.json`. If it's empty, ask the user for the URL to navigate to before doing anything else — don't guess a URL.
- `targetRepoPath` from `flow.config.json`, to read the page object files that contain the placeholders and to write the resolved values back.

## Steps

1. Read `flow.config.json` for `appBaseUrl` and `targetRepoPath`. If `appBaseUrl` is empty, ask the user for it now.
2. Note: this skill only covers **web** locators (the landing page, driven via a real browser). If the placeholder locators belong to the **desktop** iDesign app (Appium/WinAppDriver), tell the user Playwright MCP can't inspect a WinForms desktop app and this step must be done manually or with a Windows UI inspection tool instead — do not attempt it.
3. Use the `playwright` MCP server to navigate to `appBaseUrl`, and to whichever specific screen/state the test case's steps require (log in, open the relevant page/panel, etc.) — reuse the navigation steps described in the fetched test case for context.
4. For each placeholder locator, inspect the DOM/accessibility tree (via the Playwright MCP's snapshot/inspect tools) to find the element the test step is describing. Prefer, in order of stability: `id` or `data-testid` attributes, `name`, then a scoped XPath as a last resort. Avoid brittle absolute XPaths or ones based on visible text that's likely to change/localize.
5. Confirm each candidate locator actually resolves to exactly one element (use the Playwright MCP's own query/count capability, not just eyeballing the snapshot) before treating it as final.
6. Update the target repo's page object file(s), replacing each `PLACEHOLDER_LOCATOR_<Name>` constant's value with the resolved selector string. Do not touch anything else in the file — this is a value-only edit to existing fields generate-script already created.
7. Show the user a before/after table: `Field name | Placeholder | Resolved locator | How found (id/data-testid/name/xpath)`.
8. Close/stop the Playwright MCP browser session when done.

## Constraints

- Never write Playwright API calls, imports, or syntax into any `.cs` file — output is locator strings only.
- Never invent a locator you haven't actually verified against the live DOM in this run.
- Don't modify test logic, only the locator constant values that `generate-script` left as placeholders.
- Don't touch `.git`, don't commit — this skill only edits working-tree files in `targetRepoPath`.
