---
name: experiment-discipline
description: The discipline of experiments and benchmarks — verifying the identity of A/B arms, "it really ran" does not mean the condition was reproduced, never publishing a confounded number even with a caveat, the units your logs count vs. the units on the wire, the overfitting self-check before "it's fixed", measurement proves only positives, and diagnosing irreproducible environments by falsification
tags: [verification, measurement, benchmark, ab-testing]
sources:
  - feedback_ab_arm_isolation_pythonpath_src_and_verify_import
  - feedback_dont_affirm_cross_arm_differential_from_single_point
  - feedback_running_it_is_not_reproducing_the_condition
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

## 2. "It really ran" does not mean the condition was reproduced

"It actually ran" and "the run supports a conclusion" are different things. There is a real case of three measurements in a single day, each genuinely executed, that supported nothing:

1. **The condition was never reproduced.** A flaky test documented as "occurs under load" was run sequentially, one test at a time, on a quiet machine, and passed 25/25 — nearly reported as "fixed." **Load, the condition, had not been created; it had merely been avoided.**
2. **The harness broke, not the code.** Correcting for that, the same test was run six times concurrently: 17/18 failed. The failure was `FileExistsError: already exists` — **six copies fighting over one shared workspace**, not a defect in the thing under test. Reporting that number as-is would have contradicted a cause another engineer had independently isolated.
3. **The unit was in dispute** (detailed in §4): "the residue is 2 cells" turned out to count occurrences of an escape code itself, including ones with no following text — the same hole as the units-and-altitude problem below.

What finally settled this flake was the third attempt — **running different tests in parallel**, i.e. the actual shape CI takes. Only then did usable evidence appear, and it exposed **a third face** of failure (a missing model fixture entry, distinct from either of the two known causes).

> **"It ran" feels cheap, so it gets treated as the evidence — but it is only the cheap part. What makes a run evidence is (a) that the condition under which the symptom occurs was actually created, and (b) that the failure you counted is the failure you think it is.**

- Before reporting a rate, say out loud **what condition the symptom requires** (load? parallelism? a cold cache? a specific terminal?) and whether the run actually had it. "Sequential and quiet" is not "under load."
- Before it becomes a number, **open the failure and read it**. A green-vs-red count says nothing about whether the red is the one you think it is.
- Parallel execution of a parallelism-sensitive test must come from **different tests** sharing a machine. N copies of the same test fighting over shared fixtures measures a harness defect, not parallelism.

## 3. A confounded number is not published — not even with a caveat

A number whose measurement conditions turn out to have been broken (the system under test was handicapped) is **not published even with** a "reference value" or "lower bound" **annotation**. Numbers drop their annotations and walk off on their own, and later sessions and documents will quote the bare figure. Measured: the scoring harness was correct, but the system under test had been running in an environment where its own self-checks were broken — just before the number came out, it was classified as confounded, and the decision taken was to **checkpoint (record the state and the judgment so far, and stop for now) and re-measure faithfully in the next measurement slot**.

- The question before emitting: "**Is every input to this number faithful?**" (Do the measurement conditions measure the claimed capability, or the handicap of a broken condition?)
- Derived rule: a measurement whose **verification loop touches the answer key** (iterating against the test leaks the answer) **cannot be published** as an external score — it retains value as an internal stability signal (use it strictly for that purpose, and do not call it an official score).

## 4. The units your logs count and the units on the wire are at different altitudes

Measured (all on localhost, at zero billing cost): a single embedding call sent **nine** HTTP requests. The log said "attempt 1/3." The culprit was not the suspected layer:

```
The caller intends "no retries specified (None)"
→ a lower library's `max_retries or DEFAULT` converts None into 2
→ the SDK sends 3 HTTP requests per attempt (1 initial + 2 added retries)
× 3 attempts at the upper layer (its retry setting "3" is implemented as total attempts)
= 9 requests (write "3" in the config, and it silently becomes 9)
```

- **Before writing or reading "retries N times," count what is on the wire** — when retry layers nest, the **attempt counts** multiply. Worse, what a setting means differs per layer ("added retries" vs "total attempts" — the example above is 3 attempts × 3 requests = 9). So what you multiply is never the configured values but the measured attempts × requests.
- **Suspect the `x or DEFAULT` shape**: the caller's "not specified (None)" turns into the library's default. "Not specifying" and "disabling" are different things.
- **Name the unit your own log is counting**: attempt / call / request / round-trip are all different units.
- **Timeouts carry the same altitude gap**: an SDK's `timeout=` is the deadline **per retried request**, not the overall deadline — `timeout: 60` silently becomes 180 seconds. The risk is not "the library ignores your setting" but "**it honors it at a different altitude than the reader assumed**."
- A bonus measurement: an implementation that records cost only **after** the await records zero on exceptions and, even on success, only 1 of the 9 requests — a **structurally lower-bound** accounting. The owner's one-liner ("wait, have retries been inflating our costs this whole time?") was the starting point of the discovery; the author had merely written the branch and had **never measured it**.

## 5. The overfitting self-check before saying "it's fixed"

Immediately before reporting a fix as "verified," ask yourself the following (every item was actually stepped on in practice):

1. **N=1, or N≥5?** A system containing an LLM is stochastic; a single success is consistent both with "fixed" and with "it just dodged it this time."
2. **Does the test input overlap the surface of the fix itself?** Testing with a sentence nearly identical (in the measured accident, exactly identical) to the guidance text the fix added, and succeeding = **you only measured a match against your own fix**. Ask yourself: "If I deleted that guidance text, would this test still pass?"
3. **Did you separate the structural axis from the behavioral axis?** "It became reachable" (it now appears in the listing) and "it gets chosen given natural input" are different axes. Do not say "verified" on the former alone.
4. **Attribution ablation (reverting one factor at a time)**: is there other state you changed together with the fix (deleting a cache, etc.)? "If I revert only my own change, does the behavior regress?" — in the measured case, the true cause of the improvement was the state clearing done at the same time, and the fix itself was irrelevant.
5. **Did you decompose the ambiguous input?** If behavior that looked like "the wrong choice" is reasonable as a different interpretation of an ambiguous request, it is not a defect — "it was not a defect" is also a legitimate conclusion.
6. **If the fix originated from a benchmark**: first test whether the target is already reachable through the general path. **If reachable**, no priority placement or prompt micro-tuning is needed — adding it anyway is a **soft cheat** (tuning that eases only that benchmark); in the measured case exactly such an addition was sent back by the owner. **If unreachable or awkward to use**, the thing to fix is the general path itself (a form that helps every user), not a benchmark-specific shortcut.

## 6. Measurement proves only positives — do not use it as grounds for rejection

The owner's challenge (2026-06-19): "Judging an improvement proposal by measurement is wrong. **In a limited environment, how do you prove that it cannot improve things in other environments?**"

- What measurement can prove is only the **positive** (it worked in this environment). Generalizing a negative from a limited environment into "it works nowhere" is logically invalid.
- Accept or reject improvement proposals (guidelines, prompt improvements, and the like) as a **design judgment**: is it sound, low-cost, harmless, and plausibly beneficial in other environments? "I could not measure an effect in my environment = reject" is wrong.
- Exception: **structural redundancy** (the system already does the same thing structurally) is an environment-independent fact, and a legitimate ground for rejection without measurement.
- Relation to §5: §5 states the evidence requirements for whoever claims the positive ("it's fixed"); this section is its flip side — since measurement proves only positives, rejection on a negative ("it doesn't help") is the job of design judgment, not measurement. Even if N≥5 shows no improvement, that only means "we cannot say it's fixed"; adoption returns to design judgment. (5 is not a magic number — an operational floor for telling a one-off fluke from a trend in a probabilistic system.)

## 7. Performance problems in an environment you cannot reproduce are diagnosed by falsification, not confirmation

For a performance degradation or freeze that does not reproduce in your own environment (it occurs only on the other party's machine), do not ship a fix derived from static analysis (reading the code without executing it) and call it "fixed." Measured: a fix that was a genuine improvement by static analysis had **zero effect** on the other party's freeze (the true cause was on a different path). What prevented the next two wrong fixes was a chain of falsifications:

1. **Falsify the hypothesized subsystem wholesale with a config-lever A/B**: have the other party flip a setting that bypasses the hypothesized layer. If the symptom persists, the entire hypothesis (past fixes included) is rejected in one shot. Cheapest relative to its decisive power — one round trip rejects a whole subsystem — and environment-independent; fire this first.
2. **Pincer with the gaps in the event log**: the blank between two clock-synchronized events localizes the fault to uninstrumented synchronous code.
3. **A stack dump is decisive**: capture the stack once during the hang, and the function that is stuck is named outright.
4. **Cheap behavior-fork questions**: "every turn, or periodic?" "are there images in the history?" — free ways to narrow down the subsystem (weak decisive power on their own; run them alongside 1–3).

The order of fixes is also disciplined: relieve the acute symptom first with the **minimal fix that is certainly correct**, and **split off** delicate optimizations (those whose interaction with recursion/recovery is unproven) into a follow-up with a tracking number.

## Checklist

- [ ] A/B: did you print and confirm each arm's actual import path? Are you promoting a single-point differential to "causality across the whole range"? Did you search for orthogonal mechanisms?
- [ ] Before reporting a rate: did you actually satisfy the condition the symptom requires (load, parallelism, cold cache, etc.)? Did you open the failure and read its content?
- [ ] Before emitting a number: is every input faithful? If there is a confound, did you checkpoint + re-measure (rather than annotate)?
- [ ] Are you treating a measurement whose verification loop touches the answer key as a public score?
- [ ] Before writing "N times": did you count on the wire? Any `x or DEFAULT`? Did you name the unit your log counts?
- [ ] Before "it's fixed": N≥5 / overlap between input and fix surface / structural vs. behavioral axis / ablation / decomposition of ambiguity / not a soft cheat?
- [ ] Are you rejecting an improvement proposal on a limited-environment negative (did you decide by design judgment)?
- [ ] An irreproducible report: did you run the falsification chain (config lever → log gaps → stack → behavior-fork questions) instead of closing it on static analysis?

## Sources (measured during reyn development)

A/B arms: #2187 (an editable install silently shadowed the environment variable and contaminated an arm / a differential asserted from a single point → an orthogonal mechanism was found). Reproducing the condition: three in one day (a flake passed 25/25 without reproducing load; a harness broken by six concurrent copies produced 17/18 failures; a unit in dispute; the third attempt, running different tests in parallel, finally produced usable evidence and a third failure face). Confounded numbers: FP-0008 C7 (2026-05-30, a score from an environment with broken self-checks was classified as confounded just before publication; checkpointed). Altitude of units: #3047 (2026-07-17, 1 embed = 9 HTTP requests, sparked by the owner's one-liner; cost recording is a structural lower bound) + #3045 (the timeout altitude gap; the author falsified their own docstring). Overfitting: 2026-05-17 scenario D (an N=1 "verified" with a sentence identical to the guidance text → the true cause was the simultaneous cache deletion; the same author had previously verified with 430 calls × 10 hypotheses with rejection criteria). Soft cheat: arc-187 (an internal campaign label, not an issue number; an addition to priority placement was sent back by the owner — it was reachable via the general path; the episode is recorded in #1416). Positives only: 2026-06-19 owner directive. Internal signal: 2026-05-31 (scores whose verification loop leaks the answer are not published; the internal value is retained). Falsification diagnosis: #2937 (a genuine improvement with zero effect) → #2938 (true cause: an O(N×M) re-scan per message) → #2939 (delicate optimization split into a follow-up).

## Related

- [Measuring the Wrong Target](measurement-target.md) — the general principle of "does the number answer the question"
- [Before Blaming the Model](capability-attribution.md) — the attribution-side discipline (this document's counterpart)
- [Integrity of the Strip Instrument](strip-instrument-integrity.md) — when the measuring instrument itself is broken
- [Reading Green](green-reading.md) — confirming "what ran"
- [Argument Hygiene](argument-hygiene.md) — the battery of checks before writing a conclusion
