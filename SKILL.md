# bc-scan Skill

**Trigger:** `/bc-scan`, "bc scan", "behavioral contracts scan", "run bc scan"
**Version:** 1.2.0

Run a local behavioral contract scan and upload results to the Behavioral Contracts dashboard.

---

## Execution Workflow

### Step 1 — Resolve API key

Check in order:
1. `$BC_API_KEY` environment variable
2. `cat ~/.claude.json 2>/dev/null` → parse `projects.<cwd>.mcpServers["behavioral-contracts"].headers.Authorization` → strip `"Bearer "` prefix
3. `cat ~/.claude.json 2>/dev/null` → parse top-level `mcpServers["behavioral-contracts"].headers.Authorization`
4. `cat ~/.claude/claude_desktop_config.json 2>/dev/null` → same path

If no key found: tell the user "API key not found. Follow docs/setup.md to configure MCP, or set BC_API_KEY." Stop.

Store as `$API_KEY`.

### Step 2 — Find tsconfig.json

Check `.bc-scan` config file first:
```bash
cat .bc-scan 2>/dev/null
```
If it contains `"tsconfig": "<path>"`, use that path — skip discovery.

Otherwise check in order:
1. Argument `--tsconfig <path>` if provided
2. `./tsconfig.json`
3. `./apps/web/tsconfig.json`
4. `find . -name tsconfig.json -not -path "*/node_modules/*" -maxdepth 4`

If multiple found, list them and ask: "Which tsconfig? (enter path)"

After resolving (and if there was ambiguity), write `.bc-scan`:
```json
{ "tsconfig": "<resolved_path>" }
```
Add `.bc-scan` to `.gitignore` if not already present.

### Step 3 — Resolve repository ID

If `BC_REPO_ID` env var is set, use it directly — skip to Step 4.

Otherwise:
```bash
git remote get-url origin 2>/dev/null
```
Parse to `owner/repo` format (strip `.git`, handle SSH and HTTPS).

Call `list_repositories`:
```bash
curl -s -X POST https://app.behavioral-contracts.com/api/mcp \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"list_repositories","arguments":{}},"id":1}'
```

Find entry where `fullName` matches `owner/repo`. Use its `id` as `$REPO_ID`.

If no match: tell the user "Repository not found. Connect it at https://app.behavioral-contracts.com/repositories then try again." Stop.

### Step 4 — Get CLI version

```bash
CLI_VERSION=$(npx @behavioral-contracts/verify-cli@latest --version 2>/dev/null)
```

### Step 5 — Run verify-cli

```bash
npx @behavioral-contracts/verify-cli@latest \
  --tsconfig <tsconfig_path> \
  --output /tmp/bc-scan-results.json \
  --include-drafts \
  --no-terminal
```

`--include-drafts` ensures full coverage matching cloud scan behavior.

If the output file is not created, show the error and stop. (Exit code 1 is normal when violations exist.)

### Local vs Cloud Scan Scope

Local scans (this skill) and cloud scans (triggered from the dashboard) may find different violation counts. This is expected:

- **Local scan**: uses the `--tsconfig` you specify (e.g. `apps/web/tsconfig.json`), which covers only the files in that tsconfig's `include` pattern.
- **Cloud scan**: clones the full repository and generates a root-level permissive tsconfig covering `**/*.ts`, `**/*.tsx`, `**/*.js` across all packages.

**To get parity with the cloud scan count:**
1. Run from the monorepo root with a root-level tsconfig, OR
2. Create a root `tsconfig.scan.json` with `"include": ["**/*.ts", "**/*.tsx"]` (excluding node_modules/dist) and run with `--tsconfig ./tsconfig.scan.json`

The cloud scan typically finds more violations because it covers all packages in a monorepo, not just the web app.

### Step 6 — Parse results

Read `/tmp/bc-scan-results.json`. Extract:
- `violations[]` — each has: `file`, `line`, `column`, `package`, `function`, `postconditionId`, `severity` (lowercase), `message`, `codeContext`
- `summary.filesAnalyzed`

Compute `errorCount`, `warningCount`, `totalViolations`.

### Step 7 — Upload

Build payload and write to `/tmp/bc-upload-payload.json`:
```json
{
  "repositoryId": "$REPO_ID",
  "cliVersion": "$CLI_VERSION",
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
      "codeSnippet": "<violation.codeContext>",
      "subViolations": "<violation.subViolations mapped as: [{postconditionId, message, severity: UPPERCASE}] — omit field if empty>"
    }
  ],
  "summary": {
    "totalViolations": "<violations.length + sum(v.subViolations?.length ?? 0)>",
    "errorCount": "<count of ERROR severity across primary + sub-violations>",
    "warningCount": "<count of WARNING severity across primary + sub-violations>",
    "scannedFiles": "<summary.filesAnalyzed>"
  },
  "commitSha": "<git rev-parse --short HEAD 2>/dev/null || echo local>",
  "branch": "<git branch --show-current 2>/dev/null || echo local>"
}
```

Note: `severity` must be uppercased (`"error"` → `"ERROR"`).

Note: `totalViolations` must include sub-violations. Compute as:
`violations.length + sum(v.subViolations?.length ?? 0)`.
Similarly expand `errorCount` and `warningCount` to include sub-violation severities so the dashboard count matches the CLI output.

```bash
curl -s -X POST https://app.behavioral-contracts.com/api/mcp/upload \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/bc-upload-payload.json
```

If `success` is not `true`, show the error and stop.

### Step 8 — Show summary

```
Behavioral Contracts Scan Complete

CLI version:    <cliVersion>
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

### Step 9 — Cleanup

```bash
rm -f /tmp/bc-scan-results.json /tmp/bc-upload-payload.json
```

---

## Fix Mode

If invoked as `/bc-scan --fix`:

After Step 8, ask: "Fix the top ERROR violations? (yes/no)"

If yes, for each ERROR violation (top 3 max):
- Read `violation.file`, locate `violation.line`
- Wrap the call in a try-catch with appropriate error handling based on `message` and `postconditionId`
- After all fixes: "Re-run `/bc-scan` to confirm violations are resolved."

---

## Reference Docs

- **First-time setup:** read `~/.claude/skills/bc-scan/docs/setup.md`
- **Auto-scan on commit:** read `~/.claude/skills/bc-scan/docs/hooks.md`
- **Error help:** read `~/.claude/skills/bc-scan/docs/troubleshooting.md`
