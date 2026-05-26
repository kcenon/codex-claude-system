# Local Environment & Source Index (Investigation Snapshot)

> Snapshot of the local CLI environment used while compiling this references package, plus a topic-organized index of every external source cited in the four reference reports. Acts as the entry point for `docs/references/` and as the audit-trail anchor for "what was true on the investigation date."

| Field | Value |
| --- | --- |
| Snapshot date | 2026-05-27 (Asia/Seoul) |
| Working directory | `/Volumes/raphael/Sources` |
| Compilation context | Local CLI and official-docs review session |
| Companion reports | `00-overview.md`, `01-codex-cli.md`, `02-claude-code-cli.md`, `03-orchestration-patterns.md`, `04-integration-examples.md` |

---

## 1. Local CLI Snapshot

| Item | Value |
| --- | --- |
| Codex CLI path | `/Users/dongcheolshin/.npm-global/bin/codex` |
| Codex CLI version | `codex-cli 0.133.0` |
| Claude Code path | `/Users/dongcheolshin/.local/bin/claude` |
| Claude Code version | `2.1.150 (Claude Code)` |
| Codex automation entry points | `codex exec`, `codex mcp`, `codex mcp-server`, `codex app-server` |
| Claude automation entry points | `claude -p` (with `--bare`), `claude mcp`, `claude mcp serve`, `claude agents`, `claude remote-control` |

Version metadata recorded here is the **point-in-time** reference for every CLI claim in the companion reports. Re-run `codex --version` / `claude --version` and update this table before any design decision is acted on — both CLIs ship frequently.

### Locally Confirmed Flags (subset)

Verified against `--help` output on the snapshot date. Authoritative flag semantics live in the companion reports, not here; this table only confirms the flag *exists* in the installed binary.

- **`codex exec`** — `--sandbox <read-only|workspace-write|danger-full-access>`, `--json`, `--output-schema <FILE>`, `--output-last-message <FILE>`, `--ephemeral`, `--ignore-user-config`, `--ignore-rules`, `--cd <DIR>`, `--add-dir <DIR>`. See [`01-codex-cli.md` §3.2 / §3.3](01-codex-cli.md#32-non-interactive-exec-mode) for full semantics.
- **`codex mcp-server`** — stdio MCP server entry point. See [`01-codex-cli.md` §8.2](01-codex-cli.md#82-codex-as-mcp-server-codex-mcp-server).
- **`claude -p`** — locally listed in `--help`: `--output-format`, `--input-format`, `--json-schema`, `--permission-mode`, `--allowedTools`, `--disallowedTools`, `--tools`, `--max-budget-usd`, `--no-session-persistence`, `--mcp-config`, `--settings`, `--bare`, `--worktree <name>`, `--session-id`, `--fork-session`. Official docs also document headless flags not shown by local `--help`, including `--max-turns` and `--permission-prompt-tool`; treat those as doc-confirmed but not local-help-confirmed until a live invocation verifies them. See [`02-claude-code-cli.md` §3.2](02-claude-code-cli.md#32-headless-p-print-mode) and [§4](02-claude-code-cli.md#4-output-formats).
- **`claude mcp serve`** — Claude Code as an MCP server. See [`02-claude-code-cli.md` §9.2](02-claude-code-cli.md#92-as-mcp-server).

---

## 2. External Source Index — OpenAI Codex

Cited primarily in `01-codex-cli.md`. URLs verified on the snapshot date unless flagged otherwise.

| Topic | URL | Where used |
| --- | --- | --- |
| Codex CLI overview | https://developers.openai.com/codex/cli | `01` §1 |
| CLI options reference | https://developers.openai.com/codex/cli/reference | `01` §3.3 |
| Non-interactive mode | https://developers.openai.com/codex/noninteractive | `01` §3.2, §4 |
| Features | https://developers.openai.com/codex/cli/features | `01` §1 |
| Sandbox concepts | https://developers.openai.com/codex/concepts/sandboxing | `01` §5 |
| Agent approvals & security | https://developers.openai.com/codex/agent-approvals-security | `01` §5, `design/security.md` |
| Configuration | https://developers.openai.com/codex/config | `01` §6 |
| Codex SDK | https://developers.openai.com/codex/sdk | `01` §7 |
| Codex MCP client | https://developers.openai.com/codex/mcp | `01` §8.1 |
| Codex App Server | https://developers.openai.com/codex/app-server | `01` §8 / `04` §4.6 |
| Use Codex with Agents SDK (mcp-server) | https://developers.openai.com/codex/guides/agents-sdk | `01` §8.2 |
| Codex subagents | https://developers.openai.com/codex/subagents | `01` §9, `03` §8 |
| Codex hooks | https://developers.openai.com/codex/hooks | `01` §6, `design/security.md` |
| AGENTS.md guide | https://developers.openai.com/codex/guides/agents-md | `01` §6 |
| Changelog | https://developers.openai.com/codex/changelog | `01` §1 |
| Open source repo | https://github.com/openai/codex | `01` §2.1 |
| Known issue: non-TTY (closed) | https://github.com/openai/codex/issues/4219 | `04` §3.2, §6 — **verified CLOSED on 2026-05-27** |

---

## 3. External Source Index — Anthropic Claude Code

Cited primarily in `02-claude-code-cli.md`.

| Topic | URL | Where used |
| --- | --- | --- |
| Run Claude Code programmatically (headless) | https://code.claude.com/docs/en/headless | `02` §3.2 |
| CLI usage reference | https://code.claude.com/docs/en/cli-usage | `02` §3 / §5 |
| Settings (settings.json, scopes) | https://code.claude.com/docs/en/settings | `02` §6.1 |
| Permission modes | https://code.claude.com/docs/en/permission-modes | `02` §6.2 |
| Permissions (allow/deny rules) | https://code.claude.com/docs/en/permissions | `02` §6.3 |
| Sub-agents | https://code.claude.com/docs/en/sub-agents | `02` §7 |
| Parallel agents / teams | https://code.claude.com/docs/en/agents | `02` §7 / §10 |
| Hooks | https://code.claude.com/docs/en/hooks | `02` §8 |
| MCP integration | https://code.claude.com/docs/en/mcp | `02` §9 |
| Agent SDK overview | https://code.claude.com/docs/en/agent-sdk/overview | `02` §10 |
| Agent SDK TypeScript reference | https://code.claude.com/docs/en/agent-sdk/typescript | `02` §4.2 |
| Structured outputs (error handling) | https://code.claude.com/docs/en/agent-sdk/structured-outputs | `02` §4 / §10 |
| Remote control | https://code.claude.com/docs/en/remote-control | `02` §10 |

---

## 4. External Source Index — Orchestration Patterns & Prior Art

Cited primarily in `03-orchestration-patterns.md` and `04-integration-examples.md`.

| Topic | URL | Where used |
| --- | --- | --- |
| AWS CLI Agent Orchestrator (CAO) repo | https://github.com/awslabs/cli-agent-orchestrator | `03` §8.1 |
| CAO launch blog | https://aws.amazon.com/blogs/opensource/introducing-cli-agent-orchestrator-transforming-developer-cli-tools-into-a-multi-agent-powerhouse/ | `03` §8.1 |
| Overstory repo | https://github.com/jayminwest/overstory | `03` §8.2 |
| Addy Osmani — Code Agent Orchestra | https://addyosmani.com/blog/code-agent-orchestra/ | `03` §1, §4 |
| Microsoft Copilot Studio — multi-agent patterns | https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/multi-agent-patterns | `03` §1 |
| Lushbinary — supervisor/swarm/pipeline/router catalog | https://lushbinary.com/blog/multi-agent-orchestration-patterns-supervisor-swarm-pipeline-router-guide/ | `03` §2 |
| Brandon Redmond — 10 Claude instances in parallel | https://dev.to/bredmond1019/multi-agent-orchestration-running-10-claude-instances-in-parallel-part-3-29da | `03` §8.5 |
| MindStudio — multi-agent orchestration | https://www.mindstudio.ai/blog/multi-agent-orchestration-patterns | `03` §1, §5 |
| OpenAI Symphony launch coverage | https://www.helpnetsecurity.com/2026/04/28/openai-symphony-codex-orchestration-linear/ | `03` §8.3 |
| EloPhanto swarm | https://dev.to/elophanto/how-i-orchestrate-claude-code-codex-and-gemini-cli-as-a-swarm-4p3c | `03` §8.4 / `04` §4.2 |
| Codex plugin in Claude Code (Mark Chen) | https://medium.com/@markchen69/when-rivals-collaborate-installing-openais-codex-plugin-in-claude-code-5d3e503ce493 | `04` §4.1 |
| Sangho Oh — Claude + Codex agentic coding | https://medium.com/@sangho.oh/claude-codex-cli-agentic-coding-a98c83ba043e | `04` §4.3 |
| Ivan Bragin — Agentic planner vs shell-first surgeon | https://blog.ivan.digital/claude-code-vs-openai-codex-agentic-planner-vs-shell-first-surgeon-d6ce988526e8 | `04` §4.4 |
| `buildoak/agent-mux` (Nick Oak) — closest prior art for user's direction | https://github.com/buildoak/agent-mux | `04` §4.5 |
| `raysonmeng/agent-bridge` — MCP ↔ Codex app-server | https://github.com/raysonmeng/agent-bridge | `04` §4.6 |
| Termdock — Claude Code vs Codex CLI 2026 | https://www.termdock.com/en/blog/claude-code-vs-codex-cli | `04` §7 |
| Blake Crosley — Codex vs Claude Code 2026 | https://blakecrosley.com/blog/codex-vs-claude-code-2026 | `04` §7 |
| awesome-agent-orchestrators (catalog) | https://github.com/andyrewlee/awesome-agent-orchestrators | `03` §6 |

---

## 5. Trust Model & Caveats

- **Local `--help` is authoritative** for whether a flag *exists* on the installed version. The companion reports cite official docs for flag *semantics*; if `--help` disagrees with docs, the locally installed CLI wins and the discrepancy should be filed as an Open Question against the corresponding report.
- **External issues, web pages, READMEs, dependency manifests, and any scraped content** must be treated as untrusted input when fed to a worker. Prompt-injection protection is part of [`docs/design/security.md`](../design/security.md).
- **MCP wiring (`codex mcp add claude-code -- claude mcp serve`)** is plausible but the actual tool surface, schemas, and concurrency semantics are not fully documented. Validate with `codex mcp list`, the Codex TUI's `/mcp`, or an MCP inspector before depending on it. See [`02-claude-code-cli.md` §9.2 Open Questions](02-claude-code-cli.md#92-as-mcp-server) and [`04-integration-examples.md` §6](04-integration-examples.md#6-known-gaps-for-codex-orchestrator-claude-worker).
- **Live-data citations** (GitHub issue states, repo stars, release dates) are point-in-time. Any "still open / now closed" claim in the reports should be re-verified before being quoted in a design decision. The investigation date 2026-05-27 is the authoritative snapshot for this package.
- **Single-source secondary claims** (e.g., "~10% of public GitHub commits per morphllm") are flagged inline in the reports with `[UNVERIFIED — single secondary source]`. Do not promote these into design assumptions without first sourcing a primary statistic.

---

## 6. How to Use This Index

- Reading the package end-to-end: start at [`00-overview.md`](00-overview.md), then `01` → `02` → `03` → `04`. Come back to this file when you need the local CLI version or the canonical URL for a specific topic.
- Re-verifying a citation: find the topic in §2–§4, click through, and update the "Where used" column if a new report cites the same source.
- Adding a new source: append a row to the right section and link the citing report section. Keep the table sorted by topic, not by date added.
