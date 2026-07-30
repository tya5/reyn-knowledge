---
name: experiment-discipline
description: The discipline of experiments and benchmarks — verifying the identity of A/B arms, never publishing a confounded number even with a caveat, the units your logs count vs. the units on the wire, the overfitting self-check before "it's fixed", measurement proves only positives, and diagnosing irreproducible environments by falsification
tags: [verification, measurement, benchmark, ab-testing]
sources:
  - feedback_ab_arm_isolation_pythonpath_src_and_verify_import
  - feedback_dont_affirm_cross_arm_differential_from_single_point
  - feedback_no_confounded_benchmark_number
  - feedback_your_count_and_the_wire_count_are_different_altitudes
  - overfit_self_check
  - feedback_benchmark_catalog_tuning_is_soft_cheat
  - feedback_measure_negative_cannot_prove_from_limited_env
  - feedback_swe_passrate_internal_signal_not_published
  - feedback_owner_perf_freeze_falsify_before_refix_playbook
---

# The Discipline of Experiments and Benchmarks — Before Emitting a Number, Before Believing One

## Premise

A/B measurement (an experiment that compares two variants side by side under the same conditions), benchmarks, verifying that "the fix worked" — all of them end in a **number**. And a number **walks off on its own**, shedding the caveats about the conditions that produced it. This document is discipline for both sides: the one that emits numbers and the one that believes them. If [Measuring the Wrong Target](measurement-target.md) is the general principle of "does that number answer the question," this document is that principle specialized to the experimental setting.

## 1. Identity of the A/B arms — verify "which version ran" before you measure

Whether each arm of an A/B (arm = one of the two configurations being compared) **really executed the version of the code you specified** is a matter of verification, not assumption. Measured: an environment variable that was supposed to point at the code under test was **silently shadowed** by an editable install (a development-mode installation that makes Python import the in-development code directly), and no error appeared even though the wrong version ran — an A/B with a contaminated arm invalidates the entire differential.

- At the head of the measurement, **print and confirm the file path of the module that was actually imported**, then run.
- **A single-point differential can only assert existence.** "Across the whole range, this differential is caused by that mechanism" is a separate claim; it requires measuring the range and a **search for orthogonal mechanisms** (is there another mechanism in the comparison arm producing the same result?). Measured: after a differential was asserted from a single point, it turned out that a different mechanism in the comparison arm had been producing the same result.
- Honestly shrinking a claim (re-recording the asserted "the baseline arm stalls" as "it mostly recovers") is not a failure; it is correct behavior.

## 2. A confounded number is not published — not even with a caveat

A number whose measurement conditions turn out to have been broken (the system under test was handicapped) is **not published even with** a "reference value" or "lower bound" **annotation**. Numbers drop their annotations and walk off on their own, and later sessions and documents will quote the bare figure. Measured: the scoring harness was correct, but the system under test had been running in an environment where its own self-checks were broken — just before the number came out, it was classified as confounded, and the decision taken was to **checkpoint and re-measure faithfully in the next slot**.

- The question before emitting: "**Is every input to this number faithful?**" (Do the measurement conditions measure the claimed capability, or the handicap of a broken condition?)
- Derived rule: a measurement whose **verification loop touches the answer key** (iterating against the test leaks the answer) **cannot be published** as an external score — it retains value as an internal stability signal (use it strictly for that purpose, and do not call it an official score).

## 3. The units your logs count and the units on the wire are at different altitudes

Measured (all on localhost, at zero billing cost): a single embedding call sent **nine** HTTP requests. The log said "attempt 1/3." The culprit was not the suspected layer:

```
The caller intends "no retries specified (None)"
→ a lower library's `max_retries or DEFAULT` converts None into 2
→ the SDK sends 3 HTTP requests per call
× 3 retries at the upper layer
= 9 requests (write "3" in the config, and it silently becomes 9)
```

- **Before writing or reading "retries N times," count what is on the wire** — when retry layers nest, the configured values multiply.
- **Suspect the `x or DEFAULT` shape**: the caller's "not specified (None)" turns into the library's default. "Not specifying" and "disabling" are different things.
- **Name the unit your own log is counting**: attempt / call / request / round-trip are all different units.
- **Timeouts carry the same altitude gap**: an SDK's `timeout=` is the deadline **per retried request**, not the overall deadline — `timeout: 60` silently becomes 180 seconds. The risk is not "the library ignores your setting" but "**it honors it at a different altitude than the reader assumed**."
- A bonus measurement: an implementation that records cost only **after** the await records zero on exceptions and, even on success, only 1 of the 9 requests — a **structurally lower-bound** accounting. The owner's one-liner ("wait, have retries been inflating our costs this whole time?") was the starting point of the discovery; the author had merely written the branch and had **never measured it**.

## 4. The overfitting self-check before saying "it's fixed"

Immediately before reporting a fix as "verified," ask yourself the following (every item was actually stepped on in practice):

1. **N=1, or N≥5?** A system containing an LLM is stochastic; a single success is consistent both with "fixed" and with "it just dodged it this time."
2. **Does the test input overlap the surface of the fix itself?** Testing with a sentence nearly identical to the guidance text the fix added, and succeeding = **you only measured a match against your own fix**. Ask yourself: "If I deleted that guidance text, would this test still pass?"
3. **Did you separate the structural axis from the behavioral axis?** "It became reachable" (it now appears in the listing) and "it gets chosen given natural input" are different axes. Do not say "verified" on the former alone.
4. **Attribution ablation (reverting one factor at a time)**: is there other state you changed together with the fix (deleting a cache, etc.)? "If I revert only my own change, does the behavior regress?" — in the measured case, the true cause of the improvement was the state clearing done at the same time, and the fix itself was irrelevant.
5. **Did you decompose the ambiguous input?** If behavior that looked like "the wrong choice" is reasonable as a different interpretation of an ambiguous request, it is not a defect — "it was not a defect" is also a legitimate conclusion.
6. **If the fix originated from a benchmark**: first test whether the target is already reachable through the general path. "Fixing" something that is reachable by adding priority placement or micro-tuning prompts is a **soft cheat** (benchmark-specific tuning); the correct answer is a fix on the general path.

## 5. Measurement proves only positives — do not use it as grounds for rejection

The owner's challenge (2026-06-19): "Judging an improvement proposal by measurement is wrong. **In a limited environment, how do you prove that it cannot improve things in other environments?**"

- What measurement can prove is only the **positive** (it worked in this environment). Generalizing a negative from a limited environment into "it works nowhere" is logically invalid.
- Accept or reject improvement proposals (guidelines, prompt improvements, and the like) as a **design judgment**: is it sound, low-cost, harmless, and plausibly beneficial in other environments? "I could not measure an effect in my environment = reject" is wrong.
- Exception: **structural redundancy** (the system already does the same thing structurally) is an environment-independent fact, and a legitimate ground for rejection without measurement.

## 6. Performance problems in an environment you cannot reproduce are diagnosed by falsification, not confirmation

For a performance degradation or freeze that does not reproduce in your own environment (it occurs only on the other party's machine), do not ship a fix derived from static profiling and call it "fixed." Measured: a fix that was a genuine improvement by static analysis had **zero effect** on the other party's freeze (the true cause was on a different path). What prevented the next two wrong fixes was a chain of falsifications:

1. **Falsify the hypothesized subsystem wholesale with a config-lever A/B**: have the other party flip a setting that bypasses the hypothesized layer. If the symptom persists, the entire hypothesis (past fixes included) is rejected in one shot. Cheapest and environment-independent — fire this first.
2. **Pincer with the gaps in the event log**: the blank between two clock-synchronized events localizes the fault to uninstrumented synchronous code.
3. **A stack dump is decisive**: capture the stack once during the hang, and the function that is stuck is named outright.
4. **Cheap behavior-fork questions**: "every turn, or periodic?" "are there images in the history?" — free ways to narrow down the subsystem.

The order of fixes is also disciplined: relieve the acute symptom first with the **minimal fix that is certainly correct**, and **split off** delicate optimizations (those whose interaction with recursion/recovery is unproven) into a follow-up with a tracking number.

## Checklist

- [ ] A/B: did you print and confirm each arm's actual import path? Are you promoting a single-point differential to "causality across the whole range"? Did you search for orthogonal mechanisms?
- [ ] Before emitting a number: is every input faithful? If there is a confound, did you checkpoint + re-measure (rather than annotate)?
- [ ] Are you treating a measurement whose verification loop touches the answer key as a public score?
- [ ] Before writing "N times": did you count on the wire? Any `x or DEFAULT`? Did you name the unit your log counts?
- [ ] Before "it's fixed": N≥5 / overlap between input and fix surface / structural vs. behavioral axis / ablation / decomposition of ambiguity / not a soft cheat?
- [ ] Are you rejecting an improvement proposal on a limited-environment negative (did you decide by design judgment)?
- [ ] An irreproducible report: did you run the falsification chain (config lever → log gaps → stack) instead of closing it on static analysis?

## Sources (measured during reyn development)

A/B arms: #2187 (an editable install silently shadowed the environment variable and contaminated an arm / a differential asserted from a single point → an orthogonal mechanism was found). Confounded numbers: FP-0008 C7 (2026-05-30, a score from an environment with broken self-checks was classified as confounded just before publication; checkpointed). Altitude of units: #3047 (2026-07-17, 1 embed = 9 HTTP requests, sparked by the owner's one-liner; cost recording is a structural lower bound) + #3045 (the timeout altitude gap; the author falsified their own docstring). Overfitting: 2026-05-17 scenario D (an N=1 "verified" with a sentence identical to the guidance text → the true cause was the simultaneous cache deletion; the same author had previously verified with 430 calls × 10 hypotheses with rejection criteria). Soft cheat: #187 (an addition to priority placement was sent back by the owner — it was reachable via the general path). Positives only: 2026-06-19 owner directive. Internal signal: 2026-05-31 (scores whose verification loop leaks the answer are not published; the internal value is retained). Falsification diagnosis: #2937 (a genuine improvement with zero effect) → #2938 (true cause: an O(N×M) re-scan per message) → #2939 (delicate optimization split into a follow-up).

## Related

- [Measuring the Wrong Target](measurement-target.md) — the general principle of "does the number answer the question"
- [Before Blaming the Model](capability-attribution.md) — the attribution-side discipline (this document's counterpart)
- [Integrity of the Strip Instrument](strip-instrument-integrity.md) — when the measuring instrument itself is broken
- [Reading Green](green-reading.md) — confirming "what ran"
- [Argument Hygiene](argument-hygiene.md) — the battery of checks before writing a conclusion
