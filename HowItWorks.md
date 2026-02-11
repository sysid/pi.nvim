# How it works
The pi CLI process is spawned on every :PiAsk or :PiAskSelection invocation — it's not a long-running daemon.

The exact call chain:
```
:PiAsk → M.prompt_with_buffer() → M.send(input, context)
:PiAskSelection → M.prompt_with_selection() → M.send(input, context)
```

In M.send() at lua/pi/init.lua:271:

```lua
state.job = vim.fn.jobstart(cmd, { stdin = "pipe", ... })
```

Where cmd is built by get_pi_cmd() (line 98):

```lua
{ "pi", "--mode", "rpc", "--no-session" }
```

Plus optional --provider and --model flags from config.

Lifecycle: one process per request. Neovim sends a single JSON prompt on stdin, immediately closes
stdin (chanclose at line 320), then listens for stdout events. When pi sends agent_end, the process
terminates and state.job is cleared. The singleton guard (if state.job then return end) prevents
concurrent instances.

There is no persistent background process, no connection pooling, no session reuse (--no-session
flag). Each user interaction spawns a fresh pi process that lives only for that one request.


```lua
  local function get_pi_cmd()
    local cmd = { "pi", "--mode", "rpc", "--no-session" }
    if M.config.provider then
      table.insert(cmd, "--provider")
      table.insert(cmd, M.config.provider)
    end
    if M.config.model then
      table.insert(cmd, "--model")
      table.insert(cmd, M.config.model)
    end
    return cmd
  end
```

And the jobstart call (line 271) passes no cwd or env overrides.

What happens: 
The pi process inherits Neovim's working directory and environment. Whether it reads ~/.pi config
depends entirely on pi's own default behavior — the plugin doesn't pass --config ~/.pi or any
config path flag.


# Security

What the plugin itself sends — verified from code:

Yes, data is only sent when the user explicitly runs :PiAsk or :PiAskSelection, which both require:
1. User types the command
2. User enters text in vim.ui.input() and presses Enter

The JSON payload sent on stdin (lua/pi/init.lua:315-320) contains exactly:
{"type":"prompt", "message":"<user_input>\n\nContext:\n<system_prompt + buffer content>"}

No other data is sent by the plugin.

However — and this is important — the plugin is NOT the only actor here:
┌────────────────────────────────────────────────────────────────────┬─────────────┐
│                              Concern                               │   Status    │
├────────────────────────────────────────────────────────────────────┼─────────────┤
│ Plugin sends data only on explicit user action                     │ Verified ✅ │
├────────────────────────────────────────────────────────────────────┼─────────────┤
│ Plugin only sends current buffer content                           │ Verified ✅ │
├────────────────────────────────────────────────────────────────────┼─────────────┤
│ pi process inherits full environment ($HOME, API keys, etc.)       │ ⚠️ Yes      │
├────────────────────────────────────────────────────────────────────┼─────────────┤
│ pi is an agent with tool use — it can read arbitrary files on disk │ ⚠️ Yes      │
├────────────────────────────────────────────────────────────────────┼─────────────┤
│ pi has network access during execution                             │ ⚠️ Yes      │
├────────────────────────────────────────────────────────────────────┼─────────────┤
│ No sandboxing, no env restriction, no cwd jail in jobstart         │ ⚠️ Yes      │
├────────────────────────────────────────────────────────────────────┼─────────────┤
│ pi config (~/.pi) could contain arbitrary tool definitions         │ ⚠️ Unknown  │
├────────────────────────────────────────────────────────────────────┼─────────────┤
│ --no-session prevents session persistence, but not runtime access  │ Partial     │
└────────────────────────────────────────────────────────────────────┴─────────────┘

The tool_execution_start events in handle_event() (line 231) prove that pi runs tools during
execution. Those tools are defined by pi-mono, not by this plugin. Once spawned, pi is a fully
autonomous agent with your user's filesystem and network permissions.

Bottom line: The plugin itself has no side-channel. But it spawns an unrestricted subprocess that,
by design, reads and writes files and makes network calls. The security boundary is pi-mono, not
pi.nvim.
