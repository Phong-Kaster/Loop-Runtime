# Loop Runtime

A **portable autonomous execution engine**: copy one directory (`.loop/`) into any repository, drop in a feature PRD, run one script — and an AI-driven loop plans, implements, builds, tests, reviews, and verifies the feature until it is provably done, stopping for a human only at genuine decision points.

An application of Addy Osmani's **Loop Engineering** concept: the human stops being the person who prompts the agent step-by-step and becomes the system designer who owns intent.

> Human owns intent. The loop owns execution.
> The agent forgets. The repository doesn't.
> The engine requests. The human decides. The runtime enforces.

---

## Why

Prompt engineering makes the human the orchestrator: every step needs a new instruction, and progress dies with the chat context. Loop engineering replaces that role with a system:

- **Intent enters once** — as a PRD plus one approved Definition of Done.
- **Execution is continuous** — a loop of stateless iterations, each of which reconstructs everything from the repository, works to a verified checkpoint, and persists everything back.
- **Human involvement concentrates where it has the most leverage** — defining what "done" means before any code, deciding escalations the engine may not decide alone, and reviewing the final branch. Never mid-implementation babysitting.

The objective is not autonomous coding for its own sake. It is an execution system that is **safe, predictable, auditable, resumable, and human-governed** — the AI is one component inside a well-designed system, not the system itself.

## Product vs distributable

- **This repository** is the *product*: design docs, glossary, decision records, examples. Consumers never receive it.
- **`.loop/`** is the *distributable*: the only thing a consumer repository copies.

> Design principle: **every file inside `.loop/` must directly contribute to runtime execution.** If a file exists only to explain how the runtime was designed, it belongs outside `.loop/`.

---

## Repository tree

```
Loop-Runtime/
├── .loop/                                    ← THE DISTRIBUTABLE — what consumers copy
│   ├── capabilities/
│   │   └── baseline.json                     ← permanent low-risk capability ledger (read, edit, local git)
│   ├── templates/                            ← blueprints the engine instantiates into .ai/ and knowledge/
│   │   ├── DoD.template.md                   ← Definition of Done (human-approved intent contract)
│   │   ├── PLAN.template.md                  ← machine-owned execution strategy
│   │   ├── STATE.template.md                 ← execution memory: progress, history, assumptions
│   │   ├── TASK.template.md                  ← one task: description, dependencies, evidence
│   │   ├── ESCALATION.template.md            ← request/decision protocol between engine and human
│   │   ├── PROJECT-KNOWLEDGE.template.md     ← durable per-repo knowledge cache
│   │   └── capabilities.template.json        ← standing / goal-scoped capability ledgers
│   ├── ENGINE.md                             ← the Execution Engine Specification (the engine's system prompt)
│   ├── POLICIES.md                           ← generic engineering policy: tiers, retry, review, evidence, risk classes
│   └── run.ps1                               ← the Runtime: thin, dumb, mechanical outer loop
│
├── docs/
│   ├── adr/                                  ← decision records (semantic filenames — self-explanatory)
│   │   ├── ADR-001-prd-and-dod-source-of-truth.md
│   │   ├── ADR-002-stateless-iteration-dumb-runtime.md
│   │   ├── ADR-003-checkpoint-abstraction-and-git-persistence.md
│   │   ├── ADR-004-capability-permission-and-trust-chain.md
│   │   └── ADR-005-fresh-context-review-done-candidate.md
│   ├── architecture.md                       ← the complete design: planes, lifecycles, loop, contracts
│   └── consumer-guide.md                     ← how to install and operate the loop in your repository
│
├── examples/                                 ← (empty until the first real consumer validates V1)
├── CONTEXT.md                                ← the glossary — canonical vocabulary of the architecture
└── README.md                                 ← this file
```

### What lands in a consumer repository

```
consumer-repo/
├── .loop/          ← copied verbatim; never edited per-project; replaced only by runtime upgrades
├── PRD.md          ← human-owned product intent (per feature)
├── .ai/            ← generated per run; machine-owned execution state; disposable; removed from the final deliverable
└── knowledge/      ← generated at first bootstrap; survives every run; the repo's verified operational memory
```

Four artifacts because there are **four lifecycles** (install-time / per-feature / per-run / per-repository) — see [docs/architecture.md §3](./docs/architecture.md).

---

## How it works (60 seconds)

1. **Install:** copy `.loop/` into your repo. **Provide intent:** write `PRD.md`. **Start:** `powershell .loop/run.ps1`.
2. **Bootstrap** (automatic — the first invocation finds no `.ai/`): the engine reads the PRD and the repository, generates `knowledge/` and `.ai/`, creates a dedicated `loop/<prd-slug>` branch, and stops with one question: *approve the Definition of Done*.
3. **The one mandatory human gate:** review/edit the DoD (the testable meaning of "done") and the proposed toolchain capabilities; write your decision into `.ai/ESCALATION.md`; re-run.
4. **The loop runs unattended:** each iteration is a fresh process that recovers, orients from repository state, selects the highest-value task, implements, builds, tests, gets a **fresh-context review** from a clean-context subagent, reconciles everything it learned, and commits **one atomic checkpoint** (code + state together). Status `CONTINUE` → the runtime invokes it again.
5. **It stops only for real reasons:** `ESCALATE` (a decision above its authority — architecture change, intent gap, capability request) or `FAILED` (execution environment broken). You answer in a file; the loop resumes.
6. **Completion is earned, not claimed:** the iteration that finishes the last task may not declare victory. A *fresh* verifier iteration — which wrote none of the code — re-proves every DoD criterion, strips the execution state from the branch tip (Cleanup Commit, whose message is the completion summary), and only then reports `DONE`.
7. **You merge.** The engine never touches your default branch, never pushes, never merges.

Everyday commands:

```powershell
# start (or resume) the loop — this terminal gets the live timer/activity feed
cd D:\path\to\your-repo
powershell -ExecutionPolicy Bypass -File .loop\run.ps1

# cautious first run: cap the iteration budget
powershell -ExecutionPolicy Bypass -File .loop\run.ps1 -MaxIterations 5

# watch a running loop from another terminal (clean activity feed)
Get-Content "$env:TEMP\loop-run-<repo-name>.log" -Wait -Tail 20

# see what the loop has committed so far
git log --oneline loop/<prd-slug>
```

Full operating manual (parameters, watching the loop, escalations, merge): [docs/consumer-guide.md](./docs/consumer-guide.md).

---

## Core design commitments

- **Stateless iterations, dumb runtime.** Every iteration starts from repository state, so resumability is *tested continuously*, not trusted. The runtime has no judgment — its whole intelligence is a status reaction table plus two mechanical safety bounds (crash watchdog, iteration budget).
- **Status contract.** Every engine invocation ends with exactly one of `CONTINUE / DONE / ESCALATE / FAILED`; producing no status *is* the crash signal.
- **Tiered mutability.** Tier 1: the engine freely reshapes tasks (always logged). Tier 2: architecture/strategy changes hard-stop for approval. Tier 3: intent belongs to the human, forever.
- **Capability-based permissions.** No permanent allowlists — grants carry intent, command, scope, and lifetime (goal-scoped by default, auto-expiring), enforced through the trust chain *Human → Ledger → Runtime Compiler → Settings → Engine*. A guardrail against accidents and drift — documented honestly as not being a boundary against an adversarial engine.
- **Three minds.** The builder implements, a clean-context reviewer challenges (it never sees the builder's reasoning — that reasoning may contain the original mistake), and a fresh verifier confirms completion.
- **Everything auditable.** Plan amendments logged with reasons; human decisions recorded with rationale; every checkpoint a commit; the branch history *is* the execution history.

The complete vocabulary lives in [CONTEXT.md](./CONTEXT.md); the full design in [docs/architecture.md](./docs/architecture.md); the reasoning behind each hard-to-reverse choice in [docs/adr/](./docs/adr/).

---

## When to use the loop — and when not to

The loop's per-iteration overhead is roughly **constant** (fresh-process orientation ~1 min, a build per checkpoint, review passes, one extra verification iteration). Its value — unattended execution, no context rot, crash resume, audited decisions, completion you don't have to verify yourself — **scales with feature size**. So the economics invert with task size:

| Task | Right tool |
|---|---|
| Small fix, one-file feature, anything you'd finish in one sitting | An interactive AI session — the loop's overhead dominates and it will feel slow |
| A real feature PRD (hours-to-days of work you'd otherwise prompt-and-review step by step) | The loop — the overhead amortizes and the guarantees take over |
| Anything you want to run unattended (overnight, while doing other work) | The loop — that's what it's for |

Don't judge the loop by a hello-world; judge it by unattended correctness on work you didn't want to babysit.

## Status

**V1 validated against its first real consumer** (Android Jetpack Compose project, hello-world-notification PRD, 2026-07-08). Every contract fired correctly in a real run: bootstrap → DoD escalation gate → capability grants → checkpoint commits with build evidence → fresh-context review → the engine invoking the DONE-Candidate rule on itself (refusing to self-certify and deferring completion to a clean verifier iteration).

Field findings driving the next iteration of the runtime:

- **Plan granularity must scale with PRD size** — bootstrap split a 2-task feature into 5 tasks, multiplying the per-iteration overhead (policy tune, pending).
- Observability must never kill execution — a log-tail file lock once crashed the whole loop; log writes are now shared-mode and fail-silent, and a run lock allows only one runtime per repository (fixed).
- Iteration counters differ between engine (counts from STATE, includes bootstrap) and runtime (counts this session's invocations) — cosmetic, pending alignment.
- Console shows mojibake for UTF-8 punctuation on default Windows code pages — cosmetic, pending `[Console]::OutputEncoding` fix.
- Multi-engine portability (Codex, Gemini) is architecturally confined to one adapter surface in `run.ps1` — see [ADR-006](./docs/adr/ADR-006-engine-adapter-boundary.md).

The repository stays intentionally small: additional structure earns its way in through real usage, not anticipated complexity. Deferred items and their revisit-triggers: [docs/architecture.md §12](./docs/architecture.md).
