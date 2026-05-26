# References Overview

> Entry point for `docs/references/`. Reads the four investigation reports as one package: what each report covers, what the cross-cutting findings are, and which Open Questions are still outstanding. Use this file to decide which report to open first, and to keep a single audit-trail anchor for the investigation done on **2026-05-27 (Asia/Seoul)**.

---

## 1. What Lives in `docs/references/`

`docs/references/` contains **external facts** collected during the investigation. It is *frozen* in the sense that every fact has a verification date; if a fact changes, the file is updated and the verification date is bumped. Proposals built on top of these facts live in [`../design/architecture.md`](../design/architecture.md), [`../design/security.md`](../design/security.md), and [`../design/worker-contracts.md`](../design/worker-contracts.md), and may evolve separately.

| File | Scope |
| --- | --- |
| [`00-local-environment.md`](00-local-environment.md) | Local CLI versions installed on the investigation date, plus a topic-organized index of every external URL cited in this package. |
| [`00-overview.md`](00-overview.md) | Reading order, cross-cutting findings, and consolidated Open Questions. |
| [`01-codex-cli.md`](01-codex-cli.md) | OpenAI Codex CLI reference (install, exec, output channels, sandbox/approval, config, SDK, MCP). Sectioned with explicit "Codex-as-orchestrator vs Codex-as-worker" capability split (§9.1). |
| [`02-claude-code-cli.md`](02-claude-code-cli.md) | Claude Code CLI reference (headless `-p`, `--bare`, output formats incl. NDJSON wire schema, permission/tool gating, sub-agents, hooks, MCP). |
| [`03-orchestration-patterns.md`](03-orchestration-patterns.md) | Multi-agent CLI orchestration patterns: supervisor/pipeline/fan-out/swarm/router, communication channels, reliability compounding, prior-art case studies, recommendations for a Codex+Claude hybrid. |
| [`04-integration-examples.md`](04-integration-examples.md) | Existing Codex+Claude integration attempts: 16 case studies, direction-of-control taxonomy, capability comparison table, Open Questions on this specific hybrid. |

---

## 2. Reading Order

For a first read, follow this order. Each step has a clear purpose; skip a step only if you already know what it would say.

1. **`00-local-environment.md`** — anchor yourself to the snapshot. Confirm which CLI versions every claim is about.
2. **`01-codex-cli.md` §1, §3, §4, §5, §9** — Codex CLI surface for an orchestrator role. (Skim §6–§8 unless you're sure you need config/SDK/MCP detail.)
3. **`02-claude-code-cli.md` §1, §3, §4, §6, §7, §10** — Claude Code CLI surface for a worker role.
4. **`03-orchestration-patterns.md` §2, §3, §6, §10** — the architectural vocabulary. §10 specifically scopes three candidate hybrids for "Codex + Claude" with explicit trade-offs.
5. **`04-integration-examples.md` §3, §4.5, §6** — what's already been tried, where the gaps are. §4.5 is the closest prior art to the user's design direction.
6. **Hand off to `../design/`** — read [`design/architecture.md`](../design/architecture.md), then [`design/security.md`](../design/security.md), then [`design/worker-contracts.md`](../design/worker-contracts.md). Each design document cites the relevant reference sections.

A reviewer who only has 30 minutes should read §3 (Key Findings), §5 (Consolidated Open Questions), and then jump straight to `design/architecture.md`.

---

## 3. Key Findings Per Report

### 3.1 `01-codex-cli.md` — Codex CLI

1. **`codex exec --json --output-schema` is the canonical orchestration handle.** stdout streams JSONL events; stderr carries progress; `-o <path>` persists the final answer. Official docs name the top-level event families (`thread.started / turn.started / turn.completed / turn.failed / item.* / error`), while per-item fields still require defensive parsing because the full field schema is not published. This is the contract an external orchestrator parses, but it must tolerate unknown event and item subtypes.
2. **Three integration modes coexist:** (a) per-task `codex exec` subprocess, (b) `codex app-server` JSON-RPC 2.0 over stdio/WebSocket/Unix socket (with `--ws-auth signed-bearer-token`), (c) `codex mcp-server` exposing Codex itself as an MCP tool. For a Codex-as-orchestrator design (a) is the simplest starting point; (c) becomes interesting if Codex needs to be addressable by other MCP-aware agents.
3. **Sandbox × approval is a 3×3 matrix.** `--sandbox {read-only|workspace-write|danger-full-access}` × `--ask-for-approval {untrusted|on-request|never}` (plus the documented/deprecated `on-failure`). In the local CLI, `--ask-for-approval` is a top-level Codex flag, so scripted calls should place it before the subcommand, e.g. `codex --ask-for-approval never exec --sandbox read-only ...`.
4. **`§9.1` explicitly separates Codex-as-orchestrator from Codex-as-worker capabilities** — added during the fact-check pass to keep the user's design direction (Codex on top) discoverable from the reference structure.
5. **Open Questions deliberately preserved (9 items)** including the authoritative JSON event schema, `exec` exit-code semantics, and the precedence between `CODEX_API_KEY` and `OPENAI_API_KEY`. Treat any design that depends on one of these as needing live verification first.

### 3.2 `02-claude-code-cli.md` — Claude Code CLI

1. **`claude -p` + `--bare` is the deterministic CI entry point.** `--bare` skips auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and `CLAUDE.md`. Documented as "the future default for `-p`". Critical caveat: bare mode does **not** read `CLAUDE_CODE_OAUTH_TOKEN`, so headless workers must use `ANTHROPIC_API_KEY` or `apiKeyHelper`.
2. **The `--output-format stream-json` wire schema is fully documented** — message types (`system/init`, `assistant`, `user`, `stream_event`, `system/compact_boundary`, `system/api_retry`, `result`) match the SDK TypeScript types 1:1. The `result` event carries `session_id`, `total_cost_usd`, `usage`, `modelUsage`, `num_turns`, `duration_ms`, `terminal_reason`, `structured_output`, `permission_denials`. An orchestrator can make cost, retry, and failure-mode decisions directly from this payload.
3. **Session transcripts are cwd-local.** `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`, with explicit "session files are local to the machine that created them" semantics. Multi-host worker fleets must roll their own transport. `--session-id <UUID>` lets an orchestrator pre-decide the ID; `--fork-session` enables branching.
4. **Permission governance is 5-layer** (managed → CLI flag → local → project → user). For unattended fleets the recommended combination is `--permission-mode dontAsk` with an explicit `permissions.allow` rule set. `--permission-prompt-tool <mcp-tool>` lets an external orchestrator hold the gate via an MCP server. `auto` mode aborts the session on repeated denials in `-p` and is therefore unsuitable as a fleet default.
5. **Subagents come in two flavors:** files (`.claude/agents/*.md` with YAML frontmatter) and inline (`--agents '<json>'`). Inline never touches disk, lives only inside the session — an orchestrator can inject a different sub-agent topology on every call. Limitation: **subagents cannot spawn subagents** (single-level); multi-level fan-out has to go through background agents / agent teams.

### 3.3 `03-orchestration-patterns.md` — Patterns

1. **All studied systems converge on one topology:** *supervisor → workers in isolated worktrees → schema-validated output → quality-gate verifier*. Whether the channel is MCP, file mailbox, or subprocess stdio, the structural shape is the same. The actionable choice is the transport, not the topology.
2. **Reliability compounds catastrophically.** R = R₁ × R₂ × … × Rₙ; 99% reliable components chained 50 times give ~60% system reliability. The MAST taxonomy attributes 41.77% of multi-agent failures to specification ambiguity, 36.94% to coordination, 21.30% to verification gaps. *Soft failures* (plausible-but-wrong outputs) are operationally worse than *hard failures* because they pass silent quality gates.
3. **Both Codex and Claude Code provide native primitives for subprocess orchestration.** `codex exec --json --output-schema` and `claude -p --output-format stream-json --json-schema --bare` between them give: deterministic invocation, structured I/O, pre-approved tool sets, and sandbox isolation. The orchestrator does not need to invent its own RPC layer to start.
4. **`git worktree` + `tmux` is the de-facto isolation standard.** Filesystem state via worktrees, process state via tmux sessions. Two non-obvious caveats: (a) `node_modules` / `.env` must never be shared across worktrees; (b) worktree boundaries solve filesystem races but not context-window or task-coordination drift.
5. **Three candidate hybrids are scoped in §10**, presented as trade-offs rather than prescriptions: (A) supervisor over subprocess workers (simplest, recommended starting point), (B) pipeline with fan-out (2–5 workers, worktree-isolated), (C) file-based mailbox (10+ concurrent workers). A → B is the natural growth path. Across all three, common invariants are: schema-validated output, explicit `--tools` / `--allowedTools` policy, independent verifier, hard budget cap, quality gate, 3-fail rule, replay-able file logs.

### 3.4 `04-integration-examples.md` — Prior Art

1. **`buildoak/agent-mux` (Nick Oak) is the closest prior art to the user's design direction.** Documented: "A Codex main session spawns Opus 4.6 as the GSD coordinator via agent-mux." Channel = direct subprocess calls to `codex`/`claude`/`gemini` binaries with a JSON contract. Treats Codex-as-orchestrator as a first-class case (most prior art treats the reverse).
2. **The official integration is one-way (Claude → Codex), not the user's direction.** `openai/codex-plugin-cc` (released 2026-03-30) gives Claude Code the ability to call Codex; there is no symmetrical official plugin in the other direction. OpenAI Symphony — when it does Codex-as-orchestrator — only orchestrates *other Codex agents*, not Claude. Workload-division blog posts (Sangho Oh, Ivan Bragin) similarly assume Claude planner + Codex executor.
3. **Codex headless used to be a known limitation but has stabilized.** Issue `openai/codex#4219` was open at the time of several blog posts; verified **closed** on 2026-05-27. The remaining concern is long-lived parent processes that multiplex several `codex exec` children — that pattern is still self-rolled (see §4.5 agent-mux for one approach).
4. **MCP is the most-mentioned bidirectional channel.** `raysonmeng/agent-bridge` is the only fully bidirectional bridge found (MCP ↔ Codex app-server WebSocket). Most other integrations are unidirectional and pick a single transport (subprocess or MCP).
5. **Workload-division consensus from prior art:** planning to the long-context model (Claude), surgical execution to the shell-first model (Codex), adversarial review across the divide. This consensus partially conflicts with the user's design direction (Codex on top) — see §4 below for the cross-cutting tension.

---

## 4. Cross-Cutting Findings

These findings only become visible when the four reports are read together.

1. **The two CLIs are structurally symmetric for orchestration.** Both expose: a non-interactive entry (`exec` / `-p`), JSONL/streaming output (`--json` / `--output-format stream-json`), schema-validated final answers (`--output-schema` / `--json-schema`), pre-approved tool sets and sandbox flags. An adapter layer can treat them as instances of the same interface — building a `CodexWorker` and `ClaudeWorker` against a shared protocol is realistic and cheap. (Sources: `01` §3, §4, §5; `02` §3, §4, §6.)

2. **The user's design direction (Codex orchestrator → Claude worker) is the minority direction in public prior art.** Out of 16 case studies in `04`, only one (`agent-mux`) explicitly puts Codex on top with Claude as worker. The majority puts Claude on top. This is not an argument against the user's direction — it is a flag that the design will inherit *fewer recipes from the community* and need to invent more of the integration surface itself. (`04` §3, §4, §6.)

3. **Schema-validated output is the single highest-leverage primitive.** Every reliable system in `03` and every successful integration in `04` uses `--output-schema` (Codex) or `--json-schema` (Claude) at the worker boundary. Free-form text parsing for control flow is the #1 anti-pattern. This is the first thing the design should enforce, not the last. (`02` §4; `03` §4, §9.1, §10; `04` §4.5, §4.8.)

4. **Permission/sandbox is a two-headed problem because both CLIs have their own model.** Codex sandbox levels are filesystem-process boundaries; Claude permission modes are application-level allow/deny rules with a different vocabulary. When Codex runs Claude as a subprocess, the Codex sandbox alone is insufficient — the wrapper must set Claude `--permission-mode`, `--tools` when true tool availability must be restricted, and `--allowedTools` / `--disallowedTools` for permission policy. It must also refuse calls like `claude --dangerously-skip-permissions`. This is the cross-cutting concern that justifies `design/security.md` as a standalone document. (`01` §5; `02` §6; `03` §3, §9; `04` §6.)

5. **Authentication is the silent gotcha.** `--bare` (the recommended deterministic mode for Claude workers) does *not* read `CLAUDE_CODE_OAUTH_TOKEN`. Codex auth precedence between `OPENAI_API_KEY` and `CODEX_API_KEY` is an Open Question. A worker fleet that "just works" interactively can break on the first CI run unless these are set up explicitly per worker process. (`02` §2.2, §3.2; `01` §2.2, Open Questions.)

6. **MCP is a strategic option, not a first-step requirement.** Both CLIs are MCP-capable in both directions, and `raysonmeng/agent-bridge` proves the bidirectional pattern works. But subprocess-with-schema is enough to start, and avoids the protocol-surface tax. Promote MCP only when (a) the orchestrator needs to expose worker capabilities to other MCP-aware tools, or (b) per-call subprocess startup cost becomes a measurable problem. (`01` §8; `02` §9; `03` §3.3; `04` §2.)

7. **Reliability declines geometrically — design the verifier first, not last.** Three reports independently land on the same conclusion: every orchestration stage needs an independent verifier (tests, lint, type-check, schema validation), and the verifier must not be the same model as the worker. Brandon Redmond's pipeline (`03` §8.5) explicitly reverts the worktree on quality-gate failure rather than asking the worker to "try again with the failures fixed". (`03` §6, §9.8; `04` §4.8, §4.9.)

---

## 5. Consolidated Open Questions

Open Questions surfaced by individual reports, organized by impact. Items in **bold** are most likely to block design decisions.

### Codex (from `01` §11)

- **Authoritative JSON event schema for `codex exec --json`** is not published; it has to be inferred from observed traces.
- **Exit-code semantics for `codex exec`** are not documented per error class.
- **`CODEX_API_KEY` vs `OPENAI_API_KEY` precedence** when both are set — undocumented.
- GitHub `docs/exec.md`, `docs/sandbox.md`, `docs/execpolicy.md` are stubs that redirect to `developers.openai.com` — primary content lives at the latter.
- Python SDK stability label is inconsistent across pages.

### Claude Code (from `02` §12)

- **`claude mcp serve` exposed tool surface and stability guarantees** are not documented.
- **Wire format of `--input-format stream-json`** is documented only via TypeScript SDK examples; no formal JSON schema is published.
- `SDKPermissionDenial`, `NonNullableUsage`, `ModelUsage`, `SDKMessageOrigin`, `ApiKeySource` types referenced in `result` / `init` are only visible in SDK typings.
- `structured_output` vs `result.result` interaction during `error_max_structured_output_retries` retries is not exhaustively specified.
- `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` defaults to 10 per session — bounds in-session tool use, **not** a fleet of `claude -p` workers. The orchestrator must throttle externally.

### Integration-specific (from `04` §8)

- Activity / production-readiness of `buildoak/agent-mux`, `ccswarm`, `claude-codex` — single repos with limited public adoption.
- Codex-side equivalents of Claude's `--bare` flag (deterministic CI mode) — exists?
- `milisp/codexia` (referenced in §3.2) — production state not verified.
- `~10% of public GitHub commits / ~326K daily` figure (`04` §7) is single-secondary-source (morphllm); no first-party GitHub data confirms it.
- EloPhanto / Sangho Oh / Ivan Bragin workload-division claims need controlled benchmarks before they can be cited as design principles.

### Patterns / Reliability (from `03`)

- MAST 1st-party paper (NeurIPS 2025) — only cited through secondary blog summaries in `03`.
- MindStudio's "≈3× throughput from worktrees" — primary benchmark not located.

---

## 6. Hand-off to `docs/design/`

Once you've read the references, the design documents express what we propose to build with these facts:

| Design file | What it proposes | Most-cited reference sections |
| --- | --- | --- |
| [`design/architecture.md`](../design/architecture.md) | Subprocess worker pool (baseline) + MCP bridge (alternative). Task spec / result schemas. Parallelism policy. | `01` §3, §4, §9.1.1; `02` §3.2, §4.2, §10; `03` §2.1, §4, §10 |
| [`design/security.md`](../design/security.md) | Combined Codex sandbox + Claude permission matrix. Secret policy. Prompt-injection policy. Audit log requirements. | `01` §5; `02` §6; `04` §6 |
| [`design/worker-contracts.md`](../design/worker-contracts.md) | Worker calling contract. Prompt template. Result JSON schema. Failure handling matrix. Pilot sequence. | `02` §3.2, §4.2, §10; `03` §4 |

Design documents are intentionally separate from references because they **will change** as the system is built and as live verification of Open Questions comes back. References track what was true on 2026-05-27; design tracks what we currently propose. The two should never be merged.
