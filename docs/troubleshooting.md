# Troubleshooting

## "My client doesn't show an `augflow` server at all"

- Confirm `augflow --version` works from the **same environment your client
  spawns subprocesses in** (a shell PATH that's correct in your terminal but
  not in your client's launch environment is the most common cause).
- Confirm the config file is the one your client actually reads — see the
  table in [README.md](../README.md#supported-clients) for exact paths.
- Restart the client after editing config — most MCP clients only read server
  definitions at startup.
- Run `augflow mcp` by hand in a terminal. If it errors immediately, fix that
  first — your client will fail the same way silently.

## "Tools show up, but every call returns empty (no cards, no tasks)"

This is almost always a **project-scoping** issue, not a broken connection —
`augflow mcp` returns an empty result with no error when it can't resolve a
project with data. Check:

1. Is `AUGFLOW_PROJECT_PATH` set, and does it point at a project Augflow
   already knows about?
2. If not set, is the directory your client launched `augflow mcp` from, or
   one of its parent directories, linked (`.augflow/project`) or a project
   registered with Augflow? Parent directories are searched since v0.1.8.
3. On Linux and macOS, a parent directory's key file is ignored if that
   directory, its `.augflow/` folder or the file is owned by another user or
   writable by other users or a shared group. Fix the permissions, or set
   `AUGFLOW_PROJECT_PATH`.
4. Run `augflow project link <key>` from the project root if it isn't linked
   yet.

Two things that look like an empty result but aren't scoping problems:

- `task_list` returns at most 100 tasks per call (max 500 with `limit`); page
  with `offset` for more.
- Since v0.1.8, a mistyped `status` filter on `card_list` (or Hermes
  `augflow_cards_list`) returns an error listing the valid values instead of
  an empty list.

See [docs/transports-and-scoping.md](transports-and-scoping.md#project-scoping)
for the full resolution order.

## "A tool says a card/task is not found, but the ID exists"

This is the same class of problem as the empty-result case above, from the
other side: card/task ID lookups (`card_show`, `task_show`,
`deps_list`/`deps_add`, `qa_result`, `qa_list_runs`, `workspace_plan`,
`workspace_implement`, and more) are project-scoped for every caller,
including local stdio. An ID that belongs to a different project than the
one this call resolved to comes back not-found, not that other project's
data. Check which project the server actually resolved to (see
[docs/transports-and-scoping.md](transports-and-scoping.md#project-scoping))
before assuming the ID itself is wrong. Since v0.1.8 the not-found message
names the project the call resolved to, so the mismatch is visible in the
error itself.

A task that exists but has no card yet returns "has no card yet" from card
tools that act on an existing card. Use `task_show`, `card_show` (a read-only
virtual view of the task), or create the card with
`workspace_start`/`card_schedule`.

## "My MCP client rejects a result: 'expected record, received array'"

Augflow before v0.1.8 sent list results (`card_list`, `task_list`,
`deps_list`, and others) as a JSON array in `structuredContent`, which strict
clients such as Claude Code reject. Upgrade to Augflow v0.1.8 or later; list
results now carry the JSON only as text.

## "A start is rejected: model is not available via the 9router bridge"

With 9router routing on, starts and relaunches through MCP check the chosen
model against the live 9router catalog. Run `augflow router models` to list
valid models and combos, then set `agent.model` or pass one of those models
(or start with `router_enabled: false`).

## "A card was started via MCP, but the agent never came up / a follow-up `card_show` has empty `agent_provider` fields"

`workspace_start` (a fresh-card start only — resume launches are unchanged)
returns as soon as the card exists, while the agent may still be launching
asynchronously. Its response itself is only `{ok, card_id, branch, status}`
— it carries no `agent_provider*` fields at all — but a follow-up
`card_show`/`card_list` can show the card's session with empty
`agent_provider*` fields until that launch finishes and fills them in, which
is expected right after a start. If the `augflow mcp` stdio process is killed
before the launch finishes, it can be abandoned; the process prints `warning:
some agent launches were still running at shutdown; those cards may need a
restart` when it can't drain them in time. Retrying the start on the same
card, or running against a project also served by `augflow serve`/`augflow
mcp --http` (which reconcile in-progress cards on their own startup), is the
recovery path. Caveat: if the process was killed right after the prompt was
typed, the agent may actually be running in the card's tmux pane even though
its session looks provider-less — check the pane before retrying or relying
on reconcile, which can otherwise relaunch over the live pane (a known,
unclosed race).

## Hermes: `channel_not_allowlisted`

The error message (and `augflow_whereami`'s `how_to_allowlist` field) echoes
the exact identity and prints the precise `augflow hermes-direct channel add`
command to run — copy it into a terminal. Do not fall back to raw tmux access;
it's intentionally not exposed.

## Hermes: `augflow hermes-direct doctor` reports "not ready" even though the token and allowlist look correct

Check that **tmux is installed and available** — the connector needs it to
drive card sessions, and "ready" requires it even when auth is otherwise fine.

## Hermes: direct mode refuses to start

```
direct Hermes mode is disabled; enable hermes_direct.enabled and run `augflow hermes-direct token --enable`
```

Run `augflow hermes-direct token --enable` (see
[docs/clients/hermes.md](clients/hermes.md#1-enable-direct-mode-and-generate-a-token)).
If you already have a token but the connector still refuses to start, the
token in `$AUGFLOW_HERMES_DIRECT_TOKEN` doesn't match the stored hash — generate
a fresh one (this invalidates the old one).

## HTTP transport: `/api/mcp` returns nothing / connection refused

- Confirm `augflow serve` is actually running (the mount only exists while it
  is) — or use `augflow mcp --http` as a standalone alternative.
- You must be calling from a local/CLI context — the `/api/mcp` mount is
  explicitly blocked for remote-tunneled Augflow Anywhere devices, by design.
  A remote, already-paired/approved device must instead use the separate
  `/api/mcp-remote` route — see
  [docs/transports-and-scoping.md](transports-and-scoping.md).
- Set the `X-Project-Path` header to the project's absolute path — without
  it, `/api/mcp` uses the project `augflow serve` is running against.
  Standalone `augflow mcp --http` ignores this header and always serves the
  project it resolved at startup.

## `/api/mcp-remote` returns 403 "remote MCP access is disabled"

This exact message fires when `remote_access.mcp.enabled` is `false`, or its
effective `allowed_tools` is empty (the operator explicitly set
`allowed_tools: []`). Check `remote_access.mcp` in `~/.augflow/config.yaml`.
The same 403 is returned when `config.yaml` exists but can't be read, is
empty, or fails to parse — the route fails closed and the server log records
the failure. Fix the file and the next request picks it up. Only a missing
`config.yaml` falls back to the setting `augflow serve` started with.

## Still stuck

Cross-check against `augflow mcp --help`, `augflow hermes-direct --help`, and
`augflow doctor` / `augflow hermes-direct doctor`. This guide summarizes
Augflow's behavior as of when it was written; the CLI's own help output and
doctor checks are always the current truth.
