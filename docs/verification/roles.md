---
name: roles
description: An application map for operating with separate coder / tester / reviewer / dispatcher agents — separation works only when a context that does not share the implementer's assumptions holds the witness. The coder's reporting contract, the tester's falsification menu, the reviewer's independent verification and verdict flow, and what to write in the brief
tags: [verification, roles, collaboration, index]
sources:
  - feedback_reviewer_speculation_arrives_as_instruction_and_outranks_measurement
  - feedback_gate_merge_on_covet_verdict_not_batch_read_and_merge
  - feedback_wait_for_updated_verdict_before_merge_after_raising_finding
  - feedback_covet_note_gates_author_final
---

# Operating with Separate Coder / Tester / Reviewer Agents — Who Verifies What, Reports What, and Waits for What

The standard configuration for AI coding is heading toward splitting implementation (coder), testing (tester), and review (reviewer) into **separate agents**. Every document in this repository can be read as assuming that configuration — indeed, **nearly every source incident has the shape "the implementing agent's verification was green; an independent agent's verification exposed the problem,"** so this material is itself a measurement of the effect of role separation. This document is an application map that reorganizes the 25 disciplines into per-role obligations.

## Why separation works — and when it does not

Recall the core claim of [Test doubles must match the real shape](test-doubles.md): **the double and the test share their author's assumptions, so they keep agreeing with each other while missing reality**. This is not only about test doubles. When an implementer writes tests for their own implementation, **the tests are born from the same assumptions as the implementation** — the same misunderstanding of the envelope shape, the same "this can't be called here," the same focus on the happy path.

The strength of an independent agent is not capability but **non-shared context**. An agent that does not know the implementer's intent can see nothing but behavior — role separation enforces, through organizational structure, the records-of-intent vs records-of-behavior principle of [Liveness is decided by the producer](liveness-is-producer.md).

But separation does not work automatically. **The ways it stops working are known:**

1. **The verifier reuses the implementer's artifacts (fixtures, fakes, test harness)** — the closed loop comes back ([Test doubles must match the real shape](test-doubles.md), Shape 2).
2. **The verifier co-signs by inference** — grepping for "a plausible cause exists in the code" is not observation ([Proving fixture provenance](fixture-provenance.md)).
3. **The verifier's speculation arrives as an "instruction" and pushes aside the implementer's measurements** (below).

To be honest about configuration flexibility: at small scale, tester and reviewer can be one agent. An operation where the coder writes its own tests can also work — but in that case, **a falsification pass by an independent agent (strip, whole-mechanism kill, real-shape matching) becomes mandatory**. The invariant to protect is not the org chart but **"do not write — or verify — the witness with the same assumptions as the implementation."**

## The coder (implementation agent): the reporting contract

A completion report is not "done, it's green" — write it **in a form the recipient can verify**. It should include:

- **Strip-falsification details**: what (which single property) was broken, whether the replacements landed (counts), what went RED. The result of the whole-mechanism kill (the four axes of [The discipline of strip-falsification](strip-falsification.md))
- **Suite results with the skip list**: not a bare rc=0 but "N passed / M skipped: <reasons>" ([Reading green](green-reading.md))
- **Coordinates of the measurement**: which tree, which commit, which seed (if order-dependent), under what load ([Measuring the wrong target](measurement-target.md), [Proving fixture provenance](fixture-provenance.md))
- **Labels on claims**: never mix MEASURED / READ (read it in a source) / INFERRED
- **Declared landing shape**: is this the intended final form or a simplified version? If simplified, state the remainder explicitly ([Claims without context](cross-context-claims.md))
- **Leftovers made visible**: a checklist of done vs remaining ([Incomplete work delays discovery](incomplete-work.md))

## The tester (independent test-and-falsification agent): the menu

The point of separating out a tester is not to split the labor of writing tests. It is to **construct tests from outside the implementer's assumptions**. The independent tester's job is essentially falsification, and the menu already exists in this repository:

- **Run the strip falsification with your own hands**: re-execute the implementer's "RED confirmed." Verify anchor uniqueness and execution-environment identity yourself ([Integrity of the strip instrument](strip-instrument-integrity.md))
- **Whole-mechanism kill**: even when every individual strip landed, the shape where a wholesale `if False` still goes green remains ([The discipline of strip-falsification](strip-falsification.md), axis 4)
- **Falsify on the owner's (reporter's) real path**: not the sibling path the implementer's tests trod ([Prove fixes on the live path](fix-verification-live-path.md))
- **Build fixtures yourself from real traces**: do not inherit the implementer's fixtures. Pin the envelope shape from the producer's code ([Test doubles must match the real shape](test-doubles.md))
- **Measure a RED on a defective build before trusting a gate** ([How vacuous gates are born](vacuous-gates.md))
- **Step outside the happy path**: error paths, invalid inputs at runtime, the whole screen ([Beyond the happy path](beyond-happy-path.md))
- **Hit the destruction lifecycle**: truncation, eviction, retention limits ([Recovery must survive truncation](recovery-truncation.md))

## The reviewer (review agent): independent verification, and the asymmetry of instructions

The principle for how a reviewer receives things is in [Claims without context](cross-context-claims.md): **accept the map of facts; recompute the verdict in your own context**. In addition, look outside the diff ([Reviewing sweep PRs](sweep-reviews.md), [Fix-class review](fix-class-review.md), [Audits must match content](audit-content-match.md)).

And the reviewer carries a hazard no other role has:

> **A reviewer's speculation arrives as an "instruction." ∴ The recipient obeys it ahead of their own measurements.**

Real case: a reviewer handed over, as an instruction, the **unmeasured hypothesis** "this observation point is inside the external service, so it cannot exist." The implementer had **already measured the unreachability themselves in round 1**, yet demoted that to secondary and put the reviewer's speculation first. The implementer's summary: "**The reviewer-side speculation is the more dangerous one — it arrives as an instruction, so I obeyed it before re-checking my own data.**" A coder's speculation stops inside the coder; **a reviewer's speculation overwrites measurements downstream.**

- **When issuing instructions, separate speculation from measurement with labels** ("Measured: …" / "My hypothesis (unverified): …"). Written in the same register, the recipient cannot tell them apart.
- **If the other party has reported measurements, rank them above your own hypothesis.**
- **You can state explicitly "keep the conclusion, fix only the reasoning"** — a wrong reason does harm even under a correct conclusion (the wrong reason left in the code gets read by the next implementer).
- **The recipient's defense is a default procedure, not "being careful"**: ask "is this measured?" even of instructions, and measure yourself before applying. That same night, the agent that had made this procedure a standing practice was never pushed aside, and the agent that had not was — the difference is procedure, not attentiveness (principle 4 of [The system of verification knowledge](index.md)). A recipient measuring and overturning an instruction is **the normal path**; when you are overturned, be grateful and record it.

## Obligations of the dispatcher / merger (briefing and acceptance)

**What to write in the brief** — an aggregation of the "when briefing" items across the docs:

- Name the strip's exact target ("remove only the escape handling"), require confirmation of the replacement counts, and "if green, report it — green is the next question, not a conclusion"
- Demand **two separate asserts** — mechanism correctness and reachability ([Wiring tests vs mechanism tests](wiring-vs-mechanism.md))
- If the change alters text that reaches the LLM, announce the fixture re-recording in advance and require "RED with old code + new fixtures"
- The required format for "pre-existing failure" claims (seed + both main and PR + the set difference P−M)
- For user-facing changes, make the doc update a mandatory item

**Three merge-gate rules** (each from a measured accident):

1. **Do not read the verdict and merge in the same batch.** Merging reflexively on CI green puts the bug on mainline before you have processed a verdict that says "fix before merging" (real case: read + merge bundled into a single turn; a real bug the co-vet had named landed as-is). Make it separate steps: read the verdict → decide hold/merge → then merge.
2. **Once you have raised a finding yourself, the gate becomes the updated verdict.** Merging on the pre-finding CLEAR races your own escalation (real case: merged 40 seconds before the updated verdict).
3. **"PASS, but one note" waits for the author's "final."** The author may be fixing that very point right now (real case: merged 47 seconds before the improvement commit). The author's obligation is the pair of this: when you start addressing a note, announce "HOLD" first, and issue "final" when done.

## Role × pipeline map

| Stage ([The system of verification knowledge](index.md)) | coder | tester / reviewer |
|---|---|---|
| 0 Identity of the target | Attach the measurement coordinates (tree, commit, seed) to reports | Do not accept measurements without coordinates. Check your own measurement environment as check zero |
| 1 Substance of the claim | Perform the four strip axes and report the details. Attach the skip list | Independently re-run the strip (on the owner's path). Whole-mechanism kill. Build fixtures yourself |
| 2 Capability of the environment | Declare in plain language "in which environments this is unverified" | Ask "can this environment turn this red?" Step outside the happy path |
| 3 Static premises | Put a READ label on anything based on declarations or docs | Grep the producers. Match audit content. Assert absence for removals |
| 4 Review blind spots | Turn leftovers into a checklist | Count for yourself the judgments not in the diff, the population, the same-disease siblings |
| 5 Provenance and context of claims | State the landing shape (final form / simplified) | Recompute the verdict in your own context. Demand measurement labels on instructions |

## Sources (measured during reyn development)

The asymmetry of instructions: #3437 (a reviewer's unmeasured hypothesis pushed aside the implementer's measurement; the same-night contrast: another agent with first-hand re-verification as its default procedure was not pushed aside) and #3471 (the normal path: the instruction's recipient measured before applying and overturned it). Merge gates: #2774/#2775 (read+merge in one batch landed a bug the co-vet had named), #2826 (merged 40 seconds before the updated verdict), #1603/#1606 (merged 47 seconds before the author's improvement commit). The sources for the effect of role separation itself are this entire repository — see each doc's "Sources."

## Related

- [The system of verification knowledge](index.md) — the pipeline and the four principles; this document is their projection onto the role axis
- [Claims without context](cross-context-claims.md) — labels for claims that cross boundaries (the theory side of this document)
- [Test doubles must match the real shape](test-doubles.md) — the rationale for a separate tester
