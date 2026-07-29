---
name: cross-context-claims
description: Advice, delegated audits, and completion reports carry claims that depend on context the sender does not have — label each claim with the context it was computed in, or gate it on approval from whoever owns that context
tags: [verification, collaboration, review]
sources:
  - feedback_hedge_impl_dependent_design_steers
  - feedback_subagent_audit_verdict_needs_owner_context_regrounding
  - feedback_completion_report_must_match_owner_intended_architecture
---

# Claims Without Context — Labels for Advice, Delegated Audits, and Completion Reports

## The shape of the problem

When multiple agents (or people) develop together, claims constantly flow across **knowledge boundaries**. The design advisor does not have the implementation details. The delegated audit agent does not have the history behind design decisions. The implementer does not have the final shape that lives in the owner's head.

Claims crossing boundaries is, in itself, normal. What breaks things is when **a claim carries the parts that depend on context the sender does not have in assertive form, as if the sender did have that context**. Three shapes recur.

## Shape 1 — Design advice that depends on implementation details: verify it, or hedge explicitly

When a design steer (directional design advice) contains **claims that depend on implementation details** — "who owns this state," "can this value be cached," "what is this invariant," "is this function sync or async" — there are only two options:

1. **Verify the detail against the primary source (the implementation) before** asserting it
2. If you cannot verify it, hedge explicitly: "**this point awaits primary confirmation of X; please verify on the implementation side**"

Real case: within a single design wave, the lead's design advice was correctly overturned **three times** by implementation-side primary evidence — (a) the proposed implementation split broke the existing "list and detail come from a single source" invariant; (b) the target proposed for synchronous prebuilding was in fact an async function with dynamic inputs, impossible to precompute; (c) what was advised as "the caller can just hold this as its own state" was in fact a shared singleton that would collide under concurrent execution. In all three cases the high-level steer itself was useful. The only problem was **asserting the load-bearing details unverified** — hedging up front would have made the round trips unnecessary.

## Shape 2 — From a delegated audit, take only the facts; recompute the verdicts yourself

When you delegate a read-only audit to a subagent, the **verdicts** that come back ("this is a removal blocker," "this is safe") were computed **from the code alone**. The subagent has neither the owner's design decisions nor the background of the contract.

> **A subagent correctly finds what will break. But whether breaking it is the point is something it cannot know, even in principle.**

Real case: an impact audit for a removal PR was delegated, and the report came back as "this restore feature breaks = removal blocker, user-visible regression." But removing it was **precisely the point of the removal as the owner had decided it** (the removal existed to fix the collisions that restore behavior was causing). The correct handling: (1) accept the **map of facts** (call sites, each one's fallback) as an asset; (2) **recompute the verdicts yourself, in the owner's context**; (3) directly confirm the one load-bearing claim yourself. Addendum: a subagent's **scope summary can also undershoot** (a layer reported as "its sole role is X" in fact also handled Y) — re-derive the scope yourself.

## Shape 3 — "A working version landed" and "the intended design landed" are different completions

Before writing a completion report for work that originated in the owner's design proposal, explicitly determine **whether what landed is the final shape the owner intended, or a simplified version**.

Real case: in a durability design, what landed was not the final shape the owner proposed (the owning worker holds numbering and writes serially, eliminating the lock) but a **consciously staged, simplified version** (the caller assigns numbers under a lock). It was reported as "complete, landed" — and the divergence surfaced only after the owner asked **twice**: "this is not what I expected. Is this a next step? Or is something misunderstood?"

- For a simplified or staged version, **declare proactively**: "**running, but short of the final shape; X remains.**" Do not flatten it to "complete / done."
- Confirm whether the staging was **agreed with the owner or decided unilaterally on the implementation side**.
- When asked "did it land?", answer with **the degree of match to the design intent**, not with whether the feature exists.

## The unified shape — label each claim with the context it was computed in

If MEASURED / READ / INFERRED in [Measuring the wrong target](measurement-target.md) are labels for the **kind of evidence**, this document is about labels for the **scope of context**:

> **How much context was this claim computed against — verified against the implementation, from the code alone, or checked against the design intent?**

To the receiver, an unlabeled assertion is indistinguishable from a claim verified against full context. And the cost of correction (round trips, false blocks, the owner having to ask again) is always higher than one line of labeling.

## How to apply

- [ ] Before writing design advice: did you verify the implementation details it depends on against the primary source? If you could not, did you hedge explicitly?
- [ ] When receiving a delegated audit: did you separate facts from verdicts? Did you recompute the verdicts with context? Did you directly confirm the one load-bearing claim?
- [ ] Before a completion report: is the landed shape the intended final form or a simplified version? If simplified, did you state what remains? Was the staging agreed?
- [ ] For each claim written in assertive form, can you attach "the context in which it was computed"?

## Sources (measured during reyn development)

Shape 1: the tool-architecture design wave of 2026-06-14 (three corrections by primary evidence). Shape 2: #2248 PR-D (the removal audit's "blocker" verdict was the very point of the owner's decision; the scope undershoot is from the same case). Shape 3: #2246/#1765 (a simplified version reported as "complete"; the divergence surfaced only through the owner asking twice).

## Related

- [Measuring the wrong target](measurement-target.md) — MEASURED / READ / INFERRED, and the relay rule for measurements (attach the tree and commit)
- [Reviewing sweep PRs](sweep-reviews.md) — the delegation variant of "don't swallow verdicts; count the population and the evidence yourself"
