# bc-scan Skill

**Trigger:** `/bc-scan`, "bc scan", "behavioral contracts scan", "run bc scan"
**Version:** 1.3.0

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

### Step 1.5 — Resolve base URL

Check `.bc-scan` config file for `baseUrl`:
```bash
cat .bc-scan 2>/dev/null
```
If it contains `"baseUrl": "<url>"`, use that as `$BASE_URL`.

Otherwise fall back to `$BC_BASE_URL` env var, then default to `https://app.behavioral-contracts.com`.

Store as `$BASE_URL`. This URL is used for all API calls in subsequent steps.

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
Store as `$GITHUB_FULL_NAME`. If parsing fails (no remote, non-GitHub URL), set `$GITHUB_FULL_NAME=""`.

Call `list_repositories`:
```bash
curl -s -X POST $BASE_URL/api/mcp \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","params":{"name":"list_repositories","arguments":{}},"id":1}'
```

Find entry where `fullName` matches `owner/repo`. Use its `id` as `$REPO_ID`.

If no match, auto-create a local tracking entry:
```bash
curl -s -X POST $BASE_URL/api/mcp/repository \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"fullName\": \"owner/repo\"}"
```
Parse the response and use `id` as `$REPO_ID`. Set `REPO_CREATED=true`.

If the call fails, stop and show the error.

### Step 4 — Get CLI version

```bash
CLI_VERSION=$(npm show @behavioral-contracts/verify-cli@latest version 2>/dev/null)
```

This returns the npm package version (e.g. `2.1.3`), which is the actual version being run. Do not use `--version` — the CLI binary's self-reported version string may be stale and not match the npm package version.

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
    "scannedFiles": "<summary.files_analyzed from the CLI JSON>"
  },
  "commitSha": "<git rev-parse --short HEAD 2>/dev/null || echo local>",
  "branch": "<git branch --show-current 2>/dev/null || echo local>",
  "githubFullName": "$GITHUB_FULL_NAME"
}
```

Note: `githubFullName` is included when a GitHub remote is detected. The dashboard uses this to enable commit-status checks for MCP-uploaded repos. If no GitHub remote is found, omit the field entirely (do not send an empty string). Only include it when `$GITHUB_FULL_NAME` matches the `owner/repo` pattern (`[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+`).

Note: `severity` must be uppercased (`"error"` → `"ERROR"`).

Note: `totalViolations` must include sub-violations. Compute as:
`violations.length + sum(v.subViolations?.length ?? 0)`.
Similarly expand `errorCount` and `warningCount` to include sub-violation severities so the dashboard count matches the CLI output.

```bash
curl -s -X POST $BASE_URL/api/mcp/upload \
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

Dashboard: $BASE_URL/repositories/<REPO_ID>
```

If errors exist, show the top 3:
```
  [ERROR] <package>.<function> — <message>
          <file>:<line>
```

If zero violations: "No violations found. All behavioral contracts satisfied."

If `REPO_CREATED=true`, append after the summary:
```
→ Repository tracked locally. Connect GitHub at the dashboard URL above
  to enable cloud scans on push and PR gates.
```

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
