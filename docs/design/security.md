# Security & Permissions (Proposal)

> This is a design proposal, not external research. Sourced primarily from `../references/`. Subject to change as the system evolves.

In this architecture two coding agents have overlapping authority over the
same workspace: Codex spawns Claude, and Claude itself may invoke file and
shell tools. The Codex sandbox alone is not sufficient — the Claude
permission model, working-directory isolation, and secret policy must be
designed together.

## Principles

- Start at minimum privilege and raise it only per task, never globally.
- Separate read tasks from write tasks; they get different permission
  envelopes.
- Treat all external content (issues, web pages, fetched docs) as untrusted
  input.
- Do not read secret-bearing files unless the user explicitly asked for it
  and the task genuinely requires it.
- No worker pushes directly to a protected branch.
- Global configuration (`~/.codex`, `~/.claude`, shell profile,
  `~/.ssh/config`) is excluded from every task's scope.

## Codex-side boundary

Codex's sandbox limits which files and which network destinations a
model-generated command can touch, and the approval policy decides when the
agent must pause to ask. Authoritative reference:
[Codex CLI Reference §5](../references/01-codex-cli.md#5-sandbox-approval-model).

The sandbox modes are:

| Mode | Filesystem | Network |
| --- | --- | --- |
| `read-only` | Reads only; no writes. | Blocked. |
| `workspace-write` (default for `codex exec`) | Read everywhere; write inside the active workspace. `.git`, `.agents`, `.codex` stay read-only even in-workspace. | Blocked by default. |
| `danger-full-access` | Unrestricted. | Unrestricted. |

The approval policy values are `never` (only safe choice for unattended
runs), `on-request` (default; pauses at boundary crossings), `untrusted`
(approves only safe reads), and `on-failure`. The recommended combination
for a CI / non-interactive read-only run is `read-only` sandbox plus
`never` approvals, and for an orchestrator worker that must edit the
recommendation is `workspace-write` sandbox plus `never` approvals
**together with a workspace that is itself disposable** (container, scratch
directory, or worktree). In the installed CLI, `--ask-for-approval` is a
top-level flag, so a scripted invocation is ordered as
`codex --ask-for-approval never exec --sandbox read-only ...`
([Codex CLI Reference §5.4](../references/01-codex-cli.md#54-recommended-combinations)).

Notes specific to this project:

- When Codex spawns the `claude` binary, the Claude process gets its own
  filesystem and shell tool surface. Codex's `--sandbox workspace-write`
  alone is **not** enough — the wrapper must also pass the right
  `--permission-mode`, `--tools`, `--allowedTools`, and
  `--disallowedTools` to Claude.
- Codex `execpolicy` or hooks should refuse, or escalate to explicit
  approval, any attempted invocation of
  `claude --dangerously-skip-permissions` or `claude --permission-mode
  bypassPermissions`.
- For a worker that needs additional writable roots, prefer
  `--add-dir PATH` over dropping to `danger-full-access`.

Recommended Codex sandbox per orchestrator role:

| Situation | Codex sandbox |
| --- | --- |
| Planning / decomposition / review | `read-only` |
| Writing worker task spec files into the queue | `workspace-write` |
| Fully automated runs inside a dedicated container | `workspace-write`, escalating to `danger-full-access` only when the container itself is the isolation boundary |

## Claude-side boundary

Claude Code exposes per-tool allow/deny rules and a coarser permission mode.
Authoritative reference:
[Claude Code Reference §6](../references/02-claude-code-cli.md#6-tools-permissions-sandbox).

Permission modes ([§6.2](../references/02-claude-code-cli.md#62-permission-modes)):

| Mode | Auto-approves | Best for |
| --- | --- | --- |
| `default` | Reads only | Sensitive work |
| `acceptEdits` | Reads, edits, common FS commands inside cwd / `additionalDirectories` | Reviewed implementation tasks |
| `plan` | Reads only; no edits | Pre-edit exploration |
| `auto` | Everything, gated by a background classifier | Long autonomous tasks (Anthropic API only) |
| `dontAsk` | Only the `permissions.allow` rules + the built-in read-only command set; everything else auto-denies | Locked-down CI |
| `bypassPermissions` | Everything | Container / VM only |

Worker role × recommended Claude configuration:

| Worker role | Permission mode | Tool policy |
| --- | --- | --- |
| Read-only investigator / reviewer | `dontAsk` (with allow-list) or `plan` | `Read`, `Glob`, `Grep` only |
| Implementer (small change) | `acceptEdits` | `Read`, `Glob`, `Grep`, `Edit`, `Write`, narrow `Bash(<verification command>)` |
| Test runner | `dontAsk` | Only the specific test command in `permissions.allow` |
| Risk / threat analyzer | `plan` | Read-only; network disabled |
| Sandboxed experimenter (container only) | `bypassPermissions` (case-by-case approval) | Permitted only when host FS mounts are restricted |

Important distinction: `--allowedTools` pre-approves matching tool uses; it
does not by itself remove every other tool from Claude's context. Use
`--tools` when the worker must only see a closed set of built-in tools, and
use `--disallowedTools` / settings deny rules for explicit denials.

Read-only example:

```sh
claude --bare -p \
  --permission-mode dontAsk \
  --tools "Read,Glob,Grep" \
  --allowedTools "Read" "Glob" "Grep" \
  --disallowedTools "Bash" "Edit" "Write" \
  --no-session-persistence \
  --output-format json \
  "Review the assigned files and return findings only."
```

Write-capable example:

```sh
claude --bare -p \
  --permission-mode acceptEdits \
  --tools "Read,Glob,Grep,Edit,Write,Bash" \
  --allowedTools "Read" "Glob" "Grep" "Edit" "Write" "Bash(npm test *)" \
  --disallowedTools "Bash(git push *)" "Bash(git reset *)" "Bash(rm *)" \
  --output-format json \
  "Implement the assigned change only."
```

`--bare` is the recommended scripted / SDK mode and is documented as the
future default of `-p`. Important caveats:

- Bare mode **does not** read `CLAUDE_CODE_OAUTH_TOKEN`; the wrapper must
  set `ANTHROPIC_API_KEY` or configure an `apiKeyHelper`
  ([Claude Code Reference §10.3](../references/02-claude-code-cli.md#103-authentication-for-headless-ci)).
- Bare mode also skips discovery of hooks, skills, plugins, MCP servers,
  auto memory, and `CLAUDE.md`. Any of these that the worker actually needs
  must be re-injected via `--settings`, `--mcp-config`, `--agents`, or
  `--append-system-prompt-file`.

Two additional governance levers are particularly useful for this project:

- `--permission-prompt-tool <mcp-tool>` routes any permission prompt
  through an orchestrator-owned MCP tool, giving Codex programmatic say
  over individual decisions
  ([Claude Code Reference §10.5](../references/02-claude-code-cli.md#105-governance-levers)).
- `--strict-mcp-config` plus `--mcp-config <file>` guarantees the worker
  only sees the MCP servers the orchestrator authorizes — `.mcp.json` and
  user-level config are ignored.

## Combined matrix: per worker role × Codex sandbox × Claude permission mode

| Worker role | Codex sandbox (parent) | Claude permission mode (child) | Notes |
| --- | --- | --- | --- |
| Planner / aggregator (Codex only) | `codex --ask-for-approval never exec --sandbox read-only` | n/a (no Claude invocation) | Strictly machine-checkable analysis |
| Read-only investigator | `read-only` (Codex stays read-only) | `dontAsk` + read-tool allowlist | Outputs are summaries only |
| Implementer (single owner) | `workspace-write` + `never` over a disposable worktree | `acceptEdits` + scoped Bash allowlist | Edits land in `<repo>/.claude/worktrees/T-####` |
| Test runner | `workspace-write` + `never` | `dontAsk` + only `Bash(<test command>)` in allowlist | Same worktree as implementer or a fresh checkout |
| Migration / lockfile change | `workspace-write` + `on-request` | `acceptEdits`, single worker only | Always requires aggregator approval |
| Container-isolated experiment | `danger-full-access` (in a disposable container) | `bypassPermissions` | Only when the container is the trust boundary; explicit user approval |

The right column intentionally never says "default" or "auto" for the Claude
side: `default` requires a human at the terminal and `auto` is unsafe under
`-p` because repeated permission blocks abort the session
([Claude Code Reference §6.2](../references/02-claude-code-cli.md#62-permission-modes)).

## Secret policy

Default denials:

- Reading `.env`, `.env.*`, secret-manager dumps, cloud credential files,
  SSH keys, API token files.
- Passing secret values as command-line arguments.
- Embedding tokens, cookies, `Authorization` headers, or private-key
  fragments in the worker result JSON.
- Sending code, logs, commit messages, or environment variables to any
  external URL.

Permitted exceptions:

- The user explicitly requested it and the task genuinely needs it.
- The worker only confirms presence/absence (variable name, config key)
  without revealing the value.
- A vetted secret-injection path exists and log masking is verified.

Wrapper-enforced checks:

- Scan the task prompt and the worker output for secret-like patterns
  before emitting either.
- Treat any access to a forbidden path as a hard failure.
- Block obvious exfiltration tools: `Bash(curl *)`, `Bash(wget *)`,
  `Bash(nc *)`, `Bash(ssh *)`, `Bash(scp *)`. These should appear in
  `disallowedTools` by default and be auditable in the resulting argv.
- Reject any argv containing `OPENAI_API_KEY=`, `ANTHROPIC_API_KEY=`,
  `CODEX_API_KEY=`, `Authorization:`, or similar token-bearing tokens.

## External-content & prompt-injection policy

External content (GitHub issues, web pages, fetched documents) routinely
contains adversarial instructions disguised as natural language. Codex's
security documentation already warns about this for its own surface
([Codex CLI Reference §5](../references/01-codex-cli.md#5-sandbox-approval-model)).
This project adopts the following rules:

- External content is always **data to analyze**, never instructions to
  execute.
- Instructions embedded in fetched content ("run this command", "send the
  output to ...", "change this setting") are ignored.
- Any task that requires fetching an external URL is split into a
  read-only worker that fetches and summarizes, plus a separate
  implementation worker that consumes the **summary**, not the raw fetch.
- The summary is normalized (whitespace, code-fence stripping) before being
  passed to a write-capable worker.

## Working directory isolation

Recommended sequence:

1. Read-only tasks run against the original checkout, with the visible tool
   surface restricted through `--tools` and any write tools explicitly denied.
2. Write tasks run inside a per-task git worktree. Claude exposes
   `--worktree <name>` which checks out into
   `<repo>/.claude/worktrees/<name>`
   ([Claude Code Reference §10.2](../references/02-claude-code-cli.md#102-identity-isolation-concurrency)),
   and the wrapper provides the name from the task spec.
3. Tasks that touch the same file are serialized through the queue rather
   than parallelized.
4. Only the **diff** from a finished worktree is reviewed by Codex; the
   worktree itself is the worker's scratch space, not the production tree.
5. A failed worktree is preserved (not auto-deleted) along with its logs so
   the orchestrator and a human reviewer can inspect what happened.

Forbidden paths (default deny):

```text
.git/
.claude/
.codex/
.env
.env.*
id_rsa
id_ed25519
*.pem
*.key
```

The wrapper enforces both ends:

- Builds a Claude `permissions.deny` rule list from `forbidden_paths`.
- Verifies that every entry in `result.changed_files` lies inside
  `allowed_paths` before marking the task `succeeded`.

## Audit log requirements

Every worker run records:

- `task_id`, start time, end time, configured timeout.
- The **sanitized argv** used to spawn `claude` (secrets and tokens
  redacted, never logged in clear).
- The active permission mode and the `allowedTools` / `disallowedTools`
  lists actually applied.
- Working directory, git commit hash, branch / worktree name.
- Paths to the raw stdout, stderr, and (when `--output-format stream-json`
  is used) the event log file.
- The parsed Claude `result` event including `subtype`, `session_id`,
  `total_cost_usd`, `usage`, `terminal_reason`, and `permission_denials`
  ([Claude Code Reference §4](../references/02-claude-code-cli.md#4-output-formats)).
- The list of changed files with a diffstat.
- Every verification command actually executed, with its exit code.
- For Codex-side runs: the `thread_id` from `thread.started` (so the run
  can be resumed via `codex exec resume`) and the `usage` block from
  `turn.completed` ([Codex CLI Reference §4.1](../references/01-codex-cli.md#41-json-event-vocabulary-codex-exec-json)).

## Approval-required conditions

The following must **not** run unattended. They require an explicit Codex
aggregator decision or a user prompt:

- `claude --permission-mode bypassPermissions` or
  `--dangerously-skip-permissions`.
- `codex --sandbox danger-full-access` or
  `--dangerously-bypass-approvals-and-sandbox`.
- Outbound network writes.
- Dependency installation or large lockfile changes.
- Database migrations, infrastructure / IAM changes.
- Pushes to a protected branch, any force-push.
- Access to a secret-bearing file.
- Large-scale file deletion or move operations.
- Any change to project settings, agent configuration, or global agent
  configuration.

## References

- [Codex CLI Reference §5 — Sandbox & Approval Model](../references/01-codex-cli.md#5-sandbox-approval-model)
- [Codex CLI Reference §5.4 — Recommended combinations](../references/01-codex-cli.md#54-recommended-combinations)
- [Codex CLI Reference §4.1 — JSON event vocabulary](../references/01-codex-cli.md#41-json-event-vocabulary-codex-exec-json)
- [Claude Code Reference §6 — Tools, Permissions, Sandbox](../references/02-claude-code-cli.md#6-tools-permissions-sandbox)
- [Claude Code Reference §6.2 — Permission modes](../references/02-claude-code-cli.md#62-permission-modes)
- [Claude Code Reference §6.3 — Permission rule syntax](../references/02-claude-code-cli.md#63-permission-rule-syntax)
- [Claude Code Reference §10.2 — Identity, isolation, concurrency](../references/02-claude-code-cli.md#102-identity-isolation-concurrency)
- [Claude Code Reference §10.3 — Authentication for headless / CI](../references/02-claude-code-cli.md#103-authentication-for-headless-ci)
- [Claude Code Reference §10.5 — Governance levers](../references/02-claude-code-cli.md#105-governance-levers)
