---
name: sweep-reviews
description: Reviewing a sweep PR from the diff alone shows only half the picture — "left untouched" is also a decision, and decisions are verification targets. Spot-checking only answers whether the fixed items are correct
tags: [verification, review]
sources:
  - feedback_sweep_pr_review_the_untouched_decision_is_invisible_in_the_diff
---

# Reviewing Sweep PRs — Verifying the Decisions That Don't Appear in the Diff

## The shape of the problem

A **sweep PR** (bulk-fix PR) is a PR that scans the entire repository and fixes instances of the same kind of problem in one pass. Example: audit all 119 status-record files and fix the 26 that had gone stale.

The natural review for this type of PR is **spot-checking**: pick some of the fixed items and cross-check them against primary evidence (the actual state of the PRs, the existence of the commits). In the real case, 11 items were sampled and **all of them matched**. So can this sweep be trusted?

**Only half of it has been verified.** A sweep PR is dangerous in three directions, and spot-checking reaches only the first:

| Danger | Appears in the diff? | Detectable by spot-checking? |
|---|---|---|
| A fixed item is wrong | Yes | **Yes** (cross-check against primary evidence) |
| Something that should not have been fixed was fixed | Yes | Yes (but you must bring your own standard of correctness) |
| **Something that should have been fixed was not fixed / something correct was flipped** | **No** | **No** |

The third row is the heart of the matter. The 93 decisions behind "left 93 of the 119 untouched" are **written nowhere in the diff**.

> **In a sweep PR, "did not change" is also a decision, and decisions are verification targets. The diff shows only the decisions to act. The decisions not to act remain invisible forever unless you go looking for them.**

This is the document version of "a skipped test looks green": "fixed 26" is no evidence that "the remaining 93 were correct." **An unchanged item does not distinguish "inspected and found correct" from "never looked at."**

## A real review that closed the gap

What the independent second verifier did on this sweep serves directly as the model:

1. **Actively selected the strongest candidate for "looked likely to be flipped, but was not."** A file left in place as "partially complete" — exactly the kind of item that mechanical processing would likely have flipped to "complete."
2. **Measured the grounds for leaving it against primary evidence.** The file's claim ("the module in question is still large") was measured with `wc -l` (7488 lines / 3903 lines), confirming that leaving it untouched was correct.
3. **Did not trust the PR numbers quoted in the diff; re-pulled them with `gh pr view`.**

Only then did "**the decision not to flip was correct**" become evidence.

## How to apply

When reviewing a sweep / bulk-fix PR, in addition to spot-checking:

1. **Pick 1–2 "looked likely to be flipped, but was not" items yourself, and confirm against primary evidence that leaving them was correct.** How to pick: items that mechanical processing would have wrongly changed (partially implemented, conditional, or exception-status items).
2. **Count the population yourself.** Confirm the N in "fixed M of N" with something like `git ls-tree | grep -c`. Writing "the full set" in the task request is no guarantee that the full set was actually examined.
3. **Be suspicious of zero abstentions.** In a sweep involving semantic judgment, zero abstentions more likely means "**has no criterion for abstaining**" than "understood everything." In the real case, 2 items were held back as abstentions, and that in itself was a quality signal.
4. **Do not trust quotations inside the diff.** Re-pull PR numbers, commits, and line counts from primary sources (`gh pr view` / `wc -l`), not from what the diff says.

## Sources (measured during reyn development)

#3186 (a full sweep of 119 files with 26 fixes; the reviewer, with all 11 spot-checked items matching, was about to conclude "trustworthy" — an independent second verifier closed the gap by verifying the "not flipped" side).

## Related

- [The discipline of enumeration](enumeration-discipline.md) — the general form of counting the population yourself
- [Measuring the wrong target](measurement-target.md) — the structure of "feeling verified"
- [How vacuous gates are born](vacuous-gates.md) — the many variants of "green does not distinguish 'inspected and correct' from 'never looked'"
