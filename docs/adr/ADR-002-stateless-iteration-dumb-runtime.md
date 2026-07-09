# Stateless iterations driven by an intentionally dumb runtime

The outer loop is a thin script (the Runtime) that repeatedly invokes the Execution Engine as a **fresh process**. Each Iteration reconstructs all context from durable repository artifacts, executes to one Stable Checkpoint, persists everything, and reports exactly one Execution Status (CONTINUE / DONE / ESCALATE / FAILED). The Runtime contains no planning, reconciliation, or scheduling logic — its only intelligence is the reaction table for the four statuses plus the crash Watchdog.

## Considered Options

- **One long-lived interactive session that self-loops** — rejected: "never terminate" and "hard stop on Tier 2" become prompt-level wishes; context drift and session death are unhandled.
- **Session-per-task hybrid with a supervising harness** — rejected for V1 as added complexity without the key benefit below.

## Consequences

- The principle "the agent forgets, the repository doesn't" is **tested on every iteration**, not only on rare crashes. Insufficient state surfaces on iteration 2, not in production.
- Tier-2 hard stops are enforced mechanically (the process exits; the Runtime stops) rather than by model obedience.
- Startup cost per iteration (re-reading PRD/DoD/STATE/Knowledge) is accepted; portability and enforceability outrank iteration speed.
- Crash detection must live in the Runtime (the Watchdog) because an engine cannot supervise its own death: an invocation that produces no status **is** a Crash.
