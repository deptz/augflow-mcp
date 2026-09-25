# Connecting Hermes Agent to Augflow MCP

Hermes does **not** use the general-purpose `augflow mcp` server. It uses a
separate, purpose-built connector — `augflow hermes-direct serve` — that speaks
MCP over stdio, requires a token, and only exposes 13 tools scoped to
driving one bound card session from a chat channel. This is Augflow's **direct
Hermes integration**, currently a local-first proof of concept (disabled by
default). Team Hub remains the recommended path for multi-user/production
routing — this connector is for a single trusted local user.

> **One-click alternative:** the Augflow web UI has **Settings → Direct Hermes →
> ⚡ Quick setup**, which runs the whole flow below in one wizard, including
> writing the MCP entry into Hermes' own config. If you use that, you can skip
> straight to [Verify](#4-verify).

## 1. Enable direct mode and generate a token

```bash
augflow hermes-direct token --enable
```

This prints a token **once** and stores only its SHA-256 hash in
`~/.augflow/config.yaml` (`hermes_direct.token_hash`). Copy the printed value —
running this command again **rotates** (invalidates) the previous token.

## 2. Allowlist your Hermes channel/user

No channel can bind, switch, prompt, or stop the connector until it's
explicitly allowlisted — a Hermes chat channel is never treated as
authorization on its own, and there is no wildcard allowlist.

Start least-privilege — grant only what you need:

```bash
augflow hermes-direct channel add \
  --platform telegram --chat-id <CHAT_ID> --user-id <USER_ID> \
  --permissions read,capture,summary

augflow hermes-direct channel list      # confirm it's there
```

Permission classes: `read`, `bind`, `switch`, `capture`, `prompt_queue`,
`summary`, `stop`. Mutating permissions (`bind`, `switch`, `prompt_queue`,
`stop`) require a `--user-id` — a channel-wide (bot/webhook) entry can only be
read-only.

Grant the **full command set** (every mutating class) only once you've
verified the read-only setup works and you actually need chat-driven
prompting/stop control:

```bash
augflow hermes-direct channel add \
  --platform telegram --chat-id <CHAT_ID> --user-id <USER_ID> \
  --permissions read,bind,switch,capture,prompt_queue,summary,stop
```

Mutating tools are also gated by global config switches that default to
**off**, independent of the per-channel permission above:
`hermes_direct.allow_prompt_queue`, `allow_stop_agent`, and `allow_full_debug`
must each be explicitly enabled (in `~/.augflow/config.yaml` or via the web UI)
before the corresponding tool works at all, even for an allowlisted user. An
optional `rate_limit_per_minute` further caps calls per actor.

## 3. Point Hermes at the connector

Configure Hermes' MCP client with:

```yaml
command: augflow
args: ["hermes-direct", "serve"]
env:
  AUGFLOW_HERMES_DIRECT_TOKEN: "<the token from step 1>"
  AUGFLOW_PROJECT_PATH: "/path/to/your/project"   # or use --project instead
```

See [examples/hermes.yaml](../../examples/hermes.yaml). This block is what
Augflow's Quick-setup wizard writes into `mcp_servers.augflow` in Hermes'
`~/.hermes/config.yaml` (honoring `$HERMES_HOME` if set) — see
[Hermes Agent](https://hermes-agent.nousresearch.com) for how that file is
structured on the Hermes side.

**If you write this file by hand, lock down its permissions yourself** —
it now holds a plaintext bearer token. The Quick-setup wizard does this for
you automatically (atomic write, mode `0600`, parent directory `0700`); a
manually-edited file does not get that protection for free:

```bash
chmod 600 ~/.hermes/config.yaml
```

Useful `serve` flags: `--project <path>`, `--token-env <VAR>`, `--readonly`
(force-deny all mutating tools).

**Optional hardening — restrict which project(s) the connector will ever
serve:** set `hermes_direct.allowed_projects` in `~/.augflow/config.yaml` to a
list of project paths/keys; `serve` then refuses to start for any project not
on that list. Leaving it empty (the default) means **no restriction** — any
project resolved via `--project`/`AUGFLOW_PROJECT_PATH`/`default_project` is
served. This is an operator-side control (a chat user can never change which
project is being served), but setting it is recommended defense in depth.

## 4. Verify

```bash
augflow hermes-direct doctor
```

This checks: direct mode enabled, token configured, transport is stdio, raw
tmux-send disabled, at least one channel allowlisted, tmux available, project
resolved, and the database reachable. **A "ready" result also requires tmux to
be installed** — without it the connector can't drive card sessions even if
the token and allowlist are correct.

## Tool surface

Read-only: `augflow_status`, `augflow_cards_list`, `augflow_card_show`,
`augflow_groups_list`, `augflow_context_capture`, `augflow_whereami`,
`augflow_events_poll`, `augflow_events_wait`.
Routing: `augflow_bind_channel`, `augflow_active_card_set`.
Mutating: `augflow_prompt_queue`, `augflow_summary_generate`, `augflow_stop_agent`.

The connector deliberately never exposes raw `tmux_send`, shell exec, or git
mutation tools (no commit/push/PR/branch-delete from Hermes). It also exposes an
MCP resource, `augflow://skills/hermes-direct`, that Hermes can read
(`resources/read`) to learn the tool set and safe workflow directly.

## Security notes

- Every remote command writes an audit receipt; prompt delivery is classified
  by agent state (delivered only when `needs_input`; queued if `running`;
  rejected if `completed` unless `allow_restart` is set, or if
  `stopped`/`failed`/no-session; also rejected if the pane's foreground is a
  shell/editor rather than the agent).
- Context sent to Hermes is cleaned (ANSI stripped) and redacted (secrets,
  auth URLs, local paths unless `expose_local_paths: true`) before it leaves
  your machine.
- Do not commit the plaintext token anywhere. If Hermes' Quick-setup wizard
  wrote it into `~/.hermes/config.yaml`, that file is `0600` — keep it that
  way and rotate the token (`augflow hermes-direct token`) if you suspect
  exposure.

See [docs/troubleshooting.md](../troubleshooting.md) for `channel_not_allowlisted`
and other common errors.
