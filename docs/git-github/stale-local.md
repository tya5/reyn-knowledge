---
name: stale-local
description: Your local tree goes stale silently — a server-side merge does not move your local checkout. A stale grep produces false findings, a branch cut from a stale base becomes a quiet mass revert, and line numbers do not carry across a moving main
tags: [git-github, git, measurement]
sources:
  - feedback_gh_merge_leaves_local_tree_stale_sync_before_local_grep
  - feedback_sync_local_main_before_sequential_worktree_dispatch
  - feedback_local_checkout_branch_instability
  - feedback_multiref_fetchhead_resolves_to_first_ref_not_branch
  - feedback_line_numbers_are_not_identifiers_across_a_moving_main
---

# Your Local Tree Goes Stale Silently — "The Tree You Looked At" and "The Tree That Counts" Differ

## Background — why this bites agent operations in particular

`gh pr merge` (a server-side merge via the GitHub CLI) **advances origin/main — the remote source of truth — but does nothing to your local checkout**. In human development, "merge, then pull" gets inserted naturally; but when an agent server-side-merges several PRs in a single session, **local main silently falls 5–15 commits behind**. Greps, diagnostics, line numbers, and branch cuts against that old tree mass-produce the accidents below.

This is the git edition of [Measuring the wrong target](../verification/measurement-target.md) and [Integrity of the strip instrument](../verification/strip-instrument-integrity.md): **"the tree you looked at" and "the tree that counts" are different objects**, and the gap **widens silently over time**.

## Accident 1 — a stale grep produces false findings and false reassurance

Observed (false finding): a reviewer fired a completeness sweep ("are there other unfixed siblings of the same class?") as a grep at the local tree and filed the hits as "unfixed siblings" in a change request. **The symbol had been deleted five commits earlier.** The implementer refuted it with `git show origin/main:<path> | grep -c ...` → 0. Filing false findings against a healthy PR — the same damage as the "false accusation against a healthy gate" in [Integrity of the strip instrument](../verification/strip-instrument-integrity.md).

Observed (false alarm): on another day, after merging a signature-change PR, a local grep reported "10 remaining sites using the deleted argument" — yet the PR's full suite and CI were green, a contradiction. The cause: local HEAD was **15 commits behind**, and all 10 sites had already been migrated.

> **Run `git fetch origin main` first, then aim greps that back completeness or "present/absent here" claims at `git show origin/main:<path>` or `git grep <pattern> origin/main`, not at the local tree.** For ordinary exploration, staleness does little harm — but claims like "that's all of them" or "it exists here too" **become false merely because the tree is old**.

Adjacent: a behavior-check `python -c "import mypkg; ..."` may also **be resolving a different clone** (don't trust it until you have confirmed the resolution target with `mypkg.__file__` — failure mode 3 of [Integrity of the strip instrument](../verification/strip-instrument-integrity.md)).

## Accident 2 — a branch cut from a stale base becomes a "quiet mass revert"

In a phased arc (phase N+1 depends on phase N's merged code), cutting the next phase's working branch **from local main** means the base lacks the previous phase. The result is bad in two stages:

1. **If it merely conflicts, you're still lucky** (you can notice).
2. The real damage is that **the implementer's design judgment rots**. In the observed case, because the base lacked the previous phase, the implementer scoped the work on the premise that "that feature doesn't exist yet" — resolving the conflicts was not enough; **the design decisions invalidated by the stale base had to be revisited** as well.

It generalizes further (beyond worktree dispatch): **it happens even to docs-only PRs**. In the observed case, the diff of a "+111-line documentation PR" cut from a detached HEAD 9 commits behind actually contained **−789 lines = a nine-commit rollback**. A "+N docs" headline is no proof that a change is additive.

- Sync before cutting: `git fetch origin main && git merge --ff-only origin/main`. Also include an opening `git fetch origin main && git rebase origin/main` in the implementer's brief.
- **Pre-merge tripwire**: a supposedly small PR shows unexpected deletions or a large negative line count, or `git merge-base <branch> origin/main` does not match current origin/main → stale base. Don't merge; rebase and re-take the diff.

## Accident 3 — your checkout ends up on a different branch without you noticing

Observed: a post-merge `git pull origin main` failed with "divergent branches." The cause: **the checkout had ended up on someone else's feature branch** (another agent had run checkout in the shared working directory).

- Before operations that depend on local state (`git pull`, `git reset`, ...), **confirm that `git branch --show-current` says main**.
- Better still, anchor verification on **branch-independent references**: `git fetch origin main` → read via the `origin/main` ref. Correct regardless of local checkout state.
- The root fix is **placement**: agents don't work in a shared tree; each gets its own dedicated clone / worktree.

## Accident 4 — FETCH_HEAD points at "the first ref fetched"

After **fetching multiple refs at once**, as in `git fetch origin main <branch>`, `git show FETCH_HEAD:<file>` reads **the file from the first ref (usually main), not from the branch**.

Observed: whether a rename had landed on a branch was "inspected" this way; the inspection saw main's old content and filed a false report to the implementer that "the rename isn't on the branch" (it had in fact been there since the first commit). **The worst form of measurement error** — the stale content is "visible," so you act with confidence, and it comes bundled with unjust attribution to another party.

- Inspect branch content with an **explicit ref**: `git show origin/<branch>:<file>`. Don't use FETCH_HEAD after a multi-ref fetch.
- If the other party disputes your finding, re-take **both sides via explicit refs** before maintaining your position. Once shown wrong, **retract immediately** — the finding, and everywhere you propagated it.

## Accident 5 — line numbers are not identifiers across a moving main

Observed: during a long-running analysis (hundreds of line-numbered classifications), the shared tree was pulled by other work and the target file shifted by **+107 lines**. All the analysis data was anchored to a stale snapshot. What saved it was **content-based re-anchoring immediately before applying** (checking that the content each line number points at matches expectations): 87 mismatches out of 100 surfaced the drift — without that check, the application would have gone ahead and destroyed the file.

- For work where **the moment of measurement and the moment of use are separated** (analyze → bulk apply, keeping line numbers from grep output, "strip line N"-style instructions), **re-anchor by content immediately before applying**.
- Judge the cross-check by **the count of mismatches, not the match rate** (a handful of offsets drowns in a match rate).
- When emitting **identifiers others will use** (line numbers, symbol positions, "there are N of them"), measure against `origin/<ref>`, not local HEAD — "they happened to be identical" is not discipline.
- A diff whose comparison baseline **contradicts your own working tree** should make you suspect a sync problem before a regression.

### Note — worktrees share most git state

A worktree **separates the working directory but does not separate much of git's state**. Measured: the `git stash` stack is **a single stack across all worktrees**, and a `git stash pop` in an isolated worktree **unpacked another session's old stashed content into that tree** (recovered immediately, zero damage).

- **Don't use stash** (park temporary work in a WIP commit confined to your own branch). Even if the stash list is non-empty, don't assume it's yours.
- "I'm in an isolated worktree, so I'm safe" is **true for the filesystem, false for git state (stash / refs / config)**.

## Checklist

- [ ] Fired completeness/existence-claim greps at `origin/main` refs (not at the local tree)?
- [ ] Synced local main to origin before cutting a branch or dispatching a dependent phase?
- [ ] Any negative line counts or unexpected deletions in a supposedly small PR? Is the merge-base current origin/main?
- [ ] Confirmed the current branch before operations that depend on local state?
- [ ] Not using FETCH_HEAD to inspect branch content (explicit refs instead)?
- [ ] When handing line numbers or counts to others, measured against an origin ref? Re-anchored by content immediately before applying?

## Sources (measured during reyn development)

Accident 1: #3385 (5 commits behind, false finding on a deleted symbol; the implementer also correctly refused to record an exemption, on the grounds that "an exemption entry for a nonexistent symbol is itself drift") and #3149 (15 commits behind, 10 false leftovers). Accident 2: #2817 (P4 cut from a base missing P3, contaminating even the design judgment) and #2919 (docs PR from a detached HEAD containing a −789-line revert, self-caught). Accident 3: #1069 (checkout on someone else's feature branch, caused by the shared directory — resolved by moving to dedicated clones). Accident 4: #2818 (FETCH_HEAD resolved to main; false finding and unjust attribution). Accident 5: #3082 (+107-line drift; re-anchoring found 87/100 mismatches and averted destruction; recovery 381/389) and the cross-worktree stash contamination case (2026-07-28; the reference number inside the source pin is an internal work-item id, not a GitHub issue).

## Related

- [Measuring the wrong target](../verification/measurement-target.md) — the relay rule for "which tree, at which commit, was measured"
- [Integrity of the strip instrument](../verification/strip-instrument-integrity.md) — identity of the measured object (the venv edition)
- [Verifying removals](../verification/removal-verification.md) — absence asserts against an explicitly fetched tip
