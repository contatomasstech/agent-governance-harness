<div align="center">

<img src="./docs/assets/mass-logo-dark.svg#gh-dark-mode-only" alt="MASS." width="140">
<img src="./docs/assets/mass-logo-light.svg#gh-light-mode-only" alt="MASS." width="140">

### Agent Governance Harness

_Deterministic Architecture & Runtime Governance for Autonomous Systems_

<p align="center">
  <a href="./README.md"><b>English</b></a> | <a href="./README.pt-BR.md"><b>Português</b></a>
</p>

---
</div>

**Hermetic worktrees, deterministic concurrency, and adversarial gates for autonomous code generation.**

A reference architecture for letting AI coding agents claim tasks, write code, run tests, and open pull requests **without unattended write access to your main branch** — and without falling back to the throughput-killing pattern of a human approving every single file edit by hand.

---

## The problem

Autonomous coding agents in a production engineering org tend to fail in one of two directions:

1. **Manual candidate staging.** The agent proposes an edit, a human approves it file by file before anything is written to disk. Safe, but it does not scale: every task becomes a queue of one-by-one approvals, and the "autonomy" is mostly theater.
2. **Unrestricted write access.** The agent gets a real branch, real commit rights, sometimes real push rights to shared infrastructure. This scales, until the agent hallucinates a plausible-looking but wrong implementation, and it lands in a shared branch — or worse, in `main` — before anyone reads it.

Neither is acceptable once agents are doing real, unattended work against a real issue tracker. The harness in this repository is the middle path we converged on after living with both failure modes.

## The core idea

Give the agent **real authority to write, test, and commit — but only inside a disposable, isolated worktree, and only up to the edge of a pull request that a second, independent process has to approve before a human ever needs to look at it.**

Nothing the agent does is visible outside that worktree until:
- its own change passes the project's real test suite, and
- an independent adversarial review — a second model or process with no stake in "looking productive" — approves the diff.

Only then does a **draft** pull request get promoted for human review. Merge authority and the final "done" transition stay exclusively human, always.

## Architecture

```
                     [ Inbound Task / Issue ]
                              │
                              │  strict WIP ceiling enforced
                              │  (global + per-category capacity)
                              ▼
                     ┌─────────────────┐
                     │   Orchestrator  │  claims the task, tracks liveness,
                     │                 │  never self-approves to "Done"
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │     Executor    │  ephemeral git worktree, created
                     │    (coding)     │  fresh from origin — never touches
                     │                 │  the shared local checkout
                     │                 │
                     │                 │  policy enforcement: denied_paths
                     │                 │  is fail-closed (deny wins, always)
                     │                 │
                     │                 │  runs the project's real test suite
                     │                 │  inside the worktree; non-zero exit
                     │                 │  aborts everything downstream
                     └────────┬────────┘
                              │  tests green
                              ▼
                     ┌─────────────────┐
                     │   Draft PR /    │  commit + push to a dedicated
                     │  Patch Emitted  │  branch (agent/<issue-id>)
                     │                 │  zero direct writes to main
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │   Adversarial   │  independent static/security review:
                     │      Gate       │  auth boundaries, tenant isolation,
                     │                 │  secret exposure, data-handling risk
                     │                 │
                     │  REJECT ─────── │  close the draft PR, no merge, no
                     │                 │  retry — the branch stays for audit
                     │                 │
                     │  APPROVE ────── │  promote the PR out of draft
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │ Human Approval  │  exclusive merge authority,
                     │                 │  exclusive "Done" transition
                     └─────────────────┘
```

## Guiding principles

- **Hermetic filesystem.** All agent work happens in a worktree created fresh from `origin`, never from a shared local checkout that other processes might be using. The worktree is disposable; a corrupted or abandoned one costs nothing.
- **Fail-closed policy enforcement.** `denied_paths` are checked against literal, resolved paths — no glob syntax to get subtly wrong, no permissive default. Anything not explicitly inside `allowed_paths` is unreachable by construction, not by convention.
- **Deterministic concurrency.** A global WIP ceiling, plus per-category sub-limits (e.g. "at most one concurrent CLI-backed coding session," "at most N lightweight operations") — so throughput scales without any single scarce resource collapsing under concurrent load.
- **Adversarial review before human review.** The gate that decides whether a change is even worth a human's time is a second, independent process — not the same model grading its own homework, and not a unit test suite, which proves behavior but not intent.
- **Never-done rule.** No agent, no adapter, no orchestrator in this design transitions a task to its terminal "done" state. That is a human action, always, with no override.
- **Reject leaves evidence.** A rejected change is never silently discarded. The draft PR is closed, not deleted — the diff, the test run, and the adversarial verdict all stay attached to it for audit.

## Repository structure

```
docs/
  adrs/                    architecture decision records documenting how
                            this design was arrived at (and what it replaced)
  architecture/
    lifecycle-flow.md       a more detailed walk-through of the diagram above
templates/
  policies.canonical.yaml   an annotated, fill-in-the-blanks runtime policy
.github/
  workflows/
    pr-audit-template.yml   an illustrative CI hook for the adversarial gate
```

## Status

This repository documents the pattern, not a packaged, install-and-run tool. The reference implementation this pattern was extracted from is integrated with an internal orchestrator, issue tracker, and a specific local review model — none of which are portable in their current form. What's published here is the architecture and the governance reasoning behind it, sanitized of any internal system names, credentials, or business logic, so it can be implemented against whatever orchestrator, tracker, and review tooling you already run.

## License

MIT — see [`LICENSE`](./LICENSE).

## About

Maintained by [MASS](https://github.com/contatomasstech). Issues and discussion around the architecture are welcome; this is published as a reference design, not as a supported product.
