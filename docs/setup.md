# bc-scan — First-Time Setup

You only need to do this once.

## Step 1: Create an API key

1. Go to [Settings → API Keys](https://app.behavioral-contracts.com/settings?tab=api-keys)
2. Click **Create API Key**, name it `claude-code`, copy the key
3. Keep it secret — it grants full access to your organization's scan data

## Step 2: Add to your MCP config

This configures the Behavioral Contracts MCP server so Claude Code can call cloud tools **and** so `/bc-scan` can read your key automatically.

Find your Claude Code config file:
- **Claude Code CLI:** `~/.claude.json`
- **Claude Desktop:** `~/.claude/claude_desktop_config.json`

Add this inside the `mcpServers` object:

```json
{
  "mcpServers": {
    "behavioral-contracts": {
      "url": "https://app.behavioral-contracts.com/api/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

Replace `YOUR_API_KEY` with your key from Step 1.

## Step 3: Connect your repository

The repo you're scanning must be connected in the dashboard:

1. Go to [Repositories](https://app.behavioral-contracts.com/repositories)
2. Click **Connect Repository** and authorize via GitHub
3. Select the repo

Once connected, `/bc-scan` will auto-match it by GitHub remote URL — no manual ID needed.

## Step 4: Restart Claude Code

After saving the config, restart Claude Code (or reload the window). Test with:

```
/bc-scan
```

The skill will detect your API key from the MCP config and your repo from `git remote` automatically.

## Alternative: env var

If you prefer not to use the MCP config, set:

```bash
export BC_API_KEY=your_key_here
```

Add to `~/.zshrc` or `~/.bashrc` to persist across sessions.
