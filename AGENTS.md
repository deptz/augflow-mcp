# AGENTS.md

Dense, machine-readable summary for an AI agent configuring itself to connect
to Augflow MCP. Prose guides for humans live under `docs/`; this file is the
fast path.

## Facts

- Binary: `augflow`, must be on `PATH`. Check: `augflow --version`.
- Two independent MCP servers exist. Pick the right one:
  - `augflow mcp` — general purpose, 94 tools, no auth, stdio or `--http`.
  - `augflow hermes-direct serve` — Hermes-only, 13 tools, token + allowlist auth, stdio only.
- Default project scope = launch directory, unless `AUGFLOW_PROJECT_PATH` is set. See "Project scoping" below.
- Card/task ID lookups (`card_show`, `task_show`, `deps_list`/`deps_add`,
  `qa_result`, `qa_list_runs`, `workspace_plan`, `workspace_implement`,
  etc.) are project-scoped: an ID that belongs to a different project than
  the resolved scope returns not-found, not a cross-project result. If a
  specific known ID comes back not-found, check project scope before
  assuming the ID is wrong.
- No config values in this repo are fabricated. Anything not directly sourced from the `augflow` repo is explicitly marked ASSUMPTION or unverified (see `docs/clients/other-mcp-clients.md`).
- Reflects Augflow v0.1.6.

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

A separate route, `/api/mcp-remote`, is intended for an already-paired,
approved Augflow Anywhere remote device — but local/admin callers can also
reach it, always limited to the `allowed_tools` subset below. `/api/mcp`
itself is unchanged and is never reachable by a paired remote device. The
remote route exposes a curated, config-driven tool subset (`remote_access.mcp`
in `~/.augflow/config.yaml`, hand-edit only — no Settings UI yet): `enabled`
defaults to true when unset (a remote device additionally needs to be
paired and approved to reach it), `allowed_tools`
defaults to a curated, non-destructive set (not strictly read-only; some
reads do internal bookkeeping) — nothing that commits, pushes, deletes,
creates/updates a PR, or starts/retries a cloud job (`card_reopen` is not in
the default) — and an explicit `allowed_tools: []` is a deliberate deny-all.
`X-Project-Path` (or `?project=`) is mandatory on this route, with no
fallback project. See
[docs/transports-and-scoping.md](docs/transports-and-scoping.md) for the full
picture.

## Project scoping resolution order (stdio servers)

1. `AUGFLOW_PROJECT_PATH` env var
2. `.augflow/project` link file in cwd (not parent dirs)
3. literal cwd absolute path as legacy fallback key

Empty tool results with no error == almost always a scoping miss, not a
connection failure. Set `AUGFLOW_PROJECT_PATH` explicitly if unsure.
A tool call that returns a not-found error for a card/task ID you know exists
is the same underlying issue from the other side: the ID belongs to a
different project than the one this server resolved (see "Facts" above).

`workspace_start` (a fresh-card start only — resume launches are unchanged)
can return before the agent finishes booting: the launch continues on its own
goroutine, so a follow-up `card_show`/`card_list` may show the card's session
with empty `agent_provider*` fields until the launch fills them in. If the
stdio `augflow mcp` process is killed before that goroutine finishes, the
launch may be abandoned; retrying the start on the same card, or running it
against a project also served by `augflow serve`/`augflow mcp --http` (which
reconcile in-progress cards on their own startup), is the recovery path.
Caveat: if the process was killed right after the prompt was typed, the agent
may actually be running in its tmux pane even though the session looks
provider-less; check the pane before retrying or relying on reconcile, which
can otherwise relaunch over the live pane (a known, unclosed race).

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
