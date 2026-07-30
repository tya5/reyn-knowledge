---
name: worktree-parallel
description: Worktrees separate the filesystem but not git state (stash, refs, the shared checkout) — git discipline for parallel work, from fan-out across concurrent agents and the stash ban to post-commit tree verification and linters after renames
tags: [git-github, git, multi-agent]
sources:
  - feedback_parallel_coders_shared_central_file_hazard
  - feedback_line_numbers_are_not_identifiers_across_a_moving_main
  - feedback_same_role_concurrent_session_branch_divergence
  - feedback_explicit_git_add_not_blanket_in_artifact_heavy_session
  - feedback_verify_pushed_tree_matches_working_tree_before_pr
  - feedback_falsify_restore_never_git_checkout_uncommitted
  - feedback_rename_refactor_i001_full_scope_ruff
---

# Parallel Agents and Git — What Worktrees Separate, and What They Don't

## Background

When multiple coding agents run concurrently in the same repository, the standard move is to give each agent a **worktree** (a parallel checkout of the same repository, created with `git worktree add`). A worktree **separates the working directory** — but **much of git's internal state is shared across all worktrees** (there is a single `.git`). This boundary — separated-looking, yet not separated — is where parallel-operation accidents come from.

> **"I'm in an isolated worktree, so I'm safe" is true for the filesystem, false for git state (stash, refs, the shared checkout).**

## 1. Before fan-out, look for the shared central file

Parallelization is right when each unit touches **mutually disjoint files**. In an observed case where work was fanned out three-wide with every unit editing the same central file (a registry, a ledger, an enum, a dispatch table), all it produced was (1) merge conflicts in that one file and (2) **cross-worktree stash contamination** the moment someone used `git stash`.

- Before fan-out: **check whether the units edit the same central file**. If they do, choose one of: (a) serialize (A merges → B rebases → ...), (b) **one owner for the central file** (everyone else touches only the leaves), (c) a structure that touches the central file exactly once.
- In every configuration, **merge serially** (a chain of rebases) — so that each edit to the central file lands on top of the previous result.

## 2. Don't use stash — the stack is one per repository

The `git stash` stack is **a single stack shared by all worktrees**. Measured: `git stash` in an isolated worktree (with nothing to stash) → `git stash pop` **unpacked stale stashed content another session had pushed into this tree**.

- **Don't use stash.** Park temporary work in a WIP commit confined to your own branch.
- Even if the stash list is non-empty, **don't assume it's yours**. Never pop or drop someone else's stash.

A related restoration trap: after stripping a single line for verification, **don't restore with `git checkout <file>` / `git restore <file>`** — these roll the working tree **all the way back** to HEAD, silently erasing not just the stripped line but **the file's entire uncommitted diff**. The most dangerous scenario is verifying one point on top of a large uncommitted diff under review. Before the strip, `cp <file> /tmp/<file>.bak`; restore by copying back, or by reverse-editing just that one line.

## 3. The first git command can land in the shared checkout

The **first** `git checkout -b` of an agent that had been assigned a worktree ran not in the worktree but in **the shared primary checkout** (the main working tree other sessions read) — observed for two consecutive agents. Even when the instructions say "work in the worktree," **the first command can fire before the agent has oriented itself**.

- Put it in the brief: "**before your first git command, run `git rev-parse --show-toplevel` and confirm it is the worktree path**."
- In the two observed cases the implementing agents themselves noticed and recovered before committing; on top of that, the dispatcher's **`git status` check of the shared checkout after each agent finished** independently confirmed both were clean — trust-but-verify.

## 4. Concurrent sessions under the same role name — a push reject is primary evidence

Two sessions with the same role name can run at once, and a coordination message can **reach only one of them**. An empty inbox does not mean you have received every current instruction. In the observed case, a change of direction (a HOLD plus a policy change) reached only the other session; this one implemented on stale assumptions and pushed — the **non-fast-forward reject** (the other side had pushed first) was the first sign of the conflict.

- **A push reject is primary evidence that another session is touching this branch.** Immediately `git fetch` and read the other side's commits with `git log HEAD..origin/<branch>`. **Never force-push.**
- If the other side's commits cite newer authority (a HOLD, a policy reversal), **discard your unpushed commits and converge** (`git reset --hard origin/<branch>`). Unpushed commits have affected no one and remain in the reflog, so this discard is safe.
- After converging, **report transparently what stale assumptions you acted on and what you discarded**, and retract earlier announcements such as "I will revise this next."

## 5. Commit hygiene — the substance is post-commit tree verification, not the add style

The commit stage holds **two traps pointing in opposite directions**; guarding against only one lands you in the other:

- **Trap A (sweep-in)**: long sessions accumulate untracked working files (verification backups, probe output, experiment scripts). `git add -A` then sweeps them all in — measured: what was meant to be 4 files dragged in **46 files, +8019 lines**, and was blocked from merging.
- **Trap B (drop-out)**: explicit-path `git add <p1> <p2> <p3>` **aborts atomically and stages nothing if even one path does not exist**. The subsequent commit stacks only what was staged beforehand — new files silently drop out, **local tests stay green (they run against the working tree)**, and a PR ships without the code.

∴ The unifying rule is not an "add style" but **verification after the fact**:

```bash
git status --porcelain          # empty = working tree matches HEAD (guarantee that tests measured what will be pushed)
git ls-tree -r HEAD --name-only | grep <new-file>       # the new artifact made it into the tree
git show HEAD:<file> | grep <new-symbol>                # the wiring made it into the commit
git diff --stat origin/main...HEAD                      # matches the expected file set
gh pr view <N> --json files                             # the authority after push: the list of changed file names (changedFiles is only a count)
```

In artifact-heavy sessions, use explicit adds (plus a `git diff --cached --name-only` cross-check), and delete verification backups (`.bak` and the like) at the end of the session.

## 6. Linters after a rename — the test files bite

Renaming a package or module changes **the alphabetical position** of import statements, so import-ordering linters (isort / ruff I001) fire in **every rewritten importer** — often hundreds of test files. If the local linter run was scoped to `src/` or predates the rebase, you falsely believe you're "clean" and CI goes red (two observed cases on the same day).

- After a rename-class refactor, run `ruff check src tests --fix` (the same full scope as CI) **on the final rebased HEAD** before pushing.
- Review side: on a red CI for a rename PR, suspect I001 in tests/ first.

## Checklist

- [ ] Before fan-out: is there a shared central file all units touch? Are merges serialized?
- [ ] Not using stash? Not using `git checkout <file>` to restore after a strip?
- [ ] (In the brief) worktree check before the first git command? (On the receiving side) checked the shared checkout's status?
- [ ] Not answering a push reject with force? Read the other side's commits and converged?
- [ ] After committing: verified the tree via empty porcelain, ls-tree, show, diff --stat, and the PR's file list?
- [ ] After a rename: ran the full-scope linter on the rebased HEAD?

## Sources (measured during reyn development)

Central-file fan-out and stash contamination: the #2681 arc (three-wide conflicts plus refs/stash clobber) and the empty-stash pop that unpacked someone else's WIP (2026-07-28; the reference number inside the source pin is an internal work-item id, not a GitHub issue). First command in the shared checkout: the 0061 arc (two consecutive agents, self-caught before committing). Same-role concurrency: #2838 (the HOLD went to the other session; detected via the reject and converged). Sweep-in: #1502 (46 files +8019 lines, reconstructed via `git reset --soft`). Drop-out: a partial commit from a pathspec abort (caught by the reviewer's pre-read). I001: #1685/#1687 (two on the same day).

## Related

- [Your local tree goes stale silently](stale-local.md) — stale bases, shared trees, line-number drift over time
- [Integrity of the strip instrument](../verification/strip-instrument-integrity.md) — the shared venv, the same kind of "not actually separated" surface
- [Completeness sweeps in practice](../verification/completeness-sweeps.md) — the three reference classes of a rename
