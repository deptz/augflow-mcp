# Tool reference

Two different tool catalogs, depending on which server you connect (see
[README.md](../README.md) for which one your client should use). Reflects
Augflow v0.1.6.

## `augflow mcp` (general-purpose, 94 tools)

Representative categories — this list will drift as Augflow adds tools; treat
it as a map of what exists, not an exact/exhaustive spec:

- **Cards** — `card_list`, `card_show`, `card_worktree_status`, `card_diff`,
  `card_move`, `card_done`, `card_reopen`, `card_reset`, `card_start_over`,
  `card_wont_do`, `card_archive`, `card_unarchive`, `card_stop_session`,
  `card_switch_provider`, `card_add_repos`, `card_validate`, `card_push`,
  `card_pr_preview`, `card_pr_generate_description`, `card_pr_create`,
  `card_pr_update`, `card_clean`, `card_delete`

  `card_reopen` moves a Done card back into In Progress in the worktree it
  already has, resuming its agent conversation where the provider supports
  it — unlike `card_start_over`, which rebuilds from the branch. It requires
  that worktree (or a parked pod's retained PVC) to still exist; otherwise use
  `card_start_over`. It does not reverse the Jira Done transition or un-release
  dependent cards that already started — those are reported back in
  `unreversed_side_effects`. `card_move`/`card move` still cannot move a Done
  card; Reopen is the only route back into In Progress (`card_start_over`
  sends it back to To Do instead).
- **Knowledge** — `knowledge_tree`, `card_knowledge_get`, `card_knowledge_add`,
  `card_knowledge_remove`, `card_knowledge_sync`
- **Workspace** — `agent_options`, `workspace_start`, `workspace_plan`,
  `workspace_implement`

  `workspace_start`, `card_move`, `card_reopen`, and `card_switch_provider` all
  take an optional `router_enabled` input — a per-start override of the global
  9router bridge toggle. When the router is enabled, `agent_options` also
  reports its live combos/models (never `base_url`/`api_key`; models capped at
  200, combos uncapped).

  `workspace_start` (a fresh-card start only — resume launches are unchanged)
  returns as soon as the card exists, while the agent may still be launching
  asynchronously. Its response is only `{ok, card_id, branch, status}`; a
  follow-up `card_show`/`card_list` can show the card's session with empty
  `agent_provider*` fields until the launch finishes and fills them in.
- **Plans** — `plan_list`, `plan_save`, `plan_decide`, `plan_append_to_task`
- **Runs** — `run_list`, `run_show`, `run_log`, `run_cancel`
- **Review threads** — `card_review_list`, `card_review_create`,
  `card_review_patch`, `card_review_delete`, `card_review_send`
- **Cloud agent** — `card_start_cloud`, `card_cloud_status`,
  `card_cloud_cancel`, `card_cloud_retry`, `card_cloud_request_changes`
- **Peer handoff** — `peer_list`, `peer_self`, `peer_pair`, `peer_unpair`,
  `card_send_to_peer`, `card_handoff_accept`, `card_handoff_reject`,
  `card_handoff_list_incoming`
- **Tasks / deps / QA** — `task_*`, `deps_*`, `qa_*` (notably `task_sync` and
  `jira_import_task`)
- **Bitbucket PR review** — `bb_pr_review_list_prs`, `bb_pr_review_list_drafts`,
  `bb_pr_review_get_draft`, `bb_pr_review_get_batch`,
  `bb_pr_review_update_draft_body` (requires `pr_provider: bitbucket` +
  Bitbucket Cloud credentials configured in Augflow)
- **Scheduling** — `card_schedule`, `card_schedule_cancel`,
  `card_schedule_preview`, `card_schedule_list`, `card_schedule_resume_now`,
  `card_queue_add`, `card_queue_remove`, `card_queue_list`

For the current, authoritative list, ask your MCP client to list tools from
the `augflow` server (its `tools/list` call).

Every successful tool result (both `augflow mcp` and `augflow hermes-direct
serve`) now carries the real JSON payload in `structuredContent`, not an
empty object — `Content[0].Text` still carries the same JSON alongside it, so
existing clients that only read the text are unaffected. An error result
carries no `structuredContent`.

## Remote-device subset (`/api/mcp-remote`)

`/api/mcp-remote` is intended for an already-paired, approved Augflow
Anywhere remote device, but local/admin callers can reach it too — either
way, it's always limited to a separate, curated tool subset, not the full
list above — see [docs/transports-and-scoping.md](transports-and-scoping.md)
for the transport and auth details. Unless narrowed by
`remote_access.mcp.allowed_tools`, the default allowlist is: `card_list`,
`card_show`, `card_diff`, `card_worktree_status`, `card_pr_preview`,
`card_schedule_list`, `card_schedule_preview`, `card_queue_list`,
`card_review_list`, `task_list`, `task_show`, `run_list`, `run_show`,
`run_log`, `plan_list`, `knowledge_tree`, `deps_list`, `deps_link_types`,
`qa_list_runs`, `qa_result`, `agent_options` — a curated, non-destructive set
(not strictly read-only; some reads do internal bookkeeping): nothing that
commits, pushes, deletes, creates/updates a PR, or starts/retries a cloud
job, and `card_reopen` is not included. A disallowed tool simply doesn't
appear in that caller's `tools/list`.

## Not covered by MCP: repository set management

Registering a local git checkout into a **repository set** (the device-level
slug→path mappings in `~/.augflow/config.yaml`'s `repos_profiles`, surfaced in
the web UI as Settings → Repos) is **not exposed via either MCP server** — it
was checked against the full list of all 94 `augflow mcp` tools at v0.1.6, and
none of them touch `repos_profiles`. You have to do this once, outside MCP,
via:

- **CLI**: `augflow repos add [--profile <id-or-name>]`
- **Web UI**: Settings → Repos (backed by `PUT`/`GET /api/config/repos`, a
  REST endpoint, not MCP)

Two MCP tools sound similar but only *consume* an already-registered set,
they don't manage it:

- `card_add_repos` — attaches an already-configured repo to an in-progress
  card's worktree
- `task_set_repo` — assigns an already-configured repo to a task

So an MCP client (Claude Code, Cursor, Hermes, etc.) can use repos that are
already registered, but can't onboard a brand-new local checkout itself —
that first step always requires the CLI or web UI.

## `augflow hermes-direct serve` (Hermes-specific, 13 tools)

- **Read-only** — `augflow_status`, `augflow_cards_list`, `augflow_card_show`,
  `augflow_groups_list`, `augflow_context_capture`, `augflow_whereami`,
  `augflow_events_poll`, `augflow_events_wait`
- **Routing** — `augflow_bind_channel`, `augflow_active_card_set`
- **Mutating** — `augflow_prompt_queue`, `augflow_summary_generate`,
  `augflow_stop_agent`

Full detail, including what each tool requires and how permissions gate them,
is in [docs/clients/hermes.md](clients/hermes.md).

Deliberately **excluded** from the Hermes tool surface (by design, not a gap):
raw `tmux_send`, shell exec, `git commit`/`push`, PR creation, and starting new
local cards from chat.
