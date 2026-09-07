# Connecting GitHub Copilot CLI to Augflow MCP

GitHub Copilot CLI reads MCP server definitions from `~/.copilot/mcp-config.json`.

## Option A — let Augflow install it for you

Augflow web UI: **Settings → MCP → augflow → Install for GitHub Copilot**.

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

See [examples/github-copilot.json](../../examples/github-copilot.json).

## Project scope

`augflow mcp` scopes to the directory it's launched from, or `AUGFLOW_PROJECT_PATH`
if set. See [docs/transports-and-scoping.md](../transports-and-scoping.md).

## Verify

Check Copilot CLI's MCP/tool list for an `augflow` server. If tools list but
calls return empty results, that's almost always a project-scoping issue, not a
connection failure — see [docs/troubleshooting.md](../troubleshooting.md).

## No authentication required

`augflow mcp` has no token/API-key check — it trusts the local process and
scopes access by project path only.
