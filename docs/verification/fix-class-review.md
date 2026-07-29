---
name: fix-class-review
description: Reviewing a fix in the correctness frame lets sibling defects survive — "does it break other callers?" and "do other callers carry the same pathology?" are different questions, and by default only the former gets asked
tags: [verification, review, performance]
sources:
  - feedback_perf_fix_needs_fixclass_question_not_correctness_frame
---

# Fix-Class Review — "Does It Break Others?" and "Do Others Share the Disease?" Are Different Questions

## The two frames

When reviewing a performance fix (or the removal of any "pathology"), there are two frames your head can be in:

- **The correctness frame** (the default): "Does this change **break** other callers?" → The shared function has not changed by a single byte → no impact → **pass**.
- **The fix-class frame** (what pathology removal actually requires): "Do other callers carry **the same pathology this fix is removing**?"

Answering the former correctly answers nothing about the latter. And **review, by default, asks only the former**. This is the mechanism by which defects of the same shape as the fix (the fix-class siblings) pass review and survive.

Real case (took three PRs): a spot that re-scanned the entire log on every message processed was fixed by introducing a "build the predicate once and reuse it" helper. Both reviewers passed it. One of them had, **in their own pre-investigation, even enumerated the sibling callers**, yet dismissed them with "the shared function is unchanged, so the callers are unaffected" — a **correct answer** to the correctness question. As a result, **seven sibling sites survived**. One of them was O(n²) on the cold-start path, invisible to local measurement because the dev-environment log had only 14 lines.

## The actionable signal — a new helper applied in exactly one place

> **If a fix introduces a hoist/dedup primitive (a reuse helper) and applies it in only one place, the very existence of that primitive is evidence that the pathology is a class.**

Nobody extracts a helper for a computation that has only one caller. The hand that did the extracting knows "there are others of the same shape". In review, ask immediately: **"Who else has this shape?"**

And place the authority for the enumeration in **grep** — not the author's declared list. In the real case, the target sites written in the request were **all wrong**, and the implementer's grep found seven sites absent from the request. ([The discipline of enumeration](enumeration-discipline.md))

## Inspecting the rationale — is that "safe" a census?

The rationale for excluding a sibling site as "safe here" is usually census-shaped ("this site's caller doesn't loop" — true today, silently false tomorrow). There is a real case where the original pathology itself grew exactly that way: against an API that takes the log as an argument and re-scans it internally, a looping caller **sprouted later**.

- **If the shape of the API forces the pathology, caller-side fixes will not close it. Change the shape** (pass a predicate, not the log).
- **Never fix only one half of a "mirror pair."** If you symmetrize only one side of a self-proclaimed mirror, the surviving old symmetry docstring starts to **actively argue for reintroducing the defect** ("these two are supposed to be identical — let's go back to pulling the log").

For the census/structural classification of the rationale itself, see [Census vs structure](census-vs-structure.md).

## Checklist

- [ ] Can you state in one sentence what pathology this fix removes?
- [ ] **Separately from** the correctness question (does it break anything?), did you ask the fix-class question (does anyone share the disease)?
- [ ] If a newly introduced helper is applied in one place, did you enumerate "who else has this shape?" with grep (not trusting the author's list)?
- [ ] Is the rationale for "safe here" structural? Or merely "today's callers don't happen to do that"?
- [ ] Does the API's shape itself force the pathology? Have you left one half of a mirror pair behind?

## Sources (measured during reyn development)

#2937/#2938 (predicate helper introduced, applied in one place, passed review) → #2948 (seven sibling sites, including O(n²) on the cold-start path; the sites in the request were all wrong, grep had the right answer). Real case of a census rationale and the true structural rationale: #2945.

## Related

- [Census vs structure](census-vs-structure.md)
- [The discipline of enumeration](enumeration-discipline.md)
- [Shared-helper widening](shared-helper-widening.md) — the danger of consolidation always lies in the call sites it flattens
