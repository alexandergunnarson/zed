# Changes from upstream

All changes on this fork relative to `zed-industries/zed` (merge-base `e784c92e3c`), organized by area.

## Feature: Agent panel master-detail layout with sidebar

Replaces the stacked agent panel layout with a persistent sidebar (thread list) + right pane (active thread). Includes:

- **Sidebar with thread list** always visible alongside the active conversation, with a "New Chat" button.
- **Draggable resizer** between sidebar and thread pane (min 150px, max 600px).
- **Active session highlighting** — the sidebar tracks and highlights the currently open thread.
- **Thread deletion** immediately removes the entry from the sidebar and switches to a new thread if the active one was deleted.

Files: `crates/agent_ui/src/agent_panel.rs`

## Feature: Thread history enhancements

- **Content search** — search now queries thread message bodies (not just titles), showing matched snippets with highlighted positions. Uses `AgentSessionList::search_sessions()` backed by full-text scan of thread data.
- **Inline rename** — thread titles can be renamed in-place via an editor widget, persisted through `AgentSessionList::set_session_title()`.
- **Collapsible tool calls** in the thread view.
- **Switched from `uniform_list` to `ListState`** for more flexible item rendering.

Files: `crates/agent_ui/src/thread_history.rs`, `crates/agent_ui/src/connection_view/thread_view.rs`

## Feature: Agent thread status & diff stats tracking

Tracks per-thread status and cumulative diff statistics:

- **`AgentStatus` enum** (`Idle`, `Working`, `AwaitingInput`, `Error`) on `AgentSessionInfo`.
- **Diff stats** (`files_changed`, `lines_added`, `lines_deleted`) extracted from `streaming_edit_file` tool output.
- **Persisted to SQLite** via `ALTER TABLE` migrations adding `status`, `last_action_summary`, `files_changed`, `lines_added`, `lines_deleted` columns.
- **Status transitions** plumbed from `NativeAgentConnection` (set `Working` on prompt, `Idle` on stop) through `ThreadStore` events to the UI.
- **Animated working indicator** in the sidebar for threads with `Working` status.

Files: `crates/acp_thread/src/connection.rs`, `crates/agent/src/agent.rs`, `crates/agent/src/thread.rs`, `crates/agent/src/thread_store.rs`, `crates/agent/src/db.rs`, `crates/agent_servers/src/acp.rs`, `crates/sqlez/src/bindable.rs`, `crates/sidebar/src/sidebar.rs`

## Feature: `terminal::SendToTerminal` action

Workspace-level action that sends text to the active terminal with `$ZED_*` task variable substitution. Analogous to VSCode's `workbench.action.terminal.sendSequence`.

```json
["terminal::SendToTerminal", { "text": "$ZED_SELECTED_TEXT\n", "paste": true }]
```

- `text`: supports all task variables (`$ZED_SELECTED_TEXT`, `$ZED_FILE`, etc.) and `${VAR:default}` syntax.
- `paste`: when `true`, wraps in bracketed paste escape sequences.
- Registered as a workspace action (works from the editor, not just terminal focus).
- Reuses the task system's `substitute_variables_in_str()` for variable resolution.

Files: `crates/terminal_view/src/terminal_panel.rs`

## Feature: OpenAI `stream_options` with `include_usage`

Passes `stream_options: { include_usage: true }` when streaming from OpenAI-compatible providers, so token usage is reported in stream responses.

Files: `crates/open_ai/src/open_ai.rs`, `crates/language_models/src/provider/open_ai.rs`

## Feature: Google Gemini thinking mode improvements

- **`ThinkingConfig` serialization** conditionally omits `thinkingBudget` when set to 0 (sentinel for "auto" mode), and always includes `includeThoughts: true`.
- **`thought` field on `TextPart`** distinguishes thinking content from regular text, mapped to `LanguageModelCompletionEvent::Thinking`.
- Default thinking-capable models use `budget_tokens: Some(0)` instead of `None` to enable thought streaming.

Files: `crates/google_ai/src/google_ai.rs`, `crates/language_models/src/provider/google.rs`

## Bugfix: Grep tool infinite loop on overlapping ranges

When merging overlapping match ranges, the code unconditionally set `range.end = next_range.end`, which could *shrink* the range if the next match ended earlier, causing the merge loop to never terminate. Fixed to only expand.

Files: `crates/agent/src/tools/grep_tool.rs`

## Bugfix: Cannot open files outside workspace in remote projects

- `terminal_path_like_target.rs`: replaced direct `fs.canonicalize()`/`fs.metadata()` calls (which fail on remote) with `project.resolve_abs_path()`.
- `workspace.rs`: for remote projects, uses `DirectoryLister::Project` and opens paths directly via `open_paths()` instead of trying to create a new local window.

Files: `crates/terminal_view/src/terminal_path_like_target.rs`, `crates/workspace/src/workspace.rs`

## Bugfix: Crash in `Grammar::parse_text` when tree-sitter returns `None`

`parse_with_options()` can return `None` (e.g. cancelled parse). Changed return type from `Tree` to `Option<Tree>` and updated `highlight_text` to skip highlighting gracefully instead of panicking.

Files: `crates/language/src/language.rs`

## Bugfix: Show token ring instead of split display for Anthropic custom models

Anthropic custom models don't support split input/output token counts. Overrides `supports_split_token_display()` to return `false`.

Files: `crates/language_models/src/provider/anthropic.rs`

## Bugfix: Compilation error from `stream_options` addition

Added `stream_options: None` to Mercury edit-prediction request body.

Files: `crates/edit_prediction/src/mercury.rs`

## Performance: Git diff view with large change sets

Fixes severe slowdowns when the uncommitted changes view has thousands of files (e.g. ~2,000), especially over remote development:

- **Skip binary files early** — `load_buffers()` in `BranchDiff` now checks file extensions against a list of known binary formats (images, archives, executables, fonts, etc.) before attempting `open_buffer`. Previously every binary file triggered a full RPC round-trip that failed with `"Binary files are not supported"`. Files are still listed in the git panel for staging/committing.
- **Debounce `DiffChanged` refreshes** — Each registered buffer's diff subscription triggers a full refresh of the project diff. With N buffers, a diff change at a `yield_now()` point during the refresh loop would cancel the in-progress refresh and restart from scratch, potentially preventing completion. `DiffChanged` now has a 50ms debounce so events coalesce before work begins. `StatusesChanged` and `EditorSaved` remain immediate.

Files: `crates/project/src/git_store/branch_diff.rs`, `crates/git_ui/src/project_diff.rs`

## Build: Faster remote server uploads

- **Release builds for remote server** — `cargo zigbuild --release` instead of debug, with corresponding binary path change.
- **`strip = true`** added to release profile in root `Cargo.toml`.
- **Reduced dev debug info** — `debug = 1` for workspace crates, `debug = false` for dependencies and build-override to speed up incremental builds.

Files: `Cargo.toml`, `crates/remote/src/transport.rs`
