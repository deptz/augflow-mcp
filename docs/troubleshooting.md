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
2. If not set, is there a `.augflow/project` link file in the directory your
   client launched `augflow mcp` from? (Not a parent directory — it only
   checks the exact cwd.)
3. Run `augflow project link <key>` from that directory if it isn't linked
   yet.

See [docs/transports-and-scoping.md](transports-and-scoping.md#project-scoping)
for the full resolution order.

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
- Confirm the `X-Project-Path` header is set — without it, project scope
  can't resolve.

## Still stuck

Cross-check against `augflow mcp --help`, `augflow hermes-direct --help`, and
`augflow doctor` / `augflow hermes-direct doctor`. This guide summarizes
Augflow's behavior as of when it was written; the CLI's own help output and
doctor checks are always the current truth.
