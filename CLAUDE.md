# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

pi.nvim is a Neovim plugin that bridges the `pi` CLI coding agent into Neovim. It sends buffer content (or visual selection) as context with a user prompt to `pi` via JSON-RPC over stdin/stdout, shows a spinner while `pi` works, then reloads the buffer from disk after `pi` edits the file directly.

## Commands

```bash
make deps          # Clone mini.test into deps/ (required before first test run)
make test          # Run all tests headless
make test-interactive  # Open Neovim with test UI for debugging
make clean         # Remove deps/
```

No linter is configured. No single-test runner exists — the test suite runs as one file.

## Architecture

Two-file plugin with a single module-level `state` table enforcing one concurrent job:

```
plugin/pi.nvim.lua          # Auto-loaded by Neovim. Registers :PiAsk and :PiAskSelection
                            # commands with lazy require("pi").
lua/pi/init.lua             # All business logic (~430 lines). Module table M returned at end.
```

### Data Flow

```
:PiAsk / :PiAskSelection
  → validate buffer is file-backed
  → vim.ui.input() for user prompt
  → build context string (SYSTEM_PROMPT + file content [+ selection])
  → M.send(): spawn `pi --mode rpc --no-session` via jobstart()
  → send single JSON {"type":"prompt","message":...} on stdin, then close stdin
  → parse streaming JSON events on stdout via handle_event()
  → on "agent_end": vim.cmd("edit!") to reload buffer from disk
```

Key design: `pi` edits files directly on the filesystem. Neovim never receives edited content over the RPC channel — it reloads from disk.

### Module Internals

- **Public API**: `M.setup(opts)`, `M.send(message, context)`, `M.prompt_with_buffer()`, `M.prompt_with_selection()`, `M.get_buffer_context()`, `M.get_visual_context()`
- **State**: Single `state` table (job, buf, win, spinner_idx, spinner_timer, ns_id, extmark_id). Singleton pattern — refuses to start if `state.job` is set.
- **Spinner UI**: Floating window with extmark-based virtual text. Buffer stays `modifiable=false`. 10-frame Braille animation at 200ms intervals.
- **Event types from pi**: `message_update`, `tool_execution_start`, `tool_execution_end`, `agent_end`, `response`

### Config

```lua
require("pi").setup({
  provider = nil,  -- optional, e.g. "openrouter", "anthropic", "openai"
  model = nil,     -- optional, e.g. "claude-haiku-4-5"
})
```

## Testing

Uses `mini.test` framework with `MiniTest.new_child_neovim()` for full Neovim process isolation.

Tests mock `vim.fn.jobstart`, `vim.fn.chansend`, `vim.fn.chanclose`, and `vim.ui.input` to capture the command and JSON prompt without spawning `pi`. Test file: `tests/test_pi_commands.lua`. Test init: `tests/minimal_init.lua` (overrides `vim.notify` to capture notifications).

Three test groups: PiAsk (6 tests), PiAskSelection (6 tests), Configuration (2 tests).

## Conventions

- All `on_stdout`/`on_stderr`/`on_exit` callbacks use `vim.schedule()` for safe Neovim API access
- Load guard via `vim.g.loaded_pi_nvim` prevents double-loading
- `vim` global is expected (Neovim runtime) — LSP "undefined global" warnings for `vim` are false positives
- Runtime dependency: `pi` CLI (`npm install -g @mariozechner/pi-coding-agent`)
