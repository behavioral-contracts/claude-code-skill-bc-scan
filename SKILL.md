# bc-scan Skill

**Trigger:** `/bc-scan`, "bc scan", "behavioral contracts scan", "run bc scan"
**Version:** 1.1.0

Run a local behavioral contract scan and upload results to the Behavioral Contracts dashboard.

---

## Execution Workflow

### Step 1 — Resolve API key

Check in order:
1. `$BC_API_KEY` environment variable
2. `cat ~/.claude.json 2>/dev/null` → parse `mcpServers["behavioral-contracts"].headers.Authorization` → strip `"Bearer "` prefix
3. `cat ~/.claude/claude_desktop_config.json 2>/dev/null` → same path

If no key found: tell the user "API key not found. Follow docs/setup.md to configure MCP, or set BC_API_KEY." Stop.

Store the resolved key as `$API_KEY`.

### Step 2 — Find tsconfig.json

Check in order:
1. Argument `--tsconfig <path>` if provided
2. `./tsconfig.json`
3. `./apps/web/tsconfig.json`
4. `find . -name tsconfig.json -not -path "*/node_modules/*" -maxdepth 4`

If multiple found, list them and ask: "Which tsconfig? (enter path)"
If none found, stop with instructions to specify `--tsconfig`.

### Step 3 — Resolve repository ID

```bash
git remote get-url origin 2>/dev/null
```

Parse the output to `owner/repo` format (strips `.git`, handles both HTTPS and SSH formats).

Call `list_repositories` via the MCP REST endpoint:

```bash
curl -s -X POST https://app.behavioral-contracts.com/api/mcp \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"tool": "list_repositories", "args": {}}'
```

Find the entry where `fullName` matches `owner/repo`. Use its `id` as `$REPO_ID`.

If no match: "Repository not found. Connect it first at https://app.behavioral-contracts.com/repositories then try again." Stop.

If `BC_REPO_ID` is set as an env var, skip auto-detection and use it directly.

### Step 4 — Run verify-cli

```bash
npx @behavioral-contracts/verify-cli \
  --tsconfig <tsconfig_path> \
  --output /tmp/bc-scan-results.json \
  --no-terminal
```

If the output file is not created, show the error and stop. (Exit code 1 is normal when violations exist — only stop if the file is missing.)

### Step 5 — Parse results

Read `/tmp/bc-scan-results.json`. Extract:
- `violations[]` — each has: `file`, `line`, `column`, `package`, `function`, `postconditionId`, `severity` (lowercase), `message`, `codeContext`
- `summary.filesAnalyzed`

Compute `errorCount`, `warningCount`, `totalViolations`.

### Step 6 — Upload

Build payload:

```json
{
  "repositoryId": "$REPO_ID",
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
    "totalViolations": 0,
    "errorCount": 0,
    "warningCount": 0,
    "scannedFiles": "<summary.filesAnalyzed>"
  },
  "commitSha": "<git rev-parse --short HEAD 2>/dev/null || echo local>",
  "branch": "<git branch --show-current 2>/dev/null || echo local>"
}
```

Note: `severity` must be uppercased (`"error"` → `"ERROR"`).

```bash
curl -s -X POST https://app.behavioral-contracts.com/api/mcp/upload \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/bc-upload-payload.json
```

If `success` is not `true`, show the error and stop.

### Step 7 — Show summary

```
Behavioral Contracts Scan Complete

Files scanned:  <filesAnalyzed>
Violations:     <total> (<errors> errors, <warnings> warnings)
Scan ID:        <scanId from response>

Dashboard: https://app.behavioral-contracts.com/repositories/<REPO_ID>
```

If errors exist, show the top 3:
```
  [ERROR] <package>.<function> — <message>
          <file>:<line>
```

If zero violations: "No violations found. All behavioral contracts satisfied."

### Step 8 — Cleanup

```bash
rm -f /tmp/bc-scan-results.json /tmp/bc-upload-payload.json
```

---

## Fix Mode

If invoked as `/bc-scan --fix`:

After Step 7, ask: "Fix the top ERROR violations? (yes/no)"

If yes, for each ERROR violation (top 3 max):
- Read the file at `violation.file`, locate `violation.line`
- Wrap the call in a try-catch with appropriate error handling based on `message` and `postconditionId`
- After all fixes, say: "Re-run `/bc-scan` to confirm violations are resolved."

---

## Reference Docs

- **First-time setup:** read `~/.claude/skills/bc-scan/docs/setup.md`
- **Auto-scan on commit:** read `~/.claude/skills/bc-scan/docs/hooks.md`
- **Error help:** read `~/.claude/skills/bc-scan/docs/troubleshooting.md`
