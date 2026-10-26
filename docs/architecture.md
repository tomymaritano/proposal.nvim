# Architecture

`proposal.nvim` is a single Neovim process. The model is reached with one `curl` POST per turn. Neovim itself is the tool runtime: buffers, diagnostics, `rg`, and `vim.system`.

```
ui  -->  loop  -->  provider  -->  curl  -->  chat/completions
              |          ^
              v          |
            tools -----> jail
              |
              +---- read_file | grep | diagnostics     (run now)
              +---- propose_edit | propose_shell       (queue)
                        |
                        v
                      diff  -->  buffer bytes, once per file
                             -->  vim.system, after y
```

`diff` does not call the network. `tools` does not change buffer bytes. `provider` does not know about hunks.

## File layout

```
lua/proposal/init.lua           setup, user commands
lua/proposal/config.lua         defaults and opts
lua/proposal/ui.lua             prompt, notify, keys
lua/proposal/loop.lua           message list and turns
lua/proposal/provider.lua       complete(messages, tools)
lua/proposal/context.lua        first-turn context pack
lua/proposal/jail.lua           realpath under project root
lua/proposal/diff.lua           review and apply
lua/proposal/tools/init.lua     registry and dispatch
lua/proposal/tools/read_file.lua
lua/proposal/tools/grep.lua
lua/proposal/tools/diagnostics.lua
lua/proposal/tools/propose_edit.lua
lua/proposal/tools/propose_shell.lua
```

Tests live under `spec/` and load these modules headless. They inject a fake `complete`. They do not spawn `curl`.

## Call rules

| From | May call | Must not call |
| --- | --- | --- |
| `init` | `config`, `ui` | `provider`, `diff` |
| `ui` | `loop` | `curl`, buffer writes |
| `loop` | `provider`, `context`, `tools`, `diff`, `ui` (notify) | `nvim_buf_set_lines` |
| `provider` | `vim.system` (`curl`) | tools, buffers |
| `tools` | `jail`, `vim.diagnostic`, `rg` | `provider`, `nvim_buf_set_lines`, `vim.system` for shell |
| `diff` | buffers, `vim.diff`, `vim.system` for an accepted command | `provider`, `tools` |

`propose_shell` only records the command. `diff` is the module that runs it.

## Project root

`jail.root()`:

1. Run `git rev-parse --show-toplevel` with cwd = Neovim's cwd.
2. On success, that path is the root.
3. On failure, the root is the cwd.

A path is inside the root when `vim.uv.fs_realpath` succeeds and the real path is the root or a descendant. A symlink inside the tree whose target is outside the root is rejected. `..` has no separate case. Realpath covers it.

## Configuration

`setup(opts)` deep-extends these defaults. Unknown keys are ignored.

```lua
{
  provider = {
    base_url = "https://api.openai.com/v1",
    model = "gpt-4.1",
    api_key_env = "OPENAI_API_KEY",
  },
  max_turns = 8,
}
```

`base_url` has no trailing-slash requirement. The provider joins it as `{base_url}/chat/completions`, stripping one trailing slash.

The LazyVim spec is added at `~/dotfiles/nvim/lua/plugins/proposal.lua` during implementation. `coder/claudecode.nvim` is not edited. `<leader>a` stays that plugin. `<leader>A` is the which-key group `Proposal`.

## What v1 leaves out

Inline completion, embeddings, a repo index, ACP, a second agent, session resume, streaming, and a GUI shell. A new `:Proposal` does not continue the previous ask.
