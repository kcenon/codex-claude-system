# Claude Code CLI Reference (for orchestration use)

> Reference compiled for orchestrators that drive Claude Code (`claude`) as a worker. Focus: headless execution, structured I/O, session lifecycle, permission/tool gating, sub-agents, hooks, and MCP. Sourced from the official Claude Code / Agent SDK documentation at `code.claude.com` and `docs.claude.com`. All quoted CLI/JSON/YAML excerpts are taken verbatim from the linked pages; speculation is marked `[INFERRED]`.

---

## 1. Overview

Claude Code (`claude`) is Anthropic's official coding agent CLI. The same binary serves three roles relevant to an external orchestrator:

1. An **interactive REPL** (default `claude` invocation).
2. A **headless / "print" mode** (`claude -p "<prompt>"`) that runs one task, emits structured output, and exits. This is the entry point the docs explicitly recommend for "scripts and CI/CD" and "non-interactive mode" ([Run Claude Code programmatically](https://code.claude.com/docs/en/headless)).
3. An **embedded agent loop** via the Agent SDK (Python / TypeScript). Both SDKs and `claude -p` "give you the same tools, agent loop, and context management that power Claude Code" ([Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)).

For an external orchestrator (e.g. Codex CLI as master), the practical contract is:

- Spawn `claude` as a subprocess.
- Pipe a prompt in (`-p "..."` or stdin).
- Read structured events from stdout (`--output-format json|stream-json`).
- Govern capability via flags (`--permission-mode`, `--tools`, `--allowedTools`, `--disallowedTools`, `--mcp-config`, `--settings`, `--agents`).
- Resume context across calls via `--session-id`, `--resume`, or `--continue`.

> Billing note: "Starting June 15, 2026, Agent SDK and `claude -p` usage on subscription plans will draw from a new monthly Agent SDK credit, separate from your interactive usage limits." ([Run Claude Code programmatically](https://code.claude.com/docs/en/headless))

## 2. Installation & Authentication

### 2.1 Install

Source: [Advanced setup](https://code.claude.com/docs/en/setup).

The native installer is recommended ("Native installations automatically update in the background"):

```bash
# macOS, Linux, WSL
curl -fsSL https://claude.ai/install.sh | bash

# Homebrew (does not auto-update)
brew install --cask claude-code           # stable channel
brew install --cask claude-code@latest    # rolling

# npm (same native binary, pulled in via per-platform optional dependency)
npm install -g @anthropic-ai/claude-code
```

Verify with `claude --version` and `claude doctor`. System requirements include macOS 13+, Windows 10 1809+/Server 2019+, Ubuntu 20.04+, Debian 10+, Alpine 3.19+, "4 GB+ RAM, x64 or ARM64 processor", and a live internet connection ([setup](https://code.claude.com/docs/en/setup#system-requirements)).

A specific version may be pinned: `claude install 2.1.118` / `claude install stable` / `claude install latest` ([CLI commands](https://code.claude.com/docs/en/cli-usage)).

### 2.2 Authenticate

Source: [Authentication](https://code.claude.com/docs/en/authentication).

Credential precedence (highest first):

1. Cloud provider creds when `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, or `CLAUDE_CODE_USE_FOUNDRY` is set.
2. `ANTHROPIC_AUTH_TOKEN` — sent as `Authorization: Bearer` (LLM gateway / proxy).
3. `ANTHROPIC_API_KEY` — sent as `X-Api-Key`. "In non-interactive mode (`-p`), the key is always used when present."
4. `apiKeyHelper` script output (rotating creds, vault-backed tokens). Default refresh: 5 min or on HTTP 401; tunable via `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`.
5. `CLAUDE_CODE_OAUTH_TOKEN` — long-lived (one-year) OAuth token issued by `claude setup-token`; requires Pro/Max/Team/Enterprise plan.
6. Subscription OAuth from `/login`.

> "[Bare mode](https://code.claude.com/docs/en/headless#start-faster-with-bare-mode) does not read `CLAUDE_CODE_OAUTH_TOKEN`. If your script passes `--bare`, authenticate with `ANTHROPIC_API_KEY` or an `apiKeyHelper` instead."

Credential storage:
- macOS: encrypted Keychain.
- Linux: `~/.claude/.credentials.json` mode `0600`.
- Windows: `%USERPROFILE%\.claude\.credentials.json` (inherits profile ACL).

`claude auth status` exits 0 if logged in, 1 if not — usable as a CI gate ([CLI commands](https://code.claude.com/docs/en/cli-usage)).

## 3. Execution Modes

### 3.1 Interactive (REPL)

```bash
claude                       # bare REPL
claude "explain this project"  # REPL with initial prompt
```

Not relevant to most orchestrator paths because permission prompts and Shift+Tab mode cycling require a TTY. Mode is set by flag in non-interactive runs.

### 3.2 Headless / `-p` (print) mode

```bash
claude -p "Find and fix the bug in auth.py" \
  --tools "Read,Edit,Bash" \
  --allowedTools "Read" "Edit" "Bash(pytest *)"
```

Quoting [Run Claude Code programmatically](https://code.claude.com/docs/en/headless): "Add the `-p` (or `--print`) flag to any `claude` command to run it non-interactively. All [CLI options](https://code.claude.com/docs/en/cli-usage) work with `-p`."

Key headless-only / headless-aware flags ([CLI flags](https://code.claude.com/docs/en/cli-usage)):

| Flag | Purpose |
|---|---|
| `-p`, `--print` | Print response without interactive mode. |
| `--bare` | Skip auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and `CLAUDE.md` for reproducible CI runs. Sets `CLAUDE_CODE_SIMPLE`. |
| `--input-format text\|stream-json` | Specify input format for print mode. |
| `--output-format text\|json\|stream-json` | Output format for print mode (see §4). |
| `--include-partial-messages` | Include partial streaming events; requires `--print --output-format stream-json`. |
| `--include-hook-events` | Include all hook lifecycle events in the output stream; requires `--output-format stream-json`. |
| `--max-turns N` | Limit agentic turns (print mode only). Exits with an error on hit. |
| `--max-budget-usd N` | Stop after spending N USD (print mode only). |
| `--fallback-model <id>` | Auto-fallback when the default model is overloaded (`-p` and background only; ignored in interactive). |
| `--exclude-dynamic-system-prompt-sections` | Move per-machine sections out of the cached system prompt to improve cache reuse across users/machines. Recommended for scripted multi-user workloads. |
| `--permission-prompt-tool <mcp-tool>` | Have an MCP tool handle permission prompts in non-interactive mode. |
| `--json-schema <schema>` | Return validated JSON matching the schema (print mode only). |
| `--no-session-persistence` | Don't save the session to disk (print mode only). Same effect as `CLAUDE_CODE_SKIP_PROMPT_HISTORY=1`. |
| `--replay-user-messages` | Re-emit user messages from stdin back on stdout (requires `stream-json` both ways). |
| `--init`, `--maintenance` | Run Setup hooks with the matching matcher before the session (print mode only). |
| `--init-only` | Run Setup + SessionStart hooks then exit. |

> *Local-CLI caveat (snapshot 2026-05-27):* `--max-turns` and `--permission-prompt-tool` are documented at the linked sources but were **not** listed in the locally installed `claude --help` output for `claude 2.1.150` (see [`00-local-environment.md` §1](00-local-environment.md#1-local-cli-snapshot)). Treat them as doc-confirmed; verify with a live invocation before depending on either in an orchestrator implementation.

Bare mode tools and what you must re-add yourself ([Run Claude Code programmatically](https://code.claude.com/docs/en/headless#start-faster-with-bare-mode)):

| To load | Use |
|---|---|
| System prompt additions | `--append-system-prompt`, `--append-system-prompt-file` |
| Settings | `--settings <file-or-json>` |
| MCP servers | `--mcp-config <file-or-json>` |
| Custom agents | `--agents <json>` |
| A plugin | `--plugin-dir <path>`, `--plugin-url <url>` |

> "`--bare` is the recommended mode for scripted and SDK calls, and will become the default for `-p` in a future release."

Stdin is read in non-interactive mode; "As of Claude Code v2.1.128, piped stdin is capped at 10MB. If you exceed the cap, Claude Code exits with a clear error and a non-zero status."

### 3.3 Session resume (`--resume`, `--continue`)

Source: [CLI](https://code.claude.com/docs/en/cli-usage), [Work with sessions](https://code.claude.com/docs/en/agent-sdk/sessions).

```bash
claude -c                                    # continue most recent session in cwd
claude -c -p "Check for type errors"         # continue via -p
claude -r "<session-id-or-name>" "Finish this PR"
claude --resume auth-refactor
claude --session-id "550e8400-e29b-41d4-a716-446655440000"   # use a specific UUID
claude --resume <id> --fork-session          # branch a new session_id from <id>
```

Key behaviours:

- `--continue` / `-c` resumes "the most recent conversation in the current directory" (includes sessions that called `/add-dir`).
- `--resume` / `-r` resumes by session ID or display name. As of v2.1.144 background sessions show up in the picker marked `bg`.
- `--session-id <UUID>` lets the caller pin a UUID for the new conversation — useful when the orchestrator wants to mint IDs externally.
- `--fork-session` creates a new ID from an existing session's history so the original is preserved.
- `--name`, `-n` sets a display name, also usable with `claude --resume <name>`.
- Session transcripts are stored locally at `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`, where `<encoded-cwd>` is the absolute working directory with every non-alphanumeric character replaced by `-` (so `/Users/me/proj` becomes `-Users-me-proj`). "Session files are local to the machine that created them."

[INFERRED] To resume on another host, either copy the `.jsonl` file to the same path / cwd, or re-derive state from the orchestrator's own log.

## 4. Output Formats

Source: [Run Claude Code programmatically — Get structured output](https://code.claude.com/docs/en/headless#get-structured-output), [TypeScript SDK reference](https://code.claude.com/docs/en/agent-sdk/typescript), [Stream responses in real-time](https://code.claude.com/docs/en/agent-sdk/streaming-output).

`--output-format` options:

- `text` (default) — plain text.
- `json` — single JSON object with the result + session metadata. "The response payload includes `total_cost_usd` and a per-model cost breakdown, so scripted callers can track spend per invocation."
- `stream-json` — newline-delimited JSON; one event per line.

### 4.1 `json` example

```bash
claude -p "Summarize this project" --output-format json
```

The textual result lives in `result`. Recommended parse:

```bash
claude -p "Summarize this project" --output-format json | jq -r '.result'
```

With `--json-schema`, the structured payload appears under `structured_output`:

```bash
claude -p "Extract the main function names from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}' \
  | jq '.structured_output'
```

### 4.2 `stream-json` event types

```bash
claude -p "Explain recursion" --output-format stream-json --verbose --include-partial-messages
```

Each stdout line is a JSON object. The TypeScript SDK exposes the same types Claude Code emits ([typescript reference](https://code.claude.com/docs/en/agent-sdk/typescript)), so the schemas below also describe the line-delimited JSON wire format:

**`system / init`** — first event, session metadata:

```ts
type SDKSystemMessage = {
  type: "system";
  subtype: "init";
  uuid: UUID;
  session_id: string;
  agents?: string[];
  apiKeySource: ApiKeySource;
  betas?: string[];
  claude_code_version: string;
  cwd: string;
  tools: string[];
  mcp_servers: { name: string; status: string }[];
  model: string;
  permissionMode: "default" | "acceptEdits" | "bypassPermissions" | "plan" | "dontAsk" | "auto";
  slash_commands: string[];
  output_style: string;
  skills: string[];
  plugins: { name: string; path: string }[];
};
```

Init also reports plugin load errors via `plugin_errors` and is preceded by `system/plugin_install` events when `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` is set ([headless](https://code.claude.com/docs/en/headless)).

**`assistant`** — completed Claude turn:

```ts
type SDKAssistantMessage = {
  type: "assistant";
  uuid: UUID;
  session_id: string;
  message: BetaMessage; // id, content, model, stop_reason, usage
  parent_tool_use_id: string | null;
  error?: SDKAssistantMessageError;
};
```

`SDKAssistantMessageError` values: `'authentication_failed' | 'oauth_org_not_allowed' | 'billing_error' | 'rate_limit' | 'invalid_request' | 'model_not_found' | 'server_error' | 'max_output_tokens' | 'unknown'`.

**`user`** — user-side messages (real or synthetic, e.g. tool_use_result):

```ts
type SDKUserMessage = {
  type: "user";
  uuid?: UUID;
  session_id?: string;
  message: MessageParam;
  parent_tool_use_id: string | null;
  isSynthetic?: boolean;
  shouldQuery?: boolean;
  tool_use_result?: unknown;
  origin?: SDKMessageOrigin;
};
```

**`stream_event`** — emitted only when `--include-partial-messages` is set; wraps a raw Claude API event:

```ts
type SDKPartialAssistantMessage = {
  type: "stream_event";
  event: BetaRawMessageStreamEvent;
  parent_tool_use_id: string | null;
  uuid: UUID;
  session_id: string;
};
```

Common `event.type` values: `message_start`, `content_block_start`, `content_block_delta` (with `delta.type` of `text_delta` or `input_json_delta`), `content_block_stop`, `message_delta`, `message_stop` ([streaming-output](https://code.claude.com/docs/en/agent-sdk/streaming-output)).

Filter pattern (text only):

```bash
claude -p "Write a poem" --output-format stream-json --verbose --include-partial-messages | \
  jq -rj 'select(.type == "stream_event" and .event.delta.type? == "text_delta") | .event.delta.text'
```

**`system / compact_boundary`** — emitted when context compaction runs:

```ts
type SDKCompactBoundaryMessage = {
  type: "system";
  subtype: "compact_boundary";
  uuid: UUID;
  session_id: string;
  compact_metadata: { trigger: "manual" | "auto"; pre_tokens: number };
};
```

**`system / api_retry`** — emitted on retryable API errors ([headless](https://code.claude.com/docs/en/headless)):

| Field | Type | Description |
|---|---|---|
| `type` | `"system"` | message type |
| `subtype` | `"api_retry"` | identifies this as a retry event |
| `attempt` | integer | current attempt number, starting at 1 |
| `max_retries` | integer | total retries permitted |
| `retry_delay_ms` | integer | milliseconds until the next attempt |
| `error_status` | integer or null | HTTP status, or null for connection errors |
| `error` | string | `authentication_failed`, `oauth_org_not_allowed`, `billing_error`, `rate_limit`, `invalid_request`, `model_not_found`, `server_error`, `max_output_tokens`, or `unknown` |
| `uuid` | string | unique event identifier |
| `session_id` | string | session the event belongs to |

**`result`** — last event of the run. Two variants ([typescript reference](https://code.claude.com/docs/en/agent-sdk/typescript)):

```ts
type SDKResultMessage =
  | {
      type: "result";
      subtype: "success";
      uuid: UUID;
      session_id: string;
      duration_ms: number;
      duration_api_ms: number;
      is_error: boolean;
      api_error_status?: number | null;
      num_turns: number;
      result: string;             // the human-readable final answer
      stop_reason: string | null;
      ttft_ms?: number;
      total_cost_usd: number;
      usage: NonNullableUsage;    // input/output/cache token counts
      modelUsage: { [modelName: string]: ModelUsage };
      permission_denials: SDKPermissionDenial[];
      structured_output?: unknown; // present when --json-schema was passed
      deferred_tool_use?: { id: string; name: string; input: Record<string, unknown> };
      terminal_reason?: TerminalReason;
      fast_mode_state?: "on" | "off" | "cooldown";
      origin?: SDKMessageOrigin;
    }
  | {
      type: "result";
      subtype:
        | "error_max_turns"
        | "error_during_execution"
        | "error_max_budget_usd"
        | "error_max_structured_output_retries";
      uuid: UUID;
      session_id: string;
      duration_ms: number;
      duration_api_ms: number;
      is_error: boolean;
      num_turns: number;
      stop_reason: string | null;
      total_cost_usd: number;
      usage: NonNullableUsage;
      modelUsage: { [modelName: string]: ModelUsage };
      permission_denials: SDKPermissionDenial[];
      errors: string[];
      terminal_reason?: TerminalReason;
      fast_mode_state?: "on" | "off" | "cooldown";
      origin?: SDKMessageOrigin;
    };
```

`TerminalReason` values: `"completed" | "max_turns" | "tool_deferred" | "aborted_streaming" | "aborted_tools" | "hook_stopped" | "stop_hook_prevented" | "blocking_limit" | "rapid_refill_breaker" | "prompt_too_long" | "image_error" | "model_error"`.

Practical contract for an orchestrator:
- Parse line-by-line, dispatch on `type` / `subtype`.
- Treat any `result.subtype` not equal to `"success"` as a worker failure; `result.subtype === "error_max_structured_output_retries"` specifically means a `--json-schema` constraint could not be satisfied ([structured outputs](https://code.claude.com/docs/en/agent-sdk/structured-outputs#error-handling)).
- Record `session_id` from the first `init` event for follow-up `--resume`.
- Accumulate `total_cost_usd` per worker call.

## 5. Input Channels

Source: [Run Claude Code programmatically](https://code.claude.com/docs/en/headless), [Streaming Input](https://code.claude.com/docs/en/agent-sdk/streaming-vs-single-mode).

Five ways to give Claude Code work:

1. **Prompt argument**: `claude -p "task"`.
2. **Piped stdin (text)**:
   ```bash
   cat build-error.txt | claude -p 'concisely explain the root cause' > output.txt
   ```
   Capped at 10 MB. For larger inputs, write to a file and reference the path in the prompt.
3. **`--input-format stream-json`** — newline-delimited JSON `user` messages on stdin. The TypeScript example demonstrates the line shape ([streaming-vs-single-mode](https://code.claude.com/docs/en/agent-sdk/streaming-vs-single-mode)):
   ```json
   {"type":"user","message":{"role":"user","content":"Analyze this codebase for security issues"},"parent_tool_use_id":null}
   ```
   With image attachments:
   ```json
   {"type":"user","message":{"role":"user","content":[{"type":"text","text":"Review this architecture diagram"},{"type":"image","source":{"type":"base64","media_type":"image/png","data":"<base64>"}}]},"parent_tool_use_id":null}
   ```
   Use with `--input-format stream-json --output-format stream-json` for a true streaming I/O channel. `--replay-user-messages` echoes each input on stdout for ack.
4. **Files referenced in the prompt** — Claude reads files using the `Read` tool subject to permission rules.
5. **Slash commands** — only available in interactive mode. "User-invoked [skills](https://code.claude.com/docs/en/skills) like `/commit` and [built-in commands](https://code.claude.com/docs/en/commands) are only available in interactive mode. In `-p` mode, describe the task you want to accomplish instead." ([headless](https://code.claude.com/docs/en/headless)).

## 6. Tools, Permissions, Sandbox

### 6.1 Configuration scopes

Source: [Settings](https://code.claude.com/docs/en/settings), [Permissions](https://code.claude.com/docs/en/permissions).

Effective settings precedence (highest first):

1. Managed (MDM / org policy) — `/Library/Application Support/ClaudeCode/managed-settings.json` and equivalents.
2. CLI flags.
3. Local — `.claude/settings.local.json` (gitignored).
4. Project — `.claude/settings.json` (in VCS).
5. User — `~/.claude/settings.json`.

`--settings <file-or-json>` overrides matching keys for the session; `--setting-sources user,project,local` lets the orchestrator restrict which scopes load.

### 6.2 Permission modes

Source: [Choose a permission mode](https://code.claude.com/docs/en/permission-modes).

| Mode | Auto-approved | Best for |
|---|---|---|
| `default` | Reads only | Sensitive work, getting started |
| `acceptEdits` | Reads, file edits, common FS commands (`mkdir`, `touch`, `mv`, `cp`, `rm`, `rmdir`, `sed`) inside cwd / `additionalDirectories` | Iterating on reviewed code |
| `plan` | Reads only; no edits | Pre-edit exploration |
| `auto` | "Everything, with background safety checks" via a separate classifier; not on Bedrock/Vertex/Foundry | Long autonomous tasks |
| `dontAsk` | Only `permissions.allow` rules + the built-in read-only command set; everything else auto-denies | Locked-down CI |
| `bypassPermissions` | Everything (including writes to `.git`, `.claude`, `.vscode`, `.idea`, `.husky` as of v2.1.126). Only `rm -rf /` and `rm -rf ~` still prompt as circuit breaker | Containers / VMs only |

Set via `--permission-mode <name>` or `permissions.defaultMode` in settings. `--dangerously-skip-permissions` is equivalent to `--permission-mode bypassPermissions` and is blocked when running as root/sudo on Linux/macOS unless inside a recognised sandbox.

For orchestrators, the most CI-safe pairings ([headless](https://code.claude.com/docs/en/headless#auto-approve-tools)):
- `--permission-mode dontAsk` + an exhaustive `permissions.allow` list (deterministic, no prompts).
- `--permission-mode acceptEdits` for "lint fix" style tasks where edits are reviewed downstream.
- `--permission-mode auto` only on accounts that meet the auto requirements (Sonnet 4.6 / Opus 4.6 / 4.7, Anthropic API only, admin opt-in on Team/Enterprise). In `-p` mode, "repeated blocks abort the session since there is no user to prompt" ([permission-modes](https://code.claude.com/docs/en/permission-modes#when-auto-mode-falls-back)).

### 6.3 Permission rule syntax

Source: [Permissions](https://code.claude.com/docs/en/permissions#permission-rule-syntax).

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run build)",
      "Bash(npm run *)",
      "Bash(git commit *)",
      "Read(~/.zshrc)",
      "WebFetch(domain:example.com)",
      "mcp__github__*",
      "Agent(my-custom-agent)"
    ],
    "ask":   ["Bash(git push *)"],
    "deny":  ["Bash(curl *)", "Read(./.env)", "Read(./.env.*)", "Read(./secrets/**)", "WebFetch"],
    "defaultMode": "acceptEdits",
    "additionalDirectories": ["../docs/"]
  }
}
```

Evaluation order: **deny → ask → allow**, first match wins. A managed-level deny cannot be overridden by `--allowedTools`. `--disallowedTools` adds restrictions beyond managed.

CLI equivalents:
- `--allowedTools "Bash(git log *)" "Bash(git diff *)" "Read"` — pre-approve without prompts.
- `--disallowedTools "Bash(rm *)" "Edit"` — bare tool name removes the tool from context; scoped form blocks matching calls only.
- `--tools "Bash,Edit,Read"` — restrict which built-in tools are even available (use `""` to disable all, `"default"` for all).

Notable Bash specifics:
- Wildcards: `Bash(ls *)` matches `ls -la` but not `lsof`; `Bash(ls*)` matches both.
- Compound commands (`&&`, `||`, `;`, `|`, `|&`, `&`, newline) must each match independently.
- Process wrappers `timeout`, `time`, `nice`, `nohup`, `stdbuf`, bare `xargs` are stripped before matching.
- Built-in read-only allowlist runs without prompts in every mode: `ls`, `cat`, `echo`, `pwd`, `head`, `tail`, `grep`, `find`, `wc`, `which`, `diff`, `stat`, `du`, `cd`, and read-only `git` forms.

Read/Edit rules follow gitignore semantics with anchor prefixes: `//abs`, `~/home`, `/project-root`, `path` or `./path` (cwd-relative).

### 6.4 Sandboxing (Bash)

Source: [Configure the sandboxed Bash tool](https://code.claude.com/docs/en/sandboxing).

OS-level isolation for the Bash tool only (macOS Seatbelt, Linux/WSL2 bubblewrap + socat). Independent of permission modes.

```json
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": false,
    "excludedCommands": ["docker *", "kubectl *"],
    "filesystem": {
      "allowWrite": ["~/.kube", "/tmp/build"],
      "denyWrite": ["/etc"],
      "denyRead":  ["~/.aws/credentials"],
      "allowRead": ["."]
    },
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org"],
      "deniedDomains":  ["sensitive.cloud.example.com"]
    }
  }
}
```

Useful properties for an orchestrator:

- "When sandboxing is enabled with `autoAllowBashIfSandboxed: true`, which is the default, sandboxed Bash commands run without prompting even if your permissions include `ask: Bash(*)`."
- `allowUnsandboxedCommands: false` disables the `dangerouslyDisableSandbox` escape hatch — Claude can no longer fall back to unsandboxed execution.
- Network proxy enforces allowlist by hostname only; "the contents of encrypted connections are not examined."
- Native Windows is unsupported; use WSL2.
- `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` strips Anthropic and cloud credentials from subprocesses.

## 7. Subagents

Source: [Create custom subagents](https://code.claude.com/docs/en/sub-agents).

A subagent is "a specialized AI assistant that handles specific types of tasks" with its own fresh context window, system prompt, tool set, and permission mode. Built-ins: `Explore` (read-only Haiku), `Plan` (used by plan mode), `general-purpose`, plus helpers `statusline-setup` and `claude-code-guide`.

### 7.1 Definition

File-based: Markdown with YAML frontmatter under `.claude/agents/` (project) or `~/.claude/agents/` (user). Both directories are scanned recursively.

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
```

Supported frontmatter fields (only `name` and `description` are required):

| Field | Notes |
|---|---|
| `name` | Unique lowercase-hyphen identifier; hooks receive this as `agent_type`. |
| `description` | Used by Claude to decide when to delegate. |
| `tools` | Allowlist (omitted = inherit all). Use `Skill` only via the `skills` field, not here. |
| `disallowedTools` | Denylist; applied before `tools`. |
| `model` | `sonnet`, `opus`, `haiku`, full ID (e.g. `claude-opus-4-7`), or `inherit`. |
| `permissionMode` | `default | acceptEdits | auto | dontAsk | bypassPermissions | plan`. Ignored for plugin subagents. |
| `maxTurns` | Cap on agentic turns. |
| `skills` | Preload Skill content at startup. |
| `mcpServers` | Scoped MCP servers (inline or name reference). Ignored for plugin subagents. |
| `hooks` | Lifecycle hooks active only while this agent runs. |
| `memory` | `user | project | local` — enables a persistent memory dir at `~/.claude/agent-memory/<name>/` etc. |
| `background` | `true` to always run in background. |
| `effort` | `low | medium | high | xhigh | max`. |
| `isolation` | `worktree` for an isolated git worktree. |
| `color` | UI color. |
| `initialPrompt` | Auto-submitted first user turn when this agent is the main session agent. |

Scope priority (highest first): Managed → `--agents` CLI flag → `.claude/agents/` → `~/.claude/agents/` → plugin `agents/`.

### 7.2 Orchestrator-relevant flags

```bash
# Inline session-only definitions (no file on disk)
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer. Use proactively after code changes.",
    "prompt": "You are a senior code reviewer. Focus on code quality, security, and best practices.",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "sonnet"
  },
  "debugger": {
    "description": "Debugging specialist for errors and test failures.",
    "prompt": "You are an expert debugger. Analyze errors, identify root causes, and provide fixes."
  }
}'

# Run the entire session as a specific agent (replaces default system prompt)
claude --agent code-reviewer

# Disable a built-in agent
claude --disallowedTools "Agent(Explore)"
```

`--agents` accepts the same frontmatter fields as files plus a `prompt` field for the system prompt. `--agent` flips the main thread itself to that agent's prompt/tools/model and persists across `--resume`.

### 7.3 Subagent invariants

- Subagents cannot spawn other subagents (deepest level only).
- Subagents do not see the parent's conversation; the delegating call passes a fresh task message plus `CLAUDE.md` and git status (except `Explore` and `Plan` which skip both).
- Subagent transcripts persist independently at `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl` and survive parent compaction; cleanup respects `cleanupPeriodDays` (default 30).
- Background subagents auto-deny anything that would prompt — they share the session's already-granted permissions only.

## 8. Hooks

Source: [Hooks reference](https://code.claude.com/docs/en/hooks).

Hooks are user-defined integrations that fire on Claude Code lifecycle events. Five types: `command`, `http`, `mcp_tool`, `prompt`, `agent`.

### 8.1 Lifecycle events

- **Session-level**: `SessionStart`, `Setup` (with matchers `init`, `maintenance`), `SessionEnd`.
- **Per-turn**: `UserPromptSubmit`, `UserPromptExpansion`, `Stop`, `StopFailure`.
- **Agentic loop**: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `PermissionRequest`, `PermissionDenied`, `SubagentStart`, `SubagentStop`.
- **Async**: `Notification`, `FileChanged`, `CwdChanged`, `ConfigChange`, `InstructionsLoaded`, `PreCompact`, `PostCompact`, `WorktreeCreate`, `WorktreeRemove`, `Elicitation`, `ElicitationResult`, `TaskCreated`, `TaskCompleted`, `TeammateIdle`.

Hooks defined in subagent frontmatter automatically convert `Stop` → `SubagentStop` at runtime.

### 8.2 Hook contract

Hooks receive JSON on stdin with at minimum:

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse"
}
```

Tool-use events add `tool_name`, `tool_input`, and `tool_use_id`. Subagent contexts add `agent_id`, `agent_type`.

Exit code semantics:

- **0** — success; stdout is parsed for JSON output.
- **2** — blocking. For `PreToolUse`, `PermissionRequest`, `UserPromptSubmit`, `UserPromptExpansion` it blocks the action. For `Stop`, `SubagentStop`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `ConfigChange`, `PreCompact` it prevents the action. For `PostToolUse*` it surfaces stderr to Claude. Other events ignore the exit code.
- **Other** — non-blocking error.

JSON output (any event):

```json
{
  "continue": true,
  "suppressOutput": false,
  "stopReason": "...",
  "systemMessage": "Warning to user",
  "terminalSequence": "<OSC escape>",
  "hookSpecificOutput": { "hookEventName": "PreToolUse", "additionalContext": "..." }
}
```

`PreToolUse` decision control:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow" | "deny" | "ask" | "defer",
    "permissionDecisionReason": "Destructive command blocked by hook",
    "additionalContext": "Optional context for Claude",
    "modifiedInput": { "command": "modified command here" }
  }
}
```

Important precedence: "Hook decisions do not bypass permission rules. Deny and ask rules are evaluated regardless of what a PreToolUse hook returns, so a matching deny rule blocks the call and a matching ask rule still prompts even when the hook returned `\"allow\"` or `\"ask\"`. ... A hook that exits with code 2 stops the tool call before permission rules are evaluated." ([permissions](https://code.claude.com/docs/en/permissions#extend-permissions-with-hooks)).

### 8.3 Hook configuration

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "if": "Bash(rm *)",
        "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh",
        "timeout": 30
      }]
    }],
    "PostToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "/path/to/lint-check.sh",
        "timeout": 60
      }]
    }]
  },
  "allowedHttpHookUrls": ["https://hooks.example.com/*"],
  "disableAllHooks": false
}
```

HTTP hook:

```json
{
  "type": "http",
  "url": "http://localhost:8080/hooks/pre-tool-use",
  "timeout": 30,
  "headers": { "Authorization": "Bearer $MY_TOKEN" },
  "allowedEnvVars": ["MY_TOKEN"]
}
```

Matcher syntax: exact name (`Bash`), `|`-separated list (`Edit|Write`), or JS regex (`mcp__memory__.*`). Default timeouts: 600s for command/http/mcp_tool, 30s for prompt, 60s for agent.

Orchestrator note: with `claude -p --output-format stream-json --include-hook-events`, all hook lifecycle events appear in the output stream, so the master process can observe and intercept tool calls without writing scripts.

## 9. MCP Integration

Source: [MCP](https://code.claude.com/docs/en/mcp).

Claude Code is both an MCP **client** (connecting to external MCP servers) and an MCP **server** (`claude mcp serve` exposes Claude Code itself to other apps).

### 9.1 As MCP client

Transports: `http` (recommended), `sse` (deprecated), `stdio` (local).

CLI:

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp
claude mcp add --transport stdio db -- npx -y @bytebase/dbhub --dsn "postgresql://..."
claude mcp add --transport http github https://api.githubcopilot.com/mcp/ \
  --header "Authorization: Bearer $GITHUB_PAT"

claude mcp list
claude mcp get github
claude mcp remove github

# Pre-built JSON
claude mcp add-json weather-api '{"type":"http","url":"https://api.weather.com/mcp"}'

# Within Claude Code
/mcp           # auth status + connectors
```

Scopes (highest precedence first): Local → Project (`.mcp.json`) → User (`~/.claude.json`) → Plugin → Claude.ai connectors.

`.mcp.json` example with env interpolation:

```json
{
  "mcpServers": {
    "api-server": {
      "type": "http",
      "url": "${API_BASE_URL:-https://api.example.com}/mcp",
      "headers": { "Authorization": "Bearer ${API_KEY}" }
    },
    "internal": {
      "type": "http",
      "url": "https://mcp.internal.example.com",
      "headersHelper": "/opt/bin/get-mcp-auth-headers.sh"
    }
  }
}
```

`--mcp-config <file-or-json>` loads servers for the session; `--strict-mcp-config` uses only those, ignoring `.mcp.json` / settings.

Authentication: OAuth 2.0 happens via `/mcp` in an interactive session; for headless, pre-configure credentials with `--client-id` / `--client-secret` / `--header` and use `--callback-port` to pin the OAuth redirect port.

Tool naming: rules use `mcp__<server>__<tool>` (e.g. `mcp__github__list_prs`), and MCP-exposed prompts surface as slash commands: `/mcp__github__pr_review 456`.

Output limits: default warn at 10k tokens, max 25k; override with `MAX_MCP_OUTPUT_TOKENS=50000`. Per-tool override via `_meta.anthropic/maxResultSizeChars`.

Tool deferral / "tool search" is enabled by default — only tool names load at session start, full schemas on demand. Disable with `ENABLE_TOOL_SEARCH=false`, or auto-tune via `ENABLE_TOOL_SEARCH=auto:5` (5% of context). Exempt a server with `"alwaysLoad": true`.

### 9.2 As MCP server

```bash
claude mcp serve
```

Claude Desktop config example:

```json
{
  "mcpServers": {
    "claude-code": {
      "type": "stdio",
      "command": "/full/path/to/claude",
      "args": ["mcp", "serve"],
      "env": {}
    }
  }
}
```

[INFERRED] An orchestrator could expose Claude Code as a tool to another MCP-aware agent runtime via this mode rather than spawning `claude -p` for each task. The docs don't elaborate on the contract beyond installation; treat this as exploratory.

## 10. Orchestration-Relevant Capabilities

This is the synthesis aimed at the Codex-CLI-as-orchestrator scenario.

### 10.1 Worker invocation primitives

- **Single shot**: `claude --bare -p "<prompt>" --output-format json --tools "Read,Glob,Grep" --allowedTools "Read" "Glob" "Grep"` → parse `result` / `structured_output` / `total_cost_usd` / `usage`.
- **Long-running with mid-stream visibility**: `claude --bare -p "<prompt>" --output-format stream-json --include-partial-messages --include-hook-events` → consume NDJSON.
- **Schema-constrained answer**: append `--json-schema '<schema>'` and read the `structured_output` field from the final `result` event; on failure inspect `subtype === "error_max_structured_output_retries"`.
- **Background**: `claude --bg "<prompt>"` returns immediately with a session ID; manage with `claude logs <id>`, `claude attach <id>`, `claude respawn <id>`, `claude stop <id>`, `claude rm <id>`.

### 10.2 Identity, isolation, concurrency

- Each `claude -p` invocation is a fresh process and (by default) a fresh session. Pass `--session-id <UUID>` to mint deterministic IDs from the orchestrator.
- `--no-session-persistence` (or `CLAUDE_CODE_SKIP_PROMPT_HISTORY=1`) prevents writing transcripts to disk — useful for ephemeral CI workers.
- Multiple sessions can run concurrently in different cwds; transcripts segregate by `~/.claude/projects/<encoded-cwd>/`.
- `--worktree <name>` (or `-w`) starts Claude in `<repo>/.claude/worktrees/<name>` for isolated filesystem state per worker.
- `--add-dir` extends file access without granting `.claude/` config discovery from those paths.
- `--exclude-dynamic-system-prompt-sections` improves prompt-cache hit rate across workers running the same template.

### 10.3 Authentication for headless / CI

- Preferred: `claude setup-token` to mint a one-year `CLAUDE_CODE_OAUTH_TOKEN`, then export it in CI.
- API-key path: set `ANTHROPIC_API_KEY`; in `-p` mode, the key is used unconditionally if set.
- Bare mode (`--bare`) does **not** read `CLAUDE_CODE_OAUTH_TOKEN`. Use `ANTHROPIC_API_KEY` or an `apiKeyHelper` instead. This matters because bare mode is the recommended scripted path.
- `claude auth status` for liveness checks (exit 0/1).

### 10.4 Cost, budget, and stop signals

- `total_cost_usd` is on every `result` event (both success and error variants).
- `--max-budget-usd N` aborts the run when exceeded (yields `result.subtype === "error_max_budget_usd"`).
- `--max-turns N` caps agentic loops (`error_max_turns`).
- `terminal_reason` in `result` distinguishes `completed`, `max_turns`, `tool_deferred`, `aborted_streaming`, `aborted_tools`, `hook_stopped`, `stop_hook_prevented`, `blocking_limit`, `rapid_refill_breaker`, `prompt_too_long`, `image_error`, `model_error`.
- `system/api_retry` events surface transient failures during the run.

### 10.5 Governance levers

- `--settings <inline-json>` injects per-call permission and hook config without touching disk.
- `--permission-prompt-tool <mcp-tool>` routes prompts (if any) to an orchestrator-owned MCP tool — the cleanest pattern for headless approval pipelines.
- `--strict-mcp-config` + `--mcp-config <file>` guarantees only orchestrator-supplied MCP servers are visible.
- `--disable-slash-commands` blanks out skills/commands for tight prompt isolation.
- Managed settings keys like `allowManagedPermissionRulesOnly`, `allowManagedHooksOnly`, `allowManagedMcpServersOnly`, `allowManagedReadPathsOnly`, `allowManagedDomainsOnly` let an MDM-style policy lock the box down even if the spawned process tries to widen.

### 10.6 Known limits / gotchas

- Stdin cap: 10 MB.
- Auto mode in `-p` aborts the session after repeated blocks; not safe as the default for unattended fleets unless you've configured trusted infrastructure.
- Subagents cannot spawn subagents; for parallel fan-out beyond one level, use background agents or agent teams.
- Slash commands and `AskUserQuestion` don't work in `-p`.
- Session files don't move across hosts automatically; bring your own transport or shared FS.

## 11. Examples

### 11.1 `claude -p` + JSON result parsing

```bash
# CI-friendly: bare mode, deterministic tools, JSON out.
result_json="$(
  claude --bare -p "Summarize the changes in CHANGELOG.md" \
    --tools "Read" \
    --allowedTools "Read" \
    --permission-mode dontAsk \
    --output-format json \
    --no-session-persistence
)"

text=$(jq -r '.result' <<<"$result_json")
cost=$(jq -r '.total_cost_usd' <<<"$result_json")
sid=$(jq -r '.session_id' <<<"$result_json")
subtype=$(jq -r '.subtype' <<<"$result_json")

if [[ "$subtype" != "success" ]]; then
  echo "Worker failed: $subtype" >&2
  exit 1
fi
echo "session=$sid cost=\$$cost"
printf '%s\n' "$text"
```

Schema-constrained variant:

```bash
claude --bare -p "Extract the main function names from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}' \
  --tools "Read,Glob,Grep" \
  --allowedTools "Read,Glob,Grep" \
  --permission-mode dontAsk \
  | jq '.structured_output.functions'
```

### 11.2 Multi-turn with `--resume`

```bash
# Turn 1 — capture session_id
sid=$(claude --bare -p "Review the database layer for hot queries" \
  --tools "Read,Glob,Grep" \
  --allowedTools "Read,Glob,Grep" \
  --permission-mode dontAsk \
  --output-format json | jq -r '.session_id')

# Turn 2 — same context, no re-exploration
claude --bare -p "Now propose a fix for the worst offender" \
  --resume "$sid" \
  --tools "Read,Edit,Glob,Grep" \
  --allowedTools "Read,Edit,Glob,Grep" \
  --permission-mode acceptEdits \
  --output-format json | jq -r '.result'

# Turn 3 — branch into an alternative without losing turn 2
sid_alt=$(claude --bare -p "Instead, refactor it asynchronously" \
  --resume "$sid" --fork-session \
  --tools "Read,Edit,Glob,Grep" \
  --allowedTools "Read,Edit,Glob,Grep" \
  --output-format json | jq -r '.session_id')

echo "main=$sid alt=$sid_alt"
```

### 11.3 Stream-json output with hook + custom subagent gating

`.claude/agents/db-reader.md`:

```markdown
---
name: db-reader
description: Execute read-only database queries. Use when analyzing data or generating reports.
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "${CLAUDE_PROJECT_DIR}/.claude/hooks/validate-readonly-query.sh"
---

You are a database analyst with read-only access. Execute SELECT queries to answer questions about the data.
```

`.claude/hooks/validate-readonly-query.sh`:

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if echo "$COMMAND" | grep -iE '\b(INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|TRUNCATE|REPLACE|MERGE)\b' > /dev/null; then
  echo "Blocked: Only SELECT queries are allowed" >&2
  exit 2
fi
exit 0
```

Drive it from the orchestrator:

```bash
claude -p "Use the db-reader agent to compute monthly revenue for Q1" \
  --output-format stream-json --verbose \
  --include-partial-messages --include-hook-events \
  --allowedTools "Agent(db-reader)" \
  --permission-mode dontAsk \
| jq -c '
    select(
      .type == "result"
      or (.type == "system" and .subtype == "init")
      or (.type == "stream_event" and .event.type == "content_block_start" and .event.content_block.type == "tool_use")
      or (.type == "system" and .subtype == "api_retry")
    )
  '
```

The hook intercepts every Bash command issued by the subagent; the orchestrator only sees aggregated NDJSON it can route into its own task store.

## 12. Open Questions / Gaps

- **`claude mcp serve` contract**: docs cover installation but not the exposed tool surface or stability guarantees. Validate empirically before designing an orchestrator that depends on it.
- **Wire format of `--input-format stream-json`**: documented via TypeScript SDK examples but no formal JSON schema is published. Treat the example shapes as the de-facto contract.
- **`SDKPermissionDenial`, `NonNullableUsage`, `ModelUsage`, `SDKMessageOrigin`, `ApiKeySource`**: referenced in `result` and `init` schemas but their field-level definitions are only visible in SDK typings. Pull from `@anthropic-ai/claude-agent-sdk` types when implementing a strict parser.
- **`structured_output` vs `result.result`** when both `--json-schema` and a verbose answer exist: the docs say `result` carries the agent's human answer and `structured_output` the validated payload, but ordering rules during retries (`error_max_structured_output_retries`) are not exhaustively specified.
- **Session resume across hosts**: docs explicitly warn this is local-only; the recommended `SessionStore` adapter exists only in the SDK, not via the CLI. Plan a transport in the orchestrator.
- **Auto mode classifier latency / cost**: documented to add "a round-trip before execution"; not quantified.
- **Cross-version stability**: many flags include version notes (e.g. `--enable-auto-mode` removed in 2.1.111, stdin cap added in 2.1.128, background-session resume UX in 2.1.144, bypassPermissions protected-path behaviour changed in 2.1.126). Pin a `minimumVersion` in managed settings or check `claude --version`.
- **`max_thinking_tokens` + streaming**: explicitly incompatible; if extended thinking is needed, drop streaming.
- **Concurrency limits**: `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` defaults to 10 per session — useful but doesn't bound a fleet of `claude -p` workers; the orchestrator must throttle externally.

## 13. References

- Run Claude Code programmatically (headless): https://code.claude.com/docs/en/headless
- CLI reference: https://code.claude.com/docs/en/cli-usage
- Settings: https://code.claude.com/docs/en/settings
- Hooks: https://code.claude.com/docs/en/hooks
- Permissions: https://code.claude.com/docs/en/permissions
- Permission modes: https://code.claude.com/docs/en/permission-modes
- Sandboxing: https://code.claude.com/docs/en/sandboxing
- Subagents: https://code.claude.com/docs/en/sub-agents
- MCP integration: https://code.claude.com/docs/en/mcp
- Agent SDK overview: https://code.claude.com/docs/en/agent-sdk/overview
- Agent SDK — Streaming output: https://code.claude.com/docs/en/agent-sdk/streaming-output
- Agent SDK — Streaming vs single mode: https://code.claude.com/docs/en/agent-sdk/streaming-vs-single-mode
- Agent SDK — Sessions: https://code.claude.com/docs/en/agent-sdk/sessions
- Agent SDK — Structured outputs: https://code.claude.com/docs/en/agent-sdk/structured-outputs
- Agent SDK — TypeScript reference: https://code.claude.com/docs/en/agent-sdk/typescript
- Authentication: https://code.claude.com/docs/en/authentication
- Setup: https://code.claude.com/docs/en/setup
- Environment variables: https://code.claude.com/docs/en/env-vars
- Repository (anthropics/claude-code) — example settings: https://github.com/anthropics/claude-code/tree/main/examples/settings
- TypeScript SDK source: https://github.com/anthropics/claude-agent-sdk-typescript
- Python SDK source: https://github.com/anthropics/claude-agent-sdk-python
