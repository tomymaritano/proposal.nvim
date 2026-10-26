# Testing

Tests run inside `nvim --headless`. They do not listen on a socket and they do not exec `curl`. `loop` takes `complete` as an argument. Production passes the curl provider. Tests pass a function.

A test that needs a project root creates a temporary directory, runs `git init` there, and sets cwd to it. Symlink cases create the link inside that directory and the target outside it.

## Cases

### Jail

| Case | Expect |
| --- | --- |
| File under the git toplevel | Allowed |
| Path outside the toplevel | `error: …` from `read_file` |
| Symlink inside the tree whose real path is outside | Rejected |
| Directory with no `.git` | Root is the cwd. A child file is allowed. A sibling of the cwd is rejected. |

### propose_edit

| Case | Expect |
| --- | --- |
| `old` absent | `error: …`, nothing queued |
| `old` twice | `error: …`, nothing queued |
| `old` once | Queued. Virtual lines equal the replacement. Buffer bytes unchanged. |
| Two edits, one file, second `old` missing in the virtual document | First stays queued. Second is `error: …`. |
| Empty `old` | `error: …` |

### Apply

Build a buffer with two index hunks. Mark one accepted and one rejected. Finish the file.

- Buffer text equals the merge.
- One `u` restores the snapshot.
- A review where every hunk is rejected does not call `nvim_buf_set_lines` and does not add an undo entry.
- The file on disk is unchanged in all three situations.

### Loop

| Fake `complete` | Expect |
| --- | --- |
| First call: one `propose_edit`. After the tool result: plain text, no tool call. | Review runs. After a scripted accept, the next call receives a tool result that names the path and the hunk decision and does not contain the new file body. The text is shown. The ask ends. |
| Every call returns a tool call. | The provider is called 8 times. The 8th response's tool calls are not run and no review opens. Notify `proposal.nvim: stopped after 8 turns`. |
| First call raises a transport error. | Notify. No tool runs. A later `:Proposal` does not see the failed list. |

Shell tests use a fake runner injected into `diff` instead of `vim.system`. They assert the runner is not called until `y`, and that `n` yields `user rejected this command` without calling the runner.

## Out of the automated suite

The LazyVim keymap check and a live model call are manual, listed as the success checks in the [design spec](superpowers/specs/2026-10-26-proposal-nvim-design.md#success). The headless suite is the bar for a change to `jail`, `tools`, `diff`, or `loop`.
