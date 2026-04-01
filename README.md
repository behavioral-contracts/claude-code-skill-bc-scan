# bc-scan — Behavioral Contracts Claude Code Skill

> **⚠️ DEPRECATED** — This skill is no longer the recommended way to trigger scans. See [below](#migration) for alternatives.

---

## Migration

**bc-scan is superseded by direct MCP usage.** Scans are now triggered via:

- **Dashboard** — hit the Rescan button at [app.behavioral-contracts.com](https://app.behavioral-contracts.com)
- **Push webhook** — scans run automatically on every push to connected repositories
- **MCP tool** — ask Claude to `trigger_scan` directly via the behavioral-contracts MCP server

If you have MCP configured, Claude can scan your repo natively without this skill:

```
Trigger a scan for my repository, wait for it to complete, and show me all ERROR violations.
```

For the full setup guide, see [app.behavioral-contracts.com/developer](https://app.behavioral-contracts.com/developer).

---

## Want to fix violations?

**[bc-fix](https://github.com/behavioral-contracts/claude-code-skill-bc-fix)** is still actively maintained and handles the full scan-and-fix workflow. It does not depend on bc-scan.

```bash
gh repo clone behavioral-contracts/claude-code-skill-bc-fix ~/.claude/skills/bc-fix
```

---

## Status

This repository is **archived**. Existing installs will continue to work — the skill calls the cloud API so it won't break as long as your MCP config is valid. But it won't receive updates or bug fixes.

---

## Original documentation

<details>
<summary>Show original README</summary>

A Claude Code skill that scans your TypeScript codebase for behavioral contract violations and uploads results to the [Behavioral Contracts](https://app.behavioral-contracts.com) dashboard — all without leaving your editor.

### What it does

- Finds your `tsconfig.json` automatically
- Triggers a cloud scan via the Behavioral Contracts API
- Detects your repository from `git remote` — no manual ID needed
- Reads your API key from your existing MCP config — no extra env vars needed
- Shows a summary with a dashboard link

### Usage

```
/bc-scan
```

Or say: **"bc scan"**, **"run behavioral contracts scan"**

### Options

| Command | Description |
|---------|-------------|
| `/bc-scan` | Trigger scan and show results |
| `/bc-scan --tsconfig ./apps/api/tsconfig.json` | Specify tsconfig path explicitly |

### Prerequisites

1. **A Behavioral Contracts account** — [app.behavioral-contracts.com](https://app.behavioral-contracts.com) (Solo plan or higher for API access)
2. **MCP configured** — Follow the setup guide at [app.behavioral-contracts.com/developer](https://app.behavioral-contracts.com/developer)
3. **The repository connected** — The GitHub repo must be connected in your dashboard

### Docs

- [docs/setup.md](docs/setup.md) — First-time MCP setup
- [docs/troubleshooting.md](docs/troubleshooting.md) — Common errors

</details>
