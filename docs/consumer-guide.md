# Consumer Guide — operating the loop in your repository

How to install, run, and govern the Loop Runtime as the human in the loop. Design rationale lives in [architecture.md](./architecture.md); this is the operating manual.

---

## Prerequisites

- A git repository (clean working tree recommended before starting a run)
- Claude Code CLI installed and authenticated
- Windows PowerShell (V1 ships `run.ps1`; the contract is shell-agnostic)

## 1. Install

Copy the `.loop/` directory into your repository root. That is the entire installation — never edit its contents per-project (project-specific truth belongs in `knowledge/`, which the loop maintains itself).

## 2. Write the PRD

Create `PRD.md` at the repository root: business objective, requirements, constraints. Write it for a competent engineer who cannot ask you questions in real time — ambiguity you leave here either becomes a recorded assumption or an escalation that stops the loop.

## 3. Start the runtime

```powershell
powershell -File .loop/run.ps1
```

Useful parameters:

| Parameter | Default | Purpose |
|---|---|---|
| `-MaxIterations` | 50 | Iteration budget per run — deterministic safety stop, not a judgment |
| `-MaxConsecutiveCrashes` | 3 | Watchdog bound for invocations that die without reporting |
| `-Model` | (CLI default) | Model override for engine invocations |
| `-QuietEngine` | off | Suppress the live engine activity feed |
| `-DangerouslySkipPermissions` | off | Full permission bypass — only for sandboxed/VM environments |

### Watching the loop (is it running?)

The terminal that launches `run.ps1` owns the live view: a timestamped activity feed with a per-iteration stopwatch, one line per engine action —

```
=== Iteration 2 / 50 === started 20:35:30 | total elapsed 00:04:44
[20:35:41 +00:00:10] engine> Read .ai\STATE.md
[20:36:02 +00:00:31] engine> Bash ./gradlew.bat assembleDebug
Status: CONTINUE (iteration took 00:04:44, total elapsed 00:09:28)
```

New lines appearing = the loop is alive; the `+HH:MM:SS` stopwatch shows how deep into the iteration it is.

To follow the same feed from **any other terminal** (the runtime prints both log paths at startup):

```powershell
# clean, human-readable activity feed (safe to tail while the loop runs)
Get-Content "$env:TEMP\loop-run-<repo-name>.log" -Wait -Tail 20

# raw engine event stream — debugging only
Get-Content "$env:TEMP\loop-run-<repo-name>.raw.jsonl" -Wait -Tail 5
```

Other quick liveness checks:

```powershell
# is an engine invocation alive right now?
Get-Process claude -ErrorAction SilentlyContinue | Sort-Object StartTime | Select-Object -Last 3

# what has the loop committed so far?
git log --oneline loop/<prd-slug>
```

Only one runtime may run per repository — a second `run.ps1` refuses to start while another holds the run lock (`$env:TEMP\loop-run-<repo-name>.lock`; stale locks from dead processes are taken over automatically).

The first invocation finds no `.ai/` and therefore bootstraps: it reads your PRD, inspects the repository (including `CLAUDE.md` and READMEs — it never edits them), generates `knowledge/` and `.ai/`, creates the `loop/<prd-slug>` branch, and stops with `ESCALATE`.

## 4. The one mandatory gate: approve the Definition of Done

Open `.ai/DoD.md` and `.ai/ESCALATION.md`. The DoD is the exam the whole run will be graded against — this is your highest-leverage five minutes:

1. Edit the criteria freely: tighten vague ones, delete wrong ones, add missing ones. Every criterion must be provable by evidence.
2. Review the proposed standing capabilities (your repo's build/test/lint commands). Narrow anything too broad; paste approved entries into the named ledger file.
3. Write your decision **and rationale** under `## Decision` in `.ai/ESCALATION.md`.
4. Re-run `run.ps1`.

After approval the DoD is immutable to the engine: it may propose changes, never apply them.

## 5. While the loop runs

Nothing is required from you. The loop stops only for:

- **`ESCALATE`** — a decision above the engine's authority: an architecture change (Tier 2), an intent gap (Tier 3), a capability request, missing product information. Read `.ai/ESCALATION.md`, write decision + rationale, re-run. One pending escalation at a time, always.
- **`FAILED`** — execution itself is broken (environment, corruption, exhausted resources). Fix the environment, re-run; the engine resumes from the last checkpoint.
- **Watchdog / budget stops** — mechanical bounds tripped. Inspect, re-run to continue; the loop never loses verified work.

Interrupting is always safe: kill it whenever you like. Every iteration ends at a Stable Checkpoint (one atomic commit of code + state); the next invocation recovers mechanically — even from a mid-iteration crash, which it detects as a dirty working tree.

Watching progress: `git log --oneline` on the loop branch is the execution history; `.ai/STATE.md` is the engine's current memory; `.ai/AMENDMENTS.md` is the audited log of every plan mutation.

## 6. Completion and merge

`DONE` is only ever reported by a fresh verifier iteration that wrote none of the implementation and re-proved every DoD criterion. At that point the branch tip contains the implementation, updated `knowledge/`, and a **Cleanup Commit** whose message is the completion summary (criteria → evidence, notable amendments) — and no `.ai/` (execution state is the loop's memory, not your product; its full history remains in the branch's earlier commits).

Review the branch like any contribution. **Merging is your act — the engine never merges, never pushes, never touches your default branch.**

## 7. The next feature

Write a new `PRD.md`, run `run.ps1` again. `knowledge/` persists — verified commands and hard-won environmental lessons carry over; a new `.ai/` and a new loop branch are created for the run.

To abandon a run: delete `.ai/` and the loop branch. Nothing else to clean.

---

## What the engine may never do

Enforced mechanically (runtime deny rules) or by hard-stop protocol:

- Modify `.loop/`, any capability ledger, or its own permission settings
- Modify `PRD.md` or the approved `DoD.md`
- Widen a capability beyond what you approved
- Touch your default branch, push, merge, or rewrite history
- Declare `DONE` from the same invocation that implemented the final work
- Proceed past an unanswered escalation

## Honest limitations

- The capability system is a guardrail against accidents and drift — not containment for an adversarial process. For untrusted PRDs or maximum isolation, run the whole loop inside a VM/container (where `-DangerouslySkipPermissions` becomes reasonable).
- V1 assumes git and PowerShell; both are persistence/transport details, not architecture.
