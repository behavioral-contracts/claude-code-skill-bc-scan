# bc-scan — Troubleshooting

## API key not found

The skill checks `BC_API_KEY` env var, then `~/.claude.json`, then `~/.claude/claude_desktop_config.json`.

**Fix:** Either set `export BC_API_KEY=your_key` or configure the MCP server — see `docs/setup.md`.

## 401 Unauthorized

Your API key is invalid or expired.

**Fix:** Regenerate at [Settings → API Keys](https://app.behavioral-contracts.com/settings?tab=api-keys). Update your MCP config or env var.

## 403 Plan restriction

The MCP upload endpoint requires Solo plan or higher.

**Fix:** Upgrade at [Settings → Billing](https://app.behavioral-contracts.com/settings?tab=billing).

## Repository not found

The skill calls `list_repositories` and matches against your `git remote get-url origin`. No match means the repo isn't connected.

**Fix:**
1. Go to [Repositories](https://app.behavioral-contracts.com/repositories) → Connect Repository
2. Make sure the GitHub remote matches what's connected (`git remote get-url origin`)

If you want to override auto-detection, set `BC_REPO_ID` to the cuid from the dashboard URL:
```
https://app.behavioral-contracts.com/repositories/cmmkp5imy0008h1pp9r1wjt5y
                                                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                                   this is your BC_REPO_ID
```

## verify-cli not found

```bash
npm install -g @behavioral-contracts/verify-cli
```

Then use `behavioral-contracts` instead of `npx @behavioral-contracts/verify-cli`.

## No tsconfig.json found

Specify the path explicitly:

```
/bc-scan --tsconfig ./apps/api/tsconfig.json
```

## Scan runs but shows 0 violations unexpectedly

Check the corpus has contracts for your dependencies:

```bash
npx @behavioral-contracts/verify-cli --help
```

Confirm the version is 2.x or later (V2 analyzer is default). Also check that the tsconfig includes the source files you expect to be scanned (not just config files).

## npm EPERM error / root-owned cache files

You may see errors like:

```
npm error code EPERM
npm error syscall open
npm error path /Users/.../.npm/_cacache/tmp/...
npm error Your cache folder contains root-owned files, due to a bug in
npm error previous versions of npm which has since been addressed.
npm error To permanently fix this problem, please run:
npm error   sudo chown -R 501:20 "/Users/.../.npm"
```

**What it means:** The npm cache has files owned by root (from a previous `sudo npm` invocation). This is a macOS system issue, not a bc-scan bug.

**Impact:** The scan still completes correctly — the output file is created and results are uploaded. The error is cosmetic. `CLI_VERSION` will show as `"unknown"` in the scan summary.

**Permanent fix** (run once in your terminal):
```bash
sudo chown -R $(id -u):$(id -g) ~/.npm
```

After running this, npm and npx commands will work without the EPERM error.

## Upload succeeds but scan doesn't appear in dashboard

1. Confirm the `repositoryId` in the payload matches your repo's cuid (not the GitHub repo ID)
2. Check the scan appears under the repo's **Scans** tab — it may take a few seconds to process
3. If it still doesn't appear, check the `scanId` returned in the upload response
