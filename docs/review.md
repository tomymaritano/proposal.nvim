# Review

Review runs when a turn queued at least one `propose_edit` or `propose_shell`. Immediate tool results wait in memory. They are sent with the proposal summaries, in tool-call order, after review returns.

File order is the order of each file's first `propose_edit` in the assistant message. The command line shows `Proposal i/n` for the file currently open. Shell commands come after every file, in the order they were queued.

## File diff

1. Snapshot the original lines. Loaded buffer text wins over disk.
2. An unloaded file is `bufadd` + `bufload` before the split.
3. Build proposed lines by applying that file's queued `old`/`new` replacements to the snapshot, in order.
4. Hunks are `vim.diff(original, proposed, { result_type = "indices" })`. Each hunk starts undecided.
5. Open a vertical diff. Left: the real buffer, `nomodifiable` while this file is in review. Right: a scratch buffer, `buftype=nofile`, `nomodifiable`, filled with the proposed lines.

Both windows, normal mode, buffer-local:

| Key | Effect |
| --- | --- |
| `]c` / `[c` | Next and previous hunk. Vim's diff motions. |
| `ga` | Mark the hunk that contains the cursor line accepted. |
| `gr` | Mark that hunk rejected. |
| `gA` | Mark every undecided hunk accepted. |
| `q` | Finish the file. Undecided hunks count as rejected. |
| `<C-c>` | Abort the ask. |

`ga` and `gr` replace the previous mark. They do not change bytes. A signcolumn sign reads `accepted` or `rejected`. The user can flip a hunk until `q`.

On `q`:

1. Clear `nomodifiable` on the real buffer.
2. Build merged lines: accepted hunks take the proposed side, rejected and undecided hunks take the snapshot.
3. If the merge differs from the snapshot, `nvim_buf_set_lines` once for the whole buffer. That is one undo entry.
4. If the merge equals the snapshot, do not call `nvim_buf_set_lines`. The buffer's modified flag is left as it was.
5. Close the scratch window.
6. Do not `:w`.

A loaded file that was already modified is snapshotted from the buffer, so the proposal applies on top of unsaved text.

## Shell confirm

After the last file, each queued command gets a floating window with the command and the project root.

| Key | Effect |
| --- | --- |
| `y` | Run it. |
| `n` | Tool result `user rejected this command`. |
| `<C-c>` | Abort the ask. |

`y` runs `vim.system` with `cwd` = project root and a 60 second timeout. The plugin does not wrap the command in a sandbox and does not reject metacharacters. `:ProposalStop` during the wait kills the process and aborts the ask, so that command produces no tool result.

## Abort

`<C-c>` or `:ProposalStop` during review:

- Discard proposals that have not been applied.
- Clear `nomodifiable` on any buffer this review set.
- Close scratch and floating windows.
- Stop the loop. No further provider call.
- Files already applied earlier in this review stay applied, each with its own undo entry.

Notify `proposal.nvim: aborted`.
