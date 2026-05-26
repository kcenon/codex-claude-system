# codex-claude-system

> Design package for a system in which **Codex CLI acts as the orchestrator and Claude Code CLI acts as the worker**. Current phase: investigation references (`docs/references/`) + initial design proposals (`docs/design/`). No runtime code yet.

---

## Status

| Phase | State | Output |
| --- | --- | --- |
| 1. Reference investigation | Done (2026-05-27) | `docs/references/` (source index, overview, and four core reports) |
| 2. Initial design proposal | Done (2026-05-27) | `docs/design/` (architecture, security, worker contracts) |
| 3. Pilot implementation | Not started | — |
| 4. Production hardening | Not started | — |

---

## Repository Layout

```
codex-claude-system/
├── README.md                                      ← you are here
└── docs/
    ├── references/                                ← external facts (frozen, dated)
    │   ├── 00-local-environment.md                ← local CLI snapshot + URL index
    │   ├── 00-overview.md                         ← reading order, key findings, open questions
    │   ├── 01-codex-cli.md                        ← Codex CLI surface (orchestrator role)
    │   ├── 02-claude-code-cli.md                  ← Claude Code CLI surface (worker role)
    │   ├── 03-orchestration-patterns.md           ← multi-agent CLI patterns + prior art
    │   └── 04-integration-examples.md             ← Codex + Claude integration case studies
    └── design/                                    ← system proposals (evolving)
        ├── architecture.md                        ← baseline architecture + alternatives
        ├── security.md                            ← combined permission/sandbox/secret policy
        └── worker-contracts.md                    ← worker calling contract + pilot runbook
```

References and design are deliberately separated:

- `docs/references/` is **frozen in time**: every fact has a verification date. If a referenced fact changes upstream, the file is updated and the date bumped — not the design.
- `docs/design/` is **evolving**: it changes as Open Questions get answered and as the pilot reveals what works.

The two should never be merged.

---

## How to Read

Different starting points for different roles:

- **First read (anyone)** — start at [`docs/references/00-overview.md`](docs/references/00-overview.md). It gives the reading order, the key findings per report, and the cross-cutting findings that only show up when the four reports are read together.
- **System designer** — read the overview, then [`docs/design/architecture.md`](docs/design/architecture.md), following the cited reference sections as you go.
- **Security reviewer** — [`docs/design/security.md`](docs/design/security.md) → [`docs/references/01-codex-cli.md` §5](docs/references/01-codex-cli.md) → [`docs/references/02-claude-code-cli.md` §6](docs/references/02-claude-code-cli.md).
- **Prior-art researcher** — [`docs/references/04-integration-examples.md`](docs/references/04-integration-examples.md) end-to-end; the closest prior art is `§4.5` (`buildoak/agent-mux`).
- **Reviewer with only 30 minutes** — `00-overview.md` §3 (Key Findings) + `04-integration-examples.md` §3 (Direction of Control) + `design/architecture.md` (Baseline).

---

## Audit Anchor

This package is the **single source of truth for the investigation that produced it**.

| Field | Value |
| --- | --- |
| Investigation date | 2026-05-27 (Asia/Seoul) |
| Codex CLI version | `codex-cli 0.133.0` |
| Claude Code version | `2.1.150 (Claude Code)` |
| Working directory | `/Volumes/raphael/Sources` |
| Authoritative URL index | [`docs/references/00-local-environment.md`](docs/references/00-local-environment.md) |
| Consolidated Open Questions | [`docs/references/00-overview.md` §5](docs/references/00-overview.md) |

Any claim that cites a live data source (GitHub issue state, repo stars, release dates, blog publication dates) is point-in-time as of the investigation date. Re-verify before quoting in a design decision.

---

## Design Direction (Brief)

The user's design direction is **Codex CLI on top, Claude Code CLI underneath** — i.e., Codex plans and dispatches, Claude executes scoped units of work and returns structured results. This is the *minority* direction in public prior art (most prior art assumes Claude on top, Codex as adversarial reviewer or executor); see `docs/references/04-integration-examples.md` §3 for the taxonomy and `§4.5` for the one prior art that matches this direction.

The baseline architecture proposed in [`docs/design/architecture.md`](docs/design/architecture.md) is:

> A Codex `exec` planner produces task specs; a wrapper layer dispatches each task to a `claude -p --bare --output-format stream-json --json-schema …` subprocess in an isolated worktree; results are schema-validated and aggregated back into Codex, which decides verification, retry, or escalation.

This is the simplest pattern that satisfies the cross-cutting findings in [`docs/references/00-overview.md` §4](docs/references/00-overview.md). MCP bridges, file-based mailboxes, and other architectures are documented as alternatives, not as the starting point.

---

## Operational Lessons

The investigation produced a few process lessons that should carry into the pilot:

- Keep `docs/references/` and `docs/design/` separate. References answer "what was true on the verification date"; design documents answer "what we propose to build from those facts." Mixing the two makes audit review and future updates harder.
- Give worker agents exactly one output target when using multi-agent document generation. Supplementary helper files and duplicated `*-reference.md` reports are easy to create and hard to reconcile later.
- Use a producer-reviewer pattern for source-heavy documents. The `openai/codex#4219` status change is the concrete example: a separate fact-check pass caught that the headless-mode issue had closed, changing the correct design assumption from "Codex lacks headless support" to "long-lived multiplexing remains self-rolled."
- Design the verifier first. Worker output is a hypothesis until schema validation, permission checks, diff review, and tests confirm it.

---

## Conventions

| Topic | Rule |
| --- | --- |
| Language | Current package convention: English documentation. |
| Attribution | No generated-content attribution banners inside artifacts. |
| Citations | Markdown links, scoped to file + section anchor (e.g. `[01 §5.4](docs/references/01-codex-cli.md#54-recommended-combinations)`). |
| Verification dates | Inline (`Verified: 2026-05-27`) wherever a live data source is cited. |
| Speculation labels | `[INFERRED]` or `[UNVERIFIED — single secondary source]` when the underlying source is weak. |
| Reference updates | Update verification date in the citing file, not just the index. |
| Design updates | Treat each design change as a new revision; preserve the rationale in commit messages. |

---

## What's Not Here Yet

The following are explicitly *out of scope for this phase* — the package is investigation + proposal, not implementation. Items below will appear in later phases.

- Runtime code (orchestrator script, worker wrapper, aggregator)
- JSON Schema files (`schemas/worker-result.schema.json`, `schemas/task-spec.schema.json`)
- Executable scripts (`scripts/run-worker.sh`, etc.)
- Run-history directory (`runs/<task-id>/{task.json,prompt.txt,stdout.json,...}`)
- CI configuration
- Test suites
- Monitoring / cost-tracking / budget enforcement implementation
- Pilot execution logs

When pilot implementation starts, the directory structure proposed in [`docs/design/worker-contracts.md`](docs/design/worker-contracts.md) is the planned target layout.
