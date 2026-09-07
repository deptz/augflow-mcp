# AGENTS.md

Dense, machine-readable summary for an AI agent configuring itself to connect
to Augflow MCP. Prose guides for humans live under `docs/`; this file is the
fast path.

## Facts

- Binary: `augflow`, must be on `PATH`. Check: `augflow --version`.
- Two independent MCP servers exist. Pick the right one:
  - `augflow mcp` — general purpose, ~95 tools, no auth, stdio or `--http`.
  - `augflow hermes-direct serve` — Hermes-only, ~12 tools, token + allowlist auth, stdio only.
- Default project scope = launch directory, unless `AUGFLOW_PROJECT_PATH` is set. See "Project scoping" below.
- No config values in this repo are fabricated. Anything not directly sourced from the `augflow` repo is explicitly marked ASSUMPTION or unverified (see `docs/clients/other-mcp-clients.md`).

## Minimal stdio connection (most clients)

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

Variants by client (exact key names differ — see `docs/clients/<name>.md`):

| Client | Config file | Notes |
|---|---|---|
| claude-code | `~/.claude.json` | add `"type": "stdio"` |
| cursor | `.cursor/mcp.json` (project or `~`) | as shown above |
| github-copilot | `~/.copilot/mcp-config.json` | as shown above |
| gemini-cli | `~/.gemini/settings.json` | as shown above |
| opencode | `~/.config/opencode/opencode.json` | different shape: top-level `"mcp"` key (not `mcpServers`), `"type":"local"`, `"command":["augflow","mcp"]`, `"enabled":true` |
| hermes | `~/.hermes/config.yaml` under `mcp_servers.augflow` | different server entirely: `command: augflow`, `args: ["hermes-direct","serve"]`, requires `AUGFLOW_HERMES_DIRECT_TOKEN` env — see `docs/clients/hermes.md`, do not skip the token/allowlist steps |
| anything else (incl. openclaw) | client-specific, unverified | adapt the generic shape above; confirm against the client's own MCP docs first |

## HTTP alternative

```
augflow mcp --http --host 127.0.0.1 --port 4401
```
or, while `augflow serve` runs: `http://localhost:4400/api/mcp` with header
`X-Project-Path: <absolute path>`. Standalone `--http` has no auth at all;
`/api/mcp` inherits `serve`'s auth (effectively none for an unconfigured
loopback `api_token`, enforced if you set one) but is always blocked for
remote/tunneled devices. Localhost binding is the real boundary either way —
never expose beyond it.

## Project scoping resolution order (stdio servers)

1. `AUGFLOW_PROJECT_PATH` env var
2. `.augflow/project` link file in cwd (not parent dirs)
3. literal cwd absolute path as legacy fallback key

Empty tool results with no error == almost always a scoping miss, not a
connection failure. Set `AUGFLOW_PROJECT_PATH` explicitly if unsure.

## Hermes-specific setup (do in order)

```bash
augflow hermes-direct token --enable          # 1. get token, enables direct mode
augflow hermes-direct channel add \           # 2. allowlist identity (start least-privilege)
  --platform <p> --chat-id <id> --user-id <id> \
  --permissions read,capture,summary
augflow hermes-direct doctor                  # 3. verify (needs tmux too)
```
Then configure Hermes with `command: augflow`, `args: ["hermes-direct","serve"]`,
`env.AUGFLOW_HERMES_DIRECT_TOKEN=<token>`, and `chmod 600` the config file if
you wrote it by hand (the Quick-setup wizard does this automatically).

Full mutating permission set (`bind,switch,prompt_queue,stop` added) also
requires the matching global switches (`allow_prompt_queue`, `allow_stop_agent`)
to be enabled in `~/.augflow/config.yaml` — off by default even if allowlisted.

## Tool catalog

Full lists: `docs/tools-reference.md`. Do not assume a tool exists beyond what
is listed there or returned by the client's own `tools/list` call at runtime —
the catalog grows over time. This repo is a guide, not the source — if any
command/flag/config-key here seems wrong, prefer `augflow mcp --help` /
`augflow hermes-direct --help` or a live `tools/list` call over this file.
