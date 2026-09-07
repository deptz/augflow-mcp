# Connecting Gemini CLI to Augflow MCP

Gemini CLI reads MCP server definitions from `~/.gemini/settings.json`.

## Option A — let Augflow install it for you

Augflow web UI: **Settings → MCP → augflow → Install for Gemini**.

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

See [examples/gemini.json](../../examples/gemini.json).

## Project scope

`augflow mcp` scopes to the directory it's launched from, or `AUGFLOW_PROJECT_PATH`
if set. See [docs/transports-and-scoping.md](../transports-and-scoping.md).

## Verify

Check Gemini CLI's connected MCP servers for `augflow`. See
[docs/troubleshooting.md](../troubleshooting.md) if tools don't appear or
return empty results.

## No authentication required

`augflow mcp` has no token/API-key check — it trusts the local process and
scopes access by project path only.
