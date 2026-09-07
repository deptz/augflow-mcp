# Connecting Claude Code CLI to Augflow MCP

Claude Code speaks MCP over stdio and reads server definitions from a top-level
`mcpServers` object in `~/.claude.json`.

## Option A — let Augflow install it for you

In the Augflow web UI: **Settings → MCP → augflow → Install for Claude Code**.
This writes the entry below into `~/.claude.json` automatically. If **Auto-install
augflow MCP** is enabled in Settings, Augflow also does this the first time you
open an ad-hoc **project-workspace terminal** (New Agent Terminal with no repos
selected) — not when a card session starts.

## Option B — configure it by hand

Add this to the top-level `mcpServers` object in `~/.claude.json`:

```json
{
  "mcpServers": {
    "augflow": {
      "type": "stdio",
      "command": "augflow",
      "args": ["mcp"]
    }
  }
}
```

See [examples/claude-code.json](../../examples/claude-code.json) for the full
file shape if `~/.claude.json` doesn't exist yet.

## Project scope

`augflow mcp` scopes its tools to whichever project it resolves at startup — by
default, the directory Claude Code launches the process from. If you're running
Claude Code somewhere that isn't a linked project root or an active card
session, set `AUGFLOW_PROJECT_PATH` explicitly, either in your shell or as an
`env` block alongside `command`/`args` above:

```json
"env": { "AUGFLOW_PROJECT_PATH": "/absolute/path/to/your/project" }
```

See [docs/transports-and-scoping.md](../transports-and-scoping.md) for the full
resolution order.

## Verify

1. Restart Claude Code (or reload MCP servers if your version supports it).
2. Ask it to list your Augflow cards, or check its MCP server status — you
   should see tools like `card_list`, `card_show`, `workspace_start`, etc.
3. If nothing shows up, see [docs/troubleshooting.md](../troubleshooting.md).

## No authentication required

`augflow mcp` has no token/API-key check — it trusts the local process and
scopes access by project path only. Don't expose it over anything other than
localhost.
