# bc-scan — Behavioral Contracts Claude Code Skill

> **READ-ONLY.** This skill never modifies your code or creates commits. It only reads your codebase, runs an analysis, and uploads results. Safe to run at any time.

A Claude Code skill that scans your TypeScript codebase for behavioral contract violations and uploads results to the [Behavioral Contracts](https://app.behavioral-contracts.com) dashboard — all without leaving your editor.

## What it does

- Finds your `tsconfig.json` automatically
- Runs `@behavioral-contracts/verify-cli` locally
- Detects your repository from `git remote` — no manual ID needed
- Reads your API key from your existing MCP config — no extra env vars needed
- Uploads scan results and shows a summary with a dashboard link

## Want to fix violations, not just find them?

Use **[bc-fix](https://github.com/behavioral-contracts/claude-code-skill-bc-fix)** — the companion skill that reads the full violation list, proposes a DRY fix plan, gets your approval, then applies fixes in batches with atomic commits.

```bash
gh repo clone behavioral-contracts/claude-code-skill-bc-fix ~/.claude/skills/bc-fix
```

Use `bc-scan` when you want to **audit** your codebase or run a check in CI.
Use `bc-fix` when you want to **remediate** violations and have Claude write the fixes.

`bc-fix` always runs a `bc-scan` internally as its first step, so you never need to run both manually.

## Install

```bash
gh repo clone behavioral-contracts/claude-code-skill-bc-scan ~/.claude/skills/bc-scan
```

Or manually:

```bash
git clone https://github.com/behavioral-contracts/claude-code-skill-bc-scan ~/.claude/skills/bc-scan
```

## Prerequisites

1. **A Behavioral Contracts account** — [app.behavioral-contracts.com](https://app.behavioral-contracts.com) (Solo plan or higher for API access)
2. **MCP configured** — Follow the setup guide at [app.behavioral-contracts.com/developer](https://app.behavioral-contracts.com/developer) or see [docs/setup.md](docs/setup.md)
3. **The repository connected** — The GitHub repo you're scanning must be connected in your Behavioral Contracts dashboard

Once MCP is configured, no additional environment variables are required. The skill reads your API key from your existing MCP config automatically.

## Usage

In any Claude Code session, type:

```
/bc-scan
```

Or say: **"bc scan"**, **"run behavioral contracts scan"**, **"behavioral contracts scan"**

### Options

| Command | Description |
|---------|-------------|
| `/bc-scan` | Run scan and upload results |
| `/bc-scan --fix` | Run scan, then offer to fix top violations |
| `/bc-scan --tsconfig ./apps/api/tsconfig.json` | Specify tsconfig path explicitly |

## How it finds your API key

The skill checks in this order:
1. `BC_API_KEY` environment variable (if set)
2. `~/.claude.json` → `mcpServers.behavioral-contracts.headers.Authorization` (Claude Code CLI)
3. `~/.claude/claude_desktop_config.json` → same path (Claude Desktop)

If MCP is already configured, you're already set. No extra setup needed.

## How it finds your repository

The skill runs `git remote get-url origin`, normalizes the GitHub URL to `owner/repo` format, then calls `list_repositories` to find the matching entry in your dashboard. No manual repo ID required.

## Docs

- [docs/setup.md](docs/setup.md) — First-time MCP setup
- [docs/hooks.md](docs/hooks.md) — Auto-scan after every git commit
- [docs/troubleshooting.md](docs/troubleshooting.md) — Common errors

## Contributing

Issues and PRs welcome at [github.com/behavioral-contracts/claude-code-skill-bc-scan](https://github.com/behavioral-contracts/claude-code-skill-bc-scan).
