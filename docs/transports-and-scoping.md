# Transports, project scoping, and auth model

## Transports

`augflow mcp` supports two transports from one binary:

| Transport | How to start | Typical use |
|---|---|---|
| stdio (default) | `augflow mcp` | Your MCP client spawns this as a subprocess — this is what every client guide in this repo uses |
| Streamable HTTP | `augflow mcp --http [--host 127.0.0.1] [--port 4401]` | A client that wants to talk to a long-lived network endpoint instead of spawning a process |
| HTTP mount under `augflow serve` | n/a — automatic while `augflow serve` runs | Same tool surface as `--http`, at `http://localhost:<serve-port>/api/mcp` (serve's default port is 4400) |

The HTTP transport requires an `X-Project-Path` header (or `?project=` query
param) to resolve project scope — same mechanism the REST API uses.

**Auth: `augflow mcp` (stdio or standalone `--http`) has no authentication at
all** — it trusts whatever can reach `127.0.0.1` (its default bind host). The
`/api/mcp` mount under `augflow serve` is different: it sits behind `serve`'s
general session/local-admin auth, which for a loopback caller with no
`api_token` configured (the common local-dev default) treats the request as
the trusted local operator — i.e. still effectively no credential check for a
same-machine caller, but if you *have* configured `serve`'s `api_token`, that
check applies here too. Either way, `/api/mcp` is explicitly excluded from
Augflow Anywhere / tunnel remote-device access — it only works for local/CLI
callers, never a remote approved device, regardless of `api_token`.

**None of this is a substitute for network isolation.** Do not put any of
these transports behind a public listener, reverse tunnel, or port-forward —
localhost-only binding is the actual security boundary here, not a login
check.

`augflow hermes-direct serve` only supports stdio — there is no HTTP variant,
by design (no network listener at all).

## Project scoping

Every MCP tool call needs a resolved project to operate on. For `augflow mcp`
(stdio) and `augflow hermes-direct serve`, resolution order is:

1. `AUGFLOW_PROJECT_PATH` environment variable, if set.
2. A `.augflow/project` link file in the **current directory only** (no
   parent-directory search) — its contents are the project key, written by
   `augflow project link <key>`.
3. The literal absolute current-directory path, as a legacy fallback key —
   this only matches if some existing card/task's `project_path` happens to
   equal that exact string.

**If none of these resolve to a project with actual data, tools like
`card_list`/`task_list` return an empty result with no error** — they do not
fail loudly. An empty tool response is very often a scoping problem, not a
broken connection. See [docs/troubleshooting.md](troubleshooting.md).

Augflow auto-sets `AUGFLOW_PROJECT_PATH` for you in contexts it controls
directly — inside a card's tmux session, the browser floating terminal, and a
configured Hermes-direct binding. You only need to set it by hand when running
`augflow mcp` yourself from a directory that isn't a linked project root or an
active card session.

For the HTTP transport, project scope comes from the `X-Project-Path` header
(or `?project=`) on each request instead of the environment/link-file chain
above.

## Auth model summary

| Server | Auth |
|---|---|
| `augflow mcp` (stdio or `--http`) | None — local-process/localhost trust only |
| `/api/mcp` under `augflow serve` | Inherits `serve`'s session/local-admin auth — effectively no check for an unconfigured loopback `api_token` (the common default), enforced if `api_token` is set; always blocked for remote/tunneled devices either way |
| `augflow hermes-direct serve` | Required — `AUGFLOW_HERMES_DIRECT_TOKEN` (SHA-256-hashed at rest) plus a per-channel/user allowlist and per-command permission classes |
