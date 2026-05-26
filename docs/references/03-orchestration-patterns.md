# Multi-Agent CLI Orchestration Patterns

> Research date: 2026-05-27. Scope: prior art for systems that orchestrate multiple AI coding CLIs (Claude Code, Codex CLI, Gemini CLI, Aider, Cursor, etc.) under a single coordinator. This document is the foundation for the Codex-as-orchestrator + Claude-Code-as-worker design discussions. It is descriptive (what others have built) — not prescriptive — except for the explicit Recommendations section at the end, which presents trade-offs rather than verdicts.

---

## 1. Problem Statement

### 1.1 Why multi-CLI orchestration

A single coding-agent CLI is bounded by three structural walls that no amount of prompt tuning can move:

1. **Context window**: large refactors, multi-package monorepos, and long debugging chains exceed what any single session can hold coherently. Addy Osmani frames this directly — single agents face "context limits, inability to specialize, and coordination challenges" ([Osmani](https://addyosmani.com/blog/code-agent-orchestra/)).
2. **Specialization**: a model and CLI tuned for "reason about a 12k-line refactor" produces different outputs than one tuned for "quickly iterate UI" or "explain a race condition." EloPhanto's swarm explicitly assigns Codex to backend logic and race conditions, Claude Code to frontend iteration, and Gemini CLI to UI polish and security ([EloPhanto](https://dev.to/elophanto/how-i-orchestrate-claude-code-codex-and-gemini-cli-as-a-swarm-4p3c)).
3. **Throughput**: humans review at human speed, but a single agent generates code serially. Parallel agents on isolated worktrees lift throughput roughly 3× in reported case studies ([Mindstudio worktrees](https://www.mindstudio.ai/blog/parallel-agentic-development-git-worktrees)).

There is also a fourth motivation that is less often stated but more important to acknowledge: **cross-vendor risk management**. A pipeline that hard-codes one vendor's CLI inherits that vendor's quota policy, model deprecations, and outages. Mixing Codex and Claude inside one orchestrator lets you swap engines without rewriting the workflow.

### 1.2 The master/worker separation as a design tool

Putting a master/orchestrator on top of one or more worker CLIs gives you four levers you do not have with a single CLI:

- **Decomposition control**: the master can break a request into smaller, individually verifiable tasks. The worker never sees the whole problem and never has to keep the whole problem in context.
- **Policy and budget control**: the master enforces permissions, budgets, and timeouts the worker would otherwise have to negotiate with the user per-tool-call.
- **Verification gate**: the master inspects worker output before integrating it. The output of an LLM call is treated as a hypothesis, not as committed work — which is the principal mitigation for the compounding-failure problem in §6.
- **Multi-vendor routing**: the master picks the worker per task based on cost, latency, capability fit, or fallback policy.

Microsoft's guidance on multi-agent design captures the discipline this requires: "The parent orchestrator should have clear criteria for when to hand off ... Treat the entire connected agent as an agentic 'tool' with a description" ([Microsoft Copilot Studio guidance](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/multi-agent-patterns)). The worker is a callable contract, not a colleague.

---

## 2. Architectural Patterns

The community has converged on roughly five patterns. They are composable — most production systems mix two or three — but it is worth understanding each on its own first. The pattern catalog below cross-references the Lushbinary survey ([Lushbinary patterns guide](https://lushbinary.com/blog/multi-agent-orchestration-patterns-supervisor-swarm-pipeline-router-guide/)) and the LangGraph supervisor-vs-swarm analysis ([focused.io](https://focused.io/lab/multi-agent-orchestration-in-langgraph-supervisor-vs-swarm-tradeoffs-and-architecture)).

### 2.1 Supervisor / Worker (master-worker)

A single supervisor agent plans, delegates to one or more workers, collects results, and synthesizes the final output. The supervisor owns the conversation with the user; workers are silent specialists.

- **When to use**: structured workflows that require a coherent final output, audit trails, or strict step ordering. AWS CAO ([CAO repo](https://github.com/awslabs/cli-agent-orchestrator)) and the LangGraph supervisor pattern ([LangChain ref](https://reference.langchain.com/python/langgraph-supervisor)) are textbook examples.
- **Pros**: centralized control, clear audit trail, single point of compliance enforcement, easy to insert verification gates.
- **Cons**: supervisor is a single point of failure ("if planning fails, all downstream work is wasted" — [Lushbinary](https://lushbinary.com/blog/multi-agent-orchestration-patterns-supervisor-swarm-pipeline-router-guide/)); latency grows with sequential hops; the supervisor can become a token-cost hotspot since every routing decision is an LLM call.
- **Task fit**: anything where a human would normally hold a project-management role — multi-step research, regulated changes, anything needing an explicit plan-approval gate.

### 2.2 Pipeline (sequential handoff)

Agents are chained: each transforms input and passes it forward. There is no central orchestrator; the topology is hard-coded. Research → draft → edit → publish is the canonical example.

- **When to use**: predictable workflows with clear stage boundaries — lint → test → build → deploy, or extract → transform → validate → load.
- **Pros**: very debuggable (you can replay any stage with the recorded input), natural place to insert quality gates between stages, easy to reason about.
- **Cons**: serial latency (sum of all stages), no parallelism for independent sub-tasks, brittle without checkpoints — a stage-3 failure stalls stages 4+ ([Lushbinary](https://lushbinary.com/blog/multi-agent-orchestration-patterns-supervisor-swarm-pipeline-router-guide/)).
- **Task fit**: content pipelines, CI-like automation, anything where the value is in the deterministic ordering rather than in agent intelligence at the join points.

### 2.3 Fan-out / Fan-in (parallel)

A coordinator splits one task into N independent sub-tasks, runs them in parallel, and merges the results. AutoGen documents this as the "Concurrent Agents" pattern ([AutoGen docs](https://microsoft.github.io/autogen/stable//user-guide/core-user-guide/design-patterns/concurrent-agents.html)). CAO's `assign` primitive implements this for CLI agents — workers run in parallel tmux sessions and report back via `send_message` ([CAO](https://github.com/awslabs/cli-agent-orchestrator)).

- **When to use**: a single high-level task that decomposes into N peer sub-tasks (e.g., review N files, run N variations of an analysis, refactor N similar modules).
- **Pros**: latency drops to roughly max(sub-task) rather than sum; isolation prevents one slow worker from blocking others; naturally fits git-worktree isolation.
- **Cons**: merge/fan-in is the hard part — you need a deterministic strategy for combining results, and you need to handle partial failure (3 of 4 workers succeeded — do you retry the fourth, drop it, or block?).
- **Task fit**: independent reviews, parallel exploration, large refactors that decompose by feature boundary (not by file — see §5).

### 2.4 Swarm (peer agents)

Fully decentralized: agents pull from a shared queue or message bus, claim work, and communicate peer-to-peer. There is no supervisor. Overstory's SQLite mail bus with typed message protocols is a working example ([Overstory](https://github.com/jayminwest/overstory)); Kimi K2.6's internal swarm spawning up to 300 sub-agents on SWE-Bench Pro is an extreme commercial case ([Lushbinary](https://lushbinary.com/blog/multi-agent-orchestration-patterns-supervisor-swarm-pipeline-router-guide/)).

- **When to use**: very large, parallelizable workloads where decomposition is more important than central coherence (massive multi-file refactors, broad vulnerability scans).
- **Pros**: highest throughput potential, native fault isolation (one death does not stop the swarm), no single bottleneck.
- **Cons**: lowest predictability and debuggability, agents can spawn sub-agents and costs spike unpredictably, infinite-loop risk without guardrails, distributed tracing is essential and non-trivial.
- **Task fit**: research exploration at scale, fleet refactors, anything where you can tolerate variance in individual results because you're aggregating.

### 2.5 Router (dispatch by capability)

A lightweight classifier inspects each incoming request and dispatches it (entire request, end-to-end) to the right specialist. There is no plan, no decomposition, no aggregation — the router's only job is "which specialist."

- **When to use**: mixed-workload entry points — for example, "DMs go to the coordinator, #engineering messages go to the dev profile" ([Lushbinary](https://lushbinary.com/blog/multi-agent-orchestration-patterns-supervisor-swarm-pipeline-router-guide/)).
- **Pros**: minimal latency (one classification call, then direct handoff), no central bottleneck, simple to implement.
- **Cons**: cannot handle requests that genuinely need multiple specialists collaborating; misclassification sends the entire request to the wrong agent; escalation requires re-routing logic.
- **Task fit**: front-door of a system that already has clear, non-overlapping specialist roles.

### 2.6 Composition is the norm

In practice none of these patterns is used pure. A typical production setup looks like: router at the edge → supervisor for complex requests → fan-out for the parallelizable parts → pipeline for the merge stage. The Lushbinary guide makes this explicit: "A Router at the edge classifies requests, sending structured tasks to a Supervisor and simple lookups directly to a specialist" ([Lushbinary](https://lushbinary.com/blog/multi-agent-orchestration-patterns-supervisor-swarm-pipeline-router-guide/)).

### 2.7 Pattern selection at a glance

| Pattern | Control | Latency | Fault tolerance | Debuggability | Cost predictability |
| --- | --- | --- | --- | --- | --- |
| Supervisor | High | Medium-high | Medium | Good | Predictable |
| Pipeline | High | Sum of stages | Low | Excellent | Predictable |
| Fan-out/in | Medium-high | Max of branches | Medium-high | Medium | Predictable |
| Swarm | Low | Low (parallel) | High | Poor | Variable |
| Router | Medium | Low | High | Good | Predictable |

Adapted from [Lushbinary](https://lushbinary.com/blog/multi-agent-orchestration-patterns-supervisor-swarm-pipeline-router-guide/) with fan-out added.

---

## 3. Communication Mechanisms

How agents actually move bytes between each other is the part most often skipped in pattern discussions, and it is also the part most likely to bite you in production. The four mechanisms below cover essentially every prior-art system.

### 3.1 subprocess + stdin/stdout/stderr

The orchestrator spawns the worker CLI as a child process, writes a prompt to stdin (or as a `-p` argument), and reads structured output from stdout. stderr is for human-readable progress logs.

This is the dominant pattern because both Codex and Claude Code support it natively:

- **Claude Code**: `claude -p "<task>" --output-format stream-json --verbose --include-partial-messages` emits NDJSON events. With `--bare`, the subprocess skips auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and CLAUDE.md so behavior is identical on every machine — important for orchestrators that need determinism ([Claude headless docs](https://code.claude.com/docs/en/headless); see [02-claude-code-cli.md §3.2](02-claude-code-cli.md#32-headless-p-print-mode) for the per-flag breakdown and [01-codex-cli.md §3.2 / §4.1](01-codex-cli.md#32-non-interactive-exec-mode) for the Codex event vocabulary cited below). `--permission-mode`, `--tools`, `--allowedTools`, and `--disallowedTools` together give the parent process control over the child's visible and pre-approved tool surface.
- **Codex CLI**: `codex exec --json "<task>"` streams JSONL events (`thread.started`, `turn.started`, `item.started`, `item.completed`, `turn.completed`, `turn.failed`, `error`). `--output-schema <path>` requests that the final message conform to a JSON Schema; combined with `-o <path>` you get a deterministic structured artifact for the parent to consume ([Codex non-interactive](https://developers.openai.com/codex/noninteractive)).

The Claude Agent SDK is the cleanest published account of how the protocol actually works: the SDK spawns the CLI, writes JSON messages to stdin, and reads JSON-lines from stdout. There are two message categories — **regular** messages (agent responses, tool outputs, cost tracking) flow CLI → SDK, and **control** messages (permission requests, hook callbacks) are bidirectional with a `request_id` to multiplex concurrent decisions ([Inside the Claude Agent SDK](https://buildwithaws.substack.com/p/inside-the-claude-agent-sdk-from)).

- **Latency**: lowest of the four mechanisms (everything is in-process pipes).
- **Debuggability**: excellent — you can `tee` stdout to a file and replay.
- **Permission isolation**: medium — the child inherits the parent's environment unless you scrub it.
- **Failure modes**: pipe buffering (use `--line-buffered` style flushing on the worker), stdin/stdout deadlocks on large payloads (Claude caps piped stdin at 10MB and exits with non-zero status on overflow — [Claude headless docs](https://code.claude.com/docs/en/headless)), and the worker waiting on a permission prompt the parent never sees.

### 3.2 File-based (shared workspace, mailbox, queue)

Agents read and write a shared location — files in a workspace, a `tasks/` directory of JSON jobs, a `mail/` directory of messages, or a SQLite database used as a message bus.

Overstory's mail bus is the most developed example: SQLite in WAL mode, ~1-5ms query latency, 8 typed message protocols (`worker_done`, `merge_ready`, `dispatch`, `escalation`, ...), broadcast addresses (`@all`, `@builders`), async fire-and-forget plus synchronous request-response patterns ([Overstory](https://github.com/jayminwest/overstory)). Brandon Redmond's "10 Claude instances" setup uses Redis as a task queue plus file locks ([dev.to](https://dev.to/bredmond1019/multi-agent-orchestration-running-10-claude-instances-in-parallel-part-3-29da)).

- **Latency**: higher than subprocess pipes; polling intervals dominate.
- **Debuggability**: best of all four — every message is a persisted artifact you can grep, replay, and version-control. Overstory's `ov feed` / `ov replay` rests on this.
- **Permission isolation**: best — the shared workspace is the only surface, and you can enforce locks and ACLs there.
- **Failure modes**: clock skew between agents, stale locks if a worker crashes, polling waste, and ambiguity about "is this file in progress or abandoned?" (sentinel files help — see §6).

### 3.3 MCP (Model Context Protocol) channel

MCP is the structured-tools protocol used internally by Claude Code, Codex, and most modern CLI agents. An orchestrator can either (a) run an MCP server and expose orchestration primitives as tools that workers call, or (b) make the worker's MCP surface the channel. Codex can be exposed in role (b) via `codex mcp-server` (see [01-codex-cli.md §8.2](01-codex-cli.md#82-codex-as-mcp-server-codex-mcp-server)); Claude Code can play either role (see [02-claude-code-cli.md §9](02-claude-code-cli.md#9-mcp-integration)).

CAO chose (a): the central server runs on `localhost:9889` and exposes `handoff`, `assign`, and `send_message` as MCP tools. Each agent terminal carries a `CAO_TERMINAL_ID` env var so the server can route inter-agent messages and track status (IDLE / PROCESSING / COMPLETED / ERROR) ([CAO](https://github.com/awslabs/cli-agent-orchestrator)). This means the supervisor and workers all "speak MCP" natively and the orchestration primitives look like ordinary tool calls.

- **Latency**: comparable to HTTP local — a few ms per round trip.
- **Debuggability**: good if the MCP server logs every request, less good if you only see the agent-side tool call history.
- **Permission isolation**: strong — MCP servers are explicit, addressable, and can be tightly scoped per agent.
- **Failure modes**: protocol-level mismatch between client and server, MCP tool schemas drifting from agent expectations, version skew between MCP libraries in the orchestrator vs. the worker.

### 3.4 IPC / socket / HTTP

The general escape hatch — a local HTTP API, a Unix socket, or a websocket. CAO uses a WebSocket PTY endpoint over `localhost:9889` for terminal control alongside its MCP tools ([CAO](https://github.com/awslabs/cli-agent-orchestrator)). Codex's `app-server` mode exposes a JSON-RPC API to drive Codex programmatically ([Codex non-interactive](https://developers.openai.com/codex/noninteractive)).

- **Latency**: comparable to MCP (loopback is fast).
- **Debuggability**: medium — you can `tcpdump`/`socat` to inspect, but it is not log-by-default the way file-based is.
- **Permission isolation**: must be enforced explicitly; CAO defaults to localhost-only with DNS-rebinding protection because "exposing the server to untrusted networks without adding authentication" is the obvious footgun ([CAO](https://github.com/awslabs/cli-agent-orchestrator)).
- **Failure modes**: port collisions, partial writes if the protocol is not framed, and the classic distributed-systems hazards (timeouts, dropped messages, head-of-line blocking).

### 3.5 Comparison

| Mechanism | Latency | Debuggability | Permission isolation | Where it's hard |
| --- | --- | --- | --- | --- |
| subprocess + stdio | Lowest | Excellent (tee + replay) | Medium (env inheritance) | Buffering, deadlocks, 10MB stdin caps |
| File-based / DB mailbox | Higher (polling) | Best (persisted) | Best (filesystem ACLs) | Polling waste, stale locks |
| MCP channel | Low (local) | Good (with server logs) | Strong (scoped per agent) | Schema drift, version skew |
| IPC / socket / HTTP | Low (loopback) | Medium | Manual | Port collisions, framing, network exposure |

The dominant production combination is **subprocess + stdio for the worker call, file-based for state and audit, MCP for cross-agent operations**. This is essentially what CAO, Overstory, and Symphony all do, with different weights.

---

## 4. Task Unit Definition

A "task" is the contract that an orchestrator hands to a worker. The quality of this contract is the single biggest predictor of whether a multi-agent system works. Osmani is blunt about it: "Your spec is the leverage. Vague requirements propagate errors across parallel runs. Precise specs — with architecture, boundaries, edge cases, invariants — multiply into consistent implementations across the fleet" ([Osmani](https://addyosmani.com/blog/code-agent-orchestra/)).

### 4.1 What goes in a task

Across the prior art, a well-formed task carries:

- **Identity**: a stable ID (`id`), and ideally an issue link so the audit trail is anchored to the existing tracker.
- **Scope**: which files or directories the worker may touch, and which it must not. Overstory enforces this as a runtime guard, not just a prompt — "tool-call guards specific to each runtime to mechanically enforce agent permissions" ([Overstory](https://github.com/jayminwest/overstory)).
- **Dependencies**: array of prerequisite task IDs. Brandon Redmond's setup builds a dependency graph and topologically sorts before dispatch ([dev.to](https://dev.to/bredmond1019/multi-agent-orchestration-running-10-claude-instances-in-parallel-part-3-29da)).
- **Acceptance criteria**: explicit, testable. The output is verified against these — not against the worker's claim of completion.
- **Budget**: max turns, max wall-clock, max USD. Claude Code's `--max-turns` and `--max-budget-usd` give per-invocation enforcement; the orchestrator should track aggregate budget too.
- **Tool surface and allowlist**: not "any tool" — a closed set. Codex's `--sandbox workspace-write` constrains process/filesystem scope; Claude's `--tools` constrains visible built-in tools; `--allowedTools` pre-approves matching calls.
- **Output schema**: ideally a JSON Schema. Codex's `--output-schema` and Claude's `--json-schema` both enforce structured returns so the parent does not parse free-form text.

### 4.2 Schema representations in the wild

CAO writes tasks as agent profiles with YAML frontmatter on a markdown body ([CAO](https://github.com/awslabs/cli-agent-orchestrator)):

```yaml
---
name: reviewer
role: code_reviewer
provider: claude_code
allowedTools:
  - file_read
  - file_browser
  - bash
---
You are a code reviewer. Review the diff at $ARG and emit findings as JSON.
```

Overstory uses a two-layer system: base `.md` files define workflows (the HOW), and per-task overlays inject scope (the WHAT) — created with `ov spec write <task-id>` and tracking issue-to-agent bindings through task groups ([Overstory](https://github.com/jayminwest/overstory)).

Symphony treats the Linear board itself as the task store, with each issue being a task and each issue's state (Todo / In Progress / In Review / Done) being the orchestration state ([OpenAI Symphony](https://openai.com/index/open-source-codex-orchestration-symphony/)).

### 4.3 Input/output contract enforcement

The "contract" is meaningless if the worker can violate it silently. Three enforcement mechanisms appear:

1. **Schema-validated output**: `--output-schema` (Codex) or `--json-schema` (Claude) reject final messages that do not match. The parent treats a non-conforming response as a hard failure.
2. **Mechanical scope guards**: the worker's tools are restricted such that it physically cannot read or write outside its scope. Overstory bills this as "instruction overlays and tool-call guards specific to each runtime to mechanically enforce agent permissions — builders modify code, scouts read-only explore, reviewers validate without changing" ([Overstory](https://github.com/jayminwest/overstory)).
3. **Post-hoc verification**: the orchestrator runs lint/test/typecheck/security-scan and reverts on failure. Brandon Redmond's quality gate "Failed validations trigger automatic reversion and task reassignment" ([dev.to](https://dev.to/bredmond1019/multi-agent-orchestration-running-10-claude-instances-in-parallel-part-3-29da)).

### 4.4 Partial failure

When a fan-out of N tasks produces M < N successes, the orchestrator has four options: (1) retry the failed subset, (2) drop and continue with M, (3) block the whole merge, (4) escalate to human. Most systems implement (1) with a retry budget then fall through to (4). Overstory's `ov sling <task-id> --recover` bypasses status checks to spawn a fresh worker on incomplete tasks — explicit acknowledgement that workers die for non-recoverable reasons and you sometimes need to start clean ([Overstory](https://github.com/jayminwest/overstory)).

---

## 5. State & Isolation

Multiple agents touching the same working tree is the most common source of "it works for one but breaks for N" failures. Three isolation strategies dominate.

### 5.1 git worktrees

Each agent gets its own checked-out copy of the repo on its own branch, all sharing the same `.git` object store. Changes in one worktree are invisible to another until explicitly merged ([Augment Code](https://www.augmentcode.com/guides/git-worktrees-parallel-ai-agent-execution)).

- **Pros**: filesystem isolation, no lock contention on `.git/index`, each agent has its own dependency directory and env file, merging is normal git from there.
- **Cons**: worktrees solve filesystem isolation but **not** context-window health or task coordination ([MindStudio worktrees](https://www.mindstudio.ai/blog/git-worktrees-parallel-ai-coding-agents)). You still need processes for both.
- **Gotchas**: `node_modules` and language-specific lock files must be regenerated per worktree (do not symlink — concurrent installs corrupt the shared dir); `.env` files should be copied rather than symlinked; some pre-commit hooks assume a single worktree and break.

Overstory, CAO, Symphony, and EloPhanto all use this pattern. The MindStudio playbook recommends naming worktrees by feature (`../project-feat-auth`, `../project-feat-api`), automating creation, and "decomposing tasks by domain or feature boundary — not by file — to minimize merge conflicts" ([MindStudio playbook](https://www.mindstudio.ai/blog/parallel-agentic-development-git-worktrees)).

### 5.2 tmux session isolation

Each agent runs in its own tmux window or pane. The process is isolated, the PTY is real (so the agent can drive interactive tools), the session persists across orchestrator restarts, and you can `tmux attach` for human intervention mid-task ([CAO](https://github.com/awslabs/cli-agent-orchestrator)).

This is layered on top of worktrees, not instead of them. CAO requires tmux 3.3+ and uses tmux for "process isolation with real PTY access, session persistence and inspection ... human intervention capability mid-task, context separation preventing worker cross-contamination."

- **Pros**: human-attachable, persistent, gives you a real terminal for tools that detect TTY.
- **Cons**: tmux startup is slow (Overstory exposes `runtime.shellInitDelayMs: 3000` to tune around this — [Overstory](https://github.com/jayminwest/overstory)); session names collide if you do not namespace; killing a session kills the agent without cleanup hooks.

### 5.3 Container isolation

Each agent runs in its own Docker container or VM. Brandon Redmond's setup uses Docker "with CPU/memory quotas" and monitors resource exhaustion before spawning more ([dev.to](https://dev.to/bredmond1019/multi-agent-orchestration-running-10-claude-instances-in-parallel-part-3-29da)).

- **Pros**: strongest isolation, can run untrusted code, network policy enforceable.
- **Cons**: slowest startup, heaviest resource footprint, image management overhead, harder to debug interactively.
- **Fit**: required if you are running agents on untrusted prompts or against untrusted code; overkill for a trusted local developer workflow.

### 5.4 Shared state vs. message passing

The deeper architectural choice underneath isolation is whether agents share state or pass messages.

**Shared state**: workers read/write a common store (workspace, database, lock manager). This is the "snapshot + reconcile" model. Brandon Redmond's key lesson is to snapshot the repository state at agent spawn rather than maintain live updates, then "reconcile diffs between jobs instead of maintaining real-time sync" ([dev.to](https://dev.to/bredmond1019/multi-agent-orchestration-running-10-claude-instances-in-parallel-part-3-29da)). The reported breakdown was ~70% clean completion, ~25% single retry, ~5% human review.

**Message passing**: workers communicate only through typed messages, never through a shared mutable store. Overstory's mail bus is this — the SQLite DB is technically shared, but workers do not read each other's working directories.

Message passing is easier to debug (every transition is a message), but it requires the orchestrator to do more work in marshalling state. Shared state is easier to bootstrap but requires lock discipline and is harder to audit. Most production systems hybridize: shared state for the actual code (a git worktree), message passing for everything else.

---

## 6. Reliability & Compounding Failure

This is the section most teams underestimate, and it is the single biggest reason multi-agent prototypes fail in production.

### 6.1 The compounding math

System reliability under sequential dependence is multiplicative: **R_total = R₁ × R₂ × … × Rₙ**. The MindStudio analysis spells it out: "Five components at 99% reliability each yield only 95.1% system reliability. With 50 components, this drops to 60.5%. By the time you have 50 components — not unusual in a production agent system — a system where every individual piece is '99% reliable' will fail roughly four out of ten times" ([MindStudio reliability](https://www.mindstudio.ai/blog/reliability-compounding-problem-ai-agent-stacks)).

This effect is worse for LLM agents than for traditional services because of three amplifiers:

1. **Non-determinism**: an LLM call does not produce identical outputs for identical inputs, so failures are silent and propagate.
2. **Long chains by default**: a typical agent workflow involves 8+ steps (parse, search, summarize, cross-reference, draft, review, format, ...).
3. **Error propagation**: "A bad intermediate output gets fed into the next LLM call, which may confidently reason about the bad data and produce something even further from correct" ([MindStudio](https://www.mindstudio.ai/blog/reliability-compounding-problem-ai-agent-stacks)).

### 6.2 The MAST taxonomy

The MAST (Multi-Agent System Failure Taxonomy), from a NeurIPS 2025 analysis of 1,600+ execution traces, groups failures into three categories ([Augment Code](https://www.augmentcode.com/guides/why-multi-agent-llm-systems-fail-and-how-to-fix-them)):

| Category | Share | What it looks like |
| --- | --- | --- |
| Specification problems | 41.77% | Role ambiguity, unclear task definitions, missing constraints |
| Coordination failures | 36.94% | Communication breakdowns, state sync issues, conflicting objectives |
| Verification gaps | 21.30% | Inadequate testing, missing validation, absent quality checks |

Infrastructure issues (rate limits, context overflow, cascading timeouts) sit outside MAST but compound the categories above.

### 6.3 Soft vs. hard failures

Hard failures are detectable: the worker exits non-zero, the JSON does not validate, the test suite fails. Soft failures are the dangerous category: "Hallucinations, ambiguous interpretations, and reasoning drift show up as soft deviations that propagate silently, with no stack trace and no alert" ([MindStudio reliability](https://www.mindstudio.ai/blog/reliability-compounding-problem-ai-agent-stacks)). A pipeline that only checks for hard failures will compound soft ones until something visible breaks far downstream.

### 6.4 Mitigation strategies

From the prior art, the dominant mitigations are:

- **Shorten the critical path**: every component you can remove improves reliability more than tuning individual components. If your supervisor is asking the same question of two workers and reconciling, ask whether one good worker is enough.
- **Independent validation**: a separate verifier agent with an isolated prompt and a separate context window scores outputs the producer never sees. "Independent validation is the most underused reliability mechanism in multi-agent systems" ([Augment Code](https://www.augmentcode.com/guides/why-multi-agent-llm-systems-fail-and-how-to-fix-them)). Microsoft's guidance reinforces this by demanding that subagents do not communicate with the user — only the parent does — so the verifier role stays meaningful ([Microsoft](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/multi-agent-patterns)).
- **Idempotent retries with exponential backoff**: the worker call must be safe to repeat. Claude Code's `system/api_retry` event reports `attempt`, `max_retries`, `retry_delay_ms`, and `error` category for orchestrators that need to react to retry pressure ([Claude headless docs](https://code.claude.com/docs/en/headless)).
- **Circuit breakers**: pause traffic to consistently failing dependencies — for example, stop dispatching to a worker provider that is rate-limited rather than keep retrying.
- **Output validation as a first-class step**: schema validation, test runs, lint, type-check, security scan, all between worker call and integration. Brandon Redmond's pipeline runs all five before allowing a merge ([dev.to](https://dev.to/bredmond1019/multi-agent-orchestration-running-10-claude-instances-in-parallel-part-3-29da)).
- **Kill criteria**: Osmani recommends "reassign agents stuck 3+ iterations on the same error" ([Osmani](https://addyosmani.com/blog/code-agent-orchestra/)). This generalizes a global Claude habit: stop after three identical failures and ask for direction rather than burn more tokens.
- **Hard token and turn budgets**: per-agent caps with auto-pause at ~85% prevent a single runaway agent from eating the budget for the rest of the workflow.

### 6.5 Determinism and reproducibility

LLM calls are non-deterministic, but the orchestrator does not have to be. The deterministic pieces are: input contracts, schemas, dependency graphs, retry policies, verifier criteria, and post-processing. Make these reproducible (version-controlled task templates, pinned model IDs, recorded prompts, logged tool calls) and you can replay any worker call to investigate a failure. "Bernstein" in the agent-orchestrator list pitches itself explicitly as a "deterministic orchestrator — spawns parallel AI coding agents" — the determinism is in the orchestration, not in the agents themselves ([awesome-agent-orchestrators](https://github.com/andyrewlee/awesome-agent-orchestrators)).

---

## 7. Observability

If you cannot see what the swarm is doing, you cannot run it in production. Three layers of observability appear consistently across the prior art.

### 7.1 Per-agent transcripts and event streams

Every agent's input, output, tool calls, and lifecycle events must be captured to disk. Claude Code's `stream-json` output gives you `system/init` (model, tools, MCP servers, loaded plugins), `system/api_retry` (with attempt count and error class), per-message events, and per-stream-event deltas ([Claude headless docs](https://code.claude.com/docs/en/headless)). Codex's `--json` produces a similarly structured JSONL with `thread.started`, `turn.started`, `item.started`, `item.completed`, `turn.completed`, `turn.failed`, `error` ([Codex non-interactive](https://developers.openai.com/codex/noninteractive)).

CAO writes terminal scrollback and metadata to `~/.cao/logs/terminal/` before deleting a worker terminal on success — explicitly so administrators can restore deleted terminals for debugging via `cao terminal restore <terminal_id>` ([CAO](https://github.com/awslabs/cli-agent-orchestrator)).

### 7.2 Cross-agent timelines and traces

Per-agent logs are necessary but not sufficient — the interesting question is usually "what was the sequence of cross-agent messages that led to this state?" Overstory ships this as a first-class feature: `ov serve` (web UI with fleet status, mail bus, agent timelines), `ov dashboard` (TUI), `ov feed` / `ov replay` for chronological inspection across the whole swarm ([Overstory](https://github.com/jayminwest/overstory)).

The principle generalizes: every cross-agent message should carry the originating task ID, the message ID, the sender, the receiver, and a timestamp. Then "find every message related to task T-1234" is a single grep.

### 7.3 Cost tracking and budget enforcement

LLM calls cost money per call and the cost is highly variable. Without per-agent budget tracking you discover the bill after the fact. The mitigations:

- Claude Code's `--output-format json` returns `total_cost_usd` and a per-model cost breakdown in every response, so the orchestrator can sum costs per task and per agent ([Claude headless docs](https://code.claude.com/docs/en/headless)).
- Codex's `turn.completed` event includes a `usage` object with `input_tokens`, `cached_input_tokens`, and `output_tokens` ([Codex non-interactive](https://developers.openai.com/codex/noninteractive)).
- Overstory exposes `ov costs` for "tracking tokens and spending per agent/capability" ([Overstory](https://github.com/jayminwest/overstory)).
- Osmani's "token budgets: hard per-agent limits with auto-pause at 85%" is the standard guard rail ([Osmani](https://addyosmani.com/blog/code-agent-orchestra/)).

### 7.4 Progress UI

The dashboard is not optional once you have more than three concurrent workers. CAO offers `cao session status <name> --workers`. Overstory's `ov dashboard` is a TUI with the mail bus visible. EloPhanto polls every 10 minutes for "process health, PR creation, CI passage, and code review approvals" ([EloPhanto](https://dev.to/elophanto/how-i-orchestrate-claude-code-codex-and-gemini-cli-as-a-swarm-4p3c)). The exact UI matters less than the underlying invariant: a single place to see "what is every agent doing right now, and what is the queue."

---

## 8. Case Studies (prior art)

### 8.1 AWS CLI Agent Orchestrator (CAO)

- **One-line architecture**: hierarchical supervisor-worker over MCP, with tmux for process isolation and a local HTTP+WebSocket server for routing.
- **Communication**: MCP tools (`handoff`, `assign`, `send_message`) exposed by a central server on `localhost:9889`; terminal status (IDLE/PROCESSING/COMPLETED/ERROR) tracked per `CAO_TERMINAL_ID`.
- **Isolation**: tmux 3.3+ sessions, optionally per-provider auth.
- **Task definition**: markdown files with YAML frontmatter (`name`, `provider`, `role`, `allowedTools`, instructions). Installable from a registry, local file, or URL via `cao install`.
- **Provider matrix**: Kiro, Claude Code, Codex, Gemini, Kimi, Copilot, Q Developer, OpenCode. Profiles can pin a provider or inherit from the spawning terminal.
- **Strengths**: native auth preservation per provider, true PTY isolation, scheduled execution via `cao flow`, REST API for programmatic control, localhost-only security with DNS-rebinding protection.
- **Weaknesses**: MCP-only orchestration primitives mean every cross-agent operation is a tool call (rich semantics, but more moving parts than a flat subprocess+stdio approach); enforcement of `allowedTools` is hard on only 5 of 7 providers, soft on the rest.
- **Sources**: [repo](https://github.com/awslabs/cli-agent-orchestrator), [AWS blog](https://aws.amazon.com/blogs/opensource/introducing-cli-agent-orchestrator-transforming-developer-cli-tools-into-a-multi-agent-powerhouse/).

### 8.2 Overstory

- **One-line architecture**: hierarchical (Orchestrator → Coordinator → Supervisor → Workers) with shared SQLite mail bus and per-agent git worktrees.
- **Communication**: custom SQLite mail bus (WAL mode, 1-5ms latency, 8 typed message protocols, broadcast addresses, async + synchronous patterns); Claude agents run in headless mode (subprocess + NDJSON) or tmux mode.
- **Isolation**: one git worktree per agent, runtime adapters with tool-call guards enforcing permission roles (builder / scout / reviewer).
- **Task definition**: two-layer — base `.md` workflows + per-task overlays (`ov spec write <task-id>`) bound to issues via task groups.
- **Provider matrix**: Claude Code, Pi, GitHub Copilot, Gemini CLI, Cursor, Codex, Sapling, OpenCode, Aider, Goose, Amp.
- **Reliability**: tiered watchdog (mechanical daemon → AI-triage → continuous monitor agent), `ov sling --recover` for dead-agent recovery, FIFO merge queue with 4-tier conflict resolution and sentinel-file locking.
- **Observability**: `ov serve` (web), `ov dashboard` (TUI), `ov feed` / `ov replay`, `ov costs`.
- **Strengths**: probably the most-developed open observability stack, mature multi-runtime support, explicit recovery semantics.
- **Weaknesses**: in maintenance mode — active development moved to Warren, a hosted control plane for sandboxed cloud agents.
- **Sources**: [repo](https://github.com/jayminwest/overstory).

### 8.3 OpenAI Symphony

- **One-line architecture**: Linear board as a finite state machine, Codex as the worker, BEAM/OTP supervision tree as the safety net.
- **Communication**: polling Linear for new tasks, GitHub for PR state; agents post evidence (CI status, PR feedback, complexity analysis, video walkthroughs) back to Linear.
- **Isolation**: one autonomous implementation run per task (per ticket).
- **Task definition**: a Linear issue. The board's state — Todo / In Progress / In Review / Done — is the orchestration state.
- **Reliability**: OTP supervision trees restart crashed Codex instances cleanly mid-PR. "OTP's supervision trees are perfect for managing flaky AI agents" ([Help Net Security](https://www.helpnetsecurity.com/2026/04/28/openai-symphony-codex-orchestration-linear/)).
- **Reported impact**: ~500% increase in landed PRs on some teams in the first three weeks.
- **Strengths**: makes the project tracker the source of truth (no parallel task DB to drift), reuses battle-tested supervision primitives from BEAM, "manage work instead of supervising coding agents" framing.
- **Weaknesses**: tight coupling to Linear specifically (the spec is open, but the reference implementation is Linear-first); Elixir reference implementation is less approachable for Python/TS teams; engineering-preview maturity.
- **Sources**: [OpenAI announcement](https://openai.com/index/open-source-codex-orchestration-symphony/), [repo](https://github.com/openai/symphony).

### 8.4 EloPhanto (CLI swarm)

- **One-line architecture**: a single business-context-bearing orchestrator over a heterogeneous swarm of Claude Code, Codex, and Gemini CLI.
- **Communication**: orchestrator launches each agent in its own tmux session with a hand-tuned prompt; polls every 10 minutes for status (process health, PR creation, CI passage, review approvals).
- **Isolation**: one git worktree per agent on a separate branch.
- **Specialization split**: Codex for backend complexity and race conditions; Claude Code for frontend iteration and architectural validation; Gemini for UI polish and security and scalability.
- **Reliability**: failure-as-prompt-improvement — "rather than simple restarts, EloPhanto reads failures with full business context and writes a better prompt."
- **Strengths**: directly demonstrates Codex+Claude (+Gemini) cohabitation; explicit per-CLI specialization rationale; multi-model PR review (each CLI comments on PRs).
- **Weaknesses**: less formal contract definition than CAO or Symphony; the "better prompt" failure-handling loop is hard to bound (risk of churn).
- **Sources**: [EloPhanto article](https://dev.to/elophanto/how-i-orchestrate-claude-code-codex-and-gemini-cli-as-a-swarm-4p3c).

### 8.5 Brandon Redmond — "10 Claude instances in parallel"

- **One-line architecture**: meta-agent orchestrator over a Redis-backed task queue and Docker-isolated workers, with file-locking and dependency-graph topology.
- **Communication**: Redis (FIFO task queue with blocking ops, status hashing, WebSocket dashboard).
- **Isolation**: Docker per worker with CPU/memory quotas; Redis-based exclusive file locks (300s timeout, backoff on conflict).
- **Reliability**: snapshot-at-spawn (no real-time sync), quality gate (test suite + type check + conflict markers + perf benchmarks + security scan) with automatic reversion on failure.
- **Reported outcome**: 12k-line class-to-functional refactor in 2h with 6 agents, 100% test pass, zero file conflicts; ~70% clean / ~25% single retry / ~5% human review.
- **Strengths**: explicit and quantitative; documents the lessons (snapshot+reconcile beats real-time sync, coordinator-owned locking beats agent-level coordination, conflict analysis before parallelization).
- **Weaknesses**: Redis dependency adds a moving part; Docker isolation adds startup cost.
- **Sources**: [dev.to](https://dev.to/bredmond1019/multi-agent-orchestration-running-10-claude-instances-in-parallel-part-3-29da).

### 8.6 Other notable systems (compressed)

From the [awesome-agent-orchestrators](https://github.com/andyrewlee/awesome-agent-orchestrators) catalog:

- **agent-kanban**: leader-worker with cryptographic agent identity.
- **ai-maestro**: dashboard orchestrating Claude, Aider, Cursor across machines.
- **bernstein**: "deterministic orchestrator — spawns parallel AI coding agents."
- **clideck**: WhatsApp-like dashboard for multiple AI coding agents.
- **jean / parallel-code / constellagent**: desktop apps with isolated git worktrees and diff viewers.
- **agentsmesh**: "AI Agent Workforce Platform" supporting Claude Code, Codex, Gemini, Aider, OpenCode.
- **loki-mode**: 41 specialized agents across 8 swarms with "RARV cycles, 9 quality gates."
- **gnap**: "Git-Native Agent Protocol" using a shared repo as the task board.
- **wit**: function-level locking via AST parsing.
- **Dex**: structured Ralph orchestrator with human-gated planning and parallel multi-reviewer code review.

The breadth of the catalog is itself a data point — the design space is being explored from many angles, and the patterns above (supervisor, pipeline, fan-out, swarm, router) cover essentially all of them.

---

## 9. Anti-Patterns and Pitfalls

The community has accumulated a substantial set of "do not do this" findings. They are worth listing explicitly because most are non-obvious until you have hit them.

### 9.1 Free-form text parsing for control flow

Parsing the worker's prose output to decide the next step is fragile. Use `--output-schema` (Codex; see [01-codex-cli.md §3.2 / §4.1](01-codex-cli.md#32-non-interactive-exec-mode)) or `--json-schema` (Claude; see [02-claude-code-cli.md §3.2](02-claude-code-cli.md#32-headless-p-print-mode)) and treat non-conforming output as a hard failure. The orchestrator should never depend on the worker mentioning "PASS" or "FAIL" in plain text.

### 9.2 Permission prompts in non-interactive mode

If the worker is run as a subprocess without explicit `--permission-mode` and pre-approved tool rules, it can block forever on a permission prompt the parent never sees. Always provide the minimum `--tools` surface, pre-approve only the calls the worker needs with `--allowedTools`, and audit both `tools` and `allowedTools` per task. Both Codex (`--sandbox`) and Claude (`--permission-mode`) document the safe non-interactive defaults.

### 9.3 Subagents talking to the user directly

Microsoft's guidance is unambiguous: "You're a subagent. Do NOT reply to the user directly" — without this in the subagent's prompt, the worker sends messages straight to the user, producing duplicate or partial output and breaking the orchestrator's verifier role ([Microsoft](https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/multi-agent-patterns)).

### 9.4 Real-time context synchronization between agents

Brandon Redmond's hard-won lesson: "Attempting real-time context synchronization (replaced with snapshot + reconcile)" did not work above ~10 agents. Snapshot the repo at agent spawn, let agents diverge, reconcile at merge time ([dev.to](https://dev.to/bredmond1019/multi-agent-orchestration-running-10-claude-instances-in-parallel-part-3-29da)).

### 9.5 Decomposing by file instead of by feature

If two agents own different files but the same domain, they will both want to touch the seam. Decompose by feature or domain boundary, not by file. The MindStudio worktrees playbook is explicit: "Decompose tasks by domain or feature boundary — not by file — to minimize merge conflicts" ([MindStudio worktrees](https://www.mindstudio.ai/blog/git-worktrees-parallel-ai-coding-agents)).

### 9.6 Shared `node_modules` (and friends) across worktrees

Concurrent installs corrupt the shared dependency directory. Keep dependency directories per-worktree even at the cost of disk space ([MindStudio playbook](https://www.mindstudio.ai/blog/parallel-agentic-development-git-worktrees)).

### 9.7 Unbounded retry on the "better prompt" loop

The "if it failed, try again with a better prompt" loop is attractive (and EloPhanto uses it) but can churn forever without a cap. Combine with kill criteria — Osmani's "reassign agents stuck 3+ iterations on the same error" generalizes the global 3-fail rule that should be baked into the orchestrator.

### 9.8 No verifier agent (or the verifier is the same model)

Independent validation requires a separate context, ideally a different model family. The Augment Code analysis flags this as "the most underused reliability mechanism in multi-agent systems" ([Augment Code](https://www.augmentcode.com/guides/why-multi-agent-llm-systems-fail-and-how-to-fix-them)).

### 9.9 Token budgets only after the fact

If you discover your budget by reading the invoice, your budget is too high. Hard per-agent caps with pre-emptive pause at ~85% are standard. Both Claude (`--max-budget-usd`) and orchestrator-level tracking are needed.

### 9.10 Trusting the agent's "completed" claim

The agent saying it finished is not evidence it finished. Verify via tests, lint, type-check, security scan, schema validation. Brandon Redmond's pipeline reverts the worktree on quality-gate failure rather than asking the worker to "try again with the failures fixed" — because the worker has already mis-modeled the problem.

### 9.11 stdout buffering hiding partial output

Pipe buffering can delay multi-second event streams to seconds-or-minutes of silence on the orchestrator side. Force line-buffered output (`stdbuf -oL`, `--line-buffered`, equivalent flag in the worker) or use `--include-partial-messages` (Claude) / `--json` streaming (Codex) and parse incrementally.

### 9.12 Single supervisor as a token-cost hotspot

Every routing decision is an LLM call. Under heavy fan-out the supervisor becomes the most expensive component. Mitigation: cache routing decisions, use a cheaper model for the routing-only LLM call, or move to a swarm topology for cost-sensitive workloads (with the trade-offs in §2.4).

### 9.13 Exposing the orchestrator's local server

CAO defaults to localhost-only with DNS-rebinding protection because the obvious failure mode is leaving the WebSocket PTY endpoint open to the network. If the orchestrator runs an HTTP/MCP/socket server, bind to loopback by default and require explicit opt-in plus authentication for any non-loopback exposure ([CAO](https://github.com/awslabs/cli-agent-orchestrator)).

### 9.14 Machine-generated `AGENTS.md`

Research cited by Osmani shows AI-generated `AGENTS.md` files "marginally reduce success rates (~3%) while increasing costs 20%+" ([Osmani](https://addyosmani.com/blog/code-agent-orchestra/)). Have humans write the shared instructions document; agents read it.

---

## 10. Recommendations for a Codex + Claude Hybrid

The user's target is **Codex CLI as orchestrator + Claude Code CLI as worker(s)**. Three candidate patterns from the prior art map onto this directly. The right choice depends on what trade-offs you want to live with; the table below lays them out rather than picking one.

### 10.1 Candidate A — Supervisor over subprocess workers

Codex (in `codex exec` mode, driven by a small Codex-side script or by the user) plans the work, then spawns one or more `claude -p --output-format stream-json --json-schema ...` subprocesses per task. Each worker gets a single, scoped task with explicit `--permission-mode`, `--tools`, `--allowedTools`, and `--disallowedTools`. Codex parses the structured output, runs verification, and decides next steps.

This is the simplest topology and the closest analog to CAO's `handoff` primitive without the MCP server.

### 10.2 Candidate B — Pipeline with fan-out

Codex defines a deterministic pipeline: plan → decompose → (fan-out N Claude workers in worktrees) → verify → merge. Each stage has a typed contract. The fan-out stage uses git worktrees (per §5.1) for filesystem isolation and a quality gate (per §6.4) before merge. This is closest to Brandon Redmond's setup, scaled down.

### 10.3 Candidate C — File-based mailbox with Codex supervisor

Codex and Claude communicate only via files (a `tasks/` directory of JSON tasks, a `results/` directory of structured outputs, a `mail/` directory for cross-agent messages). Codex polls and dispatches; Claude workers read assigned tasks and write results. This is closest to Overstory's mail bus, simplified to filesystem.

### 10.4 Trade-off comparison

| Dimension | A — Supervisor + subprocess | B — Pipeline + fan-out | C — File-based mailbox |
| --- | --- | --- | --- |
| Setup complexity | Lowest | Medium (worktree mgmt) | Medium (mailbox protocol) |
| Latency per task | Lowest | Low (parallel branches) | Higher (polling) |
| Debuggability | Good (tee stdout) | Good (per-worktree logs) | Best (every message persisted) |
| Audit trail | Per-call JSON in logs | Per-task worktree + logs | Full message history in files |
| Parallelism | Per orchestrator design | Native to the pattern | Native (workers pull queue) |
| Permission isolation | Per `--tools` / `--allowedTools` / `--disallowedTools` invocation | Per worktree + per call | Filesystem ACLs + per call |
| Recovery from worker crash | Re-run subprocess | Re-spawn in same worktree | Re-enqueue the task |
| Cross-vendor swap (Codex→other) | Easy (swap the CLI) | Easy per worker | Easy per consumer |
| Failure compounding risk | Medium (chain depth = supervisor decisions) | Low (verifier between stages) | Low if every stage validates |
| Operational surface | One process per call | Many worktrees + tmux | Filesystem + polling daemon |
| Fits a single-developer workflow | Yes | Yes (for 2-5 workers) | Overkill below 5 workers |
| Fits a fleet of 10+ workers | Stretched | Yes | Best |
| MCP needed? | No | No | No (but a natural extension) |

### 10.5 Hybrid worth considering

A→B is the natural growth path. Start with A (single supervisor making subprocess calls, schema-validated output, per-call permissions) and add B's worktree fan-out only when you have a workload that demonstrates the need. C is the right destination if you reach 10+ concurrent workers, but it adds protocol surface that is not justified earlier.

Whichever you pick, the cross-cutting invariants from the prior art are not optional:

- Schema-validated worker output (`--output-schema` on Codex, `--json-schema` on Claude).
- Explicit `--permission-mode`, `--tools`, `--allowedTools`, and `--disallowedTools` per worker call — never run a worker with default tool access.
- An independent verifier (different prompt, ideally different model) before any merge.
- Hard per-task budget caps (turns, wall-clock, USD) with pre-emptive pause.
- Quality gate (tests, lint, type-check) between worker output and integration.
- 3-fail rule — stop and escalate rather than burn tokens on the fourth retry.
- All cross-agent state persisted to files for replay.

---

## 11. References

### Prior-art systems
- AWS CLI Agent Orchestrator — repo: <https://github.com/awslabs/cli-agent-orchestrator>
- AWS CAO introduction (blog) — <https://aws.amazon.com/blogs/opensource/introducing-cli-agent-orchestrator-transforming-developer-cli-tools-into-a-multi-agent-powerhouse/>
- Overstory — repo: <https://github.com/jayminwest/overstory>
- OpenAI Symphony — announcement: <https://openai.com/index/open-source-codex-orchestration-symphony/>
- OpenAI Symphony — repo: <https://github.com/openai/symphony>
- Help Net Security on Symphony — <https://www.helpnetsecurity.com/2026/04/28/openai-symphony-codex-orchestration-linear/>
- EloPhanto, *How I Orchestrate Claude Code, Codex, and Gemini CLI as a Swarm* — <https://dev.to/elophanto/how-i-orchestrate-claude-code-codex-and-gemini-cli-as-a-swarm-4p3c>
- Brandon Redmond, *Multi-Agent Orchestration: Running 10+ Claude Instances in Parallel (Part 3)* — <https://dev.to/bredmond1019/multi-agent-orchestration-running-10-claude-instances-in-parallel-part-3-29da>
- Awesome Agent Orchestrators catalog — <https://github.com/andyrewlee/awesome-agent-orchestrators>

### Pattern catalogs and analyses
- Addy Osmani, *The Code Agent Orchestra* — <https://addyosmani.com/blog/code-agent-orchestra/>
- Lushbinary, *Multi-Agent Orchestration Patterns: Supervisor, Swarm, Pipeline, Router* — <https://lushbinary.com/blog/multi-agent-orchestration-patterns-supervisor-swarm-pipeline-router-guide/>
- Microsoft Copilot Studio, *Multi-agent orchestration patterns and best practices* — <https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/multi-agent-patterns>
- focused.io, *Multi-Agent Orchestration in LangGraph: Supervisor vs Swarm* — <https://focused.io/lab/multi-agent-orchestration-in-langgraph-supervisor-vs-swarm-tradeoffs-and-architecture>
- LangGraph Supervisor reference — <https://reference.langchain.com/python/langgraph-supervisor>
- AutoGen, *Concurrent Agents* — <https://microsoft.github.io/autogen/stable//user-guide/core-user-guide/design-patterns/concurrent-agents.html>

### Reliability and failure analysis
- MindStudio, *Reliability Compounding Problem in AI Agent Stacks* — <https://www.mindstudio.ai/blog/reliability-compounding-problem-ai-agent-stacks>
- Augment Code, *Why Multi-Agent LLM Systems Fail and How to Fix Them* — <https://www.augmentcode.com/guides/why-multi-agent-llm-systems-fail-and-how-to-fix-them>

### CLI / SDK reference for the candidate CLIs
- Claude Code, *Run Claude Code programmatically (headless)* — <https://code.claude.com/docs/en/headless>
- Inside the Claude Agent SDK (stdin/stdout architecture) — <https://buildwithaws.substack.com/p/inside-the-claude-agent-sdk-from>
- Codex, *Non-interactive mode (`codex exec`)* — <https://developers.openai.com/codex/noninteractive>
- Codex CLI command reference — <https://developers.openai.com/codex/cli/reference>

### Isolation and git worktrees
- Augment Code, *How to Use Git Worktrees for Parallel AI Agent Execution* — <https://www.augmentcode.com/guides/git-worktrees-parallel-ai-agent-execution>
- MindStudio, *Git Worktrees for AI Coding* — <https://www.mindstudio.ai/blog/git-worktrees-parallel-ai-coding-agents>
- MindStudio, *Parallel Agentic Development With Git Worktrees: A Practical Playbook* — <https://www.mindstudio.ai/blog/parallel-agentic-development-git-worktrees>
