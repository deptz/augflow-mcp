# Tool reference

Two different tool catalogs, depending on which server you connect (see
[README.md](../README.md) for which one your client should use). Reflects
Augflow v0.1.8.

## `augflow mcp` (general-purpose, 95 tools)

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

  `card_list` matches the web board. Each real card carries
  `effective_status`, its board column. After the real cards come read-only
  **virtual To Do entries**, one per project task that has no card yet,
  taken from the project's 100 most recently created tasks
  (`id: "virtual-<taskID>"`, `status`/`effective_status: "backlog"`,
  `virtual: true`). Listing them writes nothing. The `status` filter matches
  `effective_status` and accepts `backlog`, `plan`, `in_progress` and `done`,
  plus `todo` as an alias for `backlog`. Any other value returns an error
  listing the valid values instead of an empty list. Virtual entries appear only
  with no filter or with `backlog`/`todo`, never with `archived=true`, and tasks
  whose card is archived are not listed.

  `card_show` accepts a card ID, a task ID or a `virtual-<taskID>` ID, and
  returns the card's tasks in board order. Like opening the card in the web UI,
  it moves a To Do card that already has a live agent session to In Progress.
  For a task with no card it returns the read-only virtual view without creating
  a card. `card_schedule` and `card_queue_add` accept a virtual ID and create
  the card on first use (`card_schedule_preview` accepts one without creating
  a card), and `workspace_start` creates a card from a task ID. Other card
  tools that act on an existing card return a "has no card yet" error for a
  task without a card.
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

  When 9router routing is on for the agent's provider, starts and relaunches
  through MCP (`workspace_start`, `card_move` into In Progress, `card_reopen`,
  `card_switch_provider`, `workspace_plan`, `workspace_implement`, and QA
  workflow compilation) check the chosen model against the live 9router
  catalog. An empty or unknown model is rejected with a message pointing to
  `augflow router models`. With routing off, nothing changes. A `qa_run` that
  uses LLM assertions skips them instead of failing when the model is unknown,
  and a `workspace_start` with `plan_prep` is checked only when the card moves
  to In Progress.

  `agent.auto_enter` defaults to true, so a started card's prompt is sent
  without a review step unless your config sets `auto_enter: false`.
- **Plans** — `plan_list`, `plan_save`, `plan_decide`, `plan_append_to_task`
- **Runs** — `run_list`, `run_show`, `run_log`, `run_cancel`

  `plan_list` and `run_list` accept a task ID as well as a card ID.
- **Review threads** — `card_review_list`, `card_review_create`,
  `card_review_patch`, `card_review_delete`, `card_review_send`

  `card_review_list`'s `kind` filter must be `plan`, `diff` or `html` (or
  omitted); any other value is an error.
- **Cloud agent** — `card_start_cloud`, `card_cloud_status`,
  `card_cloud_cancel`, `card_cloud_retry`, `card_cloud_request_changes`

  `card_start_cloud`'s `provider` names a configured `cloud_agent` provider:
  `edison`, `modal` or `digitalocean`.
- **Peer handoff** — `peer_list`, `peer_self`, `peer_pair`, `peer_unpair`,
  `card_send_to_peer`, `card_handoff_accept`, `card_handoff_reject`,
  `card_handoff_list_incoming`
- **Tasks / deps / QA** — `task_*`, `deps_*`, `qa_*` (notably `task_sync`,
  `jira_import_task` and `task_convert_to_jira`)

  `task_list` is paged: optional `limit` (default 100, max 500) and `offset`,
  newest first. When a page comes back full, call again with
  `offset += limit`. Before v0.1.8 the tool silently returned at most the
  newest 100 tasks and had no `limit`/`offset` parameters.

  `task_convert_to_jira` turns a manual task into a Jira-backed one, keeping
  its card and attachments. `mode: link` attaches an
  existing issue (`issue_key`; optional `content_mode`: `jira` (default),
  `keep_local` or `push_local`). `mode: create` creates a new issue (optional
  `project_key`, `epic_key`, and `story_key`, which requires `epic_key`).
  Optional `rename_branch` renames the card's branch to match. Requires Jira
  to be configured in Augflow.

  `deps_add` rejects self-dependencies and circular dependencies, like the web
  UI.
- **Bitbucket PR review** — `bb_pr_review_list_prs`, `bb_pr_review_list_drafts`,
  `bb_pr_review_get_draft`, `bb_pr_review_get_batch`,
  `bb_pr_review_update_draft_body` (requires `pr_provider: bitbucket` +
  Bitbucket Cloud credentials configured in Augflow)
- **Scheduling** — `card_schedule`, `card_schedule_cancel`,
  `card_schedule_preview`, `card_schedule_list`, `card_schedule_resume_now`,
  `card_queue_add`, `card_queue_remove`, `card_queue_list`

For the current, authoritative list, ask your MCP client to list tools from
the `augflow` server (its `tools/list` call).

Both servers return the JSON payload as text in `Content[0].Text`. Since
v0.1.8 they also set `structuredContent`, **but only when the result is a JSON
object**. Array results (`card_list`, `task_list`, `deps_list`,
`deps_link_types`, `card_schedule_list`, `card_queue_list`, and others) carry
no `structuredContent`, because strict clients such as Claude Code reject a
non-object there ("expected record, received array"). Empty lists come back as
`[]`, not `null`. An error result carries no `structuredContent`. Clients that
only read the text are unaffected.

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
job, and `card_reopen` and `task_convert_to_jira` are not included. A disallowed tool simply doesn't
appear in that caller's `tools/list`.

## Not covered by MCP: repository set management

Registering a local git checkout into a **repository set** (the device-level
slug→path mappings in `~/.augflow/config.yaml`'s `repos_profiles`, surfaced in
the web UI as Settings → Repos) is **not exposed via either MCP server** — it
was checked against the full list of all 95 `augflow mcp` tools at v0.1.8, and
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

Since v0.1.8:

- `augflow_cards_list` with no `status` returns the "relevant" view (cards in
  progress, with a live session, or with an agent status). Pass `status`
  (`backlog`/`todo`, `plan`, `in_progress`, `done`) to list one board column;
  any other value is an `invalid_args` error. Entries carry `effective_status`,
  and the response echoes the `filter` used.
- With `status: backlog`, the list also includes tasks that have no card yet,
  as read-only `virtual: true` entries (`card_id: virtual-<taskID>`). They can
  be read with `augflow_card_show`, but not bound, prompted, summarized or
  stopped.
- `card_key` falls back to the task ID when a card has no external key.
- `augflow_card_show` accepts a card ID, task ID, `card_key` or virtual ID. A
  key that matches more than one card or task returns `ambiguous_key`; use the
  `card_id` instead.
- `augflow_events_poll`/`augflow_events_wait` return only this project's events
  and include `has_more`.
- `augflow_status` and `augflow_groups_list` include a `warnings` entry when
  channel bindings can't be read, instead of reporting the channel as unbound.

Full detail, including what each tool requires and how permissions gate them,
is in [docs/clients/hermes.md](clients/hermes.md).

Deliberately **excluded** from the Hermes tool surface (by design, not a gap):
raw `tmux_send`, shell exec, `git commit`/`push`, PR creation, and starting new
local cards from chat.
