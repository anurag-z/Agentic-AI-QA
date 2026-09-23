---
name: generate-script
description: Generate a NUnit/Selenium/Appium C# test method (and any needed page-object changes) in the target repo from a structured test case produced by fetch-testcase or generate-testcase.
---

# generate-script

Turn a structured test case (from `/fetch-testcase` or `/generate-testcase`, present earlier in this conversation) into an automated test in the target repo, following that repo's existing conventions exactly. This skill does not touch ADO or GitHub.

## Inputs

- The structured test case(s) (Test Case ID/title, Steps with Action + Expected Result) from earlier in the conversation. If none are present, ask the user to run `/fetch-testcase` or `/generate-testcase` first — do not invent a test case.
- `targetRepoPath` from `flow.config.json` at the project root.

## Handling multiple test cases (bulk fetch)

If more than one structured test case is present in the conversation (e.g. from a bulk `/fetch-testcase`), process them **one at a time, in order, with an approval checkpoint between each**:

1. Announce which test case you're about to generate a script for (ID/title) and how many remain in the queue.
2. Run steps 2-7 below for that one test case only.
3. Show the result (files touched, placeholder locators) and explicitly ask whether to proceed to the next test case, skip it, or stop here.
4. Do not start generating the next test case's script until the user responds. Never silently batch-generate all of them in one pass — each one gets its own review checkpoint, since each touches real files in the target repo.

## Steps (per test case)

1. Read `flow.config.json` for `targetRepoPath`.
2. Before writing anything, study the target repo's own conventions directly from its files (do not rely on memory/assumptions):
   - `.github/copilot-instructions.md` — repo overview, Selenium (web) + Appium/WinAppDriver (desktop) only, **never Playwright** in generated code.
   - `.github/instructions/code-suggestions.instructions.md` — extend existing page objects rather than creating new ones, explicit waits over `Thread.Sleep`, no new assertion/mocking libraries, no unrelated rewrites.
   - At least one representative existing test class under `Tests/**/*.cs` and its corresponding page object under `Pages/**/*.cs` (e.g. `Tests/LandingPage_Tests/PinnedToolsTests.cs` + `Pages/LandingPages/ToolCardPage.cs`) to confirm current naming/structure — conventions may have evolved since this skill was written, so the live files are the source of truth.
3. Determine which existing test fixture and page object the new test belongs to based on the story/test case's feature area. Prefer extending an existing page object/test class over creating new files; only create new files if nothing existing fits.
4. Generate the test method following the observed conventions:
   - Class extends the appropriate base test class (e.g. `WebBaseTest` / `DesktopBaseTest` / `DesktopLoginBaseTest`).
   - Method named `TC_<CaseId>_VerifyDescriptiveName`.
   - `[Test(Description = "...")]`, `[TestCaseId("<case id>")]`, `[Category("...")]`.
   - Body translates each test-case Step (Action + Expected Result) into page-object calls + assertions; group assertions in `Assert.Multiple`.
   - Use the inherited `extent` logging helper for step-level logging, matching the observed style.
   - Include baseline/setup and cleanup, marked with the repo's existing `// baseline` / `// cleanup` comment convention.
5. For any UI element referenced that doesn't already have a locator in the relevant page object, add a `private const string` field for it, but leave its value as an obviously-fake placeholder (e.g. `"PLACEHOLDER_LOCATOR_<Name>"`) — real locators are filled in by `/find-locators`, not this skill.
6. Write the changes to the target repo's files using its existing file/folder layout — do not create a parallel structure.
7. Show the user a summary of exactly which files were created/modified and a short diff-style overview, and list every placeholder locator that still needs to be resolved by `/find-locators`.

## Constraints

- Never emit Playwright APIs or syntax — Selenium/Appium only, matching the target repo's existing driver usage.
- Never invent test steps not present in the input test case.
- Don't touch `.git`, don't commit — this skill only edits working-tree files in `targetRepoPath`.
