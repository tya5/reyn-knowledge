---
name: git-github-index
description: The map of Tier 2 — six trap areas specific to agent-driven git/GitHub operation. Automation (closing keywords, pollers, hooks) acts on surface text and state; concurrency (worktrees, parallel PRs, stale locals) breaks on surfaces that look separated but aren't
tags: [git-github, index, overview]
---

# The Map of git/GitHub Hazards — Tier 2

If Tier 1 ([The system of verification knowledge](../verification/index.md)) is the order in which to doubt an agent's completion report, Tier 2 is **the infrastructure around it** — a record of how the shared machinery of git, GitHub, and CI breaks under the usage patterns peculiar to agent operation (rapid-fire server-side merges, parallel work in worktrees, auto-merge, issue-driven task management).

Two structures run through all of it:

1. **Automation acts on surface text and state.** The closing-keyword parser does not read meaning, the merge poller keeps executing the decision made at dispatch time, and hooks react to command strings. Human intent is not an input.
2. **Concurrency breaks on surfaces that look separated but aren't.** Worktrees don't separate the stash, a local checkout doesn't follow server-side merges, and parallel PRs stay individually green while breaking in combination.

## Documents

| Document | In one line |
|---|---|
| [Closing keywords fire on surface text](closing-keywords.md) | Inside backticks doesn't close; a raw keyword in a negation does. Verify with the parser's output. Enumerate sibling PRs before closing |
| [Your local tree goes stale silently](stale-local.md) | Server-side merges don't move your checkout. False findings from stale greps, silent mass reverts from a stale base, line numbers drifting over time |
| [Parallel agents and git](worktree-parallel.md) | What worktrees separate and what they don't. Fan-out on central files, the stash ban, verifying the tree after committing |
| [Confirm merges by state](merge-gates.md) | The hook's false "merged", the poller's reopen+merge, remainders go on surfaces the gate reads, signature collisions between parallel PRs |
| [Issues decay](issue-lifecycle.md) | The body is a snapshot; resolution lives in the thread and merged PRs. Verification records at close, settling remainders, filing calibration |
| [The traps in docs-only PRs](docs-prs.md) | Tests that parse the doc, mermaid rendering, per-renderer slugs, proving links by enumeration, no internal-history refs |

## Executable checklist (skill)

- [closing-keyword-check](../../skills/closing-keyword-check/skill.md) — the full closing-keyword inspection before opening/merging a PR (the most accident-prone hazard in this area)

## Reading order

First time through: [Closing keywords fire on surface text](closing-keywords.md) → [Confirm merges by state](merge-gates.md) → [Your local tree goes stale silently](stale-local.md). Designers of parallel operation should read [Parallel agents and git](worktree-parallel.md) first.
