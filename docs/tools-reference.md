# Tool reference

Two different tool catalogs, depending on which server you connect (see
[README.md](../README.md) for which one your client should use).

## `augflow mcp` (general-purpose, ~95 tools)

Representative categories — this list will drift as Augflow adds tools; treat
it as a map of what exists, not an exact/exhaustive spec:

- **Cards** — `card_list`, `card_show`, `card_worktree_status`, `card_diff`,
  `card_move`, `card_done`, `card_reset`, `card_start_over`, `card_wont_do`,
  `card_archive`, `card_unarchive`, `card_stop_session`, `card_switch_provider`,
  `card_add_repos`, `card_validate`, `card_commit`, `card_commit_result`,
  `card_push`, `card_pr_preview`, `card_pr_generate_description`,
  `card_pr_create`, `card_pr_update`, `card_clean`, `card_delete`
- **Knowledge** — `knowledge_tree`, `card_knowledge_get`, `card_knowledge_add`,
  `card_knowledge_remove`, `card_knowledge_sync`
- **Workspace** — `agent_options`, `workspace_start`, `workspace_plan`,
  `workspace_implement`
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

## Not covered by MCP: repository set management

Registering a local git checkout into a **repository set** (the device-level
slug→path mappings in `~/.augflow/config.yaml`'s `repos_profiles`, surfaced in
the web UI as Settings → Repos) is **not exposed via either MCP server** — it
was checked against the full list of all ~95 `augflow mcp` tools, and none of
them touch `repos_profiles`. You have to do this once, outside MCP, via:

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

## `augflow hermes-direct serve` (Hermes-specific, ~12 tools)

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
