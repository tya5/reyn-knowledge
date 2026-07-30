---
name: removal-verification
description: The removal premise "X is dead" is a claim to falsify — full enumeration of producers and readers, import-green ≠ runtime-green, asserting the absence of the removal targets, confirming symbols exist during doc sync. A removal demands "proof of absence" four times
tags: [verification, removal, refactoring]
sources:
  - feedback_falsify_removal_dead_premise_all_producers
  - feedback_refalsify_own_evidence_readers_and_live_dead_names_before_removal
  - feedback_import_green_not_runtime_green_decouple_consumers
  - feedback_verify_delete_target_absent_not_just_keep_present
  - feedback_removal_docsync_kept_concept_ne_kept_symbol
---

# Verifying Removals — "It Is Dead" Is a Claim to Falsify

## Why removals are especially hard

Defects in additions usually surface as "it doesn't work." Defects in removals surface as "**silently breaking something that was still working**" or "**something supposedly deleted silently remaining**" — and both are compatible with green tests. Verifying a removal demands "proof of absence" at least four times: proof it is dead (before), proof it has no consumers (before), proof it is gone (after), proof no references remain (after).

## 1. Falsify the premise "X is dead / always empty" — enumerate every producer

When verifying a PR that says "X is no longer used, so remove it," treat that premise **as a claim to falsify, not as a given**.

Real case: a removal PR premised on "this tracking field has been always empty since an earlier change" was judged SAFE by a verifier who confirmed that "the obvious producer (writes via chat) is indeed gone." But the field was still being written, live, by a non-obvious producer — **the crash-recovery auto-resume path** — and the removal would have caused the real harm of "recovered tasks not appearing in the list."

- **Enumerate every producer of X with grep** (every path that writes or appends). Crash recovery, auto-resume, and background restoration are the classic non-obvious producers.
- **Scope the verdict to the axis you verified.** "The chat axis is clean," not "this PR is safe." An oversized PASS shines a green light on the unverified axes.

## 2. Enumerate the readers too — and re-falsify your own audit

Enumerating producers gets you only halfway. Real case: for the removal of a log event type, the initial audit concluded "deletable" on "**zero writers**, and the restore logic falls through," and review had **already approved**. When the auditor swept once more anyway, **the readers turned out to be alive (the replay mechanism, the rewind classification table, session-context retention)** — the auditor found it before implementation began and pulled the change back, with zero wrong deletions.

> **A review "approval" is not absolution. A hole found after approval but before implementation costs nothing if you simply pull the change back.**

One more important distinction discovered here:

> **"Live code that references a dead name" is not dead prose.**

A deleted tool's name surviving in a comment is prose (freely rewritable). But a deduplication set `frozenset({"invoke_skill"})`, an error category keyed by that name, and a comparison `name == "invoke_skill"` are **live code whose behavior references the deleted name**. Classify by **what the code does**, not by what the token looks like (comment-ish).

## 3. Import-green ≠ runtime-green — consumers break at runtime

Delete a field from a shared-state object (a dataclass, say), and `python -c import` and the linter **stay green**. What breaks is the `AttributeError` at the moment a remaining consumer **reads** that field — at runtime. In the real case, after the removal PR, the system was in a state where **every chat turn died with an exception**, while being reported as "import + lint green, test subset passing" (of 482 failures, 336 were the cascade from this single dangling reader).

- The removal spec must list **not just the definition site of the removal target but every consumer** (grep the entire surviving side for `.field` / `symbol(`).
- The verification gate is not "import + lint + subset" but **execution of the actual consumer paths** (the full suite, or the tests for those paths).

A deeper variant: code whose import of the deleted symbol sits **inside a function** (not at module top level) is never touched by module loading, and if no test calls that handler, it is **broken at runtime while import-green and full-suite-green**. There is a real case of a registered tool surviving in exactly this state.

- When verifying a bulk removal, **grep the registration/dispatch layer** (tool registries, `register(X)`, dispatch tables). A consumer that stays registered while dead is invisible to pytest.
- Removing such a tool cascades into **every parallel registry** (the prompt's category tables, master tables, benchmark inventories) — chase them all.

## 4. Explicitly assert the "absence" of the removal targets

After a bulk removal, "the kept files are all present," "zero dangling imports," and "full suite green" still **do not prove the removal targets are gone**. **Green does not demand a file's absence** — an orphaned corpse (a file whose only references are stale comments and which has zero functional importers) survives untouched by import, pytest, and lint. Real case: in a 13-module removal, one 35KB core module survived and sailed through every check above. What caught it was an explicit `git ls-tree` against the removal set.

- **For each item in the removal spec, assert its absence**: a loop of existence checks for files, `grep -c "^class X"` = 0 for classes, gone-from-the-registry for tools.
- **Verify against the pushed reality (the remote tip).** Further: stale remote-tracking refs can return a false "everything is gone" — fetch the target branch explicitly before looking.

## 5. Doc sync after removal — survival of the concept ≠ survival of the symbol

The doc sync that closes out a removal arc has a trap all its own: **because the concept survived, you presume the symbol bearing the same name survived too**. Real case: the field and event `control_ir_results` survived, so the `ControlIRExecutor` class and its file were assumed to exist as well, and the doc references were kept — measured (`git cat-file -e` / `git grep -c 'class X'`), **both were deleted, zero definitions**.

- In doc sync, confirm that **every file path and every symbol the edited doc references actually exists on the new main** (not just the tokens being renamed).
- **The number of flagged items is a lower bound, not the full set.** The reviewer flagged 2 → an exhaustive sweep found 4. Fixing only what was flagged is the "fix 2, miss the 3rd" shape.
- Dead references are **delete-only**. Re-pointing them at the new architecture is separate work ("rebuilding") and must not be mixed into the sync PR.

## Checklist

- [ ] The "it is dead" premise: did you enumerate every producer (including recovery, auto-resume, and background paths)?
- [ ] Did you enumerate every reader (replay, rewind, analytics)? Did you distinguish "live code with dead names" from prose?
- [ ] Did you execute the consumer paths (not settle for import-green, lint-green, subset-green)? Did you grep the registration/dispatch layer?
- [ ] Did you assert the absence of each removal target against an explicitly fetched tip?
- [ ] Did you confirm the doc's referenced symbols and paths exist on the new main? Did you mistake the flagged count for the full set?
- [ ] Did you scope the verdict to the axis you verified?

## Sources (measured during reyn development)

Producer enumeration: #2104/#2151 (the crash-recovery path missed; a reviewer caught it). Readers and dead names: the skill-vocabulary purge arc (self-re-falsification after approval, before implementation; zero wrong deletions). The import-green trap: #2434 stage1 (the 336/482 cascade) and stage3b (in-function import + a registered tool). Absence asserts: #2434 stage3b (the 35KB survivor, caught by `git ls-tree`; the stale-ref false "all gone" is the same incident). Doc sync: Control IR PR-6 (concept survival conflated with symbol survival; 2 flagged → 4 actual).

## Related

- [Liveness is decided by the producer](liveness-is-producer.md) — the liveness judgment itself; this document is the "executing and verifying the removal" that follows it
- [Audits must match content](audit-content-match.md) — the general form of doing present/absent checks by content
- [Prove fixes on the live path](fix-verification-live-path.md) — the same prescription of executing the consumer paths
- [The discipline of enumeration](enumeration-discipline.md) — techniques for exhaustive enumeration
