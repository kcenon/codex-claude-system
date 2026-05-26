# OpenAI Codex CLI Reference (for orchestration use)

> **Scope.** This document is a deep, orchestration-oriented reference for the
> OpenAI Codex CLI (sometimes referred to as `codex`). It is the result of a
> WebFetch sweep of the official documentation at `developers.openai.com/codex`
> and the public source repository at `github.com/openai/codex`.
> It is written specifically for the use case of *another system* (e.g. another
> agent CLI such as Claude Code, or a generic orchestrator) calling Codex as a
> worker process. Anything that could not be confirmed against the official
> sources is collected under [Open Questions](#11-open-questions-gaps), not
> silently inferred.
>
> **Document date.** Captured 2026-05-27 from the live developer docs (some
> pages are versioned with the changelog up to **v0.133.0**, May 2026).
>
> **A note on the "Codex" name.** "Codex" historically referred to several
> different OpenAI products. In this document, **"Codex CLI"** refers to the
> Rust-based local coding agent installable as `@openai/codex` /
> `codex` (the `openai/codex` GitHub repo), **not** the
> retired 2021 code-completion model, and **not** the cloud-only
> "Codex Cloud" / "Codex Web" surface (although the CLI integrates with
> them via `codex cloud`, `codex apply`, etc.).

---

## 1. Overview

[OpenAI Codex CLI](https://developers.openai.com/codex/cli) is described as
*"OpenAI's coding agent that you can run locally from your terminal"* — an
open-source, Rust-built agent that reads, edits, and executes code within a
selected directory under a configurable sandbox and approval model.

For an orchestrator, the relevant high-level facts are:

- It is a **local CLI binary** (`codex`). It can be driven interactively (TUI)
  or non-interactively via the `codex exec` subcommand
  ([noninteractive guide](https://developers.openai.com/codex/noninteractive)).
- It supports a **structured JSON Lines event stream** on stdout when called
  with `--json`, and an optional **JSON Schema for the final answer** via
  `--output-schema` ([noninteractive guide](https://developers.openai.com/codex/noninteractive)).
- It enforces a **sandbox** (Seatbelt on macOS, bwrap+seccomp on Linux,
  Windows AppContainer / WSL2 on Windows) and an **approval policy** that
  controls when the agent must pause to ask for permission
  ([agent approvals & security](https://developers.openai.com/codex/agent-approvals-security)).
- It can act as an **MCP client** *and* an **MCP server** (`codex mcp-server`)
  ([MCP overview](https://developers.openai.com/codex/mcp)), and exposes a
  long-running **JSON-RPC 2.0 app-server** (`codex app-server`) over stdio,
  WebSocket, or Unix socket
  ([app-server README](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md)).
- It ships **first-party SDKs in TypeScript and Python** that wrap the same
  app-server / `exec` protocol
  ([SDK overview](https://developers.openai.com/codex/sdk)).

For an orchestrator the practical choice surface is therefore:

| Integration mode | Best for | Granularity |
|---|---|---|
| `codex exec ... --json` | Stateless task: "do this one thing and exit" | Per-task subprocess |
| `codex exec resume ...` | Stateless task that builds on a prior session | Per-task subprocess, session state on disk |
| Codex SDK (TS / Python) | Long-lived in-process integration, streaming, multi-turn | Per-thread |
| `codex app-server` (JSON-RPC) | Polyglot or remote integration; the canonical protocol | Per-thread, full method surface |
| `codex mcp-server` | Plug Codex into an MCP-aware orchestrator as a tool | Per-tool-call |

The rest of this document drills into each of these surfaces.

## 2. Installation & Authentication

### 2.1 Installation

The official [overview page](https://developers.openai.com/codex/cli) advertises:

```bash
# npm (cross-platform; ships a DotSlash launcher under the hood)
npm i -g @openai/codex

# Homebrew (macOS / Linuxbrew)
brew install codex
```

Additionally, [`docs/install.md`](https://github.com/openai/codex/blob/main/docs/install.md)
documents:

- **DotSlash distribution.** GitHub releases ship a DotSlash launcher file
  named `codex` — *"a lightweight commit to source control to ensure all
  contributors use the same version of an executable."* This is the channel
  most relevant for a *pinned* orchestrator dependency.
- **Build from source.** Rust + Cargo, using `just` and `cargo-nextest` as
  build helpers; build with `cargo build`.
- Supported platforms: **macOS 12+**, **Ubuntu 20.04+/Debian 10+**, and
  **Windows 11 via WSL2** (native Windows is also supported per the
  [overview page](https://developers.openai.com/codex/cli), backed by
  AppContainer sandboxing on the security page).
- 4 GB RAM minimum, 8 GB recommended; Git 2.23+ optional but recommended.

### 2.2 Authentication

[`developers.openai.com/codex/auth`](https://developers.openai.com/codex/auth)
documents two principal flows:

1. **ChatGPT sign-in** (interactive default). Opens a local browser; access
   token cached.
2. **OpenAI Platform API key** (recommended for unattended use):

   ```bash
   codex login --with-api-key      # reads the key from stdin
   codex login status               # prints current auth status
   codex logout                     # clears credentials
   ```

3. **Device code flow** (recommended for headless / remote shells):

   ```bash
   codex login --device-auth
   ```

   The CLI prints a URL + code; the operator authenticates in a browser on
   another device. Suitable for SSH sessions and bastion hosts.

4. **Pipe an existing access token** (e.g. one minted out of band):

   ```bash
   printenv CODEX_ACCESS_TOKEN | codex login --with-access-token
   ```

Credentials are persisted to **`~/.codex/auth.json`** (or to the system
keyring if `cli_auth_credentials_store = "keyring"` is set in `config.toml`).
The CLI silently refreshes tokens before they expire.

**Orchestrator implications.**

- For a master that spawns Codex per task, the cleanest path is an API key
  exported in the parent environment (the docs use `CODEX_API_KEY` as the
  canonical name; `OPENAI_API_KEY` is also widely referenced — see
  [Open Questions](#11-open-questions-gaps) for the exact precedence).
- For Docker / ephemeral workers, `docker cp ~/.codex/auth.json
  <container>:/root/.codex/auth.json` is documented as a supported pattern.
- `codex logout` and `codex login status` give an orchestrator a deterministic
  way to assert the worker is authenticated before dispatching a task.

## 3. CLI Surface

The authoritative inventory of subcommands and flags lives at
[`developers.openai.com/codex/cli/reference`](https://developers.openai.com/codex/cli/reference).

### 3.1 Interactive mode

`codex` with no subcommand launches the **Terminal UI** (TUI). It accepts all
global flags plus an optional initial prompt and `--image/-i` attachments.
For an orchestrator this mode is **not** the integration target — its
relevance is limited to ad-hoc human use and to validating prompts before
embedding them in automation.

The TUI exposes a number of `/`-slash commands documented under
[features](https://developers.openai.com/codex/cli/features) and
[best-practices](https://developers.openai.com/codex/learn/best-practices),
e.g. `/model`, `/review`, `/mcp`, `/resume`, `/fork`, `/compact`, `/agent`.

### 3.2 Non-interactive `exec` mode

This is the **primary orchestration entry point**.

```bash
codex exec [OPTIONS] [PROMPT]
codex e    [OPTIONS] [PROMPT]   # alias

# Resume an existing session
codex exec resume [OPTIONS] [SESSION_ID] [PROMPT]
codex exec resume --last "follow-up task"
```

**Output contract** ([noninteractive guide](https://developers.openai.com/codex/noninteractive)):

- **stderr** carries progress / human-oriented chatter.
- **stdout** carries either:
  - the **final agent message** (default), or
  - a **JSONL event stream** when `--json` is set.
- `-o, --output-last-message PATH` additionally writes the final message to a
  file (and still prints to stdout).
- `--output-schema PATH` enforces a JSON Schema on the final answer; combined
  with `--json` it produces a validated, machine-readable result.

**Stdin** is supported in two patterns:

```bash
# (a) prompt is an arg; stdin becomes context concatenated to the prompt
npm test 2>&1 | codex exec "summarize failures and propose fixes"

# (b) explicit '-' makes Codex read the whole prompt from stdin
cat prompt.txt | codex exec -
```

**Resume semantics.** `codex exec resume` re-attaches to a stored session
(`~/.codex/sessions/...`), preserving prior items/turns. As of
[changelog v0.132.0](https://developers.openai.com/codex/changelog),
`exec resume` also accepts `--output-schema`, so a chained orchestration step
can both inherit context *and* enforce a schema.

### 3.3 Command line options (key flags only)

Sourced verbatim from
[`/codex/cli/reference`](https://developers.openai.com/codex/cli/reference).
Flags marked **(global)** apply to most subcommands.

#### Global flags

| Flag | Type | Purpose |
|---|---|---|
| `--add-dir PATH` | path | Grant write access to additional directories beyond the workspace. |
| `--ask-for-approval, -a {untrusted\|on-request\|never}` | enum | Approval policy. |
| `--cd, -C PATH` | path | Set working directory. |
| `--config, -c KEY=VALUE` | k=v | Override a `config.toml` key for this invocation. |
| `--dangerously-bypass-approvals-and-sandbox` | bool | Disable *all* safety. |
| `--dangerously-bypass-hook-trust` | bool | Run hooks without trust validation. |
| `--disable FEATURE` / `--enable FEATURE` | string | Toggle feature flags for this run. |
| `--image, -i PATH[,PATH...]` | paths | Attach images to the prompt. |
| `--model, -m NAME` | string | Override configured model. |
| `--no-alt-screen` | bool | Disable alt-screen TUI mode. |
| `--oss` | bool | Use the local Ollama provider. |
| `--profile, -p NAME` | string | Load a named config profile. |
| `--remote ws://… \| wss://…` | url | Connect this CLI to a remote `app-server`. |
| `--remote-auth-token-env ENV_VAR` | string | Env-var name holding a bearer token for `--remote`. |
| `--sandbox, -s {read-only\|workspace-write\|danger-full-access}` | enum | Sandbox policy. |
| `--search` | bool | Enable *live* web search (default: cached). |
| `PROMPT` | string | Trailing positional initial instruction. |

#### `codex exec` flags

| Flag | Purpose |
|---|---|
| `--cd, -C PATH` | Set workspace root. |
| `--color {always\|never\|auto}` | Output coloring. |
| `--ephemeral` | Do **not** persist session files. |
| `--ignore-rules` | Skip loading `execpolicy` / `.rules`. |
| `--ignore-user-config` | Skip `$CODEX_HOME/config.toml`. |
| `--image, -i PATH[,...]` | Attach images. |
| `--json` | Emit newline-delimited JSON events on stdout. |
| `--output-last-message, -o PATH` | Write final agent message to a file. |
| `--output-schema PATH` | Validate final answer against a JSON Schema. |
| `--skip-git-repo-check` | Allow running outside a Git repo. |

#### Other subcommands of interest

- `codex login [--device-auth | --with-access-token | --with-api-key]`,
  `codex login status`, `codex logout`.
- `codex resume [--all] [--last] [SESSION_ID]` — interactive resume picker.
- `codex fork [--all] [--last] [SESSION_ID]` — branch a session.
- `codex apply TASK_ID` — apply diffs from a Codex Cloud task locally.
- `codex cloud [--attempts 1-4] [--env ENV_ID] [QUERY]`,
  `codex cloud list [--cursor STR] [--env ENV_ID] [--json] [--limit 1-20]`.
- `codex app PATH` — launch the bundled desktop app.
- `codex app-server [--listen ...] [--ws-auth ...] [--ws-shared-secret-file ...]`
  — the JSON-RPC server (see §8 / §9).
- `codex mcp {add | get | list | login | logout | remove}` — manage MCP clients.
- `codex mcp-server` — run Codex itself as an MCP **server** over stdio.
- `codex sandbox [--permissions-profile NAME] [--cd DIR] [--allow-unix-socket PATH]
  [--log-denials] [--include-managed-config]` — run an arbitrary command under
  the same sandbox Codex uses, useful for orchestrator-side sandboxing.
- `codex features {list | enable | disable}` — persistent feature-flag toggles.
- `codex execpolicy --rules PATH... [--pretty] COMMAND...` — evaluate a
  command against an execution policy ruleset.
- `codex completion {bash|zsh|fish|power-shell|elvish}` — shell completions.
- `codex update` — self-update.
- `codex plugin marketplace {add | list | remove | upgrade}` — plugin
  marketplace management.
- `codex debug app-server send-message-v2 USER_MESSAGE` — low-level debug hook.
- `codex debug models [--bundled]` — inspect the model catalog.
- `codex remote-control` — ensure the local app-server daemon runs with
  remote control enabled.

## 4. Output Channels

For an orchestrator the relevant facts about `codex exec` output are
([source](https://developers.openai.com/codex/noninteractive)):

- **Default mode.**
  - `stdout` = the final agent message (think: a single Markdown answer).
  - `stderr` = progress, sandbox notes, model spinner lines, warnings.
- **`--json` mode.** `stdout` becomes a **JSON Lines** stream; every event the
  agent emits during the run is rendered as a single-line JSON object.
- **`--output-schema PATH`.** Enforces JSON Schema validation on the final
  answer. Combine with `--json` for `item.completed` events carrying the
  schema-validated payload.
- **`-o, --output-last-message PATH`.** Final agent message is *also* written
  to a file, while still being printed to stdout — useful for an orchestrator
  that wants to consume stdout *and* preserve a canonical artifact.
- **`--color {always|never|auto}`.** For pipelines, prefer `--color never`
  (auto correctly detects pipes today but explicit is safer).

### 4.1 JSON event vocabulary (`codex exec --json`)

The official noninteractive guide names these event types — *"thread.started,
turn.started, turn.completed, item.\*, and error"* — but the full schema for
each event type is not enumerated on `developers.openai.com`. The most
complete community summary that lines up with observed Codex output is the
[takopi cheatsheet](https://takopi.dev/reference/runners/codex/exec-json-cheatsheet/).
The shape it documents (and which matches the events surfaced by the
[app-server JSON-RPC spec](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md)):

**Stream-level events**

| Type | Required fields | Notes |
|---|---|---|
| `thread.started` | `type`, `thread_id` | One per `exec` invocation. |
| `turn.started` | `type` | Marks beginning of a turn. |
| `turn.completed` | `type`, `usage.input_tokens`, `usage.cached_input_tokens`, `usage.output_tokens` | Success. |
| `turn.failed` | `type`, `error.message` | Failure of the turn. |
| `error` (stream) | `type`, `message` | May be non-fatal (e.g. `"Reconnecting... 1/5"`). |

**Item events** — wrap incremental agent output. Every line has
`type` ∈ {`item.started`, `item.updated`, `item.completed`}, and an
`item` object with `id`, `type`, and per-subtype fields.

| Item `type` | Lifecycle | Key fields |
|---|---|---|
| `agent_message` | completed only | `item.text` (final response) |
| `reasoning` | completed only (if reasoning summaries enabled) | `item.text` |
| `command_execution` | started + completed | `item.command`, `item.aggregated_output` (truncated to ~64 KiB), `item.exit_code`, `item.status` ∈ `in_progress \| completed \| failed` |
| `file_change` | completed only | `item.changes[].path`, `item.changes[].kind` ∈ `add \| delete \| update`, `item.status` |
| `mcp_tool_call` | started + completed | `item.server`, `item.tool`, `item.arguments`, `item.result`, `item.error`, `item.status` |
| `web_search` | completed only | `item.query` |
| `todo_list` | started + updated + completed | `item.items[].text`, `item.items[].completed` (main `item.updated` source) |
| `error` (item) | completed only | `item.message` (non-fatal warning) |

The same field set appears as a JSON-RPC notification surface in the
app-server (e.g. `item/started`, `item/agentMessage/delta`,
`thread/status/changed`).

**Orchestration guidance.** A master that wants exhaustive coverage should:

1. Treat `turn.failed` and `error` (both stream- and item-level) as the
   failure signal. *Do not* rely on `agent_message` presence alone.
2. Use `command_execution.status == "failed"` and `exit_code != 0` to detect
   shell-tool failures that Codex itself recovered from.
3. Persist `thread_id` from `thread.started` to enable `codex exec resume`.
4. Bound the JSONL stream consumer's per-line buffer: `command_execution`
   `aggregated_output` is the largest realistic field and is truncated to
   ~64 KiB per the cheatsheet.

> **Coverage caveat.** Since the per-field schemas are not enumerated on the
> official OpenAI docs site, an orchestrator should defensively treat
> *unknown event types and unknown item subtypes* as opaque rather than
> erroring — Codex has added new item types historically (see changelog
> entries on `todo_list`, `mcp_tool_call`, etc.).

## 5. Sandbox & Approval Model

### 5.1 Sandbox levels

[`developers.openai.com/codex/agent-approvals-security`](https://developers.openai.com/codex/agent-approvals-security)
defines three sandbox modes (flag: `--sandbox, -s` or `sandbox_mode` in
`config.toml`):

| Mode | Filesystem | Network | Notes |
|---|---|---|---|
| `read-only` | Reads only; no writes. | Blocked. | Safest. Useful for "review/analyze and answer" tasks. |
| `workspace-write` (default) | Read-everywhere; write inside the active workspace. `.git`, `.agents`, `.codex` are recursively read-only even inside the workspace. | Blocked by default. | Default for `codex exec` when nothing else is set. Crossing the workspace boundary requires an approval. |
| `danger-full-access` | Unrestricted writes. | Unrestricted network. | Disables technical guardrails entirely. Reserved for isolated VMs/containers. |

Additionally, `--add-dir PATH` grants extra writable roots without dropping
to `danger-full-access`. The official guidance is to **prefer `--add-dir`
over `danger-full-access`**.

### 5.2 Platform implementations

From the [security page](https://developers.openai.com/codex/agent-approvals-security):

- **macOS** — Seatbelt profiles, executed via `sandbox-exec`.
- **Linux** — `bwrap` (namespaces) + `seccomp` syscall filtering.
- **Windows (native)** — AppContainer, with `unelevated` or `elevated`
  variants selectable via a permissions profile.
- **Windows (WSL2)** — inherits the Linux mechanism.

The `codex sandbox` subcommand exposes this same machinery to run *arbitrary*
commands under the Codex sandbox, which an orchestrator can use to enforce
the same trust boundary around its *own* helper commands.

### 5.3 Approval policies

Flag: `--ask-for-approval, -a` (or `approval_policy` in `config.toml`). Per
[security page](https://developers.openai.com/codex/agent-approvals-security):

| Policy | Behavior |
|---|---|
| `never` | Never prompt; the agent operates strictly within the sandbox. **The only safe choice for unattended runs.** |
| `on-request` (default) | Agent runs commands and edits within the workspace automatically; pauses to ask before crossing the workspace boundary, hitting the network, or doing side-effecting actions. |
| `untrusted` | Auto-approves safe read operations; pauses before any state-mutating or externally-invoking command. |
| `on-failure` | Asks for approval when a command/tool fails. |

Plus a **granular** form: `approval_policy = { granular = { … } }` in
`config.toml` keeps specific tool categories interactive while auto-rejecting
others, and `-c approvals_reviewer=auto_review` routes eligible interactive
approvals through an automated reviewer agent.

### 5.4 Recommended combinations

From the same page:

| Intent | Recommended combination |
|---|---|
| Local interactive work | `--sandbox workspace-write --ask-for-approval on-request` |
| Safe browsing / Q&A | `--sandbox read-only --ask-for-approval on-request` |
| **CI / non-interactive read-only** | **`--sandbox read-only --ask-for-approval never`** |
| **Orchestrator worker that must edit** | `--sandbox workspace-write --ask-for-approval never` *plus* a workspace dir that is itself disposable (container, tempdir, or scratch worktree) |
| Auto-review-mediated approvals | add `-c approvals_reviewer=auto_review` |
| Bypass all safety (only in disposable VM/container) | `--dangerously-bypass-approvals-and-sandbox` |

CLI-order caveat for the installed `codex-cli 0.133.0`: `--ask-for-approval`
is accepted as a top-level Codex flag, not after `exec`. In scripts, write
`codex --ask-for-approval never exec --sandbox read-only ...`, not
`codex exec --ask-for-approval never ...`.

### 5.5 What happens when the sandbox blocks an action

Per the [security page](https://developers.openai.com/codex/agent-approvals-security):

1. **Filesystem.** Writes outside the workspace, or to protected `.git` /
   `.agents` / `.codex`, raise an approval request (or fail silently with
   `--ask-for-approval never`).
2. **Network.** Outbound connections fail unless network access is enabled
   *and* approved. With `network_proxy` configured, denials are policy-based
   and DNS-rebinding protected.
3. **Command execution.** Restricted operations either fail or prompt
   depending on `--ask-for-approval`.

For an orchestrator running `--ask-for-approval never`, the practical signal
is: the corresponding `command_execution` item will land as
`status: "failed"` with a non-zero `exit_code`, and the agent message will
typically explain that the sandbox denied the operation. The orchestrator
should **not** treat a sandbox denial as a Codex bug — it is the agent
faithfully reporting "I could not do X under your policy."

## 6. Configuration

### 6.1 Files and precedence

[`developers.openai.com/codex/config-basic`](https://developers.openai.com/codex/config-basic)
defines the lookup chain:

- **User-level**: `~/.codex/config.toml`
- **Project-level**: `.codex/config.toml` (closest to CWD wins, trusted only)
- **System-level**: `/etc/codex/config.toml` on Unix
- **Managed**: `requirements.toml` (admin-pinned, can disable user/project
  overrides — e.g. `allow_managed_hooks_only = true`)

**Precedence, highest to lowest:**

1. CLI flags and `--config key=value` overrides
2. Active profile (`--profile NAME`)
3. Project `.codex/config.toml` (closer to CWD wins)
4. User `~/.codex/config.toml`
5. System `/etc/codex/config.toml`
6. Built-in defaults

Project-scoped config **cannot** override machine-local provider, auth,
notification, profile, or telemetry routing keys.

### 6.2 Key configuration keys

From [`config-reference`](https://developers.openai.com/codex/config-reference):

```toml
# Model
model = "gpt-5.5"
model_provider = "openai"        # or "ollama", "lmstudio", or a custom one
model_context_window = 200000
model_reasoning_effort = "high"   # minimal | low | medium | high | xhigh
model_verbosity = "medium"        # low | medium | high

# Safety
sandbox_mode = "workspace-write"  # read-only | workspace-write | danger-full-access
approval_policy = "on-request"    # untrusted | on-request | never | { granular = { ... } }
default_permissions = ":workspace"

# Search
web_search = "cached"             # cached | live | disabled

# Auth
cli_auth_credentials_store = "auto"  # file | keyring | auto

# Behavior
personality = "pragmatic"          # friendly | pragmatic | none

# Project doc lookup
project_doc_fallback_filenames = ["CLAUDE.md", "AGENT.md"]
project_doc_max_bytes = 32768

[features]
multi_agent = true
memories = true
network_proxy = true
unified_exec = true

[shell_environment_policy]
inherit = "core"                  # all | core | none

[mcp_servers.example]
command = "node"
args = ["./scripts/mcp.js"]
enabled_tools = ["read_file", "list_dir"]
startup_timeout_sec = 10
tool_timeout_sec = 60
default_tools_approval_mode = "auto"   # auto | prompt | approve

[permissions.tight]
# named permissions profile

[profiles.deep-review]
model = "gpt-5.5"
model_reasoning_effort = "xhigh"
sandbox_mode = "read-only"
approval_policy = "never"

[otel]
exporter = "otlp-http"
endpoint = "https://otel.example.com"
```

### 6.3 Environment variables

Confirmed from various official pages:

- `CODEX_API_KEY` — API key for non-interactive auth (referenced by the
  noninteractive guide).
- `CODEX_ACCESS_TOKEN` — ChatGPT access token piped via
  `codex login --with-access-token`.
- `CODEX_HOME` — overrides `~/.codex` for state, config, sessions, auth.
- `RUST_LOG` — standard Rust log filter (set on the parent process of
  `codex` to get more verbose logs).
- `LOG_FORMAT=json` — `app-server` emits structured tracing logs on stderr.

`OPENAI_API_KEY` is referenced in `model_providers` (e.g.
`env_key = "OPENAI_API_KEY"`) but the exact relationship to `CODEX_API_KEY`
is not pinned down in the official docs — see
[Open Questions](#11-open-questions-gaps).

### 6.4 Profiles

Profiles are first-class. They are addressed at runtime with `--profile
NAME` and define an entire scoped override block:

```toml
[profiles.ci]
sandbox_mode = "read-only"
approval_policy = "never"
model = "gpt-5.3-Codex-Spark"
web_search = "cached"
```

Then: `codex exec --profile ci --json "summarize PR"`.

For an orchestrator this is the recommended way to encode *"my worker
configuration"* — keep all worker tuning in a `ci` (or similar) profile
rather than passing dozens of `--config` flags per call.

## 7. SDK

Source: [`developers.openai.com/codex/sdk`](https://developers.openai.com/codex/sdk)
and [`sdk/typescript/README.md`](https://github.com/openai/codex/blob/main/sdk/typescript/README.md).

### 7.1 Languages

- **TypeScript** (production): `npm install @openai/codex-sdk`, Node 18+.
- **Python** (currently labeled experimental in the SDK overview, but bumped
  to first-class auth and Turn API in
  [changelog v0.131.0–v0.132.0](https://developers.openai.com/codex/changelog)):
  the documented installation path is `cd sdk/python && python -m pip install -e .`,
  Python 3.10+. Note the changelog also calls out a migration to the
  `openai-codex` PyPI package, with pinned types, concurrent routing, and
  approval modes.

### 7.2 Architecture

The TypeScript SDK README is unambiguous about how it works:

- The `Codex` class **spawns the `codex` binary** and communicates over
  **JSONL events on stdin/stdout** — i.e., the SDK is a wrapper around the
  same protocol an orchestrator would speak directly.
- Conversations are persisted under `~/.codex/sessions/`. Threads are
  resumable.

### 7.3 API surface

**TypeScript**

```ts
import { Codex } from "@openai/codex-sdk";

const codex = new Codex({
  env: { ...process.env, CODEX_API_KEY: process.env.CODEX_API_KEY },
  config: { sandbox_mode: "workspace-write", approval_policy: "never" },
  baseUrl: "https://api.openai.com",  // override if proxying
});

const thread = codex.startThread({
  workingDirectory: "/path/to/repo",
  skipGitRepoCheck: false,
});

// Buffered run
const turn = await thread.run("Diagnose the test failure");
console.log(turn.finalResponse);

// Streamed run — yields the JSONL events documented in §4.1
for await (const event of thread.runStreamed("Now fix it")) {
  // event.type === "item.completed" | "turn.completed" | ...
}

// Structured output
await thread.run("Return a Failure summary", {
  outputSchema: failureSchema,
});

// Resume by thread_id from a prior process
const resumed = codex.resumeThread(threadId);
```

**Python** (snapshot from SDK docs / changelog):

```python
from openai_codex import Codex, AsyncCodex

with Codex() as codex:
    thread = codex.thread_start(model="gpt-5.5")
    result = thread.run("Your prompt here")  # plain str input supported

# Async variant
async with AsyncCodex() as codex:
    thread = await codex.thread_start(model="gpt-5.5")
    result = await thread.run("Your prompt here")
```

`AppServerConfig(codex_bin=...)` lets a Python embedder point at a custom
Codex binary path.

### 7.4 SDK vs CLI exec

Use **CLI `exec`** when:

- Your orchestrator is shell-native or polyglot.
- Each task is a clean subprocess (no shared state with the orchestrator).
- You want the strongest fault isolation (a Codex crash kills its own
  process, not yours).

Use the **SDK** when:

- You want streaming events in-process without parsing JSONL yourself.
- You want a long-lived `Thread` object with multi-turn `run()` calls
  without re-launching the binary per turn.
- You want structured-output / image-input ergonomics.

Use the **`app-server` directly** (no SDK) when:

- You are in a language without an SDK.
- You want a *single, long-lived* Codex process that an orchestrator drives
  over WebSocket / Unix socket — see §8 / §9.

## 8. MCP Support

Source: [`developers.openai.com/codex/mcp`](https://developers.openai.com/codex/mcp)
and the [CLI reference](https://developers.openai.com/codex/cli/reference).

### 8.1 Codex as MCP client

Codex can call out to external MCP servers. They are registered either via
the CLI:

```bash
codex mcp add github            -- npx @modelcontextprotocol/server-github
codex mcp add docs --url https://docs.example.com/mcp \
    --bearer-token-env-var DOCS_TOKEN
codex mcp list
codex mcp get github
codex mcp login github --scopes repo,read:user
codex mcp logout github
codex mcp remove github
```

…or in `~/.codex/config.toml`:

```toml
[mcp_servers.github]
command = "npx"
args    = ["@modelcontextprotocol/server-github"]
env     = { GITHUB_TOKEN = "..." }
startup_timeout_sec = 10
tool_timeout_sec = 60
enabled_tools = ["create_issue", "search_repos"]
default_tools_approval_mode = "auto"

[mcp_servers.docs]
url = "https://docs.example.com/mcp"
bearer_token_env = "DOCS_TOKEN"
```

Supported transports: **stdio** (local process) and **streamable HTTP**.
Authentication for HTTP servers: bearer-token env var, or OAuth via
`codex mcp login` (configurable via `mcp_oauth_callback_port` /
`mcp_oauth_callback_url`).

### 8.2 Codex as MCP server (`codex mcp-server`)

`codex mcp-server` runs Codex itself as an MCP server speaking over stdio.
This is the single most powerful **integration mode** for an orchestrator
that is already MCP-aware: Codex becomes a *tool* you can call, with all the
sandbox/approval guarantees in place.

Combined with `codex sandbox`, `codex execpolicy`, and the granular
approval policy, this is the easiest way to expose Codex as a hardened
worker to any MCP host without rolling your own JSON-RPC client.

## 9. Orchestration-Relevant Capabilities

This is the section the rest of the document feeds.

### 9.1 Useful capabilities

> **Note.** Codex's role inside an orchestration system can be either the
> orchestrator ("master") that drives other agents or the worker ("subagent")
> that another orchestrator drives. The two roles need different feature
> surfaces — the master cares about long-lived dispatch, multi-channel
> control, and policy enforcement; the worker cares about deterministic I/O,
> structured output, and per-call sandboxing. The capabilities below are
> grouped accordingly. Several capabilities (`--output-schema`, profiles,
> sandbox, `mcp-server`, OpenTelemetry, image inputs) are useful in *both*
> directions and are listed under the role where they are most load-bearing.

#### 9.1.1 When Codex is the orchestrator

- **`codex app-server`.** The same agent, exposed as JSON-RPC 2.0 over
  stdio / WebSocket / Unix socket. Methods like `thread/start`,
  `turn/start`, `turn/steer`, `turn/interrupt`, `thread/list`,
  `command/exec`, `process/spawn`, `fs/readFile`, `fs/writeFile`,
  `fs/watch`, `review/start`, `model/list`, `config/read` make it a
  drop-in primitive for a long-lived orchestrator. Has built-in
  backpressure: rejects with JSON-RPC error `-32001` ("Server overloaded;
  retry later") when overloaded. This is the canonical surface for a host
  process to keep Codex alive across many turns rather than re-spawning
  per task.
- **WebSocket auth.** `--ws-auth capability-token` or `signed-bearer-token`,
  with `--ws-shared-secret-file`, `--ws-audience`, `--ws-issuer`,
  `--ws-max-clock-skew-seconds`, `--ws-token-file`. An orchestrator can mint
  short-lived signed bearer tokens and drive Codex from another host.
- **`codex execpolicy`.** Independent command-evaluator the orchestrator can
  use *before* dispatching a shell command to Codex (or to any other agent),
  to assert it would be allowed by a given policy. Lets a Codex-as-master
  process pre-check workloads that will be handed off to *other* workers
  (Claude, Gemini, a non-LLM tool).
- **Subprocess dispatch via `codex exec --json`.** When Codex is the master,
  the worker-call primitive is still `codex exec` — but here it is
  *invoking* a child agent. The same event vocabulary (`thread.started`,
  `turn.started`, `item.completed`, …) gives the master a clean handle
  on child progress. See §10.2 for a parser pattern that is symmetric in
  this direction.
- **OpenTelemetry.** `[otel]` block exports API requests, prompts, tool
  approvals as structured spans — making the master's dispatch pattern
  attributable in a multi-agent dashboard.

#### 9.1.2 When Codex is a worker called from another orchestrator

- **Non-interactive `exec --json` with `--output-schema`.** Deterministic
  contract: a per-line event stream on stdout, a final schema-validated
  payload, progress on stderr, exit code reflects overall success/failure.
  This is the load-bearing primitive when an *external* orchestrator
  (Claude, a shell pipeline, agent-mux) drives Codex.
- **Session persistence via `thread_id`.** Capture `thread_id` from
  `thread.started`, then chain follow-up `codex exec resume <id>` calls.
  Cheap, stateless from the orchestrator's POV (state lives on disk in
  `~/.codex/sessions`).
- **Profiles.** Encode worker config (`--profile ci`) once, reference by
  name. Keeps subprocess invocations short and reviewable from the
  parent's side.
- **Sandbox + approval policy.** Combined with `--add-dir`, this gives the
  orchestrator a defensible "least privilege" story per task type:
  read-only/never for analysis, workspace-write/never (in a disposable
  workspace) for edits.
- **`codex mcp-server`.** Codex as a tool callable by any MCP host —
  the standard way to expose Codex as a worker to a Claude-led or
  CAO-led orchestrator.
- **Image inputs and structured prompts.** Both CLI (`-i PATH[,PATH...]`)
  and SDK accept images, which lets the orchestrator forward screenshots /
  diagrams from upstream tasks without leaving the protocol.

### 9.2 Limits and caveats

- **Long-running tasks.** Codex's exec stream is single-process; there is no
  documented persistent daemon mode for `exec` itself. For very long tasks,
  prefer the `app-server` (which is designed to be daemon-like) or break
  work into smaller `exec resume` turns.
- **Authentication for parallel workers.** `~/.codex/auth.json` is a single
  file. For high-concurrency orchestration each worker should either (a) use
  its own `CODEX_HOME`, or (b) use API keys via environment variables rather
  than the shared auth.json.
- **Concurrency / rate limits.** The `app-server` has explicit backpressure
  via error `-32001`. The `exec` CLI does not document a comparable signal —
  orchestrators that fan out aggressively should be ready to interpret
  generic non-zero exits and `turn.failed` events as rate-limit failures.
- **Git repo requirement.** `codex exec` by default refuses to run outside
  a Git repository; use `--skip-git-repo-check` in container/scratch
  environments, or always set up a minimal `git init`.
- **Workspace boundary.** "Workspace" is the CWD/Git-root chosen at startup;
  to grant write access to other dirs use `--add-dir`, *not*
  `danger-full-access`.
- **Log volumes.** `command_execution.aggregated_output` is truncated to
  ~64 KiB per the event cheatsheet — an orchestrator that needs the *full*
  log of a long subcommand should re-run the command itself rather than
  rely on the Codex truncation.
- **Approvals in unattended mode.** With `-a never`, *any* operation that
  would normally prompt is silently denied — orchestrators should always
  inspect `turn.failed` and item-level errors, not just `agent_message`.
- **Output schema vs free text.** `--output-schema` only constrains the
  *final* answer, not intermediate events.
- **Hooks trust.** `.codex/` project hooks load only when the project layer
  is trusted; orchestrators that materialize a scratch worktree need to be
  explicit about whether they want project hooks to run.

## 10. Examples

### 10.1 `codex exec` in a shell pipeline

A pipeline runner that lets Codex summarize and triage failing tests:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Read-only / no approvals: the worker analyzes but never writes.
codex --ask-for-approval never exec \
  --sandbox read-only \
  --color never \
  --skip-git-repo-check \
  --output-last-message ./out/last-message.md \
  "$(cat <<'PROMPT'
Read the failing test output from stdin. Categorize each failure as:
- flaky
- product regression
- test bug
Return a Markdown table with columns: test_id, category, evidence, fix_owner.
PROMPT
)" < ./out/test-output.log

# stdout already contains the final Markdown answer; ./out/last-message.md mirrors it.
```

### 10.2 An external orchestrator parsing Codex output

A worker dispatcher that wants both the schema-validated final answer *and*
the per-item event stream (TypeScript, called from a parent Node process):

```ts
import { spawn } from "node:child_process";
import { createInterface } from "node:readline";

type CodexEvent =
  | { type: "thread.started"; thread_id: string }
  | { type: "turn.started" }
  | { type: "turn.completed"; usage: { input_tokens: number; cached_input_tokens: number; output_tokens: number } }
  | { type: "turn.failed"; error: { message: string } }
  | { type: "error"; message: string }
  | { type: "item.started" | "item.updated" | "item.completed"; item: any };

async function runCodexTask(prompt: string, schemaPath: string): Promise<unknown> {
  const proc = spawn("codex", [
    "--ask-for-approval", "never",
    "exec",
    "--json",
    "--sandbox", "workspace-write",
    "--color", "never",
    "--profile", "ci",
    "--output-schema", schemaPath,
    "--output-last-message", "./out/final.json",
    prompt,
  ], {
    cwd: process.env.WORKSPACE,
    env: { ...process.env, CODEX_API_KEY: process.env.CODEX_API_KEY! },
    stdio: ["ignore", "pipe", "pipe"],
  });

  const lines = createInterface({ input: proc.stdout });
  let threadId: string | undefined;
  let final: string | undefined;
  let failed = false;

  for await (const line of lines) {
    if (!line.trim()) continue;
    let evt: CodexEvent;
    try {
      evt = JSON.parse(line);
    } catch {
      // Defensive: treat malformed JSON as a stream-level warning, not fatal.
      console.warn("codex: non-JSON line:", line);
      continue;
    }
    switch (evt.type) {
      case "thread.started":
        threadId = evt.thread_id;
        break;
      case "turn.failed":
        failed = true;
        console.error("codex: turn failed:", evt.error.message);
        break;
      case "item.completed":
        if (evt.item?.type === "agent_message") final = evt.item.text;
        if (evt.item?.type === "command_execution" && evt.item.status === "failed") {
          console.warn("codex command failed:", evt.item.command, "exit", evt.item.exit_code);
        }
        break;
      case "error":
        // Non-fatal unless turn.failed follows.
        console.warn("codex: stream error:", evt.message);
        break;
    }
  }

  const code = await new Promise<number>((res) => proc.on("close", res));
  if (code !== 0 || failed) {
    throw new Error(`codex exec failed (exit=${code}, thread=${threadId ?? "n/a"})`);
  }
  // ./out/final.json holds the schema-validated payload; stdout's final agent_message is the human-friendly view.
  return JSON.parse(await Bun.file("./out/final.json").text());
}
```

This pattern is the load-bearing one for the use case the user described —
*"a master CLI that calls Codex (or another agent) as a worker"*: the
orchestrator owns process lifecycle, the worker owns its sandbox, and the
JSONL stream is the single source of truth for what happened.

Note: a long-lived parent multiplexing several `codex exec` children is still
a self-rolled concern; [openai/codex#4219][codex-headless-4219] (the original
"headless / non-interactive mode" feature request) is **closed** (verified
2026-05-27), but multi-child orchestration patterns are not yet documented
officially.

[codex-headless-4219]: https://github.com/openai/codex/issues/4219 "openai/codex#4219 — headless / non-interactive mode for Codex CLI (closed)"

## 11. Open Questions / Gaps

Items that the official documentation either does not pin down or where the
documentation is internally inconsistent. These should be confirmed
empirically before being relied on in production orchestration.

1. **`CODEX_API_KEY` vs `OPENAI_API_KEY` precedence.** The
   [noninteractive guide](https://developers.openai.com/codex/noninteractive)
   uses `CODEX_API_KEY` as the canonical CI variable. The
   [config-reference](https://developers.openai.com/codex/config-reference)
   examples for `model_providers` use `env_key = "OPENAI_API_KEY"`. The
   *exact* order in which these are consulted (and whether
   `OPENAI_API_KEY` is automatically picked up by the default `openai`
   provider) is not stated explicitly.

2. **Authoritative JSON event schema for `exec --json`.** The official
   noninteractive guide names the event categories but does not publish a
   schema. The shapes used in §4.1 are the [takopi
   cheatsheet](https://takopi.dev/reference/runners/codex/exec-json-cheatsheet/)
   reconciled against the [app-server
   README](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md).
   Field names may differ (e.g. `agent_message` event-shape vs `agentMessage`
   JSON-RPC notification). Treat as "stable enough for prototyping; verify
   per Codex version before locking it in."

3. **Exit-code semantics for `codex exec`.** The noninteractive guide does
   not enumerate exit codes. It is safe to treat exit 0 as success and
   non-zero as failure, but the *meaning* of specific non-zero codes
   (auth error vs sandbox denial vs model rate-limit) is not documented.
   Orchestrators that want fine-grained classification should rely on the
   JSONL `error` / `turn.failed` payloads, not on the exit code alone.

4. **`exec.md` is a stub.** [`docs/exec.md`](https://github.com/openai/codex/blob/main/docs/exec.md)
   in the source repo contains only a single sentence pointing back to
   `developers.openai.com/codex/noninteractive`. There is therefore no
   single authoritative spec file in the repo; the developer-portal page is
   the de facto authority.

5. **Rate limits and concurrency.** The `app-server` documents one
   backpressure signal (`-32001`). The `exec` CLI documents none. Whether
   parallel `codex exec` invocations from one machine are throttled (and
   how) is not described.

6. **Python SDK stability.** The [SDK overview](https://developers.openai.com/codex/sdk)
   still calls Python "experimental", but the
   [changelog](https://developers.openai.com/codex/changelog) describes
   first-class auth, turn APIs, and a migration to `openai-codex`. The
   *current* stability label is therefore ambiguous.

7. **`codex execpolicy` rules format.** The CLI accepts
   `--rules PATH [--rules PATH ...]` and a `COMMAND...` argument, but
   [`docs/execpolicy.md`](https://github.com/openai/codex/blob/main/docs/execpolicy.md)
   is a stub pointing to `developers.openai.com/codex/exec-policy`, which
   was not directly fetchable during this survey. Rule file schema needs
   primary-source confirmation before use.

8. **`requirements.toml` admin model.** The
   [config docs](https://github.com/openai/codex/blob/main/docs/config.md)
   reference `allow_managed_hooks_only = true` in `requirements.toml`, but
   the broader set of admin/managed keys is not enumerated in one place.

9. **Stable `--remote` + WebSocket auth examples.** The flag surface
   (`--remote`, `--remote-auth-token-env`, plus all the `--ws-*` flags on
   `app-server`) is documented in the CLI reference, but no end-to-end
   example of "spin up `codex app-server` on host A with signed-bearer-token
   auth, attach `codex --remote` on host B" was located on the official
   docs site.

## 12. References

- [Codex CLI overview — developers.openai.com/codex/cli](https://developers.openai.com/codex/cli)
- [Codex CLI features — developers.openai.com/codex/cli/features](https://developers.openai.com/codex/cli/features)
- [Codex CLI reference (flags & subcommands) — developers.openai.com/codex/cli/reference](https://developers.openai.com/codex/cli/reference)
- [Non-interactive mode — developers.openai.com/codex/noninteractive](https://developers.openai.com/codex/noninteractive)
- [Codex SDK — developers.openai.com/codex/sdk](https://developers.openai.com/codex/sdk)
- [Codex Changelog — developers.openai.com/codex/changelog](https://developers.openai.com/codex/changelog)
- [MCP overview — developers.openai.com/codex/mcp](https://developers.openai.com/codex/mcp)
- [Agent approvals & security — developers.openai.com/codex/agent-approvals-security](https://developers.openai.com/codex/agent-approvals-security)
- [Authentication — developers.openai.com/codex/auth](https://developers.openai.com/codex/auth)
- [Basic configuration — developers.openai.com/codex/config-basic](https://developers.openai.com/codex/config-basic)
- [Advanced configuration — developers.openai.com/codex/config-advanced](https://developers.openai.com/codex/config-advanced)
- [Configuration reference — developers.openai.com/codex/config-reference](https://developers.openai.com/codex/config-reference)
- [AGENTS.md guide — developers.openai.com/codex/guides/agents-md](https://developers.openai.com/codex/guides/agents-md)
- [Best practices — developers.openai.com/codex/learn/best-practices](https://developers.openai.com/codex/learn/best-practices)
- [openai/codex repo — github.com/openai/codex](https://github.com/openai/codex)
- [docs/exec.md (stub) — github.com/openai/codex/blob/main/docs/exec.md](https://github.com/openai/codex/blob/main/docs/exec.md)
- [docs/sandbox.md (stub) — github.com/openai/codex/blob/main/docs/sandbox.md](https://github.com/openai/codex/blob/main/docs/sandbox.md)
- [docs/install.md — github.com/openai/codex/blob/main/docs/install.md](https://github.com/openai/codex/blob/main/docs/install.md)
- [codex-rs/app-server/README.md — github.com/openai/codex/blob/main/codex-rs/app-server/README.md](https://github.com/openai/codex/blob/main/codex-rs/app-server/README.md)
- [TypeScript SDK README — github.com/openai/codex/blob/main/sdk/typescript/README.md](https://github.com/openai/codex/blob/main/sdk/typescript/README.md)
- [exec --json event cheatsheet (community reference) — takopi.dev](https://takopi.dev/reference/runners/codex/exec-json-cheatsheet/)
