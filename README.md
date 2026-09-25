# augflow-mcp

A connection guide for **Augflow's Model Context Protocol (MCP) servers** — how to
point an AI coding agent or chat tool at your local Augflow instance so it can see
your Kanban cards, run workspace/plan/implement flows, drive Hermes chat sessions,
and more.

This repo does not contain Augflow itself. It documents how to *connect to* the
MCP servers that ship inside the `augflow` binary. Augflow must already be
installed for any of this to work — see [Prerequisites](#prerequisites).

## There are two different MCP servers

Augflow exposes **two separate MCP servers** with different transports, tool
surfaces, and trust models. Using the wrong one for your client is the most common
setup mistake — read this table before picking a guide below.

| | `augflow mcp` | `augflow hermes-direct serve` |
|---|---|---|
| Who it's for | IDE/CLI coding agents: Claude Code, Cursor, GitHub Copilot CLI, Gemini CLI, OpenCode, and other generic MCP clients | The [Hermes Agent](https://hermes-agent.nousresearch.com) chat connector specifically |
| Transport | stdio (default) or streamable HTTP (`--http`) | stdio only |
| Tool surface | 94 tools: card lifecycle, delivery, workspace, QA, runs, tasks, PR review, scheduling, peer handoff, etc. | 13 purpose-built tools scoped to driving one card session from chat: `augflow_status`, `augflow_cards_list`, `augflow_bind_channel`, `augflow_prompt_queue`, etc. |
| Auth | None — relies on local process/localhost trust and project-path scoping (a separate, tool-scoped route is intended for an already-paired, approved remote device, though local/admin callers can reach it too — see [docs/transports-and-scoping.md](docs/transports-and-scoping.md)) | Required — `AUGFLOW_HERMES_DIRECT_TOKEN`, channel allowlist, per-command permissions |
| Enabled by default | No (opt-in auto-install for CLI agents; always available to run manually) | No — disabled until you explicitly enable direct mode |
| Guide | See [client guides](#supported-clients) below | [docs/clients/hermes.md](docs/clients/hermes.md) |

## Prerequisites

1. Augflow is installed and on your `PATH` (`augflow --version` should print a
   version) — see your organization's Augflow installation instructions if you
   don't have it yet. This repo does not re-document installing Augflow itself.
2. `tmux` is installed and available (required for Augflow to drive coding-agent
   sessions; `augflow doctor`/`augflow hermes-direct doctor` will flag it if missing).
3. You have a project Augflow already knows about (linked via `augflow project
   link <key>`, or you'll run the MCP server from inside that project directory).

## Supported clients

| Client | Config file | Guide |
|---|---|---|
| Claude Code CLI | `~/.claude.json` | [docs/clients/claude-code.md](docs/clients/claude-code.md) |
| Cursor (CLI/agent) | `<project>/.cursor/mcp.json` or `~/.cursor/mcp.json` | [docs/clients/cursor.md](docs/clients/cursor.md) |
| GitHub Copilot CLI | `~/.copilot/mcp-config.json` | [docs/clients/github-copilot.md](docs/clients/github-copilot.md) |
| Gemini CLI | `~/.gemini/settings.json` | [docs/clients/gemini-cli.md](docs/clients/gemini-cli.md) |
| OpenCode CLI | `~/.config/opencode/opencode.json` | [docs/clients/opencode.md](docs/clients/opencode.md) |
| Hermes Agent | `~/.hermes/config.yaml` | [docs/clients/hermes.md](docs/clients/hermes.md) |
| OpenClaw / any other MCP client | varies by client | [docs/clients/other-mcp-clients.md](docs/clients/other-mcp-clients.md) — **unverified**, no built-in Augflow adapter exists for this client today; see that page for the generic pattern and what's assumption vs. fact |

For Claude Code, Cursor, Copilot, and Gemini, Augflow can also write its own MCP
config entry for you (Settings → MCP in the Augflow web UI, or the
**Auto-install augflow MCP** toggle) — see each client's guide for when that
applies versus doing it by hand.

## Reference docs

- [docs/tools-reference.md](docs/tools-reference.md) — what each server can do
- [docs/transports-and-scoping.md](docs/transports-and-scoping.md) — stdio vs. HTTP, project scoping, auth model
- [docs/troubleshooting.md](docs/troubleshooting.md) — common connection failures
- [AGENTS.md](AGENTS.md) — dense, machine-readable summary of everything in this repo, for an agent configuring itself

## A note on accuracy

This guide reflects Augflow v0.1.6. The tool surface especially will grow over
time — run `augflow mcp --help` / `augflow hermes-direct --help`, or ask your
MCP client to list tools live, for the current, authoritative reference if
something here looks stale.
