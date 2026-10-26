# Technical reference

These pages are the contract for v1. The [design spec](superpowers/specs/2026-10-26-proposal-nvim-design.md) is the decision record. When they disagree, the spec wins and the reference is updated to match it.

The plugin runtime is not implemented yet. Paths below are the layout the implementation will use.

| Page | What it locks |
| --- | --- |
| [Architecture](architecture.md) | Modules, who may call whom, file layout, invariants |
| [Agent loop](agent-loop.md) | States of one ask, turn budget, message list |
| [Tool protocol](tool-protocol.md) | JSON schemas, result strings, jail, context pack |
| [Review](review.md) | Diff split, hunk marks, shell confirm, apply and undo |
| [Testing](testing.md) | Headless cases and the provider seam |

## Invariants

These hold for every state:

1. A buffer's bytes change only inside `diff`, and only when a review of that file finishes with a merge that differs from the snapshot.
2. A shell command runs only after `y` on that exact command string.
3. `read_file` and `grep` refuse any real path outside the project root. Shell is not jailed. The accept step is the control.
4. The API key never appears in a message, a buffer, a notification, or a file.
5. A tool failure is a tool result (`error: …`). It is not a Lua error.
6. The plugin never `:w`.
7. One ask is one message list. The list is dropped when the ask ends. Nothing is written to shada or to the repo.
