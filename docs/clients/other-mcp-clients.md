# Connecting OpenClaw or any other MCP-compatible client

Augflow ships built-in installer templates for five clients (Claude Code,
Cursor, GitHub Copilot CLI, Gemini CLI, OpenCode — see
[the client guides](../../README.md#supported-clients)). For everything else,
including **OpenClaw**, there is no built-in Augflow adapter today.

**ASSUMPTION:** the config below follows the standard MCP stdio-server shape
that Claude Code, Copilot, Gemini, and Cursor all use. This has not been
verified against OpenClaw specifically — no OpenClaw setup instructions,
config-file path, or JSON/YAML schema were found anywhere in the `augflow`
repository. Confirm the exact key names and config file location against
OpenClaw's own MCP documentation before relying on this.

## Generic stdio config

Most MCP clients accept some variant of:

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

Adjust the wrapper keys (`mcpServers` vs. something else) and the exact field
names (`command`/`args` vs. a combined array, an `enabled` flag, etc.) to match
your client's schema — see the [OpenCode guide](opencode.md) for an example of
a client that uses a different shape (`"type": "local"`, `command` as an
array) from the same underlying server.

## If your client prefers network transport over spawning a subprocess

`augflow mcp` also supports streamable HTTP, and the same tool surface is
mounted under a running `augflow serve` process:

```bash
# Standalone HTTP server
augflow mcp --http --host 127.0.0.1 --port 4401

# Or, if `augflow serve` is already running, hit its mount directly:
# http://localhost:4400/api/mcp   (header: X-Project-Path: /path/to/project)
```

Both bind to `127.0.0.1` by default. Standalone `--http` carries no
authentication at all; `/api/mcp` inherits `serve`'s auth (effectively none
for an unconfigured loopback `api_token`, enforced if you've set one) but is
always blocked for remote/tunneled devices regardless. Localhost binding is
the real boundary — do not expose either beyond it. See
[docs/transports-and-scoping.md](../transports-and-scoping.md) for the full
picture, including why `/api/mcp` is explicitly blocked for remote/tunneled
Augflow Anywhere devices.

## What to check if it doesn't work

1. Confirm `augflow --version` runs from the same environment your client
   launches subprocesses in (PATH issues are the most common failure).
2. Confirm your client's config schema — many MCP clients silently ignore an
   entry with the wrong shape rather than erroring.
3. See [docs/troubleshooting.md](../troubleshooting.md) for project-scoping
   and empty-result issues once the connection itself is established.

If you get OpenClaw working, please update this page (and remove the
ASSUMPTION note above) with the verified config shape and file location so the
next person doesn't have to re-derive it.
