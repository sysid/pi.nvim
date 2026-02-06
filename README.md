# pi.nvim

A Neovim plugin for interacting with [pi](https://pi.dev) - the minimal terminal coding agent.

I found all AI plugins for Neovim too complicated, like they wanted to imitate the latest IDE features, with lots of windows and information. But some users just use neovim for its simplicity. I find it funny that most coding agents are trending towards the simplicity of the CLI, and [pi.dev](https://pi.dev/) is the perfect example of this philosophy. It was the perfect candidate to integrate in neovim.

## Features

- **Context aware**: Sends your current buffer + selection as context.
- **Simple configuration**: Just set your preferred AI model.
- **Gets out of your way**: You ask it. It does it. Done.

## Requirements

- [Neovim](https://neovim.io/) 0.7+
- [pi](https://github.com/badlogic/pi-mono) installed globally: `npm install -g @mariozechner/pi-coding-agent`
- Your preferred models availble in pi: `pi --list-models`

## Installation

### Using [lazy.nvim](https://github.com/folke/lazy.nvim)

```lua
{ "pablopunk/pi.nvim" }
```

### Using [packer.nvim](https://github.com/wbthomason/packer.nvim)

```lua
use "pablopunk/pi.nvim"
```

### Using [mini.deps](https://github.com/echasnovski/mini.nvim/blob/main/readmes/mini-deps.md)

```lua
MiniDeps.add("pablopunk/pi.nvim")
```

## Config

```lua
require("pi").setup({
  model = "openrouter/free",    -- default model
})
```

Run `pi --list-models` to see available options.

## Usage

### Commands

| Command | Mode | Description |
|---------|------|-------------|
| `:PiAsk` | Normal | Prompt for input, sends with full buffer as context |
| `:PiAskSelection` | Visual | Prompt for input, sends with visual selection as context |
| `:PiStop` | Normal | Stop the currently running pi session |


## License

MIT
