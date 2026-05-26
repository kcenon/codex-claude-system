# Codex + Claude Code Integration: Existing Approaches

> Status: Reference catalogue compiled 2026-05-27.
> Scope: prior art for combining OpenAI Codex CLI with Anthropic Claude Code CLI in the same workflow, with a particular focus on the **"Codex as orchestrator → Claude as worker"** direction.
> Provenance: All non-trivial claims are linked to sources. Anything labelled "Author's claim" is the linked author's opinion, not an independently verified fact.

---

## 1. Why Combine Codex and Claude Code

The published prior art consistently frames the two CLIs as complementary rather than substitutable. The most common rationale is **cross-model second opinion** — different training histories and architectures mean Claude and GPT models do not share identical blind spots, so a code-review pass from one over the other catches issues neither would catch alone ([MindStudio][mindstudio-codex-cc-review], [xda-developers][xda-claude-codex]).

A condensed summary of the strengths each author tends to assign (full numeric comparison in §7):

| Capability                | Claude Code                                              | Codex CLI                                                  |
|---------------------------|----------------------------------------------------------|------------------------------------------------------------|
| Headless / non-interactive | Mature `claude -p` and Agent SDK ([Claude Code Docs][cc-headless]) | `codex exec` is the supported non-interactive entry point; the original headless feature request ([openai/codex#4219][codex-headless-issue]) is **closed** (verified 2026-05-27). Remaining limits are around long-lived parent processes multiplexing several `codex exec` children (see §6). |
| Sandbox isolation          | Application-layer permission modes, hooks, and optional Bash sandboxing | Codex sandbox: Seatbelt on macOS, bwrap + seccomp on Linux, AppContainer on Windows ([01-codex-cli.md §5](01-codex-cli.md#5-sandbox-approval-model)) |
| Plugin / marketplace       | Native plugins (`/plugin marketplace add ...`) ([openai/codex-plugin-cc][openai-plugin-cc]) | TOML-based config; no plugin marketplace in the same sense |
| Subagents                  | Implicit delegation via Task tool ([Claude Code Docs][cc-subagents]) | Explicit only — "Codex only spawns a new agent when you explicitly ask it to" ([OpenAI Developers][openai-subagents]) |
| MCP support                | Mature MCP client; can also be configured as a server     | Both client and server: `codex mcp-server` ([OpenAI Developers — Agents SDK][openai-agents-sdk]) |
| Open source                | Closed-source CLI                                         | Apache 2.0, Rust ([Termdock][termdock])                    |

Authors recommend the combination because the *axes of weakness are roughly orthogonal*: Nick Oak puts it as "mode collapse between Claude and OpenAI models is roughly orthogonal" ([Nick Oak][nickoak-agentmux]).

---

## 2. Integration Channels

Five distinct mechanisms appear in the surveyed prior art. Many tools combine more than one.

### 2.1 MCP server / client

Both CLIs speak MCP. Codex can be exposed as an MCP server via `codex mcp-server` ([OpenAI Developers — Agents SDK][openai-agents-sdk]); Claude Code is a mature MCP client. This enables two patterns:

- **Claude Code → Codex as MCP server.** Most common direction. Examples: `tuannvm/codex-mcp-server` ([repo][tuannvm-mcp]), `ching-kuo/claude-codex` ([repo][ching-kuo-claude-codex]).
- **Bidirectional MCP / JSON-RPC bridge.** `raysonmeng/agent-bridge` runs an MCP client inside Claude Code and translates to/from Codex's app-server WebSocket; messages flow in both directions with loop prevention ([agent-bridge README][agent-bridge], [openai/codex#15374][gh-disc-15374]).

### 2.2 Claude Code plugin for Codex (`codex@openai-codex`)

OpenAI shipped an official Claude Code plugin on 2026-03-30 ([openai/codex-plugin-cc][openai-plugin-cc], [OpenAI community announcement][openai-community-announce]). Despite being described as "MCP-based" by some third-party blogs ([MindStudio][mindstudio-codex-cc-review]), the README itself describes the mechanism as wrapping "your local Codex CLI and Codex app server" — i.e. a **plugin that shells out to the local `codex` binary and its app-server**, not a pure MCP server ([codex-plugin-cc README][openai-plugin-cc-readme]). Slash commands exposed: `/codex:review`, `/codex:adversarial-review`, `/codex:rescue`, `/codex:status`, `/codex:result`, `/codex:cancel`, `/codex:setup` ([codex-plugin-cc README][openai-plugin-cc-readme]).

### 2.3 Direct subprocess invocation

A skill or wrapper script invokes `codex exec` (non-interactive mode that streams JSONL events to stdout) or `claude -p` from inside the other CLI's session.

- **`codex exec` from Claude Code.** Aman Mishra's `/run-codex` skill is a textbook example: a `~/.claude/skills/run-codex/SKILL.md` invokes `codex exec -c model=… --sandbox read-only --ephemeral "<prompt>"` and parses stdout ([Aman Mishra blog][amanhimself]). The MCP.Directory `codex-cli` skill ([mcp.directory][mcp-directory-codex-skill]) generalises this to a teaching document that lets Claude pick `--sandbox`, fan-out parallel runs, and use `codex resume`.
- **`claude` CLI from Codex.** Less common; `buildoak/agent-mux` is the clearest case (§4.7).

### 2.4 External orchestrator that drives both as workers

A third process owns the orchestration loop and treats both CLIs as interchangeable engines.

- `EloPhanto` (swarm + tmux + git worktrees, §4.2)
- `wshobson/agents` ("one source of truth, five harnesses" — generates artefacts for Claude Code, Codex CLI, Cursor, OpenCode, Gemini CLI, no live cross-harness call) ([repo][wshobson-agents])
- `cexll/myclaude` (Claude Code as orchestrator, calls `codeagent-wrapper` which dispatches to claude/codex/gemini/opencode) ([repo][cexll-myclaude])
- `catlog22/Claude-Code-Workflow` (JSON-driven, cadence-team, Codex can act as both orchestrator and worker in different phases) ([repo][catlog22-ccw])
- `nwiizo/ccswarm` (ProactiveMaster orchestrator; Codex/Aider/Claude Code are pluggable providers — but multi-provider execution is still simulated, not wired up) ([repo][nwiizo-ccswarm])

### 2.5 Agent Client Protocol (ACP) bridge

ACPX exposes Claude Code, Codex, Gemini CLI behind a unified ACP (JSON-RPC 2.0 over stdio) interface. Currently documents Claude Code → ACPX → Codex as the orchestrator direction ([casys.ai ACPX guide][acpx]).

---

## 3. Direction of Control

### 3.1 Claude as orchestrator, Codex as tool

This is **by far the most documented direction**. Every officially-shipped integration (the openai/codex-plugin-cc, the OpenAI community announcement, Sangho Oh's MCP recipe, MindStudio's analysis, MCP.Directory's `codex-cli` skill, smartscope.blog's three-level review-loop article) sits here.

### 3.2 Codex as orchestrator, Claude as worker ← the direction you want

The prior art here is **thin but real**. Evidence:

- **`buildoak/agent-mux`** is the strongest example. The author explicitly inverts the hierarchy: "A Codex main session spawns Opus 4.6 as the GSD coordinator via agent-mux — and now Opus is running inside Codex, with full orchestration powers." ([dev.to][devto-buildoak], [agent-mux repo][buildoak-agent-mux], [Nick Oak blog][nickoak-agentmux]). agent-mux invokes the `claude` CLI binary directly as a subprocess; bidirectional dispatch is symmetric ("any LLM can dispatch work to any other LLM through one JSON contract"). Built in Go.
- **`milisp/codexia`** is a Tauri-based "Agent Workstation" where Codex CLI is the orchestrator and Claude Code is one of the available agents; integration uses Codex's app-server JSON-RPC ([repo][milisp-codexia]). Author's positioning, not independently verified.
- **`leonardsellem/codex-subagents-mcp`** is *not* in this category despite the name. It lets Codex use Claude-*style* subagents (reviewer/debugger/security personas) that are themselves Codex processes — no Claude is actually invoked. Archived 2025-12-29 ([repo][leonardsellem-mcp]).
- **`catlog22/Claude-Code-Workflow`** mentions Codex as orchestrator-or-worker but the live patterns shown route through a Claude-led skill layer.

No public OpenAI-side equivalent of the `codex-plugin-cc` (a "Claude plugin for Codex") exists. Codex's official subagent docs only orchestrate other Codex agents — "no mention is made of interoperability with competing AI platforms" ([OpenAI Developers — Subagents][openai-subagents]). The GitHub issue that originally requested headless mode for Codex CLI — "Claude Code works fine in headless mode" but Codex panics or blocks in non-TTY environments — has since been resolved: [openai/codex#4219][codex-headless-issue] is **closed** (verified 2026-05-27). `codex exec` is the supported non-interactive entry point, and the remaining gap (per §6) is not the absence of headless mode but the lack of patterns for a long-lived parent multiplexing several `codex exec` children.

### 3.3 Peer / swarm

- **`EloPhanto`** treats Claude Code, Codex, and Gemini CLI as peer workers under a separate orchestrator ([dev.to][elophanto]).
- **`raysonmeng/agent-bridge`** enables real-time peer collaboration in a single working session ([agent-bridge][agent-bridge]).
- **`shakacode/claude-code-commands-skills-agents`** models them as peers and recommends switching by fluency / sticking point ([shakacode docs][shakacode]).
- **`OhadAssulin/headless-coder-sdk`** (`@headless-coder-sdk/*`) is a unifying adapter layer rather than an orchestrator: Codex, Claude, Gemini are independent backends behind one `Coder` interface ([repo][headless-coder-sdk], [openai/codex#6402][gh-disc-6402]). Reception was minimal (1 comment, later flagged for spam by a moderator).
- **AWS CLI Agent Orchestrator (CAO)** ([repo][cao-repo], [AWS blog][cao-aws-blog]) belongs in the peer/swarm bucket from the wire perspective: its provider matrix includes both Claude Code and Codex, but the topology is a central MCP server exposing `handoff`, `assign`, and `send_message` tools that any provider terminal calls. Channel-wise it is supervisor↔worker over MCP rather than the user's intended Codex-orchestrator → Claude-worker direction, but it is the most production-shaped multi-CLI orchestrator with Codex as one of the participants — relevant prior art for the harness layer even if the direction of control does not match. See [03-orchestration-patterns.md §8.1](03-orchestration-patterns.md#81-aws-cli-agent-orchestrator-cao) for the deeper write-up.

---

## 4. Case Studies

Each case below lists: one-line summary, integration channel, direction of control, workload split, strengths/weaknesses, source URL. Where a case has not been independently verified, this is noted.

### 4.1 Codex Plugin in Claude Code (`openai/codex-plugin-cc`) — Mark Chen

- **Summary.** Official OpenAI plugin (released 2026-03-30) installable from Claude Code's marketplace; exposes `/codex:*` slash commands.
- **Channel.** Claude Code plugin system; under the hood the plugin wraps the local `codex` binary and its app server (not MCP, despite some third-party blogs calling it MCP-based).
- **Direction.** Claude Code is orchestrator; Codex is delegate subagent (`codex:codex-rescue`).
- **Workload split.** Claude handles primary reasoning/instruction following. Codex runs read-only reviews (`/codex:review`), adversarial reviews (`/codex:adversarial-review`), or full task delegation with `--background` (`/codex:rescue`).
- **Strengths.** Official, one-command install; supports background jobs (`/codex:status`, `/codex:result`, `/codex:cancel`); reuses existing Codex auth & config.
- **Weaknesses.** Review-gate mode can "create a long-running Claude/Codex loop and may drain usage limits quickly" (README warning); multi-file reviews "might take a while"; requires Codex CLI installed locally. No reverse direction (no `/claude:*` commands inside Codex).
- **Sources.** [openai/codex-plugin-cc][openai-plugin-cc], [codex-plugin-cc README][openai-plugin-cc-readme], [Mark Chen — Medium][markchen], [OpenAI community announcement][openai-community-announce].

### 4.2 EloPhanto Swarm — `elophanto/EloPhanto`

- **Summary.** EloPhanto orchestrates Claude Code, Codex, and Gemini CLI as peer worker agents.
- **Channel.** Each agent runs in its own git worktree on a separate branch, hosted in persistent tmux sessions; monitoring polls every 10 minutes. **Author does not disclose the wire protocol** (no claim of MCP / subprocess / API in the post).
- **Direction.** EloPhanto is orchestrator above all three CLIs; the human is above EloPhanto ("You ← conversation → Me … ├─→ Claude Code ├─→ Codex ├─→ Gemini CLI").
- **Workload split (author's claim, not verified).** Codex: backend features, complex logic, edge cases. Claude Code: architectural concerns, quick frontend iteration. Gemini: UI polish, security reviews. EloPhanto: routing, context, multi-model PR review.
- **Strengths (author's claim).** Parallel execution; context preservation; multi-model PR review reduces false positives.
- **Weaknesses (author).** Context-window squeeze: "Fill it with code, and there's no room for business context."
- **Sources.** [dev.to — EloPhanto post][elophanto].

### 4.3 Sangho Oh — Claude + Codex CLI as agentic coding

- **Summary.** Claude Desktop calls Codex as a tool via MCP. Conceptual recipe (Claude orchestrates, Codex codes).
- **Channel.** MCP. Claude Desktop acts as MCP *client*, Codex runs as MCP *server* via the standard `claude_desktop_config.json` config.
- **Direction.** Claude orchestrator → Codex worker. Unidirectional.
- **Workload split.** Claude: orchestration, decision-making, prompt interpretation. Codex: code generation and scaffolding ("Use the codex tool to scaffold a React app layout").
- **Strengths.** Clean separation; Codex can also speak to additional MCP servers, so the worker is itself extensible.
- **Weaknesses.** Manual configuration; predates the official codex-plugin-cc and so requires DIY bridging.
- **Sources.** [Sangho Oh — Medium][sangho-oh].

### 4.4 Ivan Bragin — Agentic Planner vs Shell-First Surgeon

- **Summary.** Comparative analysis, no automated integration. Recommends running both in parallel ("Claude Code as the main driver for large refactors, Codex as the reviewer or 'second opinion'").
- **Channel.** Human-mediated; the only programmatic option referenced is "exposing Codex CLI as an MCP server so agents can invoke it as a tool" ([OpenAI Developers — Agents SDK][openai-agents-sdk]).
- **Direction.** N/A (human is orchestrator).
- **Workload split (author).** Claude Code for large, style-sensitive refactors, multi-tool automation, work that needs visible project memory (CLAUDE.md). Codex for surgical edits, framework migrations, terminal-native CLI work, step-controlled builds with tiered autonomy.
- **Sources.** [Ivan Bragin][ivan-blog].

### 4.5 `buildoak/agent-mux` (Nick Oak) — Codex as main, Claude Opus as coordinator ← key prior art for the user's design direction

- **Summary.** Cross-engine dispatch CLI in Go. Any engine can dispatch work to any other through a single JSON contract. **Concretely demonstrates Codex-as-orchestrator → Claude-Opus-as-worker (and coordinator).**
- **Channel.** Direct subprocess invocation of each engine's native CLI (`claude`, `codex`, `gemini`). Not an SDK / API wrapper. JSON contract: `{ engine, prompt, cwd, model?, timeout?, effort? }`. Worker identity lives in `~/.agent-mux/prompts/<name>.md` with YAML frontmatter.
- **Direction.** Symmetric. The author explicitly inverts the usual hierarchy: a Codex main session spawns Opus as the "Get Shit Done" coordinator via the `--coordinator` flag ("Codex gets a brain. The brain gets an army."). The Claude side of the spawn is the standard headless invocation surface — `claude -p` plus the headless-only flags documented in [02-claude-code-cli.md §3.2](02-claude-code-cli.md#32-headless-p-print-mode).
- **Workload split.** Opus plans the migration; Codex 5.3 high swarm executes; Codex `xhigh` audits. Skills and MCP servers are injected per worker based on task type.
- **Strengths.** Standardised JSON output (success, response, activity, timing); artefacts persist across timeout and process death; errors are returned as steering signals; tool is generic ("the calling LLM decides what to do, agent-mux handles the how").
- **Weaknesses (author).** ~2 months of iteration at time of writing; coordination overhead for simple tasks; requires careful skill/prompt design to avoid context bloat; tool availability varies across engines.
- **Sources.** [buildoak/agent-mux][buildoak-agent-mux], [dev.to — buildoak post][devto-buildoak], [Nick Oak blog][nickoak-agentmux].

### 4.6 `raysonmeng/agent-bridge` — Bidirectional MCP ↔ Codex app-server bridge

- **Summary.** Local-only, real-time, in-session bridge enabling Claude Code and Codex to co-author code in the same working session.
- **Channel.** Foreground component is an MCP *client* started by a Claude Code plugin; background daemon translates between MCP notifications and Codex's app-server WebSocket protocol (JSON-RPC; see [01-codex-cli.md §9 — `codex app-server`](01-codex-cli.md#9-orchestration-relevant-capabilities) for the method surface this daemon talks to). Includes loop prevention by tracking message source (`"claude"` vs `"codex"`).
- **Direction.** Bidirectional. Codex → daemon → Claude (via MCP notifications); Claude → daemon → Codex (via a `reply` tool).
- **Workload split.** Demonstrated by co-authoring the bridge itself; not a fixed division.
- **Strengths.** First demonstrated bidirectional bridge of its kind; local-only (no cloud intermediary).
- **Weaknesses (per README v0.1.6, May 2026).** Forwards only `agentMessage` items, not intermediate events; single Codex thread; **Codex cannot perform git operations (blocked sandbox); Claude Code handles all git workflows**; fixed ports → one instance per machine; single Claude foreground connection only.
- **Sources.** [agent-bridge][agent-bridge], [openai/codex Discussion #15374][gh-disc-15374].

### 4.7 ACPX — Agent Client Protocol bridge

- **Summary.** Headless, scriptable CLI client for ACP. Replaces PTY scraping with JSON-RPC 2.0 over stdio.
- **Channel.** ACP (JSON-RPC 2.0 over stdio).
- **Direction.** Claude Code → ACPX → Codex / Gemini. Unidirectional in the documented examples.
- **Workload split (author).** Claude for architecture/refactoring; Codex for targeted edits and tests; Gemini for large-context reviews. Named sessions allow simultaneous independent workflows.
- **Strengths.** Preserves structured metadata (tool calls, file diffs, reasoning steps, error semantics) that terminal scraping loses.
- **Weaknesses.** Same as MCP family: schema lock-in, requires both ends to speak ACP.
- **Sources.** [casys.ai ACPX][acpx].

### 4.8 `ching-kuo/claude-codex` — Plan / Implement / Review loops with smart routing

- **Summary.** Set of Claude Code skills/commands that wire Codex in as an MCP server for auditing and review.
- **Channel.** Codex as MCP server (`claude mcp add codex -s user -- codex`).
- **Direction.** Claude orchestrator → Codex MCP server worker.
- **Workload split.** `/plan-codex`: Claude Opus plans, Codex audits the plan (up to 3 rounds). `/execute-codex`: small change (≤2 files, ≤30 lines, no new logic) → Claude implements + Sonnet review; large change → Codex implements. `/claude-codex`: Claude implements, Codex reviews diff with verdict (APPROVED / WARNING / BLOCKED), Claude fixes critical issues.
- **Strengths.** Explicit smart-routing rules; structured verdicts; bounded iteration.
- **Weaknesses (inherited from review-loop pattern).** Risk of long Claude/Codex loops; latency for multi-file reviews.
- **Sources.** [ching-kuo/claude-codex][ching-kuo-claude-codex].

### 4.9 SmartScope — Three-level review loop automation

- **Summary.** Catalogues three ascending levels of automating the Claude-implements / Codex-reviews pattern.
- **Channel.** Level 1: `~/.claude/skills/codex-review/SKILL.md` invoking `codex exec`. Level 2: Claude Code plugin with a Stop Hook that intercepts session-end and auto-triggers review (up to 4 parallel Codex subagents). Level 3: full pipeline orchestration via `claude-codex`, `Claude-Code-Workflow`, or GitHub Agent HQ.
- **Direction.** All levels: Claude implementer → Codex reviewer.
- **Strengths.** Clear progression from individual to mid-team to enterprise governance.
- **Weaknesses.** Author notes Stop-Hook auto-trigger "may drain usage limits quickly."
- **Sources.** [SmartScope][smartscope-review-loop].

### 4.10 `cexll/myclaude` — Multi-backend orchestration

- **Summary.** Modular `/do`, `/omo` etc. workflows that route to Claude / Codex / Gemini / OpenCode via a wrapper.
- **Channel.** `~/.claude/bin/codeagent-wrapper` CLI; standardised JSON I/O; invokes target backend's CLI.
- **Direction.** Claude Code is orchestrator; codeagent-wrapper routes; backends execute. Top-down.
- **Workload split.** Configurable per workflow; backends are interchangeable.
- **Sources.** [stellarlinkco/myclaude][stellarlinkco-myclaude].

### 4.11 `nwiizo/ccswarm` — Multi-agent orchestration framework

- **Summary.** Rust workflow framework with ProactiveMaster orchestrator and git-worktree-isolated agents. Multi-provider (Claude Code, Aider, Codex, custom) declared but largely stubbed.
- **Channel.** Native PTY sessions via `ai-session` crate; planned Unix socket / SQLite IPC and an `ai-session` MessageBus.
- **Direction.** ProactiveMaster orchestrator → agent providers (workers).
- **Strengths.** Session-persistent manager (claimed 93% token reduction); template scaffolding; TUI monitoring.
- **Weaknesses.** As of v0.5.0 (2026-02-26), "Current Execution: Simulated responses (keyword-based matching)" — core coordination loop is "Not Started." 139 stars, 13 forks; early-stage.
- **Sources.** [nwiizo/ccswarm][nwiizo-ccswarm].

### 4.12 Aman Mishra — `/run-codex` skill (headless `codex exec` inside Claude Code)

- **Summary.** Minimal Claude Code skill that drives `codex exec` as a subprocess.
- **Channel.** Subprocess. `codex exec -c model="gpt-5.3-codex" -c model_reasoning_effort="xhigh" --sandbox read-only --ephemeral "<prompt>"`.
- **Direction.** Claude Code → Codex. Codex output captured via stdout, presented in Claude's UI.
- **Workload split.** Claude collects user input interactively (model, reasoning level) via `AskUserQuestion`, then delegates code-review / refactor / edit tasks to Codex.
- **Strengths.** Sandbox enforcement; ephemeral sessions; nothing to install beyond the skill.
- **Weaknesses (author).** Initial skill had wrong Codex model names that required manual refinement; Codex's markdown table formatting "less polished than Claude Code's presentation."
- **Sources.** [amanhimself.dev][amanhimself].

### 4.13 `tuannvm/codex-mcp-server` — Codex CLI as an MCP server

- **Summary.** Wraps Codex CLI behind an MCP server interface usable by any MCP client.
- **Channel.** MCP. Flow: Claude Code → `codex-mcp-server` → Codex CLI → OpenAI API.
- **Direction.** Claude (MCP client) → Codex (MCP server).
- **Strengths.** One-click installers for VS Code / VS Code Insiders / Cursor; tools include `codex`, `review`, `websearch`.
- **Weaknesses.** Requires Codex CLI ≥ 0.75.0; session persistence limited per server instance; thread-ID / structured-output features need Codex ≥ 0.87.
- **Sources.** [tuannvm/codex-mcp-server][tuannvm-mcp].

### 4.14 `OhadAssulin/headless-coder-sdk` — Unified adapter for headless coder SDKs

- **Summary.** Wrapper / adapter library presenting Codex, Claude Agent SDK, and Gemini CLI behind a single `Coder` interface.
- **Channel.** Native SDK calls per backend (not subprocess to CLI binary). `codex-adapter`, `claude-adapter`, `gemini-adapter`.
- **Direction.** N/A — this is an adapter, not an orchestrator. Inter-agent calls are not demonstrated; the only example shows Claude and Codex running in parallel as independent reviewers, with Gemini consuming the joined structured output.
- **Strengths.** "Switch backends with a single line of code"; standardises threads, streaming, structured outputs, permissions, sandboxing.
- **Weaknesses.** Discussion got minimal traction (1 comment, later spam-flagged).
- **Sources.** [OhadAssulin/headless-coder-sdk][headless-coder-sdk], [openai/codex Discussion #6402][gh-disc-6402].

### 4.15 `bigboybamo` — Manual `task.md` handoff for .NET features

- **Summary.** Manual two-terminal workflow: Claude plans, Codex builds, a `task.md` markdown file is the contract.
- **Channel.** Human switches terminals. No automation, no plugin, no MCP.
- **Direction.** Human-mediated; conceptually Claude planner → Codex builder.
- **Workload split.** Claude reads the whole project (via `CLAUDE.md`), produces specifications in `task.md`. Codex receives the focused brief and implements without needing full repo context.
- **Sources.** [dev.to — bigboybamo][bigboybamo].

### 4.16 Patrick D'Appolonio — `dux` TUI

- **Summary.** TUI that wraps the official CLIs (Claude, Codex, Gemini, OpenCode) and runs them in parallel git worktrees.
- **Channel.** Direct CLI invocation in tmux-style panes.
- **Direction.** Peer/swarm. "No explicit orchestrator — agents operate independently in parallel."
- **Sources.** [patrickdap blog][patrickdap].

---

## 5. Workload Division Patterns

Five patterns recur across the surveyed prior art.

1. **Planning vs execution.** Claude plans, Codex implements. Cited in `ching-kuo/claude-codex` (`/plan-codex`), `bigboybamo` (`task.md` contract), MCP.Directory `codex-cli` skill (Claude returns runnable wrappers around `codex exec`), Ivan Bragin's framing ("rapid prototyping, creative fill-in, mimicking style" vs "structured, step-controlled builds"). Rationale: Claude's long-context reasoning + CLAUDE.md memory plays well with planning; Codex's surgical edits + sandboxing play well with execution.

2. **Generation vs review (adversarial second opinion).** One model implements, the other reviews. The dominant pattern in the *official* `codex-plugin-cc` (`/codex:review`, `/codex:adversarial-review`), SmartScope's three-level loop article, MindStudio's cross-provider-review write-up. Rationale: a model that just wrote code "defends the decisions it just made" ([xda-developers][xda-claude-codex]).

3. **Long-context analysis vs surgical edit.** Authors commonly position Claude's larger-context workflows for full-repo refactors or codebase understanding, and Codex's sandbox-isolated `codex exec` for tight, mechanical edits. Treat the exact context-window numbers in §7 as point-in-time third-party claims, not design assumptions. Cited in Ivan Bragin, MCP.Directory `codex-cli` skill.

4. **Routing by size / risk.** `ching-kuo/claude-codex` `/execute-codex` routes by file/line count: small change (≤2 files, ≤30 lines, no new logic) → Claude; large change → Codex. Author's rationale not explicitly given; the heuristic appears to assume Claude has lower review overhead per change.

5. **Multi-axis review (model x model).** Claude code review of Codex output, *and* Codex code review of Claude output, often in the same pipeline. EloPhanto, agent-bridge, and the SmartScope Level-2/3 patterns. Rationale: orthogonal blind spots (Nick Oak).

---

## 6. Known Gaps for "Codex Orchestrator → Claude Worker"

This is the **explicit design direction the user is pursuing**. The honest summary:

**What exists.**

- `buildoak/agent-mux` (§4.5) — a working tool that demonstrates a Codex main session spawning Claude Opus as a coordinator subagent through a Go-based dispatch CLI. Subprocess channel; symmetric JSON contract. As close to a production-grade example as the public ecosystem has.
- `milisp/codexia` (cited in §3.2) — a desktop "Agent Workstation" where Codex is the orchestrator and Claude Code can be one of the agents, via Codex app-server JSON-RPC. Author positioning; not independently verified. (ACPX is the other §3.2-cited bridge and is detailed in §4.7.)
- Headless inversion at the user level — `codex exec` *can* be invoked from inside a Claude Code session, but the reverse (`claude -p` invoked from inside a `codex` session) is documented only by `agent-mux` and a couple of swarm-orchestrator tools that wrap both CLIs from outside.

**What is missing / what you will likely have to build.**

- **No official OpenAI plugin or skill** that mirrors `codex-plugin-cc` in reverse. OpenAI's own subagent docs are explicit: Codex orchestrates *other Codex* agents only — "no mention is made of interoperability with competing AI platforms" ([OpenAI Developers — Subagents][openai-subagents]). OpenAI does ship **Symphony** ([repo][symphony-repo], [announcement][symphony-announce]) as the official Codex-as-orchestrator tool, but Symphony's worker model is Codex; it has no first-party support for invoking Claude workers (see [03-orchestration-patterns.md §8.3](03-orchestration-patterns.md#83-openai-symphony) for the architectural write-up). So the gap is more precise than "no official Codex→Claude orchestrator exists" — *Codex-as-orchestrator over Codex* is officially supported; *Codex-as-orchestrator over Claude* is not.
- **No mature Codex MCP-client → Claude MCP-server pattern.** Claude Code is consistently positioned as an MCP *client* in published patterns. Setting it up as a server that Codex calls is plausible (the Claude Agent SDK supports programmatic invocation) but not a documented out-of-the-box flow.
- **Codex's headless support has stabilized; long-lived multiplexing is still self-rolled.** `openai/codex#4219` (the original "headless / non-interactive mode" feature request) is **closed** as of verification on 2026-05-27 — `codex exec` is the supported non-interactive entry point and is exercised by every case study above. The remaining concern is no longer the absence of headless mode but the absence of *patterns* for a long-lived parent process that multiplexes several `codex exec` children, intercepts their structured events, and survives turn timeouts. The closest published prior art for this is **§4.5 `agent-mux`** (Go-based dispatch CLI that wraps multiple engine CLIs as subprocesses) and **§4.11 `ccswarm`** (multi-agent orchestration framework); both invent their own multiplexer rather than relying on any official Codex-side primitive. There is no published pattern for *orchestrating Claude from Codex* specifically (as opposed to orchestrating Codex over Codex, which OpenAI's subagents docs cover).
- **No standardised "Claude as worker" contract.** The Headless Coder SDK adapter pattern (§4.14) and ACPX (§4.7) are the closest, but neither is widely adopted, and both effectively put a third orchestration process above both CLIs rather than making Codex the orchestrator.
- **No example with Codex as orchestrator over Claude that ships compliance / audit-trail concerns.** Deterministic message logs, traceable tool invocations, per-session sandboxing, and reproducible result contracts are not addressed in any of the surveyed prior art.

**Practical implication.** The integration channel most likely to work for "Codex orchestrator → Claude worker" is direct **subprocess invocation of `claude -p` (Claude Agent SDK headless mode)** from a host process driven by Codex — i.e., the same architecture `agent-mux` uses, generalised. See [02-claude-code-cli.md §10.1](02-claude-code-cli.md#101-worker-invocation-primitives) for the worker-call primitives and [§10.3](02-claude-code-cli.md#103-authentication-for-headless-ci) for the authentication path that a Codex-side parent process must satisfy. MCP can be added later for tool exposure, but as the *transport* for orchestration, MCP in the reverse direction is largely unexplored territory.

---

## 7. Codex vs Claude Code Capability Comparison

The two third-party comparison articles disagree on several numbers and dates — both are recorded below. Where they conflict, both values are shown.

| Dimension                  | Claude Code                                              | Codex CLI                                               | Source                                |
|----------------------------|----------------------------------------------------------|---------------------------------------------------------|---------------------------------------|
| Subscription tiers         | Pro $20 / Max 5x $100 / Max 20x $200                     | Plus $20 / Pro 5x $100 / Pro 20x $200                   | [Blake Crosley][blake-crosley], [Termdock][termdock] |
| Default model              | Opus 4.7 (Max/Team Premium); Sonnet 4.6 (Pro/Standard)   | GPT-5.4 (Mar 2026 release); GPT-5.5 (per morphllm)      | [Blake Crosley][blake-crosley], [morphllm][morphllm]  |
| Per-token API pricing      | Opus 4.7: $5 in / $25 out per MTok                       | GPT-5.4: $2.50 in / $15 out per MTok                    | [Blake Crosley][blake-crosley]        |
| Context window             | 1M tokens (default on Max/Team/Enterprise since Mar 2026); 1M on Opus 4.7 at standard pricing | 200K (morphllm) — or 272K default + experimental 1.05M long-context mode at 2× input / 1.5× output multiplier (Blake Crosley); or 1M experimental w/ GPT-5.4, 400K standard (Termdock) | [Termdock][termdock], [Blake Crosley][blake-crosley], [morphllm][morphllm] |
| Sandbox                    | Permission modes, hooks, and optional Bash sandboxing | Codex sandbox: Seatbelt (macOS), bwrap + seccomp (Linux), AppContainer (Windows) | [02-claude-code-cli.md §6](02-claude-code-cli.md#6-tools-permissions-sandbox), [01-codex-cli.md §5](01-codex-cli.md#5-sandbox-approval-model) |
| Open source                | Closed-source                                            | Apache 2.0, Rust-based                                   | [Termdock][termdock]                  |
| MCP support                | Native, mature ecosystem                                 | Native, config.toml-based; can also run as MCP server via `codex mcp-server` | [Termdock][termdock], [OpenAI Developers — Agents SDK][openai-agents-sdk] |
| Headless / programmatic    | `claude -p` + Claude Agent SDK (Python / TypeScript / CLI) | `codex exec` (non-interactive); the original headless feature request [openai/codex#4219][codex-headless-issue] is **closed** (verified 2026-05-27); long-lived parent multiplexing several `codex exec` children is still self-rolled (see §6) | [Claude Code Docs][cc-headless], [openai/codex#4219][codex-headless-issue] |
| Subagents                  | Implicit delegation via Task tool; Agent Teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, sourced from the [Agent Teams docs][cc-agent-teams] — 02-claude-code-cli.md §7 covers subagents but has no dedicated coverage of this experimental flag); per-agent git worktree isolation, no hard cap on parallel agents | Subagents GA 2026-03-14; manager-worker; **explicit only**; default `max_threads=6`, `max_depth=1`, default cap 6 concurrent; cloud-sandbox isolation per task | [Claude Code Docs — Agent Teams][cc-agent-teams], [OpenAI Developers — Subagents][openai-subagents], [morphllm][morphllm], [AddyOsmani][addyosmani] |
| SWE-bench Verified         | Opus 4.7: 87.6%                                          | GPT-5.4: ~80% (third-party); GPT-5.5: 88.7% (morphllm)  | [Blake Crosley][blake-crosley], [morphllm][morphllm] |
| Terminal-Bench 2.0         | Opus 4.7: 69.4%                                          | GPT-5.4: 75.1% (Blake Crosley); GPT-5.5: 82.7% (morphllm) | [Blake Crosley][blake-crosley], [morphllm][morphllm] |
| Plugin marketplace         | Yes, native (`/plugin marketplace add ...`)              | None of the same shape; project-level `.codex/agents/` TOML | [openai/codex-plugin-cc][openai-plugin-cc], [OpenAI Developers — Subagents][openai-subagents] |
| Cloud sandbox / delegation | Local execution; Agent Teams runs locally                | `codex cloud exec`; up to 6 concurrent cloud threads     | [Blake Crosley][blake-crosley]        |
| Token-efficiency note      | Claude reportedly uses 3-4× more tokens than Codex on identical tasks but produces "more thorough output" (morphllm — author's claim, not independently verified) | —                                                       | [morphllm][morphllm]                  |
| Public GitHub commit share | "Approximately 10% of all public GitHub commits (~326K daily)" per morphllm `[UNVERIFIED — single secondary source: morphllm; no first-party GitHub data confirms this figure]`; "~4% (135K/day)" per MindStudio  | —                                                       | [morphllm][morphllm], [MindStudio][mindstudio-codex-cc-review] |

---

## 8. Open Questions

Items that this report did not pin down during writing and that should be confirmed against primary sources before being relied on for harness design. The other three reference reports each carry their own "Open Questions / Gaps" section; this one mirrors that convention.

- **Release / latest-commit dates for several case studies.** §4 entries (`buildoak/agent-mux`, `raysonmeng/agent-bridge`, `milisp/codexia`, `nwiizo/ccswarm`, `cexll/myclaude`, `ching-kuo/claude-codex`, `OhadAssulin/headless-coder-sdk`, `bigboybamo`'s post) were summarized from the cited sources at time of writing; the activity level of each repo (last commit, open vs closed issues, release cadence) was not re-checked before publication. Re-confirm before treating any of them as production-shaped.
- **`milisp/codexia`'s Codex-as-orchestrator claim.** §4.7 / §3.2 note this is "author positioning; not independently verified" — no independent reproduction or third-party write-up of the orchestrator-direction claim was located.
- **morphllm "~10% of all public GitHub commits (~326K daily)" figure.** Cited in §7 with the new `[UNVERIFIED]` label. The morphllm comparison page is the only secondary source; the underlying methodology and the conflicting "~4% (135K/day)" figure from MindStudio were not reconciled.
- **EloPhanto, Sangho Oh, Ivan Bragin workload-split rationales.** §4.2 / §4.3 / §4.4 cite the authors' own claims; comparative benchmarks for "Codex for backend complexity vs. Claude for architectural concerns" were not located in independent sources.
- **`openai/codex#4219` resolution context.** §3.2 / §6 / §7 now record the issue as **closed (verified 2026-05-27)**. The fix commit, the PR that closed it, and the precise scope of "supported headless" (vs. the multi-child multiplexing gap §6 still calls out) were not traced through to the source for this report.
- **Symphony's Claude-worker roadmap.** §6 notes Symphony has no first-party support for Claude workers today. Whether this is a deliberate scope decision or a near-term roadmap item was not established.

---

## 9. References

[acpx]: https://casys.ai/blog/acpx-multi-agent-orchestration "ACPX Inside Claude Code: Practical Multi-Agent Orchestration"
[addyosmani]: https://addyosmani.com/blog/code-agent-orchestra/ "Addy Osmani — The Code Agent Orchestra"
[agent-bridge]: https://github.com/raysonmeng/agent-bridge "raysonmeng/agent-bridge"
[amanhimself]: https://amanhimself.dev/blog/running-headless-codex-cli-inside-claude-code/ "Aman Mishra — Running headless Codex CLI inside Claude Code"
[bigboybamo]: https://dev.to/bigboybamo/how-i-use-claude-code-and-codex-together-to-build-net-features-faster-1n40 "dev.to — bigboybamo on Claude+Codex for .NET"
[blake-crosley]: https://blakecrosley.com/blog/codex-vs-claude-code-2026 "Blake Crosley — Codex vs Claude Code (2026)"
[buildoak-agent-mux]: https://github.com/buildoak/agent-mux "buildoak/agent-mux"
[cao-aws-blog]: https://aws.amazon.com/blogs/opensource/introducing-cli-agent-orchestrator-transforming-developer-cli-tools-into-a-multi-agent-powerhouse/ "AWS Open Source Blog — Introducing CLI Agent Orchestrator"
[cao-repo]: https://github.com/awslabs/cli-agent-orchestrator "awslabs/cli-agent-orchestrator"
[catlog22-ccw]: https://github.com/catlog22/Claude-Code-Workflow "catlog22/Claude-Code-Workflow"
[cc-agent-teams]: https://code.claude.com/docs/en/agent-teams "Claude Code Docs — Agent Teams"
[cc-headless]: https://code.claude.com/docs/en/headless "Claude Code Docs — Run Claude Code programmatically"
[cc-subagents]: https://code.claude.com/docs/en/sub-agents "Claude Code Docs — Create custom subagents"
[cexll-myclaude]: https://github.com/cexll/myclaude "cexll/myclaude"
[ching-kuo-claude-codex]: https://github.com/ching-kuo/claude-codex "ching-kuo/claude-codex"
[codex-headless-issue]: https://github.com/openai/codex/issues/4219 "openai/codex#4219 — headless / non-interactive mode for Codex CLI"
[devto-buildoak]: https://dev.to/buildoak/codex-inside-claude-code-subagents-inside-codex-1oe5 "dev.to — Codex Inside Claude Code. Subagents Inside Codex."
[elophanto]: https://dev.to/elophanto/how-i-orchestrate-claude-code-codex-and-gemini-cli-as-a-swarm-4p3c "EloPhanto — How I orchestrate Claude Code, Codex, and Gemini CLI as a swarm"
[gh-disc-6402]: https://github.com/openai/codex/discussions/6402 "openai/codex Discussion #6402 — Headless Coder SDK"
[gh-disc-15374]: https://github.com/openai/codex/discussions/15374 "openai/codex Discussion #15374 — Claude Code Channels ↔ Codex App Server bidirectional bridge"
[headless-coder-sdk]: https://github.com/OhadAssulin/headless-coder-sdk "OhadAssulin/headless-coder-sdk"
[ivan-blog]: https://blog.ivan.digital/claude-code-vs-openai-codex-agentic-planner-vs-shell-first-surgeon-d6ce988526e8 "Ivan Bragin — Claude Code vs OpenAI Codex: agentic planner vs shell-first surgeon"
[leonardsellem-mcp]: https://github.com/leonardsellem/codex-subagents-mcp "leonardsellem/codex-subagents-mcp"
[markchen]: https://medium.com/@markchen69/when-rivals-collaborate-installing-openais-codex-plugin-in-claude-code-5d3e503ce493 "Mark Chen — When Rivals Collaborate"
[mcp-directory-codex-skill]: https://mcp.directory/blog/claude-codex-cli-skill-guide "MCP.Directory — Claude codex-cli skill: 10 ways to bridge Claude Code and Codex CLI"
[milisp-codexia]: https://github.com/milisp/codexia "milisp/codexia"
[mindstudio-codex-cc-review]: https://www.mindstudio.ai/blog/openai-codex-plugin-claude-code-cross-provider-review "MindStudio — What Is the OpenAI Codex Plugin for Claude Code"
[morphllm]: https://www.morphllm.com/comparisons/codex-vs-claude-code "morphllm — Codex vs Claude Code (May 2026)"
[nickoak-agentmux]: https://www.nickoak.com/posts/agent-mux/ "Nick Oak — agent-mux: Cross-Engine Subagents for Claude Code and Codex CLI"
[nwiizo-ccswarm]: https://github.com/nwiizo/ccswarm "nwiizo/ccswarm"
[openai-agents-sdk]: https://developers.openai.com/codex/guides/agents-sdk "OpenAI Developers — Use Codex with the Agents SDK"
[openai-community-announce]: https://community.openai.com/t/introducing-codex-plugin-for-claude-code/1378186 "OpenAI Developer Community — Introducing Codex Plugin for Claude Code"
[openai-plugin-cc]: https://github.com/openai/codex-plugin-cc "openai/codex-plugin-cc"
[openai-plugin-cc-readme]: https://github.com/openai/codex-plugin-cc/blob/main/README.md "openai/codex-plugin-cc README"
[openai-subagents]: https://developers.openai.com/codex/subagents "OpenAI Developers — Subagents (Codex)"
[patrickdap]: https://www.patrickdap.com/post/how-to-run-multiple-agents/ "Patrick D'Appolonio — How to run multiple Claude Code or Codex agents in parallel"
[sangho-oh]: https://medium.com/@sangho.oh/claude-codex-cli-agentic-coding-a98c83ba043e "Sangho Oh — Claude + Codex CLI: Agentic Coding"
[shakacode]: https://github.com/shakacode/claude-code-commands-skills-agents/blob/main/docs/claude-code-with-codex.md "shakacode/claude-code-commands-skills-agents — Claude Code with Codex"
[smartscope-review-loop]: https://smartscope.blog/en/blog/claude-code-codex-review-loop-automation-2026/ "SmartScope — Automating the Claude Code × Codex Review Loop"
[stellarlinkco-myclaude]: https://github.com/stellarlinkco/myclaude "stellarlinkco/myclaude (mirror of cexll/myclaude)"
[symphony-announce]: https://openai.com/index/open-source-codex-orchestration-symphony/ "OpenAI — Open-source Codex orchestration: Symphony"
[symphony-repo]: https://github.com/openai/symphony "openai/symphony"
[termdock]: https://www.termdock.com/en/blog/claude-code-vs-codex-cli "Termdock — Claude Code vs Codex CLI"
[tuannvm-mcp]: https://github.com/tuannvm/codex-mcp-server "tuannvm/codex-mcp-server"
[wshobson-agents]: https://github.com/wshobson/agents "wshobson/agents — multi-harness agentic plugin marketplace"
[xda-claude-codex]: https://www.xda-developers.com/use-claude-code-and-codex-together-combination-does-something-neither-can-do-alone/ "xda-developers — I use Claude Code and Codex together"
