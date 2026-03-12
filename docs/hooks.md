# bc-scan — Auto-scan After Every Commit

Configure Claude Code to automatically run a behavioral contract scan after every `git commit`.

## Setup

Add this to your project's `.claude/settings.json` (create the file if it doesn't exist):

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "scripts/bc-post-commit.sh"
          }
        ]
      }
    ]
  }
}
```

Create `scripts/bc-post-commit.sh` in your repo:

```bash
#!/bin/bash
# Auto-upload behavioral contract scan results after git commit
# Requires: BC_API_KEY env var OR MCP config with behavioral-contracts server

# Only run when the tool input contained "git commit"
if ! echo "$CLAUDE_TOOL_INPUT" | grep -q 'git commit'; then
  exit 0
fi

# Resolve API key
API_KEY="${BC_API_KEY:-}"
if [ -z "$API_KEY" ]; then
  API_KEY=$(python3 -c "
import json, os
for f in [os.path.expanduser('~/.claude.json'), os.path.expanduser('~/.claude/claude_desktop_config.json')]:
    try:
        d = json.load(open(f))
        key = d.get('mcpServers', {}).get('behavioral-contracts', {}).get('headers', {}).get('Authorization', '')
        if key:
            print(key.replace('Bearer ', '').strip())
            break
    except:
        pass
" 2>/dev/null)
fi

if [ -z "$API_KEY" ]; then
  echo "[bc-scan] No API key found — skipping post-commit scan"
  exit 0
fi

# Find tsconfig
TSCONFIG=""
for path in "./tsconfig.json" "./apps/web/tsconfig.json"; do
  if [ -f "$path" ]; then
    TSCONFIG="$path"
    break
  fi
done

if [ -z "$TSCONFIG" ]; then
  echo "[bc-scan] No tsconfig.json found — skipping"
  exit 0
fi

# Run scan
npx @behavioral-contracts/verify-cli \
  --tsconfig "$TSCONFIG" \
  --output /tmp/bc-hook-results.json \
  --no-terminal 2>/dev/null

if [ ! -f /tmp/bc-hook-results.json ]; then
  exit 0
fi

# Resolve repo ID from git remote
REMOTE=$(git remote get-url origin 2>/dev/null | sed 's/.*github.com[:/]//' | sed 's/\.git$//')

# Build + upload payload
python3 - <<PYEOF
import json, os, subprocess, sys

data = json.load(open('/tmp/bc-hook-results.json'))
violations = [
    {
        'packageName': v['package'],
        'rule': v.get('postconditionId', ''),
        'severity': v['severity'].upper(),
        'message': v['message'],
        'filePath': v['file'],
        'lineNumber': v['line'],
        'columnNumber': v.get('column'),
        'functionName': v.get('function', ''),
        'codeSnippet': v.get('codeContext', '')
    }
    for v in data.get('violations', [])
]
summary = {
    'totalViolations': len(violations),
    'errorCount': sum(1 for v in violations if v['severity'] == 'ERROR'),
    'warningCount': sum(1 for v in violations if v['severity'] == 'WARNING'),
    'scannedFiles': data.get('summary', {}).get('filesAnalyzed', 0)
}

# Get repo ID via list_repositories
api_key = '${API_KEY}'
remote = '${REMOTE}'
repos_resp = subprocess.run([
    'curl', '-s', '-X', 'POST', 'https://app.behavioral-contracts.com/api/mcp',
    '-H', f'Authorization: Bearer {api_key}',
    '-H', 'Content-Type: application/json',
    '-d', json.dumps({'tool': 'list_repositories', 'args': {}})
], capture_output=True, text=True)

repo_id = None
try:
    repos_data = json.loads(repos_resp.stdout)
    for repo in repos_data.get('repositories', []):
        if repo.get('fullName', '') == remote:
            repo_id = repo['id']
            break
except:
    pass

if not repo_id:
    print(f'[bc-scan] Repo not found for remote: {remote} — skipping upload')
    sys.exit(0)

payload = {
    'repositoryId': repo_id,
    'violations': violations,
    'summary': summary,
    'commitSha': subprocess.run(['git', 'rev-parse', '--short', 'HEAD'], capture_output=True, text=True).stdout.strip(),
    'branch': subprocess.run(['git', 'branch', '--show-current'], capture_output=True, text=True).stdout.strip()
}

with open('/tmp/bc-hook-payload.json', 'w') as f:
    json.dump(payload, f)

result = subprocess.run([
    'curl', '-s', '-X', 'POST', 'https://app.behavioral-contracts.com/api/mcp/upload',
    '-H', f'Authorization: Bearer {api_key}',
    '-H', 'Content-Type: application/json',
    '-d', '@/tmp/bc-hook-payload.json'
], capture_output=True, text=True)

try:
    resp = json.loads(result.stdout)
    if resp.get('success'):
        errs = summary['errorCount']
        warns = summary['warningCount']
        print(f'[bc-scan] Uploaded: {errs} errors, {warns} warnings → https://app.behavioral-contracts.com/repositories/{repo_id}')
    else:
        print(f'[bc-scan] Upload failed: {resp.get("error", "unknown")}')
except:
    print('[bc-scan] Upload failed (invalid response)')

os.remove('/tmp/bc-hook-results.json')
try:
    os.remove('/tmp/bc-hook-payload.json')
except:
    pass
PYEOF
```

Make the script executable:

```bash
chmod +x scripts/bc-post-commit.sh
```

## How it works

After every Bash tool call that contains `git commit`, the hook:
1. Resolves your API key (from env var or MCP config)
2. Runs a scan
3. Auto-detects the repository from `git remote`
4. Uploads results silently in the background
5. Prints a one-line summary: `[bc-scan] Uploaded: 3 errors, 1 warning → https://...`

## Notes

- The hook runs on the **host machine**, so `BC_API_KEY` or the MCP config must be accessible in the shell
- It skips silently if no API key, no tsconfig, or no matching repo is found — never blocks a commit
- Works per-project (in `.claude/settings.json`) — add to any repo you want to track
