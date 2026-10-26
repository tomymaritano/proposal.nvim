# proposal.nvim

A Neovim agent that never writes a buffer and never runs a command until you accept the proposal.

The model may read files, grep, and inspect diagnostics on its own. An edit or a shell command is queued. You accept or reject each hunk, and each command, in a real Neovim diff. Accepted hunks land in the buffer as one undo block. The plugin does not `:w`.

v1 is specified and not implemented yet. The decision record is the [design spec](docs/superpowers/specs/2026-10-26-proposal-nvim-design.md). The implementer's reference is the [technical docs](docs/README.md).

## Status

| | |
| --- | --- |
| Plugin runtime | Not written |
| Neovim | 0.11 or newer, when the plugin exists |
| License | MIT |
| Model | Any OpenAI-compatible `chat/completions` endpoint |
| Beside | `coder/claudecode.nvim`, which this plugin does not replace |

`<leader>a` stays the Claude Code bridge. This plugin will use `<leader>A`.

## Requirements

Planned runtime requirements:

- Neovim 0.11+
- `curl` on `PATH`
- `rg` on `PATH` for the `grep` tool. Without it, that tool returns an error to the model.
- An API key in the environment variable named by `provider.api_key_env` (default `OPENAI_API_KEY`)

No plenary. No snacks. The key is read at request time and is never written to a buffer, a log, or a file.

## Docs

- [Technical reference](docs/README.md)
- [Architecture](docs/architecture.md)
- [Agent loop](docs/agent-loop.md)
- [Tool protocol](docs/tool-protocol.md)
- [Review](docs/review.md)
- [Testing](docs/testing.md)
- [Design spec](docs/superpowers/specs/2026-10-26-proposal-nvim-design.md)
