---
name: fetch-testcase
description: Fetch Azure DevOps test case(s) linked to a story, optionally filtered to one specific test case by title or ID, and normalize them into a structured summary for the generate-script skill.
---

# fetch-testcase

Given a story/work-item ID (optionally plus a specific test case title or ID), pull the relevant test case(s) from Azure DevOps via the `ado` MCP server, and produce a structured summary that the `generate-script` skill can consume directly – one test case at a time, so `generate-script` only ever generates a script for the test case(s) you actually asked for.

## Inputs

- `story_id` (required) – the ADO work item ID, passed as an argument (e.g. `/fetch-testcase 123456`).
- `test_case_filter` (optional) – a specific test case title (or partial title) or test case ID, e.g. `/fetch-testcase 123456 "Verify invalid login"` or `/fetch-testcase 123456 4521`. When given, fetch and return **only that one test case**, not the others linked to the story.
- Org/project come from `flow.config.json` at the project root (`adoOrgProject.org`, `adoOrgProject.project`). Read this file first.

## Steps

1. Read `flow.config.json` for `org` and `project`.
2. Use the `ado` MCP server's work-item tools to fetch the work item by `story_id` in that org/project. If you're unsure of the exact tool name, list the `ado` server's available tools first and pick the one for getting a work item by ID (commonly named something like `wit_get_work_item`).
3. From the work item, identify linked Test Case work items (relation type is usually "Tested By" / a link to a work item of type "Test Case").
4. Decide which test case(s) to actually fetch and return, based on `test_case_filter`:
   - **No filter given, only one test case linked** -> fetch that one.
   - **No filter given, multiple test cases linked** -> do not fetch/return all of them silently. List them (`id | title`) and ask the user which one(s) they want, then fetch only those.
   - **Filter given as an ID** -> fetch that exact test case by ID (skip listing/asking).
   - **Filter given as a title/partial title** -> match it (case-insensitive, partial match ok) against the linked test cases' titles. If exactly one matches, fetch it. If none match, say so and list the actual linked titles so the user can correct the filter. If more than one matches, list the matches and ask which one they meant – never guess.
5. If no linked test case exists at all, stop and tell the user no test case was found for this story, and suggest running `/generate-testcase <story_id>` first to create and publish one, then re-running `/fetch-testcase`.
6. Normalize each fetched test case (only the ones resolved in step 4 – never the whole linked set unless the user explicitly asked for all of them) into this structure and print it back to the user (this becomes the input to `/generate-script`, one test case per script-generation pass):

    Story:  - 
    Test Case:  - 
    Steps:

    -> Expected:

    ...
    Notes: <anything ambiguous, missing, or that needed guessing>
If multiple test cases were resolved (user explicitly asked for more than one), print each as a separate block like the above and tell the user `/generate-script` will need to be run once per test case.

7. Do not fetch or print any secrets, PATs, or unrelated work items. This step is read-only against ADO – never create, update, or comment on work items.

## Output

The structured summary above, shown directly to the user in the chat – do not write it to a file unless asked. The user will feed it (or the next skill will read it from the current conversation) into `/generate-script`, which generates a script for exactly the test case(s) present in the conversation – never more than what was fetched here.