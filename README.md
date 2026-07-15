# Loop Runtime

A **portable autonomous execution engine** for Claude Code: install one skill, hand it a requirement — inline text or a document path — and a loop of fresh AI iterations plans, implements, builds, tests, reviews, and verifies the feature until it is provably done, stopping for you only at genuine decision points.

An application of Addy Osmani's **Loop Engineering** concept: the human stops being the person who prompts the agent step-by-step and becomes the system designer who owns intent.

> Human owns intent. The loop owns execution.
> The agent forgets. The repository doesn't.
> The engine requests. The human decides. The runtime enforces.

---

## What it is

Loop Runtime is two things working together:

1. **A Claude Code skill** (`/loop-runtime`) — the easy on-ramp. Install it once per repository, then type `/loop-runtime <requirement>` to start or resume a run. The skill materializes the runtime, stages your requirement as `PRD.md`, launches the engine in the background, streams its activity live into your conversation, and turns every decision point into a normal question instead of a file you have to open and edit.
2. **A stateless execution engine underneath** (`.loop/`) — the actual "loop": a thin, dumb PowerShell runtime (`run.ps1`) that repeatedly re-invokes Claude Code headlessly, reading one status word after each invocation (`CONTINUE` / `DONE` / `ESCALATE` / `FAILED`) and reacting mechanically. All the intelligence lives in `ENGINE.md`, the engine's system-prompt specification — never in the runtime itself.

You don't need to think about these as separate things day-to-day — the skill exists specifically so you never have to touch `.loop/` directly.

---

## Install (once per repository)

```
npx skills@latest add Phong-Kaster/Loop-Runtime
```

This installs the `loop-runtime` skill into `.claude/skills/loop-runtime/` (and `.agents/skills/loop-runtime/`) in your current repo, and records it in that repo's `skills-lock.json` — the same mechanism you'd use for any other shared Claude Code skill package. Nothing else needs to be copied by hand.

> Prefer not to use the skill installer? `.loop/` is also a plain, self-contained distributable — copy it into a repo's root and run `powershell .loop/run.ps1` directly from a terminal. See [docs/consumer-guide.md](./docs/consumer-guide.md) for that path. The skill is the recommended way in for anyone already working inside Claude Code.

---

## Use it

```
/loop-runtime <inline requirement text>
/loop-runtime <path to a requirement document>
/loop-runtime
```

- **Inline text** — e.g. `/loop-runtime add a dark mode toggle to Settings that persists via DataStore` — gets written verbatim into `PRD.md` at your repo root. No rewriting, no summarizing.
- **A document path** — e.g. `/loop-runtime C:\reqs\dark-mode.md` — gets staged as the PRD source instead.
- **No argument** — resumes with whatever `PRD.md` already exists (first run in a repo with no `PRD.md` yet will ask you for one).

From there, the skill:

1. Syncs `.loop/` at your repo root from the installed skill files (so the runtime is always current with whatever version you last installed).
2. Launches `.loop/run.ps1` in the background and attaches a live log stream to the conversation — you watch the engine work the same way you'd watch any other command's output, without it ever tying up your terminal or hitting a foreground timeout.
3. Turns every `ESCALATE` into a real question, right in the chat — the engine's own proposed options become your choices. You never open `.ai/ESCALATION.md` yourself; the skill writes your decision into it and resumes automatically.
4. Reports a **roll-up summary across every `loop/*` branch** in the repo when the run reaches `DONE` — not just the one that just finished — so you always see the full picture of what's mergeable, what's still in progress, and what's stuck waiting on a decision.
5. Never merges, pushes, or touches your default branch. That step is always yours.

---

## Worked example

This is a real, verified run — not a hypothetical. `/loop-runtime write a hello world notification` in an empty scratch repo:

**1. Bootstrap runs automatically** (no `.ai/` existed yet). The engine reads the PRD, inspects the repo (nothing there, no conventions to infer), confirms Node.js is on `PATH`, creates the branch `loop/hello-world-notification`, and generates `knowledge/PROJECT.md` plus the full `.ai/` scaffold (`DoD.md`, `PLAN.md`, three task files, `STATE.md`).

**2. First stop — a real question, not a file:**

> Approve the Definition of Done? Also: "notification" is ambiguous in the PRD — console-printed output (recommended, zero dependencies) or a real OS-level desktop toast (needs a new capability)? And approve a standing `Bash(node *)` capability for this repo's toolchain?

Answered in chat: console output, approve the capability, approve the DoD as drafted. The skill writes all three into `.ai/ESCALATION.md`'s Decision section and relaunches — no file opened by hand.

**3. The loop implements unattended:** `src/notify.js` (prints `"Hello, World!"`, exits 0), `test/notify.test.js` (asserts the greeting via `node:test`), `README.md` updated with run/test instructions. Build and tests both pass. A **fresh-context review sub-agent** — given only the diff, the task, the DoD, and the policies, never the implementation reasoning — flags two non-blocking Minor notes (stdout path not directly asserted; trivial string duplication between source and test) and nothing Critical. One atomic checkpoint commit, `CONTINUE`.

**4. A fresh Verifier iteration** — which wrote none of the code — independently re-runs all five DoD criteria from scratch. All hold. It then hits a genuine capability boundary: removing `.ai/` (the required Cleanup Commit) is a destructive git operation it isn't authorized for. Rather than force it, it escalates:

> Final verification passed. May I be granted a one-time, goal-scoped capability to run the Cleanup Commit, or would you rather do that one step manually?

Answered: grant the scoped, one-time capability. It expires the moment `.ai/` is removed — no standing deletion capability left behind afterward.

**5. `DONE`.** Roll-up summary:

| Branch | Status | Contains | Merge-ready? |
|---|---|---|---|
| `loop/hello-world-notification` | DONE | `src/notify.js`, `test/notify.test.js`, updated `README.md`, `knowledge/` cache | Yes — clean tree, `.ai/` removed, 6 commits |

Two Minor review notes carried into the summary so they aren't lost when `.ai/` disappears. Merging `loop/hello-world-notification` into your default branch is left for you to do by hand.

Total: two escalations, both answered as ordinary conversation; zero files opened; one mergeable branch at the end.

---

## Repository tree

```
Loop-Runtime/
├── skills/engineering/loop-runtime/          ← THE SKILL — what `npx skills@latest add` installs
│   ├── SKILL.md                              ← the skill's own instructions (bootstrap, supervise, escalate, roll-up)
│   ├── ENGINE.md                              ← copy of the Execution Engine Specification
│   ├── POLICIES.md                           ← copy of the engineering policy
│   ├── capabilities/baseline.json            ← copy of the permanent low-risk capability ledger
│   ├── templates/                            ← copy of the .ai/ + knowledge/ blueprints
│   └── scripts/run.ps1                       ← copy of the runtime script
│
├── .loop/                                    ← THE STANDALONE DISTRIBUTABLE — for manual/non-skill installs
│   ├── capabilities/baseline.json
│   ├── templates/
│   ├── ENGINE.md
│   ├── POLICIES.md
│   └── run.ps1
│
├── tests/                                    ← Pester unit tests for run.ps1's own mechanics
│   ├── run.Tests.ps1                         ← status reactions, PRD staging, prerequisites, via a fake-claude stub
│   └── fixtures/fake-claude.ps1              ← stands in for the real `claude` CLI — no API calls, no cost
│
├── docs/
│   ├── adr/                                  ← decision records (semantic filenames — self-explanatory)
│   ├── architecture.md                       ← the complete design: planes, lifecycles, loop, contracts
│   └── consumer-guide.md                     ← manual-install operating manual (parameters, escalations, merge)
│
├── examples/                                 ← (empty until the first real consumer validates V1)
├── CONTEXT.md                                ← the glossary — canonical vocabulary of the architecture
└── README.md                                 ← this file
```

### What lands in a consumer repository

```
consumer-repo/
├── .claude/skills/loop-runtime/   ← installed by the skill installer
├── .agents/skills/loop-runtime/   ← installed by the skill installer (identical copy)
├── .loop/                         ← synced from the skill on every /loop-runtime invocation
├── PRD.md                         ← human-owned product intent (per feature)
├── .ai/                           ← generated per run; machine-owned execution state; disposable
└── knowledge/                     ← generated at first bootstrap; survives every run
```

---

## How it works underneath (60 seconds)

1. **Bootstrap** (automatic — the first invocation finds no `.ai/`): the engine reads the PRD and the repository, generates `knowledge/` and `.ai/`, creates a dedicated `loop/<prd-slug>` branch, and stops with one question: *approve the Definition of Done*.
2. **The one mandatory human gate:** review the DoD (the testable meaning of "done") and the proposed toolchain capabilities. Answered as conversation now, not a hand-edited file.
3. **The loop runs unattended:** each iteration is a fresh process that recovers, orients from repository state, selects the highest-value task, implements, builds, tests, gets a **fresh-context review** from a clean-context subagent, reconciles everything it learned, and commits **one atomic checkpoint** (code + state together). Status `CONTINUE` → the runtime invokes it again.
4. **It stops only for real reasons:** `ESCALATE` (a decision above its authority — architecture change, intent gap, capability request) or `FAILED` (execution environment broken).
5. **Completion is earned, not claimed:** the iteration that finishes the last task may not declare victory. A *fresh* verifier iteration — which wrote none of the code — re-proves every DoD criterion, strips the execution state from the branch tip (Cleanup Commit, whose message is the completion summary), and only then reports `DONE`.
6. **You merge.** The engine never touches your default branch, never pushes, never merges.

Full operating manual for the manual-install path (parameters, watching the loop, escalations, merge): [docs/consumer-guide.md](./docs/consumer-guide.md).

---

## Core design commitments

- **Stateless iterations, dumb runtime.** Every iteration starts from repository state, so resumability is *tested continuously*, not trusted. The runtime has no judgment — its whole intelligence is a status reaction table plus two mechanical safety bounds (crash watchdog, iteration budget).
- **Status contract.** Every engine invocation ends with exactly one of `CONTINUE / DONE / ESCALATE / FAILED`; producing no status *is* the crash signal.
- **Tiered mutability.** Tier 1: the engine freely reshapes tasks (always logged). Tier 2: architecture/strategy changes hard-stop for approval. Tier 3: intent belongs to the human, forever.
- **Capability-based permissions.** No permanent allowlists — grants carry intent, command, scope, and lifetime (goal-scoped by default, auto-expiring), enforced through the trust chain *Human → Ledger → Runtime Compiler → Settings → Engine*. A guardrail against accidents and drift — documented honestly as not being a boundary against an adversarial engine.
- **Three minds.** The builder implements, a clean-context reviewer challenges (it never sees the builder's reasoning — that reasoning may contain the original mistake), and a fresh verifier confirms completion.
- **Everything auditable.** Plan amendments logged with reasons; human decisions recorded with rationale; every checkpoint a commit; the branch history *is* the execution history.
- **The skill never widens what the engine can do.** It stages input and supervises output; every capability grant — standing or goal-scoped — still flows through the same human-approved ledger the engine has always used.

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

**Skill packaging validated end-to-end** (scratch repo, hello-world-notification PRD, 2026-07-15): install → `/loop-runtime` → live background streaming → two real `ESCALATE`s handled as conversation (DoD/interpretation approval, then a capability grant) → `DONE` → roll-up summary, all without opening a single file by hand. One real bug found and fixed along the way: the skill's own frontmatter needed `disable-model-invocation: true`, otherwise a nested engine invocation running inside the same repo could see and auto-trigger the launcher skill on itself.

Field findings driving the next iteration of the runtime:

- **Compound Bash commands can get denied even when the base command is capability-approved** (e.g. `cd X && node ...`) — the engine self-corrected by retrying with simpler forms both times this was hit; worth tightening the capability-proposal format so this doesn't cost a retry.
- **Plan granularity must scale with PRD size** — bootstrap split a 2-task feature into 5 tasks, multiplying the per-iteration overhead (policy tune, pending).
- Observability must never kill execution — a log-tail file lock once crashed the whole loop; log writes are now shared-mode and fail-silent, and a run lock allows only one runtime per repository (fixed).
- Iteration counters differ between engine (counts from STATE, includes bootstrap) and runtime (counts this session's invocations) — cosmetic, pending alignment.
- Console shows mojibake for UTF-8 punctuation on default Windows code pages — cosmetic, pending `[Console]::OutputEncoding` fix.
- Multi-engine portability (Codex, Gemini) is architecturally confined to one adapter surface in `run.ps1` — see [ADR-006](./docs/adr/ADR-006-engine-adapter-boundary.md).

The repository stays intentionally small: additional structure earns its way in through real usage, not anticipated complexity. Deferred items and their revisit-triggers: [docs/architecture.md §12](./docs/architecture.md).
