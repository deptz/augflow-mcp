# Connecting OpenCode CLI to Augflow MCP

OpenCode CLI reads MCP server definitions from `~/.config/opencode/opencode.json`
(it also checks `opencode.jsonc` and `config.json` in that directory — Augflow's
installer writes `opencode.json`).

## Option A — let Augflow install it for you

Augflow web UI: **Settings → MCP → augflow → Install for OpenCode**.

## Option B — configure it by hand

OpenCode uses a different shape than the other clients: a **top-level `mcp`
key** (not `mcpServers`), with a `"type": "local"` entry and `command` as an
array:

```json
{
  "mcp": {
    "augflow": {
      "type": "local",
      "command": ["augflow", "mcp"],
      "enabled": true
    }
  }
}
```

See [examples/opencode.json](../../examples/opencode.json).

## Project scope

`augflow mcp` scopes to the directory it's launched from, or `AUGFLOW_PROJECT_PATH`
if set. See [docs/transports-and-scoping.md](../transports-and-scoping.md).

## Verify

Check OpenCode's MCP server list for `augflow` and `enabled: true`. See
[docs/troubleshooting.md](../troubleshooting.md) if tools don't appear.

## No authentication required

`augflow mcp` has no token/API-key check — it trusts the local process and
scopes access by project path only.
