# WorkflowTemplate

Reusable flow that turns an Azure DevOps story into a test script, locators, a review, and a pull request. It is a Claude Code project: MCP servers in `WorkflowTemplate/.mcp.json` and skills in `WorkflowTemplate/.claude/skills/`. It edits the target test repo from `flow.config.json`, not this repo.

<img width="886" height="414" alt="image" src="https://github.com/user-attachments/assets/a62390cc-614d-4adc-98f7-10d9d08a7a57" />

## Prerequisites

- Claude Code
- Node.js (`npx` installs the MCP servers)
- Google Chrome (Playwright MCP)
- Git, plus a GitHub PAT with `repo`, `read:org`, and `read:user`

## Setup

1. Build the ADO token. `@azure-devops/mcp` expects `PERSONAL_ACCESS_TOKEN` to be base64 of `email:pat`, with `--authentication pat` already set in `.mcp.json`.

   ```powershell
   [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("you@domain.com:yourAdoPat"))
   ```

2. Set these in the shell before you start Claude Code. `.mcp.json` expands `${VAR_NAME}` from the process environment only.

   ```powershell
   $env:ADO_PAT_B64 = "<base64 from step 1>"
   $env:GITHUB_PAT = "<GitHub PAT>"
   ```

   In cmd.exe: `set ADO_PAT_B64=...` and `set GITHUB_PAT=...` (no `$env:`, no quotes).

3. Open Claude Code in `WorkflowTemplate/` so `.mcp.json` and `.claude/settings.json` load.
4. Point `flow.config.json` at the target repo, ADO org/project, base branch, and app URL. Set the same ADO project name in the `ado` server args in `.mcp.json`.
5. Keep `permissions.additionalDirectories` in `.claude/settings.json` equal to `targetRepoPath`. Restart Claude Code after any permission or env change. That entry grants file access only. `generate-script` still reads the target repo’s own conventions from its files.

## Skills

Run these in order. Each skill stops for a manual checkpoint.

| Order | Skill | When |
|---|---|---|
| Optional first | `/generate-testcase <story-id>` | The story has no test case yet. Publishes to ADO only after you choose which cases to create. |
| 1 | `/fetch-testcase <story-id>` | Loads the story and its linked test cases. If none exist, it tells you to run `/generate-testcase`. |
| 2 | `/generate-script` | Writes the NUnit/Selenium/Appium test and page-object changes in the target repo. |
| 3 | `/find-locators` | Fills placeholder locators from the live app via Playwright MCP. Writes selector strings only. |
| 4 | `/review-script` | Blocking review against the target repo’s checklist. Findings only. |
| Optional | `/optimize-script` | Execution-time notes. Does not block `/raise-pr`. |
| 5 | `/raise-pr` | New branch and PR in the target repo, only after you confirm. Requires a clean `/review-script`. |

### Maintenance

`/heal-testcase` is separate from the pipeline. Use it when a web test that used to pass fails on a stale locator. It refuses assertion or functional failures, and it asks before editing. Desktop Appium/WinAppDriver failures stay manual.

## Still to verify

Run `generate-script` → `find-locators` → `review-script` / `optimize-script` → `raise-pr` on a real story, and run `heal-testcase` on a real failing web test.


