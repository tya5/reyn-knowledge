---
name: merge-gate-check
description: A pre/post-merge state-check procedure. Confirm merged by state → kill auto-merge/pollers → put remainders where the gate reads → check the composition after serial merges → sync local. Sources docs/git-github/merge-gates and others
---

# merge-gate-check — Confirming "merged" by state and closing a merge's side effects

**When to use**: merging a PR / receiving a "merged it" report / merging several PRs in a row.

Premise: **no report, display, or hook output saying "merged" is evidence of a merge**. The only evidence is server-side state.

## Step 1 — Confirm merged by state

```bash
gh pr view <PR> --json state,mergedAt,mergeCommit
```

- Treat it as merged only on seeing `state: MERGED` with a non-null `mergedAt`. Hooks, CI displays, and peer reports have all produced measured false "merged"s.
- An "implementation complete" report and "merged" are different things — even on receiving a completion claim, fire this one shot before you ack.

## Step 2 — Stop auto-merge and pollers first (before reversing a decision)

```bash
gh pr view <PR> --json autoMergeRequest        # is auto-merge armed?
gh pr merge --disable-auto <PR>                # disarm before reversing
```

- When withdrawing or reversing a merge decision, **kill the already-dispatched automation (auto-merge, merge pollers) first**. Measured: a poller reopened and merged a PR that was supposed to be closed.
- Automation keeps executing **the decision as of dispatch time** — a human's change of mind is not among its inputs.

## Step 3 — Put remainders where the gate reads

- Work that remains after the merge (follow-ups) goes to **the surface the merge gate actually reads** (an open issue). PR-body checkboxes, conversation logs, and your own memory are not read by the gate.
- If the PR contains `Closes #N`, run [closing-keyword-check](../closing-keyword-check/skill.md) first.

## Step 4 — After serial merges, check the composition

```bash
git fetch origin main
git log --oneline -<N> origin/main             # the PRs that landed today
# run one full check on the composed main (each PR green ≠ composition green)
```

- Parallel PRs **break in composition while each stays green** (duplicate function additions, colliding signature changes). Right after landing, run the suite once against the **composed result**, not per PR.
- Especially: right after merging a series of PRs that touched the same central file.

## Step 5 — Sync your local checkout after the merge

```bash
git fetch origin && git status                 # make the local lag visible
git pull --ff-only                             # catch up before working
```

- A server-side merge **does not move your local checkout**. Grepping a stale local produces false findings, and pushing onto a stale base becomes a silent mass revert. After merging, always sync before the next piece of work.

## Background

The source incidents for each step (the hook's false "merged," the poller's reopen+merge, remainders lost in checkboxes, signature collisions, stale bases, and more) are in [Confirm Merges by State](../../docs/git-github/merge-gates.md), [Your Local Tree Goes Stale Silently](../../docs/git-github/stale-local.md), and [Parallel Agents and Git](../../docs/git-github/worktree-parallel.md).
