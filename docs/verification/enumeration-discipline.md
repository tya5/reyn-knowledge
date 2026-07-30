---
name: enumeration-discipline
description: Completeness claims ("all call sites", "zero remnants", "applied to both backends") die by output truncation, query truncation, and surface mis-definition — never pipe through head, ask yourself which axes you queried, enumerate from the producer
tags: [verification, completeness, tooling]
sources:
  - feedback_census_vs_structure_definition_and_checked_premises
  - feedback_measured_but_the_target_was_off_my_four_instances
---

# The Discipline of Enumeration — How "I Checked Everything" Becomes a Lie

"I fixed all the call sites." "No references remain." "Applied to both backends." — Software changes constantly demand **completeness claims** of this kind. And completeness claims do not break because the investigation was sloppy. They break because **careful investigation gets truncated in predictable patterns**. The patterns come in three layers: output truncation, query truncation, and surface mis-definition.

## Layer 1 — Output truncation: never pipe a completeness grep through head

Real case: to determine whether an identifier was live, someone grepped **with the correct spelling** and got **65 hits**. They looked at only the first four — tests and log collection — and concluded "orphan (not used from anywhere)". **The live consumer was hit number five.**

The evidence was sitting in the buffer. They threw it away and **wrote the conclusion the truncation suggested**. Truncation always lies in the direction of "there is nothing more to see."

> **Never pipe a completeness grep through `head`.**
> 1. `wc -l` first.
> 2. If the count exceeds what you will actually read, **you must not generalize**. Either read everything, or **partition mechanically**:
>    ```bash
>    grep -rn "the_symbol" | awk -F: '{print $1}' | sort | uniq -c | sort -rn
>    ```
>    Drop all hits into per-file / per-area buckets and reason **bucket by bucket**.

The effectiveness of this technique was demonstrated the same day: in another sweep with roughly 98 hits, partitioning by area is precisely what made it possible to correctly detect the single live documentation remnant. **The same person, on the same day**, got it right in the sweep that partitioned and got it wrong in the sweep that didn't. What separated the outcomes was not personal attentiveness but the presence or absence of the procedure.

## Layer 2 — Query truncation: ask yourself "along which axes did I query?"

Even if you read all of the output, **the queries you issued may themselves be insufficient**.

Real case: a PR body asserted "the fix was applied to both backends", but there were in fact **three** places that issue write permissions, and the one that was missed sat on the side of **that PR's headline feature**. The implementer's self-diagnosis is on point:

> I grepped for the shape of the fix (`expanduser`) and the shape of the constructor (`SandboxPolicy(`). **Both greps were correct, and both were the wrong question.** Because I had searched exhaustively along two axes, the sweep **felt** complete. But I never once asked the axis "**who reads this field?**"

> **Thoroughness along the wrong axis produces the "feeling" of completeness. That feeling is exactly the danger signal.**

Before trusting a sweep, ask: **"Along which axes did I query? Is 'who consumes this?' among them?"**

## Layer 3 — Surface mis-definition: naming conventions and formatting cannot define the surface

If you define the "surface" to be enumerated **by naming convention or string formatting**, you silently drop the entities that don't follow the convention.

- Real case 1: an audit of orphaned index entries scanned only the **filename pattern** `*_family.md`. It missed consolidated files that didn't follow the naming convention and reported 73 orphans (the real number was 31). Note also that the error came out on the **high** side — over-reporting looks harmless, but it was one step away from asking someone else to do unnecessary work.
- Real case 2: "no internal delimiter `__` remains" was checked by grepping for the **quoted-identifier format** `"[a-z_]+__[a-z_]+"` and reported as "zero remnants". But the display text in a different file (the surface the model actually reads) did not match that format and slipped straight through.

What matters here is how to fix it:

> **When a manual grep misses a visible surface, the fix is not "a better grep". It is enumerating from the machinery that produces that surface.**

What knows "what gets advertised" is the tool registration machinery (the registry), not the formatting of the source code. Have the machine enumerate the **rendered output** of every registered entry, and inspect that. For the same reason, the gate that protects completeness should not be "a human greps carefully one more time" but **a program that derives from the registry and checks every entry** — structurally eliminating the places where a human can truncate.

## Sums and signs — if the conclusion is a "direction", enumerate every term

Numerical arguments have an enumeration discipline too. Real case: the premise "+0.9 tokens per message boundary" was **true and survived measurement**. What failed was **claiming the sign of the two-term sum `net = (+0.9 × n) − 2` from the first term alone** (the −2 is the cost of brackets that appears only on the joined side).

> **If the conclusion is a direction or a sign, and the quantity is a sum, enumerate all of the terms.**

This incident has a beautiful epilogue: two measurers produced "contradictory" results, but they were in fact **measuring different terms of the same sum** (one of the paths simply had no bracket term at all). When measurements appear to contradict, first suspect "did we measure the same thing?" ([Measuring the wrong target](measurement-target.md)).

## "I checked" is not evidence of completeness

Even a correctly spelled grep plus reading every hit misses if an axis is absent. The discipline of enumeration is not finished until it lands in a form that does not depend on a person's declaration — a registry-derived, check-every-entry gate.

## Checklist

- [ ] Completeness greps: `wc -l` first. If it exceeds what you can read, read everything or partition mechanically (bucketize)
- [ ] Did you enumerate the query axes? Is the "who consumes this?" axis included?
- [ ] Does the surface definition come from the producer (the registry, the generation machinery)? Are you substituting naming conventions or formatting for it?
- [ ] If the claim is about direction or sign, did you enumerate every term of the sum?
- [ ] Is the final defense not "pay closer attention next time" but "a program that derives and checks every entry"?

## Sources (measured during reyn development)

Layer 1: #2965 (65 hits misjudged as "orphan" via head -4), #2958 (98 hits partitioned by area, detecting the one survivor). Layer 2: #2981 (the third emitter was on the side of the PR's headline feature). Layer 3: the memory index audit (73→31), #3429/#3463 (the format grep for `__` remnants slipping through). Sign of a sum: #2951.

## Related

- [Census vs structure](census-vs-structure.md) — the chain by which incomplete enumeration becomes "a fossil of a false description"
- [Fix-class review](fix-class-review.md) — entrust the enumeration of sibling sites to grep (not the author's list)
- [Measuring the wrong target](measurement-target.md)
