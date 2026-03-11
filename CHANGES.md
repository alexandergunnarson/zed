# Changes from main

## New feature: `terminal::SendToTerminal` action

Adds a new workspace-level action `terminal::SendToTerminal` that sends text to the active terminal with support for task variable substitution (e.g. `$ZED_SELECTED_TEXT`, `$ZED_FILE`).

This is analogous to VSCode's `workbench.action.terminal.sendSequence` command, enabling workflows like sending selected editor text to a running REPL.

### Action schema

```json
["terminal::SendToTerminal", { "text": "...", "paste": false }]
```

| Field   | Type     | Default | Description |
|---------|----------|---------|-------------|
| `text`  | `string` | (required) | The text to send. Supports all `$ZED_*` task variables and `${VAR:default}` syntax. |
| `paste` | `bool`   | `false` | When `true`, uses bracketed paste mode (`\x1b[200~`...`\x1b[201~`), which is appropriate for sending multi-line text to REPLs. |

### Supported variables

All standard task variables are available:

- `$ZED_SELECTED_TEXT` — the current editor selection
- `$ZED_FILE` — absolute path of the current file
- `$ZED_RELATIVE_FILE` — path relative to worktree root
- `$ZED_FILENAME` — filename only
- `$ZED_STEM` — filename without extension
- `$ZED_DIRNAME` — absolute path of the file's parent directory
- `$ZED_WORKTREE_ROOT` — absolute path of the worktree root
- `$ZED_ROW` — cursor row (1-indexed)
- `$ZED_COLUMN` — cursor column (1-indexed)
- `$ZED_SYMBOL` — symbol at cursor position

### Example: send selected text to a Julia REPL

In `~/.config/zed/keymap.json`:

```json
[
  {
    "context": "Editor && mode == full",
    "bindings": {
      "cmd-enter": [
        "terminal::SendToTerminal",
        { "text": "$ZED_SELECTED_TEXT\n", "paste": true }
      ]
    }
  }
]
```

### Design notes

- **Purely additive.** The existing `terminal::SendText` action is unchanged. No keybindings, APIs, or docs were modified.
- **Workspace-level registration.** Unlike `terminal::SendText` (which only works when a terminal is focused), `SendToTerminal` is registered as a workspace action so it can read from the active editor and forward to the terminal panel.
- **Variable resolution reuses the task system.** The action calls `Editor::task_context()` and `substitute_variables_in_str()` from the `task` crate — the same machinery used by Zed's task runner.
- **`paste` vs raw input.** When `paste` is `false`, text is sent via `Terminal::input()` (raw bytes, no escaping). When `true`, text is sent via `Terminal::paste()`, which wraps in bracketed paste escape sequences when the terminal has that mode enabled.

### Files changed

- `crates/terminal_view/src/terminal_panel.rs`
  - Added `SendToTerminal` action struct
  - Added `TerminalPanel::send_to_terminal` workspace action handler
  - Added `TerminalPanel::send_text_to_active_terminal` method
  - Registered the action in `init()`
