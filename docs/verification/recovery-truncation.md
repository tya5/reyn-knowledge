---
name: recovery-truncation
description: Start recovery/durability verification from "what destroys the source of this rebuild" — a green write→rebuild round-trip still loses data if the source vanishes with log truncation. Make truncate-falsification a mandatory gate
tags: [verification, durability, design]
sources:
  - feedback_recovery_source_must_survive_truncation_review_gate
---

# Does the Recovery Source Survive Destruction? — A Green Round-Trip Is Not Enough

## The shape of the problem

In systems that rebuild state from an event log (event sourcing, write-ahead logs, checkpoint + replay of the delta), the log is normally subject to **truncation, compaction, and rotation** — obviously, since it cannot grow forever. The typical design is that only the snapshot (a copy of the full state at a point in time) survives.

This is home to a classic data-loss bug:

> **State needed for recovery is being rebuilt from events in a truncatable log. The moment the log is truncated, the source of the rebuild vanishes, and that state can never come back.**

And worst of all, this bug leaves **the "write → rebuild" round-trip test green from start to finish**. Nothing looks broken until truncation actually happens.

Real case: in a durability-redesign review, the owner dug up **two real data-loss bugs**. (1) Configuration values were being persisted as log events (this had passed review). (2) The executing principal's identity was being rebuilt from the log (a pre-existing bug; on rewind it could be restored as a different principal — a hole with security properties). **The root was identical** — both built recovery state from a source that does not survive truncation.

## Why it was missed (two root causes)

1. **Only positive verification was done.** "Can we write and rebuild?" (yes) was checked. Nobody asked "**what destroys the source of this rebuild?**"
2. **The new feature was never checked against the system's existing invariants.** This system had always operated on the principle "the log gets truncated; only the snapshot survives" (that is exactly why the existing primary state lives in the snapshot). When the new feature was added, it was never held up against this existing principle — the log was mentally treated as "a permanent record."

The owner's catch generalizes: "**In this system the log gets truncated. Can X still be retrieved reliably?**" — a question that **adversarially applies the system's destructive properties to the new feature**.

## The discipline

1. **Identify the source of the rebuild and run it through the destruction lifecycle.** What is the source for restoring this state? Does that source survive truncation, compaction, rotation, GC? **If it comes from log events, put it in the snapshot** (or explicitly exempt it from truncation).
2. **Make a truncate-falsification test a mandatory gate:**
   ```
   Set X
   → truncate the log past the event that was X's source
   → rebuild
   → assert that X survived
   ```
   As long as the source remains log-derived, this test mechanically produces RED — **it captures the entire class in a single test**. Note that the test is meaningless unless it is constructed to "destroy X's own record" ([The discipline of strip-falsification](strip-falsification.md), axis 4: if X's record is still there, of course it can be restored).
3. **Habit**: whenever you see recovery state, immediately ask — "Is this log-derived? → then truncation breaks it → move it to the snapshot."
4. **Do not report "it can be rebuilt" as "done."** The definition of done includes verification through the destruction lifecycle (the completion-reporting discipline of [Claims without context](cross-context-claims.md)).

## Generalization — the "destroyer" differs per system, but the question is the same

Truncation is merely this case's destroyer. Cache eviction, TTL expiry, retention policies, compaction, schema migrations — **everything in your system for which "deleting is normal operation"** demands the same question:

> **Does this source of recovery / audit / billing / identity survive the destruction the system performs as normal operation?**

In the terms of [Census vs structure](census-vs-structure.md), "log entries can be truncated" is a **structural premise** (it is the contract of what a log is), while "we haven't truncated yet" is merely a census.

## How to apply

- [ ] For a recovery/durability feature, can you state the source of the rebuild in one sentence?
- [ ] Did you enumerate the normal operations that destroy that source (truncation, eviction, retention expiry)?
- [ ] Did you add a truncate-falsification test (destroy X's source, then rebuild → assert X survives)?
- [ ] Did you check the new feature against the system's existing destruction invariants?

## Sources (measured during reyn development)

#2259/#2260 (durability redesign): two bugs — configuration persisted as log events (approved in review) and identity rebuilt from the log (pre-existing; a security hole that could escalate on rewind) — found by the owner's question "the log gets truncated, so can X still be retrieved reliably?" Making the truncate-falsification test a mandatory gate is the outcome of that review.

## Related

- [The discipline of strip-falsification](strip-falsification.md) — truncate-falsification is vacuous unless constructed to "destroy X's own record" (axis 4)
- [Census vs structure](census-vs-structure.md) — "can be truncated" is contract; "not truncated yet" is census
- [How vacuous gates are born](vacuous-gates.md) — the shape where a green round-trip witnesses nothing
