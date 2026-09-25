# Transports, project scoping, and auth model

## Transports

`augflow mcp` supports two transports from one binary:

| Transport | How to start | Typical use |
|---|---|---|
| stdio (default) | `augflow mcp` | Your MCP client spawns this as a subprocess — this is what every client guide in this repo uses |
| Streamable HTTP | `augflow mcp --http [--host 127.0.0.1] [--port 4401]` | A client that wants to talk to a long-lived network endpoint instead of spawning a process |
| HTTP mount under `augflow serve` | n/a — automatic while `augflow serve` runs | Same tool surface as `--http`, at `http://localhost:<serve-port>/api/mcp` (serve's default port is 4400) |
| `/api/mcp-remote` under `augflow serve` | n/a — automatic while `augflow serve` runs | A curated, tool-scoped subset intended for an already-paired, approved Augflow Anywhere remote device (reached through the tunnel); local/admin callers can also reach it, always limited to the `allowed_tools` subset |

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

**`/api/mcp-remote` is a separate, tool-scoped route**, intended for an
already-paired, approved Augflow Anywhere remote device reaching it through
the tunnel — but it is mounted with no device-only wrapper, so a local/admin
caller can reach it too, always limited to the `allowed_tools` subset below.
It is additive: `/api/mcp` itself is unchanged and is never reachable by a
paired remote device. Which tools are reachable is controlled by
`remote_access.mcp` in `~/.augflow/config.yaml` (hand-edit only, no Settings
UI yet): `enabled` defaults to true when unset (a remote device additionally
needs to be paired and approved to reach it), and
`allowed_tools`, unless set, defaults to a curated, non-destructive set (not
strictly read-only; some reads do internal bookkeeping) — see the
"Remote-device subset" section of
[docs/tools-reference.md](tools-reference.md) — that excludes anything that
commits, pushes, deletes, creates/updates a PR, or starts/retries a cloud job.
Setting `allowed_tools: []` is a deliberate deny-all, distinct from leaving it
unset. Disabling the route entirely (`enabled: false`, or an empty
`allowed_tools`) is re-checked on every request and takes effect on the next
request, provided `config.yaml` still exists, loads/parses, and — if the
server started with any auth configured (`api_token`, Google auth, or a
session secret) — still has at least one of them. If any of those fails, the
server silently keeps the setting it started with. Otherwise this is an
operator kill switch that reaches even an already-open session. Narrowing
`allowed_tools` to a smaller non-empty list does **not** retroactively affect
a session that's already open; it keeps the tool set it was bound to until it
closes or the process restarts. The same applies to project scope: an open
session stays bound to the project it started with, even if the device's own
project scope is narrowed afterward. Only a full disable (`enabled: false` or
`allowed_tools: []`) cuts off an open session. A disallowed tool simply
doesn't appear in that caller's `tools/list`. How a remote client presents
its device session to this route is **unverified** — not documented here.

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
above. `/api/mcp-remote` has **no fallback project** — unlike `/api/mcp`,
which falls back to whatever project `augflow serve` is running against, a
request to `/api/mcp-remote` with no `X-Project-Path` (and no `?project=`) is
refused rather than silently resolving to that fallback; a whitespace-only
header is treated as absent too.

Every card/task ID lookup (`card_show`, `task_show`, `deps_list`/`deps_add`,
`qa_result`, `qa_list_runs`, `workspace_plan`, `workspace_implement`, and
more) is project-scoped for every caller, including local stdio: an ID that
belongs to a different project than the one this call resolved to returns
not-found, not that other project's data. If a tool reports not-found for an
ID you know exists, check which project this server/request actually
resolved to before assuming the ID is wrong — this is the same class of
problem as the empty-result case above, just surfacing as an error instead of
an empty list.

## Auth model summary

| Server | Auth |
|---|---|
| `augflow mcp` (stdio or `--http`) | None — local-process/localhost trust only |
| `/api/mcp` under `augflow serve` | Inherits `serve`'s session/local-admin auth — effectively no check for an unconfigured loopback `api_token` (the common default), enforced if `api_token` is set; always blocked for remote/tunneled devices either way |
| `/api/mcp-remote` under `augflow serve` | Intended for an already-paired, approved Augflow Anywhere remote device (governed by the tunnel + pairing/approval flow, no new credential); local/admin callers can also reach it — either way, always gated by `remote_access.mcp.enabled`/`allowed_tools`; a full disable is re-checked on every request, narrowing `allowed_tools` only applies to new sessions |
| `augflow hermes-direct serve` | Required — `AUGFLOW_HERMES_DIRECT_TOKEN` (SHA-256-hashed at rest) plus a per-channel/user allowlist and per-command permission classes |
