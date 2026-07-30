---
name: capability-attribution
description: Before blaming the model — "insufficient model capability" is a terminal classification that shuts down the investigation. The four-layer inspection (structural confounds, outcome facts, context adequacy, verification loop), the provenance of exonerating evidence, stacked defects and upper-bound diagnosis, weak-model runs as mining, and the boundary between specialization and overfitting
tags: [verification, ai-generated-code, attribution, llm]
sources:
  - feedback_all_failures_structural_verify_obligation
  - feedback_context_adequacy_before_model_axis_attribution
  - feedback_context_adequacy_three_legs
  - feedback_cosign_verify_exonerating_evidence_provenance
  - feedback_multi_layer_structural_decomposition
  - feedback_upper_bound_diagnostic_capability_vs_structural
  - feedback_req0_model_verification_before_capability_verdict
  - feedback_weak_model_run_value_is_structural_defect_mining
  - feedback_no_weak_model_overfitting
  - feedback_weak_tier_subtract_or_declare_not_add_signals
---

# Before Blaming the Model — the Discipline of Failure Attribution

## Premise

When a failure shows up in a system that uses an LLM (large language model) as a component — an agent runtime, a skill, a pipeline — the classification "insufficient model capability" carries a special danger: it is a **terminal classification**. The moment you write it down, the investigation ends and the failure goes into the box of things that cannot be fixed. If the true cause was a structural defect on your own side (wiring, schemas, how context is handed over), you have just **buried a fixable bug as an unfixable one**.

The weight of the measurements: a 0/5 wipeout that was about to be ruled "the cognitive limits of a weak model" was in fact **seven layers of structural defects stacked in the runtime**; after every layer was fixed, the score became 4/5. In another campaign, **ten structural defects** had to be removed one after another before a capability verdict was possible. The owner's principle (2026-05-28): "For every scenario that has not passed, you must not allow moving on until you have confirmed it is not a structural problem. Ignoring a failure you went to the trouble of finding is not permitted."

## 1. The four-layer inspection before saying "model axis"

Before attributing a failure to the model (the "model axis" — the classification that places the cause on the model's side), clear four layers in order:

1. **Pipeline structure** (confound = a co-occurring factor that distorts a result): have you ruled out runtime-side defects, misconfiguration, and environment problems?
2. **The facts of the outcome**: did you verify at the event level what actually happened (whether the edit succeeded, exit codes, produced artifacts)?
3. **Context adequacy (input side)**: **did you read the context the model actually received?** There is only one question — "**could a competent model have produced the right behavior from this context?**" If the necessary information is absent from the context (dropped during a handoff, file contents never passed along, ambiguous instructions), that is a **fixable presentation-side bug**, not a limit of the model. Measured: after concluding "all five cases are model-axis," a suspicion surfaced that the handoff had carried no file contents and passed only a **prose description of the plan** — the model may merely have "hallucinated" from an unfair context. The verdict was downgraded to provisional. This layer is **the most easily skipped of the four** (including its full form: reading the instruction text literally and checking whether it is consistent with the shape of the actual artifact).
4. **Verification loop (output side)**: did the model have a **fair opportunity** to notice its own mistakes? The discriminating axis is not "did it use it" but "**was it reachable and discoverable**":
   - The verification means was **hidden** (never shown in listings, results never returned) → structural defect (fixable).
   - It was discoverable but the model **did not use it** (insufficient exploration, assumed success) → a model-leaning verdict is warranted. Adding a compensating mechanism at this point **masks the finding** (even when things then succeed, you can no longer tell whether the failure was model-caused).

## 2. Provenance of exonerating evidence — check who authored the "it did it right once"

"The same model produced a correct output at least once ∴ the mechanism is sound and the failure is model variance" — this exoneration requires a provenance check. Measured: the "correct output" used for the exoneration had in fact been **assembled by a preprocessor**; the model had never once produced the correct shape. The truly decisive fact — "was that option ever **advertised** to the model in the first place?" — lives on the **request side**, not the response side; when it was fetched and counted, the answer was **zero times**. A structural bug.

- Trace any observation used for exoneration back to **who produced it** (the model / a preprocessor / a fixture — a fabricated test object).
- Do not stop at the response logs; capture the **request side** (what was passed to the model, what was advertised to it).
- **The same holds in reverse**: do not conclude "structural" from event-type tallies alone ("permission_denied ×12"). Looking at the individual operations, what got denied were nonexistent paths the model had hallucinated — the tally only says "there were denials," not "what was denied."

## 3. Defects stack — a capability conclusion requires an upper-bound-diagnostic environment

A single failure instance can **hide N layers of defects at once**. Fix one layer and the next one is exposed — so "I fixed one thing and it still fails ∴ the model is weak" is a fallacy.

- **Capability conclusions may only be drawn in an environment with every structural confound removed**: all tools advertised, execution-based verification available, the filesystem correct, the session fresh — if even one confound remains, the verdict "capability, not structure" is void.
- An example discriminator: **an artifact was produced but is wrong** (the runtime worked) and **no artifact was produced** (the runtime is suspect) belong in different boxes.
- An accompanying obligation: **confirm which model actually ran, on the very first request**. A routing default had been silently downgrading to a lightweight model, so the possibility that an experiment "the strong model failed to solve" had actually run on a weak model was checked every single time. This is the LLM edition of "what ran" in [Reading Green](green-reading.md).

## 4. A weak-model run is a mining operation — counting FAILs is wasted cost

The **purpose** of running benchmarks or field trials with a weak model is not to measure the pass rate. A weak model is a **probe**: it stumbles wherever the path, the skill, or the presentation is weak, exposing structural defects that a competent model would paper over.

- Classifying "0/5, all model-axis" and stopping = an anti-pattern that throws away the cost of the run. The owner (2026-06-06): "If all you do is count the FAILs, the cost is wasted. The value is finding the structural faults in the road."
- **Ask of every failure**: "What could the path/skill have done so that even this weak model got **further**?" — even a failure with a wrong answer is mining material (was the right context presented? is anything missing from the scaffolding or guidance? does the failure recur as a pattern?).
- "A genuine model limit" is a **last resort**, not a convenient default. The deliverable of the run is not a pass rate but **a list of surfaced structural defects, each with a candidate fix**.

## 5. "Works even with a weak model" is not "specialized for weak models" — overfitting is forbidden

Owner principle (2026-06-03): "I don't want to specialize for weak models; I just want it to work even with a weak one." When a weak model stumbles on some mechanism, the branching is:

1. First run the context-adequacy inspection of §1 → if the context is inadequate for any model (a general correctness bug), make a **structural fix**. The weak model starting to work is a byproduct. Measured: a failure that looked like "a weak model's limit" was actually a general context bug that was weak for every model.
2. The context is adequate and only the weak model trips → **do not distort the design**. Route the case to an existing, simpler fallback path — "works even when weak" is satisfied by that.
3. Adding heavy retries or special-case branches tailored to a weak model's quirks = **overfitting, forbidden**.
4. A rule of thumb when you want to correct behavior: **adding does not work, removing does** — additive changes (more instructions, fields, signals) kept failing to fire, and only removing contradictions and declaring the mode actually changed behavior.

## Checklist

- [ ] Before writing "model axis": did you clear the four layers in order (structural confounds, outcome facts, context adequacy, verification loop)?
- [ ] Did you read the context the model actually received? Did you answer "could a competent model have acted correctly from this context?"
- [ ] Did you check the author of the exonerating evidence (model / preprocessor / fixture)? Did you look at the request side?
- [ ] Conversely, are you concluding "structural" from tallies or event types alone (did you look inside the individual operations)?
- [ ] Capability conclusion: is the environment free of structural confounds? Did you confirm which model actually ran?
- [ ] Did you mine the weak model's failures (asking of each FAIL "what could the path have done to get it further")?
- [ ] Is that fix a structural fix, a routing to a fallback, or overfitting? Before adding, can you remove instead?

## Sources (measured during reyn development)

Principle: 2026-05-28 owner directive (no moving on until every scenario is structurally verified). Four layers: the C7 0/5 overclaim (2026-06-01, concluded model-axis with layer 3 unverified → downgraded to provisional) + 13236 r2 (skipping leg 3 came within a step of a cognitive misverdict) + #187 (edit-precision: the verification means was discoverable → re-attributed to insufficient exploration). Exoneration provenance: #1133 flask-5014 (nearly exonerated on a preprocessor-built argv; the request side showed 0×, confirming a structural bug) + #183 13236 r2 (misinferred a "fourth structural defect" from event tallies; per-op review revealed hallucinated paths). Stacking: 13236 (a benchmark task ID, not an issue number; 7 layers; 0/5→4/5 after fixing them all). Upper-bound diagnosis: #187 arc (capability floor confirmed after removing 10 defects; the req0 check caught lightweight-model downgrades each time). Mining: 2026-06-06 owner directive (after the 0/5 classification in #183). Overfitting ban: 2026-06-03 owner directive + #1092 PR-B (the "weak model's limit" was actually a general decide-frame bug) + the 2026-06 retrospective (additive changes ineffective, subtractive ones effective).

## Related

- [Structural Blindness of the Verification Environment](environment-blindness.md) — inspecting the environment's capabilities (the neighbor of layer 1 in this document)
- [Reading Green](green-reading.md) — confirming "what ran" (the first-request model check is its LLM edition)
- [Measuring the Wrong Target](measurement-target.md) — the general form of "does the number answer the question"
- [The Discipline of Experiments and Benchmarks](experiment-discipline.md) — the discipline on the side that emits the numbers (this document's counterpart)
- [Claims Without Context](cross-context-claims.md) — "model axis" is itself a claim that loses its context and starts walking on its own
