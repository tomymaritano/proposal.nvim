# proposal.nvim

Date: 2026-10-26
Status: accepted

## Thesis

`proposal.nvim` is a Neovim plugin. The agent never writes a buffer. It queues a proposal. A proposal is one or more edits, and zero or more shell commands. The user accepts or rejects each edit hunk, and accepts or rejects each command. Only then does the plugin change a buffer or run a process.

The plugin is open source (MIT). It loads in the existing LazyVim config, beside `coder/claudecode.nvim`. It does not replace that plugin and does not reuse its keymaps.

## v1

v1 is a daily driver for one repo at a time:

- Ask from normal mode, or ask with a visual selection.
- The model reads files, greps, and reads diagnostics without a prompt.
- Edits and shell commands wait for an explicit decision.
- Accepted edits land in the real buffer as one undo block per file.
- The plugin does not write the file to disk.
- The model is any OpenAI-compatible chat-completions endpoint. The API key stays in an environment variable.

v1 does not include inline autocomplete, embeddings, a repo index, ACP, a second agent, a GUI shell, or a change to `claudecode.nvim`.

## Where it lives

- Source repo: `~/code/proposal.nvim`
- License: MIT
- Neovim: 0.11 or newer
- Runtime dependency: `curl` on `PATH`
- Neovim libraries only. No plenary, no snacks requirement.
- LazyVim spec, added during implementation, at `~/dotfiles/nvim/lua/plugins/proposal.lua`. While developing, `dir` points at the local checkout.

Default spec shape:

```lua
{
  dir = vim.env.HOME .. "/code/proposal.nvim",
  opts = {
    provider = {
      base_url = "https://api.openai.com/v1",
      model = "gpt-4.1",
      api_key_env = "OPENAI_API_KEY",
    },
  },
}
```

`base_url` and `model` are the only way to change provider. Any compatible endpoint works, including a local one.

## Modules

| Module | Responsibility |
| --- | --- |
| `config` | Defaults and user opts. |
| `provider` | One HTTP call. Function signature `complete(messages, tools) -> assistant message`. `curl` is the production implementation. Tests pass a fake. |
| `context` | Builds the user-context text for the first turn of an ask. |
| `loop` | Owns the message list, runs at most `max_turns` rounds, dispatches tools, pauses for review. |
| `tools` | `read_file`, `grep`, `diagnostics`, `propose_edit`, `propose_shell`. |
| `jail` | Decides whether a path is inside the project root. |
| `diff` | Review UI and the single buffer apply. |
| `ui` | Ask prompt, notifications, commands, keys. |
| `init` | `setup`, user commands. |

`tools` calls `jail`. `loop` calls `provider`, `tools`, and `diff`. `diff` does not call the network.

## Project root and jail

The project root is the git toplevel of the cwd when `git rev-parse --show-toplevel` succeeds. Otherwise the root is the cwd.

A path is allowed when `vim.uv.fs_realpath` succeeds and the real path is the root or a file under the root. Symlinks that resolve outside the root are rejected. `..` is not special-cased beyond realpath.

`read_file` and `grep` use the jail. A rejected path is a tool error string returned to the model. It is not a Lua error shown to the user.

Shell commands are not parsed and are not jailed. They run with cwd set to the project root, and only after the user accepts that exact command. The accept step is the control.

## Context pack

Sent once, as the first user message, after the system prompt. Later turns send tool results only.

The pack contains:

- Project root.
- Active file path, or a line saying there is no file.
- Visual selection with line numbers, when the ask started from visual mode. Otherwise 80 lines centered on the cursor (40 above, 40 below), clipped to the buffer, with line numbers.
- Diagnostics for the active buffer: severity, line, message. At most 20.
- `git diff --` for the active file, when the root is a git repo. Omitted when git fails.

The pack is truncated to 32,000 characters. Truncation drops, in order: the git diff, then diagnostics, then lines farthest from the cursor. The pack ends with the line `[context truncated]` when anything was dropped.

## System prompt

The system prompt is fixed in v1:

```
You are a coding agent inside Neovim. You cannot edit files or run commands directly.
Use read_file, grep, and diagnostics to inspect the project.
Use propose_edit to change a file. old must be copied verbatim from the file and must occur once. new is the replacement. An empty new deletes old.
Use propose_shell to run a command. The user accepts or rejects every edit and every command.
Stay inside the project. Prefer a small edit over a rewrite.
When you are done, reply with a short summary and no tool call.
```

## Provider

`POST {base_url}/chat/completions`

Headers: `Authorization: Bearer <key>`, `Content-Type: application/json`.

The key is read from `provider.api_key_env` at request time. A missing or empty variable stops the ask before any request. Notification: `proposal.nvim: set <env name>`. The key is never written to a buffer, a log, or a file.

Body fields: `model`, `messages`, `tools`, `tool_choice: "auto"`. No streaming.

Transport: `curl -sS -m 60`. Timeout is 60 seconds. A non-zero exit, a timeout, or an HTTP status outside 200–299 notifies the user with the status and ends the ask. The next `:Proposal` starts a new conversation. v1 does not retry a failed HTTP call. `:ProposalStop` kills the running `curl` process.

A 200 body that is not JSON, or a JSON body without `choices[0].message`, is retried once with an extra user message: `Reply with a tool call or a plain summary.` A second failure notifies and stops.

Assistant text and tool calls are taken from `choices[0].message`. Tool calls use the OpenAI shape `id`, `function.name`, `function.arguments` (JSON string).

## Tools

Arguments that fail validation return a tool message `error: <reason>` and do not throw.

### read_file

Parameters: `path` (required), `offset` (optional, 1-based, default 1), `limit` (optional, default 200, maximum 400).

Reads the loaded buffer when the path matches a loaded buffer, otherwise the file on disk. Returns numbered lines. A path outside the jail, a directory, or a missing file is an error.

### grep

Parameters: `pattern` (required), `path` (optional, default project root), `glob` (optional).

Runs `rg --vimgrep` under the project root. `path` must pass the jail. The plugin keeps the first 50 matching lines and drops the rest. Missing `rg` is a tool error telling the model ripgrep is not installed.

### diagnostics

Parameters: `path` (optional). Default is the active buffer.

Uses `vim.diagnostic.get`. Returns severity, line, column, message. Capped at 30 entries. An unknown path is an error.

### propose_edit

Parameters: `path`, `old`, `new`. `path` and `old` are required. `new` defaults to empty string. `old` must be non-empty.

The target text is the loaded buffer when that path is loaded, otherwise the file bytes. `old` must occur once in that text. Zero or several occurrences is an error, and nothing is queued.

On success the tool queues a proposal and returns `queued`. It does not change the buffer. The loop does not send that tool result until the user finishes the review.

Several `propose_edit` calls in one assistant message are applied, in order, against a virtual copy of each file. A later call that does not match the virtual copy is an error for that call only. The earlier calls stay queued.

### propose_shell

Parameters: `command` (required, non-empty). No cwd parameter.

Queues the command. Does not run it. Returns `queued` to the loop, held until review, same as edits.

## Loop

`max_turns` default is 8. A turn is one provider call.

Each turn:

1. Call the provider.
2. If the assistant message has text and no tool calls, show the text in a floating buffer and stop.
3. Run every `read_file`, `grep`, and `diagnostics` call immediately.
4. Queue every valid `propose_edit` and `propose_shell`.
5. If anything was queued, open the review and wait.
6. Append one tool result per tool call, in the order the model emitted them. Immediate tools already have their result. Proposals get a summary of the user's decision.
7. Continue until the model stops, the user aborts, or turn 8 completes.

After turn 8 the loop notifies `proposal.nvim: stopped after 8 turns` and shows any assistant text from that turn. It does not start another review.

Each ask owns one message list in memory. The list is dropped when the ask ends. It is not written to shada, to a file, or to the repo. A new `:Proposal` starts a new list. v1 has no resume.

## Review

The review opens only when at least one proposal was queued. Files are reviewed one at a time, in the order of the first edit for that file. The command line shows `Proposal i/n` while a file is open.

For each file:

- Snapshot the original lines.
- Build the proposed lines by applying that file's queued edits to the snapshot.
- Open a vertical diff split. Left is the real buffer, set nomodifiable for the duration of the review. Right is a scratch buffer with the proposed lines, nomodifiable, `buftype=nofile`.
- Hunks come from `vim.diff` with `result_type = "indices"` between the original lines and the proposed lines. They start undecided. The cursor line maps to the hunk that contains it.

Buffer-local keys on both sides, normal mode:

| Key | Action |
| --- | --- |
| `]c` / `[c` | Next and previous hunk. Vim's default diff motions. |
| `ga` | Mark the current hunk accepted. |
| `gr` | Mark the current hunk rejected. |
| `gA` | Mark every still-undecided hunk accepted. |
| `q` | Finish this file. Undecided hunks count as rejected. |
| `<C-c>` | Abort the whole ask. No further provider call. No buffer write. |

`ga` and `gr` toggle the mark on the current hunk. A sign in the signcolumn shows `accepted` or `rejected`. The buffers stay unchanged until `q`, so the user can flip a hunk before leaving the file.

On `q`, the plugin clears nomodifiable, then builds the merged lines (accepted hunks from the proposal, rejected and undecided hunks from the original). When the merge differs from the snapshot, it replaces the buffer with `nvim_buf_set_lines` once. That is one undo entry. When the merge equals the snapshot, it does not touch the buffer. The plugin does not `:w`.

An unloaded file is loaded with `bufadd` and `bufload` before the split. After apply it is a normal modified buffer.

When every file is finished, each queued shell command is shown in a floating window: the command text and the project root. `y` accepts, `n` rejects, `<C-c>` aborts the ask. Accept runs the command with `vim.system`, cwd = project root, and waits up to 60 seconds. `:ProposalStop` kills that process and aborts the ask. stdout and stderr are concatenated and truncated to 8,000 characters in the tool result. A non-zero exit is still a normal tool result; the model can read it. Reject produces the tool result `user rejected this command`.

The edit tool result is one text block: which hunks were accepted, which were rejected, and the path. It does not include the new file contents.

Abort (`<C-c>`) discards unapplied proposals, clears nomodifiable, closes the scratch windows, and stops the loop. Applied files from earlier in the same review stay applied. Their undo entries remain.

## Ask UI

Keys, in the which-key group `<leader>A` with description `Proposal`:

| Key | Mode | Action |
| --- | --- | --- |
| `<leader>Aa` | n | Open the prompt. |
| `<leader>Av` | v | Open the prompt with the selection as context. |

Commands: `:Proposal`, `:ProposalStop`.

The prompt is a floating scratch buffer, opened in insert mode. It is multiline. Normal-mode `<CR>` submits. Insert-mode `<C-s>` submits. `<C-c>` and normal-mode `q` cancel. An empty submit cancels.

Notifications go through `vim.notify`.

## Errors

| Case | What the user sees | What the model sees |
| --- | --- | --- |
| API key missing | notify, ask does not start | nothing |
| curl, timeout, HTTP error | notify with status, turn stops | nothing |
| Bad JSON twice | notify, turn stops | the one retry message |
| Path outside root, bad `old`, bad args | nothing, unless every tool in the turn failed and there is no proposal | `error: <reason>` |
| `rg` missing | nothing extra | tool error |
| Turn cap | notify | no further call |
| Abort | notify `proposal.nvim: aborted` | no further call |

A Lua error inside the plugin is a bug. Tool failures are data.

## Testing

Tests run with `nvim --headless` and do not open a socket or call `curl`. The provider under test is the fake passed into the loop.

1. Jail. A path inside the root is allowed. A path outside is rejected. A symlink inside the tree whose real path is outside is rejected.
2. `propose_edit`. Missing `old` errors. Two matches error. One match yields the expected proposed lines. A second edit that misses the virtual document errors without dropping the first.
3. Apply. A buffer with two hunks, one accepted and one rejected, matches the merge. One `u` restores the original snapshot.
4. Loop. The fake returns a `propose_edit` and then, after the tool result, a plain summary. The loop stops on the summary. A fake that always returns a tool call stops at turn 8.

## Dotfiles

Implementation adds `~/dotfiles/nvim/lua/plugins/proposal.lua` and does not edit `claudecode.lua`. Keys under `<leader>a` stay the Claude Code bridge. Keys under `<leader>A` are this plugin.

## Success

v1 is done when, in LazyVim, with a real API key:

- `<leader>Aa` sends the cursor window and gets a proposal back.
- Accepting one hunk and rejecting another changes the buffer, and one undo restores it.
- The file on disk is unchanged until the user writes it.
- A shell command does not run until `y`.
- `claudecode.nvim` still opens on `<leader>ac`.
- The headless tests pass.
