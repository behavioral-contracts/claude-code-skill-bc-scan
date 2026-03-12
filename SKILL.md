# bc-scan Skill

**Trigger:** User invokes `/bc-scan`, says "bc scan", "behavioral contracts scan", or "run bc scan"
**Version:** 1.0.0
**Skill name:** bc-scan

> This skill is designed as a standalone git repo, installable via:
> `gh repo clone behavioral-contracts/claude-code-skill-bc-scan ~/.claude/skills/bc-scan`

---

## Purpose

Run a local behavioral contract scan against the current TypeScript project using `@behavioral-contracts/verify-cli`, then upload results to the Behavioral Contracts dashboard. Gives developers full scan + dashboard reporting without leaving Claude Code — no GitHub App required, works offline or on private repos.

---

## One-Time Setup

Run these steps once before using `/bc-scan`:

**Step 1: Get your API key**
Visit https://app.behavioral-contracts.com/settings?tab=api-keys and create an API key.

**Step 2: Set environment variables**
```bash
export BC_API_KEY=your_api_key_here
export BC_REPO_ID=your_repository_id_here
```

Add both lines to your `~/.zshrc` or `~/.bashrc` so they persist across sessions.

**Step 3: Find your repository ID**
Log in to https://app.behavioral-contracts.com, open the repository you want to track, and copy the UUID from the URL: `https://app.behavioral-contracts.com/repositories/<YOUR_REPO_ID>`.

**Step 4: Verify the CLI is accessible**
```bash
npx @behavioral-contracts/verify-cli --help
```

If you see the help text, you are ready. If not, install globally:
```bash
npm install -g @behavioral-contracts/verify-cli
```

---

## Main Workflow

When `/bc-scan` is invoked, follow these steps in order:

### Step 1 — Find tsconfig.json

Check for a TypeScript config in order:
1. `./tsconfig.json` (current working directory)
2. `./apps/web/tsconfig.json`
3. `./packages/*/tsconfig.json` (glob — pick the first match)
4. Any other `tsconfig.json` visible via `find . -name tsconfig.json -not -path "*/node_modules/*" -maxdepth 4`

If exactly one tsconfig is found, use it automatically.
If multiple are found, show the list and ask the user: "Which tsconfig should I scan? (Enter the path)"
If none is found, say: "No tsconfig.json found. Please specify the path: e.g., `/bc-scan --tsconfig ./my-app/tsconfig.json`"

### Step 2 — Check environment variables

```bash
echo "BC_API_KEY=$BC_API_KEY"
echo "BC_REPO_ID=$BC_REPO_ID"
```

If either variable is empty or unset, show the one-time setup instructions above and stop.

### Step 3 — Run verify-cli

```bash
npx @behavioral-contracts/verify-cli \
  --tsconfig PATH_TO_TSCONFIG \
  --output /tmp/bc-scan-results.json \
  --no-terminal
```

- Replace `PATH_TO_TSCONFIG` with the path found in Step 1.
- `--no-terminal` suppresses the full interactive report; we'll show our own summary below.
- `--output /tmp/bc-scan-results.json` writes the audit JSON to a known location.

If the command exits with a non-zero code AND `/tmp/bc-scan-results.json` does not exist, show the error and stop. (Exit code 1 is normal when violations are found — do not treat it as a fatal error if the output file was written.)

### Step 4 — Parse results

Read `/tmp/bc-scan-results.json`. The file is a JSON audit record. Extract:
- `violations` array — each item has fields: `file`, `line`, `column`, `package`, `function`, `postconditionId`, `severity` (lowercase: `"error"` or `"warning"`), `message`, `codeContext`
- `summary.filesAnalyzed` — number of files scanned

Compute:
- `errorCount` = violations where `severity === "error"`
- `warningCount` = violations where `severity === "warning"`
- `totalViolations` = violations.length

### Step 5 — Upload to Behavioral Contracts

Build the upload payload by mapping verify-cli's violation format to the API format:

```json
{
  "repositoryId": "$BC_REPO_ID",
  "violations": [
    {
      "packageName": "<violation.package>",
      "rule": "<violation.postconditionId>",
      "severity": "<UPPERCASE: ERROR or WARNING>",
      "message": "<violation.message>",
      "filePath": "<violation.file>",
      "lineNumber": "<violation.line>",
      "columnNumber": "<violation.column>",
      "functionName": "<violation.function>",
      "codeSnippet": "<violation.codeContext>"
    }
  ],
  "summary": {
    "totalViolations": <computed>,
    "errorCount": <computed>,
    "warningCount": <computed>,
    "scannedFiles": <summary.filesAnalyzed from audit>
  },
  "commitSha": "<git rev-parse --short HEAD 2>/dev/null || echo local-upload>",
  "branch": "<git branch --show-current 2>/dev/null || echo local>"
}
```

**Key mapping notes:**
- `severity` must be uppercased for the API: `"error"` -> `"ERROR"`, `"warning"` -> `"WARNING"`
- `scannedFiles` in the API corresponds to `filesAnalyzed` in the verify-cli output
- `rule` in the API corresponds to `postconditionId` in the verify-cli output

Write the payload to `/tmp/bc-upload-payload.json`, then upload:

```bash
curl -s -X POST https://app.behavioral-contracts.com/api/mcp/upload \
  -H "Authorization: Bearer $BC_API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/bc-upload-payload.json
```

Parse the response JSON. If `success` is `true`, note the `scanId`. If the response contains an `error` field or the HTTP status is not 200, show the error message and stop.

### Step 6 — Report summary

Show a clean summary to the user:

```
Behavioral Contracts Scan Complete

Files scanned: <filesAnalyzed>
Violations:    <totalViolations> (<errorCount> errors, <warningCount> warnings)
Scan ID:       <scanId>

Dashboard: https://app.behavioral-contracts.com/repositories/<BC_REPO_ID>
```

If there are errors, show the top 3 by severity (errors first), formatted as:
```
  [ERROR] <package>.<function> — <message>
          <file>:<line>
```

If there are zero violations, show:
```
No violations found. Your code satisfies all behavioral contracts checked.
```

### Step 7 — Cleanup

```bash
rm -f /tmp/bc-scan-results.json /tmp/bc-upload-payload.json
```

---

## Optional: Fix Mode

If the user invokes `/bc-scan --fix` or says "bc scan and fix":

1. Run Steps 1 through 6 above.
2. After showing the summary, ask: "Fix the top 3 ERROR violations? (yes/no)"
3. If yes, work through each ERROR violation in turn:
   - Read the file at `violation.file`, jump to `violation.line`
   - Use the `message` and `postconditionId` to understand what fix is needed (the message describes the required error handling)
   - Edit the file to wrap the call site in a try-catch block with appropriate error handling
   - After all fixes, remind the user to re-run `/bc-scan` to confirm violations are resolved

---

## Optional: Claude Hooks Integration

You can configure Claude Code to automatically run a behavioral contract scan after every git commit.

Add this to your project's `.claude/settings.json` (or create the file):

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "if echo \"$CLAUDE_TOOL_INPUT\" | grep -q 'git commit'; then npx @behavioral-contracts/verify-cli --tsconfig ./tsconfig.json --output /tmp/bc-results.json --no-terminal && python3 -c \"import json,os,subprocess,sys; data=json.load(open('/tmp/bc-results.json')); viols=[{'packageName':v['package'],'rule':v['postconditionId'],'severity':v['severity'].upper(),'message':v['message'],'filePath':v['file'],'lineNumber':v['line'],'columnNumber':v.get('column'),'functionName':v['function'],'codeSnippet':v.get('codeContext','')} for v in data.get('violations',[])]; summary={'totalViolations':len(viols),'errorCount':sum(1 for v in viols if v['severity']=='ERROR'),'warningCount':sum(1 for v in viols if v['severity']=='WARNING'),'scannedFiles':data.get('summary',{}).get('filesAnalyzed',0)}; payload={'repositoryId':os.environ.get('BC_REPO_ID',''),'violations':viols,'summary':summary}; f=open('/tmp/bc-hook-payload.json','w'); json.dump(payload,f); f.close(); subprocess.run(['curl','-s','-X','POST','https://app.behavioral-contracts.com/api/mcp/upload','-H','Authorization: Bearer '+os.environ.get('BC_API_KEY',''),'-H','Content-Type: application/json','-d','@/tmp/bc-hook-payload.json']); os.remove('/tmp/bc-results.json'); os.remove('/tmp/bc-hook-payload.json')\"; fi"
          }
        ]
      }
    ]
  }
}
```

This hook fires after every `Bash` tool call that includes `git commit`, runs the scan, and silently uploads results. The hook requires `BC_API_KEY` and `BC_REPO_ID` to be set in your shell environment.

For a cleaner setup, extract the Python snippet to a script at `scripts/bc-post-commit.py` and call it from the hook command.

---

## MCP Cloud Mode (Alternative)

If you have the Behavioral Contracts MCP server configured, you can use cloud-based scanning without the local CLI:

- `trigger_scan` — kick off a scan for a connected GitHub repository
- `get_latest_scan` — retrieve the most recent scan results
- `search_violations` — search across all violations

Setup: https://app.behavioral-contracts.com/developer

The local scan mode (`/bc-scan`) is preferred when:
- The repository is not connected to GitHub
- You want to scan before committing (pre-flight check)
- You need offline support
- You want faster iteration without waiting for cloud queue

---

## Troubleshooting

**`verify-cli` not found after `npx`**
```bash
npm install -g @behavioral-contracts/verify-cli
```
Then use `behavioral-contracts` or `verify-cli` instead of `npx @behavioral-contracts/verify-cli`.

**401 Unauthorized from upload API**
Your API key is invalid or expired. Regenerate it at:
https://app.behavioral-contracts.com/settings?tab=api-keys

**403 Plan restriction**
The MCP upload endpoint requires the Solo plan or higher. Upgrade at:
https://app.behavioral-contracts.com

**Repository not found (400 or scan not appearing)**
Confirm `BC_REPO_ID` matches the UUID shown in the dashboard URL when viewing the repository. Do not use the repository's GitHub URL — use the platform UUID.

**No tsconfig.json found**
Specify the path explicitly:
```
/bc-scan --tsconfig ./apps/api/tsconfig.json
```
Or create a tsconfig if the project lacks TypeScript configuration.

**Scan runs but shows 0 violations unexpectedly**
Check that the corpus has contracts for your dependencies. Run:
```bash
npx @behavioral-contracts/verify-cli --help
```
and confirm the version is 2.x or later (V2 analyzer is the default).

---

## Publishing Note

This skill is installable as a standalone git repository:

```bash
gh repo clone behavioral-contracts/claude-code-skill-bc-scan ~/.claude/skills/bc-scan
```

Or via Claude Code's skill install mechanism when available. The skill directory is self-contained — no dependencies beyond `npx` and `curl`.

To contribute or report issues: https://github.com/behavioral-contracts/claude-code-skill-bc-scan
