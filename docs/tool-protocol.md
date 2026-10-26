# Tool protocol

The provider speaks the OpenAI chat-completions tool shape. v1 does not stream.

## HTTP

`POST {base_url}/chat/completions`

Headers:

- `Authorization: Bearer <key>`
- `Content-Type: application/json`

The key is `os.getenv(provider.api_key_env)` at the start of the ask. Missing or empty: notify `proposal.nvim: set <env name>` and do not call `curl`.

Body:

```json
{
  "model": "gpt-4.1",
  "messages": [],
  "tools": [],
  "tool_choice": "auto"
}
```

Transport: `curl -sS -m 60 -X POST`. Timeout is 60 seconds.

The assistant message is `choices[0].message`. Tool calls are `message.tool_calls[]` with `id`, `function.name`, and `function.arguments` (a JSON string). Invalid arguments JSON becomes the tool result `error: invalid JSON arguments`.

## System prompt

Fixed in v1. It is not an option.

```
You are a coding agent inside Neovim. You cannot edit files or run commands directly.
Use read_file, grep, and diagnostics to inspect the project.
Use propose_edit to change a file. old must be copied verbatim from the file and must occur once. new is the replacement. An empty new deletes old.
Use propose_shell to run a command. The user accepts or rejects every edit and every command.
Stay inside the project. Prefer a small edit over a rewrite.
When you are done, reply with a short summary and no tool call.
```

## Context pack

The first user message. Built once per ask. Cap: 32,000 characters. When anything is dropped, the pack ends with `[context truncated]`. Drop order: git diff, then diagnostics, then the lines farthest from the cursor.

Sections, each preceded by a header line:

1. `Root: <path>`
2. `File: <path>` or `File: none`
3. Selection, or the cursor window. Visual mode sends the selection with line numbers and does not also send the window. Otherwise 80 lines centered on the cursor (40 above, 40 below), clipped to the buffer. Each line is `<n>|<text>` with `n` 1-based.
4. Diagnostics for the active buffer, at most 20. One line: `<severity>:<line>:<col>:<message>`. Omitted when there are none.
5. `git diff -- <file>` when the root is a git repo and the command succeeds. Omitted on failure.

## Shared tool rules

Every tool returns a string. Validation failures return `error: <reason>` and do not raise. The user is not notified for a single tool error. If a turn's tools are all errors and nothing was queued, the errors still go back to the model on the next turn. The user sees no extra notification.

`read_file` and `grep` call `jail`. `propose_edit` calls `jail` on `path`. `diagnostics` resolves a buffer by path and does not read arbitrary files. `propose_shell` is not jailed.

## read_file

```json
{
  "name": "read_file",
  "parameters": {
    "type": "object",
    "properties": {
      "path": { "type": "string" },
      "offset": { "type": "integer", "description": "1-based. Default 1." },
      "limit": { "type": "integer", "description": "Default 200. Maximum 400." }
    },
    "required": ["path"]
  }
}
```

Reads the loaded buffer whose name realpaths to `path` when one is loaded. Otherwise reads the file. Returns `<n>|<text>` lines. Errors: outside the jail, directory, missing file, `offset` past the end.

## grep

```json
{
  "name": "grep",
  "parameters": {
    "type": "object",
    "properties": {
      "pattern": { "type": "string" },
      "path": { "type": "string", "description": "Default: project root." },
      "glob": { "type": "string" }
    },
    "required": ["pattern"]
  }
}
```

Runs `rg --vimgrep` with cwd = project root. `path` and the search root must pass the jail. The plugin keeps the first 50 output lines. `rg` missing: `error: ripgrep is not installed`.

## diagnostics

```json
{
  "name": "diagnostics",
  "parameters": {
    "type": "object",
    "properties": {
      "path": { "type": "string", "description": "Default: active buffer." }
    }
  }
}
```

`vim.diagnostic.get` on that buffer. At most 30 entries, same line shape as the context pack. Unknown path: `error: no buffer for <path>`.

## propose_edit

```json
{
  "name": "propose_edit",
  "parameters": {
    "type": "object",
    "properties": {
      "path": { "type": "string" },
      "old": { "type": "string", "description": "Non-empty. Must occur once." },
      "new": { "type": "string", "description": "Default empty. Empty deletes old." }
    },
    "required": ["path", "old"]
  }
}
```

Target text is the loaded buffer when the path matches one, otherwise the file bytes. `old` must occur once in that text, or in the virtual document when several edits for the same file arrive in one assistant message. Those edits apply in order against the virtual copy. A later miss errors that call only. Earlier calls stay queued.

Success queues the edit and does not change the buffer. The tool result is held until review finishes. The result the model eventually sees does not contain the new file:

```
<path>
hunk <start_a>+<count_a> accepted
hunk <start_a>+<count_a> rejected
```

`start_a` and `count_a` are 1-based index hunks from `vim.diff` against the original snapshot. One block per file. Hunks are listed in file order.

## propose_shell

```json
{
  "name": "propose_shell",
  "parameters": {
    "type": "object",
    "properties": {
      "command": { "type": "string" }
    },
    "required": ["command"]
  }
}
```

`command` must be non-empty. There is no cwd argument. The queue records the string. The model-facing result after review:

- Accept: `exit <code>` plus a newline and the combined stdout and stderr, truncated to 8,000 characters.
- Reject: `user rejected this command`
- Non-zero exit is still this result, not an abort.

The process runs only after `y`, with cwd = project root, timeout 60 seconds. See [Review](review.md).
