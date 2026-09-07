# Connecting Cursor to Augflow MCP

Cursor's CLI agent reads MCP server definitions from `mcp.json` — either
project-scoped (`<project>/.cursor/mcp.json`, preferred when Cursor is running
inside an Augflow card's worktree) or user-scoped (`~/.cursor/mcp.json`).

## Option A — let Augflow install it for you

Augflow web UI: **Settings → MCP → augflow → Install for Cursor**. When run from
inside a card's worktree, this writes to that worktree's `.cursor/mcp.json`;
otherwise it writes to `~/.cursor/mcp.json`.

## Option B — configure it by hand

```json
{
  "mcpServers": {
    "augflow": {
      "command": "augflow",
      "args": ["mcp"]
    }
  }
}
```

See [examples/cursor.json](../../examples/cursor.json).

## Project scope

Same rule as every other client here: `augflow mcp` scopes to the directory it's
launched from (or `AUGFLOW_PROJECT_PATH` if set). Since Cursor typically runs
with its cwd already inside the target project/worktree, you usually don't need
to set anything extra. See
[docs/transports-and-scoping.md](../transports-and-scoping.md) if tools return
empty results.

## Verify

Open Cursor's MCP/tools panel and confirm an `augflow` server with tools like
`card_list`, `card_diff`, `workspace_plan` is connected. If not, see
[docs/troubleshooting.md](../troubleshooting.md).

## No authentication required

`augflow mcp` has no token/API-key check — it trusts the local process and
scopes access by project path only.
