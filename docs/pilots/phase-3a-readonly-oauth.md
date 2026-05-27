# Phase 3a — T-0001 Read-Only OAuth-Mode Pilot (Audit Record)

> Tracked audit of the first Phase 3a pilot run. Captured **before** any baseline (`--bare` + `ANTHROPIC_API_KEY`) re-run overwrites the `runs/` artifacts, which are `.gitignore`d. This record is the durable trail of *what actually happened on the first execution*, including facts the wrapper itself does not surface.

| Field | Value |
| --- | --- |
| Pilot date | 2026-05-27 (re-verified 2026-05-27) |
| Host OS | Windows 11 Pro (10.0.26200) |
| Codex CLI | `codex-cli 0.134.0` (one patch ahead of the references snapshot `0.133.0`) |
| Claude Code | `2.1.150 (Claude Code)` (matches the snapshot) |
| Task ID | `T-0001` |
| Fixture | [`fixtures/T-0001.task.json`](../../fixtures/T-0001.task.json) — read-only enumeration of `docs/references/` |
| Wrappers exercised | [`scripts/run-worker.ps1`](../../scripts/run-worker.ps1) (PowerShell 7+), [`scripts/run-worker.sh`](../../scripts/run-worker.sh) (POSIX bash) |
| Schema gate | [`schemas/task-spec.schema.json`](../../schemas/task-spec.schema.json) (input), [`schemas/worker-result.schema.json`](../../schemas/worker-result.schema.json) (output) — both passed |
| Auth mode used | **OAuth (subscription)** via `-AllowOAuth` / `--allow-oauth` — **not** the baseline `--bare` + `ANTHROPIC_API_KEY` contract |
| Tracked run artifacts | None — `runs/` and `runs-sh/` are both `.gitignore`d. This document is the curated stand-in. |

## 1. Dispatch summary

The fixture is a deliberately minimal reader: list the direct files inside `docs/references/` alphabetically, do not read contents, no verification commands. `permission_mode` is `dontAsk`; tools are `Read,Glob,Grep`; `Bash`, `Edit`, `Write` are explicitly disallowed.

Both wrappers validated the spec, rendered the prompt, spawned `claude -p --output-format json`, validated the produced `result.json` against `worker-result.schema.json`, and reported `status: succeeded` with `claude_meta.subtype: success`.

## 2. Critical observation — `--bare` absent from the PowerShell argv

`scripts/run-worker.ps1:194` reads `if (-not $AllowOAuth) { $argv.Add("--bare") }`. The saved `runs/T-0001/argv.json` does **not** include `--bare`, so the only way this argv was produced is `-AllowOAuth` (`$true`). The pilot therefore ran **outside** the baseline determinism contract: under `--bare`, Claude ignores `CLAUDE.md` auto-discovery, hooks, plugins, MCP servers, and memory; without `--bare`, the worker can be influenced by whatever machine-local Claude configuration is present.

This fact is invisible from `result.json` (the result schema does not record auth mode). It is preserved here because the next baseline re-run will overwrite `runs/T-0001/` with a different `argv.json`.

## 3. Critical observation — sandbox disabled on Windows

`runs/T-0001/stderr.log` records, verbatim:

```
⚠ Sandbox disabled: sandbox is enabled but windows is not supported (requires macOS, Linux, or WSL2)
  Commands will run WITHOUT sandboxing. Network and filesystem restrictions will NOT be enforced.
```

Claude Code's OS-level sandbox is only available on macOS, Linux, and WSL2. The Phase 3a pilot ran on native Windows; containment relied entirely on prompt-level wording, `--allowedTools` / `--disallowedTools`, and `permission_mode dontAsk`.

This is **orthogonal** to the OAuth-vs-baseline auth axis. Switching to `--bare` + API key on Windows does **not** turn the sandbox on. Sandbox enforcement requires a separate Phase 3b re-run on WSL2 / Linux / container.

## 4. Cost and behavior differences between the two wrappers

Same fixture, same model, runs minutes apart:

| Wrapper | `claude_meta.total_cost_usd` | `result.json` `summary` first line |
| --- | --- | --- |
| `run-worker.ps1` (PowerShell) | `$0.04044175` | `` `docs/references/` 내 파일들을 알파벳 순으로 열거했습니다 (내용은 읽지 않음). `` |
| `run-worker.sh` (POSIX bash) | `$0.09415225` (2.3× the PowerShell run) | `` `★ Insight ─────────────────────────────────────` `` |

Two observations:

- **2.3× cost spread.** Plausible contributors, in no fixed order: CRLF (`\r\n`) line endings in the Bash-rendered prompt (visible inside `runs-sh/T-0001/result.json` `notes_for_codex`), input-cache miss on the second of two near-identical runs, or prompt-whitespace differences between the `cat <<EOF` here-doc and the PowerShell here-string. To be attributed experimentally before treating it as a real regression.
- **Summary extraction is defective.** Both wrappers extract `summary` as the first line of `result.result` (`run-worker.ps1:328`, `run-worker.sh:316`). When the worker leads with an `★ Insight ───` block — a side effect of Claude's *Explanatory* output style — the header becomes the summary. The Bash run shows the defect; the PowerShell run happened to land on a real first sentence by chance.

## 5. What this pilot validates

- The dispatch pipeline end-to-end: schema → spec → render → argv → spawn → result event → normalize → schema → persist. Both wrappers complete the loop.
- `worker-result.schema.json` accepts the produced result and is correctly enforced wrapper-side.
- `permission_mode dontAsk` with `Bash`/`Edit`/`Write` disallowed keeps the worker inside `docs/references/`. `permission_denials` is empty because the spec already disallowed everything the task did not need.
- The `claude_meta.subtype == "success"` path is reachable in OAuth mode.

## 6. What this pilot does NOT validate

- **Baseline determinism contract.** `--bare` was omitted; CLAUDE.md / hooks / plugins / MCP servers / memory were not blocked from contributing.
- **Sandbox enforcement.** Run was unsandboxed (Windows limitation).
- **Write-task contract.** Fixture is read-only; `changed_files` is always `[]` because the wrapper does not collect a diff. Worktree creation, diff capture, and `changed_files` reconciliation are untested.
- **`--json-schema` enforcement.** Fixture omits `output_schema`; even when set, the wrappers do not forward it.
- **Stream-json event capture.** No `events.jsonl` was produced; per-event observability (hook events, partial messages, API retries) is unverified.
- **Retry / format-repair.** Phase 3a is single-shot.
- **Path-leak / secret-scan / forbidden-path checks.** Wrapper does not verify these post-hoc; the worker happened to respect the prompt.
- **Workspace enforcement.** The PowerShell wrapper does not set `ProcessStartInfo.WorkingDirectory`; the subprocess inherits the wrapper's CWD rather than `spec.workspace`. In this fixture the two happen to coincide.

## 7. Phase 3b follow-up tasks derived from this pilot

In rough priority order:

1. **Baseline re-run.** Same fixture, `ANTHROPIC_API_KEY` set, no `-AllowOAuth`. Persist the new `runs/T-0001/argv.json` and confirm `--bare` is present. Capture baseline cost and `terminal_reason` as the comparison anchor.
2. **WSL2 / container re-run.** Same fixture under a sandbox-supporting OS; confirm the stderr warning disappears and per-tool sandbox restrictions activate.
3. **Workspace enforcement.** Set `ProcessStartInfo.WorkingDirectory` from `spec.workspace` (PowerShell); add an equivalent `cd` step in the Bash wrapper. Wrapper-side assert that resolved CWD equals `spec.workspace`.
4. **`output_schema` forwarding.** When the spec sets `output_schema`, read the file and pass its contents to `--json-schema`. Surface `subtype == "error_max_structured_output_retries"` distinctly.
5. **Summary extraction fix.** Strip leading `★ Insight ───`-style decoration before taking the first line; consider asking the worker for an explicit one-line summary inside a fenced block to remove guesswork.
6. **Stream-json adoption.** Add a stream mode that uses `--output-format stream-json`, tees to `events.jsonl`, and parses the final `result` event from the stream. Keep `--output-format json` as a fallback for tiny tasks.
7. **Diff capture and `changed_files` reconciliation.** Required before the first write-task pilot.
8. **Per-wrapper cost normalization.** Attribute the 2.3× cost spread; rule out CRLF / cache miss / prompt-whitespace differences before treating it as a real regression.

## 8. Verbatim sources

Raw artifacts live in the `.gitignore`d `runs/` and `runs-sh/` on the host machine. Key field-level citations preserved here so this record stands alone after those artifacts are overwritten.

**`runs/T-0001/argv.json` (PowerShell run, abbreviated):**

```json
[
  "-p",
  "--output-format", "json",
  "--permission-mode", "dontAsk",
  "--no-session-persistence",
  "--tools", "Read,Glob,Grep",
  "--allowedTools", "Read",
  "--allowedTools", "Glob",
  "--allowedTools", "Grep",
  "--disallowedTools", "Bash",
  "--disallowedTools", "Edit",
  "--disallowedTools", "Write",
  "--session-id", "00000000-0000-4000-8000-000000000001",
  "--max-turns", "4",
  "--max-budget-usd", "0.25",
  "<prompt text>"
]
```

Note the absence of `--bare` as the first argument.

**`runs/T-0001/stderr.log` (full):**

```
Warning: no stdin data received in 3s, proceeding without it. If piping from a slow command, redirect stdin explicitly: < /dev/null to skip, or wait longer.

⚠ Sandbox disabled: sandbox is enabled but windows is not supported (requires macOS, Linux, or WSL2)
  Commands will run WITHOUT sandboxing. Network and filesystem restrictions will NOT be enforced.
```

**`runs/T-0001/result.json` (PowerShell run, abbreviated):**

```json
{
  "task_id": "T-0001",
  "status": "succeeded",
  "summary": "`docs/references/` 내 파일들을 알파벳 순으로 열거했습니다 (내용은 읽지 않음).",
  "changed_files": [],
  "commands_run": [],
  "needs_human_review": false,
  "claude_meta": {
    "subtype": "success",
    "terminal_reason": "completed",
    "total_cost_usd": 0.04044175,
    "permission_denials": []
  }
}
```

**`runs-sh/T-0001/result.json` (Bash run, abbreviated — note the defective `summary`):**

```json
{
  "task_id": "T-0001",
  "status": "succeeded",
  "summary": "`★ Insight ─────────────────────────────────────`",
  "changed_files": [],
  "commands_run": [],
  "needs_human_review": false,
  "claude_meta": {
    "subtype": "success",
    "terminal_reason": "completed",
    "total_cost_usd": 0.09415225,
    "permission_denials": []
  }
}
```
