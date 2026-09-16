# ADR 001 — The Limits of Manual Candidate Staging

Status: superseded by [ADR 002](./002-ephemeral-worktree-adversarial-gate.md)

## Context

The first working version of agent-assisted code editing in this harness followed what we now
call the **manual candidate staging** model:

1. The agent proposes an edit to a single file, given a natural-language instruction. The proposal
   is written to a candidate record on disk — the target file itself is never touched.
2. A human reads the proposed diff and, if it's correct, explicitly approves it by name. Only that
   approval step writes the change to the real file.

This model exists for a real, observed reason: locally-hosted models are not reliably faithful to
instructions on nontrivial edits, and can silently invent content that looks plausible but is
wrong — a well-documented failure mode of small/local coding models in particular. Requiring a
named human to read every diff before it becomes real is a strong, simple mitigation for exactly
that risk, and it was the correct first decision: automatic approval with no human in the loop was
considered and explicitly rejected at the time, on principle — "the agent proposes, a human
decides" was treated as non-negotiable.

## The problem

That principle is still correct. What stopped scaling was the granularity at which it was applied.

Once an orchestrator exists that can claim many tasks from an issue tracker and run them with real
concurrency, gating every individual file edit behind a named human approval turns the queue into
a sequence of one-by-one confirmations. The orchestration layer can claim ten tasks in parallel;
the approval bottleneck still processes them one file at a time. Autonomy exists on paper — the
agent "did the work" — but the actual bottleneck on throughput is identical to no automation at
all.

We considered, and rejected, two shortcuts around this:

- **Loosen the approval requirement** (e.g. accept a non-human identifier as "approval"). Rejected:
  this doesn't remove the risk the approval step exists for, it just stops recording that the risk
  was never actually checked.
- **Batch-approve multiple candidates at once without reading each one.** Rejected: this is the
  same failure mode as the point above, with extra steps.

Neither shortcut addresses the actual problem, which is that **the unit of human review was set at
the wrong granularity** — per file, rather than per unit of shippable change.

## Decision

Move the human checkpoint from *"approve this file edit"* to *"approve this pull request for
merge,"* and replace the per-file review it used to provide with an automated, independent
adversarial review that runs before a human ever sees the change. See [ADR
002](./002-ephemeral-worktree-adversarial-gate.md) for the resulting design.

This is not a loosening of "the agent proposes, a human decides" — it's a relocation of *where*
that decision happens, from the least useful place (one file, out of context, possibly before the
rest of the change even exists) to the most useful one (a complete, tested, independently-reviewed
diff, in the context of the whole change).
