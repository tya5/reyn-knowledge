---
name: verification-index
description: The map of this knowledge area — what to doubt, in what order, between an agent's "done, all green" report and the merge. Where the 13 docs sit, and the 4 principles that cut across them
tags: [verification, index, overview]
---

# The System of Verification Knowledge — the Order in Which to Doubt "Done, All Green"

## Why this system is needed

In development performed by AI coding agents, **the same party that implements a change also verifies it and writes the report**. The roles that a human team implicitly separates — builder, verifier, reader of the report — collapse into one context. And the reports are invariably fluent and confident: **confidence does not correlate with correctness**, so verifying the report becomes an engineering step of its own.

Three more conditions stack on top: generation speed exceeds human review speed; the implementer writes its own tests (its own grading criteria); and multiple agents work concurrently in the same repository and shared environment. Together these four conditions turn a failure mode that is rare in human teams — **the industrial-scale production of self-overtrust** — into a daily occurrence. The 13 documents in this directory are a system of countermeasures extracted from field records of exactly those failures.

## The pipeline — from completion report to merge

The 13 docs form a single pipeline of "what to doubt, in what order." The further upstream a breakage occurs, the more it invalidates every verification downstream of it.

```mermaid
flowchart TD
    S0["Stage 0: identity of the target<br/>what did that measurement measure?"] --> S1
    S1["Stage 1: content of the claim<br/>what does that green witness?"] --> S2
    S2["Stage 2: capability of the environment<br/>can this environment turn red?"] --> S3
    S3["Stage 3: static premises<br/>what are we believing without running it?"] --> S4
    S4["Stage 4: review blind spots<br/>which decisions never appear in the diff?"] --> S5
    S5["Stage 5: provenance of artifacts<br/>did this artifact come from reality?"] --> M["merge"]
```

### Stage 0 — Identity of the target: what did that measurement measure?

The furthest upstream. If this is off, every green and red downstream is a claim about something else.

- [Measuring the wrong target](measurement-target.md) — "measured, but the target was off" is more dangerous than "didn't measure." Before measuring: does this measurement cover what the decision needs?
- [Integrity of the strip instrument](strip-instrument-integrity.md) — non-unique anchors, duplicated declarations, measuring another tree, load. A broken instrument yields no conclusions

### Stage 1 — Content of the claim: what does that green witness?

The distance between "a test exists" and "the property is protected."

- [The discipline of strip-falsification](strip-falsification.md) — four axes that turn a RED report into evidence (coverage, minimal break, did-it-land, whole-mechanism kill)
- [Wiring tests vs mechanism tests](wiring-vs-mechanism.md) — "the mechanism is correct" and "production reaches the mechanism" are separate asserts
- [How vacuous gates are born](vacuous-gates.md) — terminal-state-only asserts, positive controls on the wrong path, properties that exist only in prose

### Stage 2 — Capability of the environment: can it turn red?

Even with individually sound tests, the execution environment itself can be structurally unable to surface a defect.

- [Structural blindness of the verification environment](environment-blindness.md) — a gate witnesses an assumption only if its environment differs from the environment the assumption talks about

### Stage 3 — Static premises: what are we believing without running it?

How to treat claims, declarations, and descriptions derived from reading code rather than executing it.

- [Census vs structure](census-vs-structure.md) — telling "true today" from "built to be true," and how to build safely on census premises
- [The discipline of enumeration](enumeration-discipline.md) — how completeness claims die by truncation (of output, of query, of surface definition)
- [Liveness is decided by the producer](liveness-is-producer.md) — declarations, docs, and names are records of intent; only producers record behavior

### Stage 4 — Review blind spots: which decisions never appear in the diff?

- [Reviewing sweep PRs](sweep-reviews.md) — "did not change" is also a decision, and it is invisible in the diff
- [Fix-class review](fix-class-review.md) — "does it break others?" and "do others share the disease?" are different questions
- [Shared-helper widening](shared-helper-widening.md) — consolidation silently widens semantics; nobody asserts the negative

### Stage 5 — Provenance of artifacts: did it come from reality?

- [Proving fixture provenance](fixture-provenance.md) — verify "I re-recorded it" by dependence, not by the claim

## The four cross-cutting principles

The same principles reappear, in different guises, at every stage. If you forget the individual rules, you can re-derive them from these four.

### Principle 1 — Green is ambiguous

Green does not distinguish "inspected and correct" from "never looked" from "the experiment never happened." A skipped test looks green, an unchanged file looks inspected, a strip that never landed looks healthy. **Before using a green as evidence, identify which of its meanings it carries.**
(→ [strip-falsification](strip-falsification.md) axis 3, [vacuous gates](vacuous-gates.md), [sweep PRs](sweep-reviews.md))

### Principle 2 — Records of intent vs records of behavior

Schemas, docs, comments, declarations, and names are records of what someone intended; they testify to nothing about behavior. Only producers, execution, and measurement record behavior. **When you infer behavior from a record of intent, label the inference READ / INFERRED and never mix it with MEASURED.**
(→ [liveness is decided by the producer](liveness-is-producer.md), [census vs structure](census-vs-structure.md), [wiring vs mechanism](wiring-vs-mechanism.md))

### Principle 3 — Verification protects only the domain it sampled

A check protects only the shapes it touched, a test only the environments it ran in, a grep only the axes it queried. **Attach the domain (which shapes, which environments, which axes) to every "verified."** Generalizing beyond the sampled domain is the entrance to every "broke while green."
(→ [census vs structure](census-vs-structure.md), [environment blindness](environment-blindness.md), [enumeration discipline](enumeration-discipline.md))

### Principle 4 — Procedures, not attention

There are multiple records of the same person, on the same day, getting it right where a procedure was interposed and wrong where it wasn't. What reproduces is not "good judgment" but "the step that was interposed." **Land discipline as checklists, brief templates, and registry-derived gates — never leave it resting on attentiveness.**
(→ [measuring the wrong target](measurement-target.md), [enumeration discipline](enumeration-discipline.md), [strip instrument integrity](strip-instrument-integrity.md))

## Reading paths by role

- **Implementers (the agent itself, or whoever dispatches implementation)**: [strip-falsification](strip-falsification.md) → [wiring vs mechanism](wiring-vs-mechanism.md) → [fixture provenance](fixture-provenance.md). The craft of making your own reports evidence-backed.
- **Reviewers / co-vetters**: [sweep PRs](sweep-reviews.md) → [fix-class review](fix-class-review.md) → [shared-helper widening](shared-helper-widening.md) → [strip instrument integrity](strip-instrument-integrity.md). The craft of seeing outside the diff and the report.
- **Designers of verification systems and CI**: [environment blindness](environment-blindness.md) → [census vs structure](census-vs-structure.md) → [liveness is producer](liveness-is-producer.md) → [enumeration discipline](enumeration-discipline.md). The craft of deciding where gates live and what shape they take.
