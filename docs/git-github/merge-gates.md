---
name: merge-gates
description: Report "merged" only after reading the PR's state — a hook firing, the fact that you typed the command, and green CI are none of them evidence. Auto-merge pollers keep executing stale decisions, and concurrent PRs break main while each stays green
tags: [git-github, merge, automation, ci]
sources:
  - feedback_confirm_merge_before_merged_ack
  - feedback_verify_merge_state_not_impl_complete_claim
  - feedback_kill_automerge_poll_before_reversing_merge_decision
  - feedback_merge_order_signature_conflict_sweep_after_landing
  - feedback_dont_over_park_ready_prs_merge_promptly
  - feedback_outstanding_item_must_live_where_the_merge_gate_reads
---

# Confirm Merges by State — Hooks, Pollers, and Concurrent-PR Traps

## Central claim

> **You may report "merged" only after `gh pr view <N> --json state,mergedAt` returns `MERGED` plus a non-null timestamp.** The fact that you ran the merge command, an automation hook's "merged" notification, and green CI are none of them evidence of a merge.

This is the git edition of "observations don't name their referents" from [Reading green](../verification/green-reading.md). Below, the ways this claim was broken, in the order they were observed.

## 1. String-matching hooks lie about "merged"

In a setup where automation (a hook) watching command execution reacts to **the string** `gh pr merge` by announcing "PR #N merged," **the notification fires even when the merge fails**. Measured three times: a PR still in draft (failed with "still a draft"), a shell syntax error aborting the command, and a rejection because the branch was behind — in every case the hook claimed "merged," and relaying that to a person would have been misinformation.

- Don't send a conditional merge (`if [green]; then gh pr merge; fi`) and a "merged ✅" report **in parallel within the same turn**. Keep the order: run the merge → confirm state → report. If they absolutely must coincide, use **incomplete phrasing**: "will merge as soon as it's green (not yet done)."
- Run other people's "everything is merged" reports through the same gate: confirm `state==MERGED` yourself before propagating hearsay (observed: a PR that had stayed open was mixed into an "all complete" report).

## 2. Under strict branch protection, merge failure becomes the normal state

With a setting that requires the base branch to be up to date (strict status checks), **every merge flips all other open PRs to BEHIND**. In that state `gh pr merge` fails — and the hook above fires "merged."

- The default cost of serial merging: after each land, run `gh pr update-branch <N>` on the next PR → wait for CI to re-run → confirm `state,mergedAt`. **"It was green a moment ago" no longer implies mergeability.**

## 3. Auto-merge pollers keep executing the decision made at dispatch time

A background poll that merges once CI turns green is convenient, but it is **fire-and-forget with the dispatch-time decision baked in**. When the decision changes, the poller doesn't know.

Worst observed case: the owner rejected the PR's approach, so it was closed with `gh pr close` — but the poller was still alive. One minute later CI went green, and the poller's `gh pr merge` **reopened the closed PR and merged the rejected change into main** (it had to be reverted). **Closing does not disable a poller.**

- **The moment a merge decision reverses (rejection, hold, changes required), kill the poller before closing.**
- Guard the poller itself: re-confirm `state==OPEN` immediately before merging.
- If a race is suspected, the forensic check is `mergedAt > closedAt` (merged after being closed = the poller won). Revert immediately.

## 4. Put leftovers on the surface the gate reads

Observed: a conditional pass — "CLEAR, but fix one thing" — was received, the one thing was relayed to the implementer **in chat**, and the PR was left in the auto-merge poller's set. The moment CI went green, **only the fix itself merged; the condition (adding a test) was left behind**.

> **An automated gate sees only the surface it reads. "CLEAR but one thing" is conditional in human memory; to the machine, it is CLEAR.**

- **Turn the leftover into machine-readable state before arming the poller.** Reverse the order and you get exactly this accident.
- Where to put it depends on the environment. Measured: because every agent authenticates as the same GitHub user, **a change request (the official blocking mechanism) could not be used on one's own PR**; only two things actually worked: **leaving an unchecked `- [ ]` in the PR body's Test plan** (author side) and **flipping the PR back to draft** (`gh pr ready --undo` — the only machine-readable block a reviewer can apply unilaterally; it makes the PR unmergeable even when `mergeStateStatus` is CLEAN).
- When a single poller cycles over multiple PRs, **take the conditional PR out of the loop**.

## 5. Signature conflicts between concurrent PRs — main breaks while each stays green

When a PR that changes a shared signature (constructor arguments, function arguments, protocol methods) is in flight at the same time as a sibling PR that **adds new calls to the old signature**, **both CIs stay green, and main breaks the moment both land**. Git merges text (clean when nothing overlaps), but semantic contracts break under composition. **CI is structurally blind to this conflict** (the branch that diverged first never once compiles the sibling's new calls).

- After landing a signature-changing PR, **immediately grep main for surviving calls to the old signature and run the full suite on fresh main**. Don't trust the merged PR's branch CI.
- When an implementer reports "that's a pre-existing failure on main," **verify on freshly synced main and treat it as a live regression** (don't dismiss it as noise and route around it). Fix it in an independent small PR (protecting the purity of the refactor in flight).
- Distinguish sites that call the constructor directly (broken) from those going through a higher-level API (which absorbs the change internally and is unharmed), to narrow the migration set.

## 6. Don't let ready PRs sit

The reverse trap also exists: an attention-allocation instruction ("prioritize the others") was interpreted as "don't touch this ready PR," and a green, reviewed PR sat for four hours, then conflicted with a follow-up PR and went stale (owner: "don't park things on your own and accumulate staleness").

- **Green plus reviewed means merge promptly.** Merging is cheap; letting things sit accumulates conflicts.
- Hold only when a real gate exists (unresolved findings, verification pending, a design question) — and during the hold, rebase periodically to stay fresh.

## Addendum — reading verdicts

The three rules — "don't read the verdict and merge in the same batch," "once findings are filed, the gate is the post-update verdict," and "PASS-with-notes waits for the author's final" — live in the merge-gate section of [Operating with separate coder / tester / reviewer agents](../verification/roles.md) (this document is its machine-side, state-side complement).

## Checklist

- [ ] Reported "merged" only after confirming `state==MERGED` plus non-null `mergedAt`?
- [ ] Not treating a hook, a notification, or the fact of having run the command as evidence of a merge?
- [ ] When the merge decision reversed, killed the poller before closing? Does the poller have an OPEN guard?
- [ ] Turned a conditional CLEAR into machine-readable state (unchecked Test plan / draft) before arming the poller?
- [ ] Immediately after landing a signature change, grepped main and ran the full suite?
- [ ] Not letting a green, reviewed PR sit on the strength of an attention-allocation instruction?

## Sources (measured during reyn development)

Hook false fires: #2043/#2051 (parallel ack while the conditional gate never fired) and #2447 (hook misfired three times on a draft PR). Strict-mode normality: #3165→#3167 after the #3000 configuration. Poller reopen+merge: #2928 (one minute after the owner's rejection). Leftover placement: #3349 (only the fix merged, the test left behind; includes the measurement that change requests are unusable in the same-user environment). Signature conflict: the #3121 arc (#3123 × #3122, found by a third party running the full suite). Over-parking: #2840 (conflicted with #2841 after four hours).

## Related

- [Operating with separate coder / tester / reviewer agents](../verification/roles.md) — the three verdict-flow rules (the human side)
- [Closing keywords fire on surface text](closing-keywords.md) — the other automatic action a merge triggers
- [Your local tree goes stale silently](stale-local.md) — local sync after a merge
- [Reading green](../verification/green-reading.md) — observations don't name their referents
