# codex-claude-system

> Design package for a system in which **Codex CLI acts as the orchestrator and Claude Code CLI acts as the worker**. Current phase: **Phase 3a — read-only single-shot wrapper landed**. The first pilot (`T-0001`, read-only) ran successfully under OAuth-mode (non-`--bare`) on Windows; the baseline `--bare` + `ANTHROPIC_API_KEY` contract has **not** yet been verified. Investigation references (`docs/references/`) and initial design proposals (`docs/design/`) remain frozen as the audit anchor.

---

## Status

| Phase | State | Output |
| --- | --- | --- |
| 1. Reference investigation | Done (2026-05-27) | `docs/references/` (source index, overview, and four core reports) |
| 2. Initial design proposal | Done (2026-05-27) | `docs/design/` (architecture, security, worker contracts) |
| 3a. Pilot — read-only single-shot wrapper | Done; OAuth-mode pilot succeeded. Baseline `--bare` + API key run **not yet verified**. | `schemas/`, `scripts/run-worker.{ps1,sh}`, `fixtures/T-0001.task.json`; ignored `runs/`, `runs-sh/` artifacts; audit at [`docs/pilots/phase-3a-readonly-oauth.md`](docs/pilots/phase-3a-readonly-oauth.md) |
| 3b. Pilot — stream-json, diff/path verification, worktree write tasks | Not started | — |
| 4. Production hardening | Not started | — |

---

## Repository Layout

```
codex-claude-system/
├── README.md                                      ← you are here
├── docs/
│   ├── references/                                ← external facts (frozen, dated)
│   │   ├── 00-local-environment.md                ← local CLI snapshot + URL index
│   │   ├── 00-overview.md                         ← reading order, key findings, open questions
│   │   ├── 01-codex-cli.md                        ← Codex CLI surface (orchestrator role)
│   │   ├── 02-claude-code-cli.md                  ← Claude Code CLI surface (worker role)
│   │   ├── 03-orchestration-patterns.md           ← multi-agent CLI patterns + prior art
│   │   └── 04-integration-examples.md             ← Codex + Claude integration case studies
│   ├── design/                                    ← system proposals (evolving)
│   │   ├── architecture.md                        ← baseline architecture + alternatives
│   │   ├── security.md                            ← combined permission/sandbox/secret policy
│   │   └── worker-contracts.md                    ← worker calling contract + pilot runbook
│   └── pilots/                                    ← tracked pilot audit notes
│       └── phase-3a-readonly-oauth.md             ← T-0001 OAuth-mode pilot fact record
├── schemas/                                       ← Phase 3a runtime contracts
│   ├── task-spec.schema.json
│   └── worker-result.schema.json
├── scripts/                                       ← Phase 3a wrappers (read-only single-shot)
│   ├── run-worker.ps1                             ← PowerShell 7+ wrapper (Windows-first)
│   └── run-worker.sh                              ← POSIX bash parity wrapper
├── fixtures/                                      ← canonical task specs
│   └── T-0001.task.json                           ← first read-only fixture
└── (runs/, runs-sh/)                              ← .gitignored pilot artifacts; never committed
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

> **Local CLI drift since the snapshot.** The Phase 3a pilot ran on a Windows machine where `codex --version` reports `codex-cli 0.134.0` and `claude --version` reports `2.1.150 (Claude Code)`. The Codex CLI has drifted one patch ahead of the snapshot table; Claude Code matches. The references are deliberately not retroactively updated — treat `0.133.0` as the investigation anchor and re-verify specific Codex behavior against the locally installed version before relying on it.

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

## Implementation Status

The package is no longer pure investigation + proposal. Phase 3a — a read-only single-shot worker wrapper — has landed. Phase 3b and the orchestrator side remain out of scope until promoted.

**Implemented in Phase 3a (this revision):**

- JSON Schemas: `schemas/task-spec.schema.json`, `schemas/worker-result.schema.json` — consumed by the wrappers at dispatch and at result normalization.
- Wrappers: `scripts/run-worker.ps1` (PowerShell 7+, Windows-first) and `scripts/run-worker.sh` (POSIX bash parity). Read-only, single-shot, no retries. Both validate input and output against the schemas.
- Fixture: `fixtures/T-0001.task.json` (read-only `docs/references/` enumeration).
- Run-history layout: `runs/<task_id>/{task.json,prompt.txt,argv.json,stdout.json,stderr.log,result.json}` produced under `runs/` (PowerShell) and `runs-sh/` (Bash). Both directories are `.gitignore`d; the curated audit record lives in `docs/pilots/`.
- Pilot result: T-0001 succeeded under both wrappers; the PowerShell run was executed via `-AllowOAuth` (subscription OAuth), so its `argv.json` does **not** contain `--bare`. The baseline contract (`--bare` + `ANTHROPIC_API_KEY`, sandbox enforcement) is not yet verified. See [`docs/pilots/phase-3a-readonly-oauth.md`](docs/pilots/phase-3a-readonly-oauth.md) for the full pilot audit.

**Still out of scope (Phase 3b and beyond):**

- Baseline contract validation: re-run T-0001 with `ANTHROPIC_API_KEY` + `--bare` and confirm `argv.json` contains `--bare`.
- Sandbox enforcement: re-run on WSL2 / Linux / container so the `"sandbox is enabled but windows is not supported"` warning no longer applies. Orthogonal to the auth-mode axis.
- Stream-json output: `--output-format stream-json`, `events.jsonl`, `--include-hook-events`, `--include-partial-messages`. The wrappers currently use `--output-format json` only.
- Wrapper workspace enforcement: the `workspace` field is currently advisory; `ProcessStartInfo.WorkingDirectory` is not set in the PowerShell wrapper.
- Output-schema enforcement: `output_schema` is accepted in the task spec but the wrappers do not forward it via `--json-schema`.
- Write tasks: per-task git worktree creation (`--worktree <name>`), diff capture (`diff.patch`), and `changed_files` reconciliation against the actual diff.
- Path/diff verification: forbidden-path leak detection, allowed-path enforcement of `changed_files`, secret-pattern scan on stdout/events/result.
- Result-summary extraction: both wrappers currently take the first line of `result.result`, which captures `★ Insight ───…` headers when the worker leads with prose decoration.
- Orchestrator side: the Codex `exec` planner, the aggregator that consumes `result.json`, retry/escalation policy, CI configuration, monitoring, and budget enforcement.

The full target directory structure remains the layout proposed in [`docs/design/worker-contracts.md`](docs/design/worker-contracts.md), which now marks Implemented-today vs Phase 3b items inline.
