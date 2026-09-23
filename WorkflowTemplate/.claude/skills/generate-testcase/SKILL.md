
---
name: generate-testcase
description: When a story/PBI has no existing test case in ADO, analyze it with ADO-only RAG context, generate new test cases, and publish them to ADO after user confirmation. Use before fetch-testcase when no test case exists yet.
---

# generate-testcase

Act as a Manual Test Lead generating contextually relevant test cases for a given ADO work item (PBI, User Story, or Feature), using only ADO-hosted context (no other organizations' data). Use this skill when `/fetch-testcase` finds no linked test case for a story — this skill creates one.

Org/project come from `flow.config.json` at the project root (`adoOrgProject.org`, `adoOrgProject.project`).

## Phase 1 — Work Item Analysis & Fetching

Given the work item ID:
- Fetch via the `ado` MCP server: Title, Description, Acceptance Criteria, System Info, Area Path, Release Priority, Discussion/comments, and Related Work (Parent, Child, Related, Tested By, Testing, etc.) from "Linked By".
- If any details are provided as images/attachments, extract and display the text in structured form.
- Build ADO-only contextual RAG: find the closest matching Bugs/PBIs/Features from ADO using semantic/keyword search, keeping only matches with a similarity/relevance greater than roughly 60–65%. Display these in a table: `id | title | reason considered similar`.
- Use the linked/similar test cases (if any) purely as a style reference — writing tone, structure, phrasing of actions/expected results, and the mix of positive/negative/edge coverage they use.
- Do not use data from other organizations or unrelated projects. Do not invent requirements or scenarios not present in the work item or retrieved context — if something is unclear, note it rather than guessing.
- This phase is silent about data-source fallbacks (don't pause to explain RAG availability) but IS allowed to surface genuinely relevant caveats about missing/ambiguous requirements.

## Phase 2 — Test Case Generation

Generate comprehensive test cases covering:
- Positive scenarios (from acceptance criteria + contextual examples)
- Negative scenarios (invalid input/errors, styled like similar PBIs)
- Edge cases
- Integration points (APIs/system dependencies), phrased like contextual PBIs

Each test case must include: Title, Pre-conditions, Sequential Steps (Action + Expected Result), Priority (1-3), Test Type, Tags.

Output format for each test case:
Title:
[Title of the Test Case]
Pre-conditions:
[List any pre-conditions required for this test case]
Steps with Expected Results:
[Step 1]
Action: ...
Expected Result: ...
[Step 2]
Action: ...
Expected Result: ...
... [Step N]
Tags: [tag1, tag2]
Priority: [1-3]
Test Type: [Functional/Integration/UI/API/etc.]

Also construct the ADO XML steps representation for each test case (needed for Phase 4):
<extracted_text>
```xml
<steps id="0" last="N">
  <step id="1" type="ValidateStep">
    <parameterizedString isformatted="true">[Action]</parameterizedString>
    <parameterizedString isformatted="true">[Expected Result]</parameterizedString>
  </step>
  ...
</steps>

Phase 3 — Traceability Report
Produce:

A table mapping Acceptance Criteria ↔ generated Test Case(s).

A reference summary table of everything used: Reference Type | ID / Link | Title / Description | Usage Context — covering the work item itself, bugs, existing test cases, discussions/related work, and any user-provided links. Only include ADO data from the configured org/project.

Phase 4 — Confirmation, then ADO Publish
Deviation from the original prompt this skill is based on: publishing is never automatic, and publishing is always selective. Phase 2 may generate several test cases (positive/negative/edge/integration); that does NOT mean all of them get published. Before creating anything in ADO:

Show the user the full generated test case(s) from Phase 2 (title, steps, priority, tags), each clearly numbered/titled, and the traceability table from Phase 3.

Explicitly ask the user which of the generated test cases should be published to ADO, and wait for their reply. The user may answer with a specific title, a number/subset, "all", or "none" — publish exactly what they specify, nothing more. If their answer is ambiguous (e.g. a title that doesn't exactly match one you generated), confirm which one they mean before publishing anything. Do not proceed to step 3 without an explicit answer.

Only for the confirmed test case(s) — never the rest — use the ado MCP server to create the Test Case work item(s):

Populate Work Item Type, Title, Author, Priority, and the Steps field with the XML built in Phase 2 (must be valid, non-empty XML — regenerate rather than publish empty/invalid steps).

Set Area Path equal to the Area Path of the source work item.

Add the tag QA_AI_EPAM_TC_Agent to the Tags field.

Link the new test case under the "Tested By" relationship to the source work item ID.

After publishing, show the user a table of Test Case ID | Title | Status plus a link to the ADO work item(s) for verification.

Constraints
ADO-only context; never use other organizations' or unrelated projects' data.

No hallucinated features/requirements — only what's explicitly in the work item or retrieved ADO context.

Match tone, length, and structure of retrieved contextual test cases where available.

Never skip Phase 4's confirmation step, even for a single test case.