---
name: census-vs-structure
description: "If this premise breaks, does the system 'break' or merely 'become something different'?" — a two-way classification of justifications. Building on census premises is allowed only with a check and visible failure, and the check does not make the premise true
tags: [verification, reasoning, design]
sources:
  - feedback_census_vs_structure_definition_and_checked_premises
  - feedback_perf_fix_needs_fixclass_question_not_correctness_frame
---

# Census vs Structure — "True Today" vs "Built to Be True"

## The shape of the problem

Every design decision and every review argument rests on some premise. "This field can be trusted, because ..." "This optimization is safe, because ..." There are **two kinds** of "because," and confusing them means building the system on a justification that is correct today and silently breaks tomorrow.

- **Census type**: a **report of present fact**. We inspected three call sites and all had n < 100. The current provider returns this format. Every consumer today reads through a filter.
- **Structural type**: a **description of a contract**. This list is sorted, because it is the output of `sort()`. Entries in this log can be deleted, because the deletion API is defined that way.

The discriminator is a single question:

> **When this premise breaks, does the system "break" (BROKEN), or does it merely "become something different" (DIFFERENT)?**

To break a structural premise, **someone has to change the design or the contract**. A census premise breaks **merely because the world happens to change** — a new call site, a different provider, an input nobody had tried. When the former breaks, "the system broke"; when the latter breaks, "circumstances changed," nothing more.

## Why "falsifiability" fails as the discriminator

The first proposed definition was "structural = contains no falsifiable empirical claim." This **over-fires and empties the structural category**.

Example: "entries in this log can be truncated" is falsifiable (delete the truncate feature and it becomes false). It is still structural — because the only way to make it false is a **design change**. Conversely, "BPE tokenizer splitting only loses merges, so the sum of the fragments ≥ the whole, i.e. it always errs on the safe side" sounds plausible but is census — **nobody designed the tokenizer to guarantee that**. It is a fact you would like to hold, not a contract anyone signed.

What matters is not the **form** (falsifiable or not) but the **substance** (whose contract is it?).

## The same premise is structural under one regime and census under another

Example: suppose token-count estimation assumes **additivity** — "the total for a message array = a constant + the sum of the per-message counts."

- If one provider's official counting convention is **defined** as "3 + Σ(per-message)," then relative to that convention additivity is **structural** (a non-additive implementation is not an implementation of that convention = it is broken).
- But when applied to **multiple providers** through an intermediate library, additivity becomes **census**. Measurements showed five providers returning identical numbers = the library is merely applying one convention to all providers. **The improvement everyone hopes for — the library gaining accurate per-provider counting — is precisely the thing that breaks additivity.**

And **at runtime you cannot tell which regime you are in**. That is the reason for the next section.

## You may build on a census premise — but only with a check and visible failure

> **Never assume a census premise at design time. Check it at runtime, and fail loudly when it breaks.**

Two important corrections here (both settled in the back-and-forth of an actual design review):

1. **The check does not make the premise true. It only makes the failure visible.** A checked census premise is not **promoted** to structural. It is **demoted** from "hidden dependency" to "declared and monitored dependency." The value gained is operational (you find out when it breaks), not epistemic (the premise does not become more certain).
2. **You may reason from a structural premise. On a checked census premise you may only depend with your own guard in place.** Otherwise the next reader reads "checked = precondition" and **builds a second feature on top of it without a guard of their own**. "Checked" protects only the call paths that the check protects.

## A check protects only the domain it sampled

The check itself inherits the census failure mode. Real case: a check that "verifies additivity once at construction time" sampled with **two plain messages**. That check **passes even in a world where additivity is broken for multimodal attachments or tool calls** — it never touches those shapes.

```
Construction-time check (2 plain messages): T([A,B]) == k + unit(A) + unit(B) → True
  ↑ tool_calls and images never pass through
Shapes silently generalized over: tool_calls, multimodal
```

**A check protects only the domain it sampled.** The shape of the countermeasure: a **shape trigger** (run the check on first occurrence of a new shape class) + **periodic reconciliation** (every N iterations, recompute the whole and compare; a discrepancy = resync + audit event, never a silent correction). This turns "pray there is no drift" into "drift is bounded by N, and it is measured."

## The field test — can one grep falsify it?

> **"If I read the code and show today's behavior is different, does this justification collapse?" — If it collapses, it is census, and the real justification has not yet been found.**

Real case: the justification "the usage total must not go into the event log, because the log is branch-filtered and a rewind would drop the abandoned usage" — plausible, but one grep collapsed it (the filter is **per-consumer opt-in**, and the usage consumer can simply not filter). So: census. The real (structural) justification lay elsewhere: **log entries can, by design, be truncated**, so placing a lifetime-monotonic total in a truncatable log means it gets eaten somewhere in the food chain — and indeed this repository **has shipped exactly that failure once before**.

## A false description is the fossil of an incomplete enumeration

The worst path by which a census premise gets fossilized:

> **Incomplete enumeration → false assertion → the assertion gets documented → it becomes the next reader's premise.**

Real case: the comment "this value is wired up by the OpContext builders" was written while the enumeration was incomplete. **That comment became the next implementer's premise**; a feature was built on top of a recording mechanism that did not exist, and **with 8540 tests and CI all green, production was recording nothing**.

All six defects surfaced in a single day had this shape: a comment assuming wiring / a function **name** assuming execution (`install_seccomp_filter` merely "returns an installer," and both call sites discarded the return value = never once loaded in production) / a comment assuming a value range ("scores are in [0.0, 1.0]" is not a validator) / warning text assuming a fallback that never happens / a PR body assuming reachability (in reality a different field is read) / an old lie surviving in the documentation.

## How to apply

- Before calling a justification "structural," ask: **"If this premise breaks, does it break, or does it merely become something different?"**
- If it merely becomes something different, it is census. **Either find the real structural justification, or reduce it to a runtime check (shape coverage, visible failure, a note for the next reader).**
- When you write a check, **state the domain it samples**. Do not generalize the premise to shapes the check never touches.
- Before asserting anything about some domain (call sites, consumers, grep hits, terms of a sum), **enumerate it**. For the concrete discipline of enumeration, see [The discipline of enumeration](enumeration-discipline.md).

## Sources (measured during reyn development)

Settling the definition: roughly 8 rounds of mutual falsification between two reviewers on 2026-07-15/16 ("truncatable is structural" in #2945 / rejection of "BPE errs safe" / regime-dependence of additivity). The fossil chain: #2949 (comment → feature built on a nonexistent recorder), #2962 (function name assuming execution), #2963, #2961, #2960, #2958. Census verdict by a single grep: #2945 (branch filter is opt-in).

## Related

- [The discipline of enumeration](enumeration-discipline.md) — the side that mechanically prevents "incomplete enumeration"
- [Structural blindness of the verification environment](environment-blindness.md) — the environment version of "a check protects only the domain it sampled"
- [Fix-class review](fix-class-review.md) — how a census justification lets sibling defects survive review
