# Lifecycle Flow

A step-by-step walk-through of the diagram in the [README](../../README.md), with the design
rationale for each stage. See [ADR 002](../adrs/002-ephemeral-worktree-adversarial-gate.md) for the
decision this flow implements.

```
[ Inbound Task / Issue ]
          │  strict WIP ceiling enforced
          ▼
  ┌───────────────┐
  │  Orchestrator │
  └───────┬───────┘
          ▼
  ┌───────────────┐
  │   Executor    │
  └───────┬───────┘
          ▼
  ┌───────────────┐
  │  Draft PR /   │
  │ Patch Emitted │
  └───────┬───────┘
          ▼
  ┌───────────────┐
  │  Adversarial  │
  │     Gate      │
  └───────┬───────┘
          ▼
  ┌───────────────┐
  │ Human Approval│
  └───────────────┘
```

## 1. Inbound task / issue

A task enters the pipeline from whatever system of record you already use (issue tracker, ticket
queue, backlog). Before it is claimed, the orchestrator checks two independent capacity limits:

- a **global WIP ceiling** across all in-flight work, and
- a **per-category sub-limit** (a real-world resource — a single licensed CLI session, a single
  GPU, a rate-limited API — can bottleneck concurrency well before the global ceiling does; a
  single combined limit hides that and produces confusing, intermittent contention failures).

A task that exceeds either limit waits; it does not queue-jump into a resource that's already
saturated, and it does not block tasks in *other* categories from proceeding.

## 2. Orchestrator

Claims the task, tracks its liveness (a task that stops reporting progress within a bounded window
is treated as stalled, not silently retried forever), and is the only component with visibility
into the task's status in the system of record. It moves the task to an "in progress" state on
claim — and, critically, is never the component that moves a task to its terminal "done" state.

## 3. Executor

This is where ADR 002's authority actually applies. Concretely, in order:

1. An ephemeral git worktree is created from the remote default branch — never from a shared local
   checkout another process might be using.
2. The requested edit(s) are generated and written directly to files inside that worktree, scoped
   strictly to an explicit allow-list of paths the task is permitted to touch (never inferred,
   never expanded by the executor itself).
3. Every touched path is checked against the runtime policy's `denied_paths` before being written
   — see [`templates/policies.canonical.yaml`](../../templates/policies.canonical.yaml). This check
   is unconditional; there is no execution path that skips it.
4. The project's real test suite runs inside the worktree. Any non-zero exit code aborts the
   entire flow before anything is committed — nothing is ever pushed on the strength of "the edit
   looked plausible."

## 4. Draft PR / patch emitted

Only after tests pass: commit, push to a dedicated branch (never `main`/`master`), open a **draft**
pull request. This is the first point at which the change becomes visible outside the ephemeral
worktree — and it's deliberately still not visible as something ready for a human to spend time on.

## 5. Adversarial gate

An independent review process — not the same model that wrote the change, and not the test suite
that already passed — evaluates the actual pull request for the classes of problem a green test
suite does not catch: authorization boundaries, tenant/data isolation, secret exposure, unsafe
data-handling paths. Two outcomes, no third option:

- **Reject** → the draft PR is closed (never merged, never deleted), the task is reported back to
  the system of record as failed-at-gate, with the review's findings attached for audit.
- **Approve** → the PR is promoted out of draft, ready for human review.

Ambiguous or malformed review output is treated as a rejection. There is no default-approve path.

## 6. Human approval

The only step in this flow with authority to merge into the default branch, and the only step with
authority to move the originating task to its terminal "done" state. Nothing upstream of this step
can do either, under any condition.
