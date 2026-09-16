# ADR 002 — Autonomous Worktree Execution with Adversarial PR Gate

Status: accepted (canonical pattern of this harness)

Supersedes: [ADR 001](./001-limits-of-manual-staging.md)

## Context

ADR 001 established the problem: gating every individual file edit behind a named human approval
preserves "an agent never writes unsupervised code" at too fine a grain to support real
concurrency. We needed a design that keeps that guarantee meaningful while moving the human
checkpoint to a unit of work that's actually worth a person's attention.

## Decision

Within a disposable, isolated execution environment, the executing agent is granted real authority
to write files, run the project's test suite, and commit — with no per-file human approval — under
the following non-negotiable constraints:

1. **Everything happens inside an ephemeral git worktree**, created fresh from the remote default
   branch. The agent never operates against a shared local checkout that a human or another process
   might be using concurrently; a worktree that hangs, corrupts, or gets abandoned costs nothing
   and affects nothing else.
2. **Every file the agent touches is still checked against a runtime policy** (`allowed_paths` /
   `denied_paths`, see [`templates/policies.canonical.yaml`](../../templates/policies.canonical.yaml)).
   Nothing in this design removes or weakens that check — it runs identically to how it ran under
   the manual-staging model.
3. **The project's real test suite has to pass** (non-zero exit at any step aborts the entire flow
   — nothing downstream of a failing test ever executes).
4. **Before anything becomes visible outside the worktree**, the change passes through an
   independent adversarial review — a second process with no stake in the change looking
   successful, checking specifically for the classes of defect a test suite doesn't catch:
   authorization boundaries, tenant/data isolation, secret exposure, unsafe data handling.
5. **Rejection is fail-closed and non-retryable within the same run.** A rejected change never gets
   promoted by omission, timeout, or default. The pull request it produced is closed, not deleted —
   the diff, the test results, and the adversarial verdict remain attached to it as an audit trail.
6. **Merge to the default branch, and the task's terminal "done" transition, are exclusively
   human actions.** No component in this design — orchestrator, executor, or adversarial gate — is
   able to perform either.

### Why the review has to target a real pull request, not a bare diff

An adversarial reviewer that only ever sees an isolated diff, out of the context of CI status,
existing review threads, and the repository's actual state, is reviewing something meaningfully
different from what a human reviewer would see. Requiring a **real, if draft, pull request** as the
unit the adversarial gate reviews means: the review artifact is the same one a human will
eventually look at, it's inherently auditable (it exists in the platform's own history, not in a
throwaway log), and rejection has a clean, unambiguous action (`close`, no merge) instead of a
bespoke "undo" path that has to be trusted to actually undo everything.

The tradeoff this implies: a rejected change briefly exists as a real (draft) pull request before
being closed. That's an accepted, deliberate exposure — a closed, unmerged draft PR is not a
security boundary crossed, it's a paper trail.

### Where the new authority is contained

The write authority this ADR grants is **not** a change to the general-purpose editing tool or
interactive agent runtime used elsewhere in the organization. It lives entirely inside the
executor component of this harness, scoped to this one execution path. Any other, interactive use
of the underlying agent tooling continues to require the named human approval described in ADR
001, unchanged. Revoking the authority this ADR grants is a matter of disabling one component, not
reverting a platform-wide policy.

## Consequences

- Throughput scales with orchestrator concurrency (subject to WIP limits), not with how fast a
  human can review one file at a time.
- The adversarial gate is a second model, with its own failure modes (including false negatives —
  approving something it shouldn't). It reduces, but does not replace, the value of the final human
  review before merge. That's why merge stays human, not a formality.
- This pattern fits well-scoped, single- or few-file changes. It is a poor fit for changes that
  require significant cross-file architectural coordination — those still warrant normal,
  human-led development outside this harness, independent of what the adversarial gate would say
  about a partial result.
