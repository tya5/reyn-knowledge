---
name: argument-hygiene
description: Seven checks to run just before a conclusion — "cannot" can never be concluded from symptoms, absence cannot tell forgotten from decided, extrapolation doesn't die in review, a conceded argument returns under a new name, only the reason gives the class, "not urgent" and "not decided" are different axes, a plausible causal story is not evidence
tags: [verification, reasoning, review]
sources:
  - feedback_cannot_claims_require_tracing_the_mechanism
  - feedback_absence_in_code_cannot_tell_forgotten_from_decided
  - feedback_extrapolation_dies_on_use_not_on_review
  - feedback_a_conceded_argument_can_return_under_a_new_name
  - feedback_form_gives_the_instance_only_the_reason_gives_the_class
  - feedback_not_urgent_and_not_decided_are_different_axes
  - feedback_diff_against_main_before_blaming_the_change
  - feedback_pre_conclusion_observation_checklist
---

# Argument Hygiene — Seven Checks to Run Before Writing a Conclusion

Most verification failures are planted before any measurement happens — **at the stage where the argument is assembled**. This document is seven checks to run just before you write a conclusion. Every one of them comes from a measured incident. All seven share a common, simple foundation, worth applying the moment you reach for a high-confidence word like "conclusion," "100%," "every one," or "consistently": **can you list, one by one, the specific observations that support this claim? Is each observation primary data (a log, a number, a grep result), or an inference from some other observation? Did you look for an observation that would refute it? If you are about to write "N of N" or "100%," did you actually look at all N yourself?**

## Check 1 — "X cannot do Y" can never be concluded from symptoms

> **"Cannot" is a claim about a mechanism. But it gets built from symptoms. Output tells you what happened; it never tells you what the thing is capable of.**

Real case: seeing a library return the string `'None'` for a corrupt PDF, the claim "this library **cannot** distinguish success from failure, so a wrapper can't either" was made three times — and was wrong all three times. Measured: the underlying parser **was raising an exception** → the library caught it and even **had** a dedicated exception class → a single missing None check in the final converter was returning `str(None)` as a "success". **The signal existed at every layer; it died at exactly one final spot** — and the only thing being watched was the last box.

- Before writing "cannot", "there is no way to", or "unsupported": **can you name the layer you observed and the layer directly beneath it?** If you cannot name the layer below, you are inferring capability from observed symptoms.
- **If behavior splits on the same input class (one case raises, another returns `'None'`), that is not evidence of "cannot distinguish" — it is a marker that there are two code paths.**
- Variant 1 (from declarations): "this path's gate is deny" does not mean "**unreachable from any path**" — in a system with multiple dispatch paths, **do not conclude unreachability until you have counted every path**.
- Variant 2 (from textual absence): an argument name not appearing in a grep of the caller's file is not evidence it isn't passed — the argument can be passed **at call time, inside an imported shared helper**. Opening one level down breaks the claim.
- **"Cannot" prose is self-sealing**: the lie "it is protected" can be falsified by the absence of a deny, but the lie "it cannot" **stays a lie because nobody ever tries**.
- If the owner or a peer says "it actually works", that is primary data. **Look for the hole in your own trace first.** Reading the library's API surface (options, exception classes, result fields) takes thirty seconds and can flip the conclusion.

## Check 2 — Absence in code cannot tell "forgotten" from "decided"

> **Code says what happens; it cannot say whether that was chosen.** Under grep, "absent because forgotten" and "absent by decision" look identical.

Real case: a directory was found to have no TTL, no size cap, and no cleanup, and filing it as an "unimplemented gap" was recommended. But the docstring read: "**disk usage grows without bound — documented as a known operational caveat**", "**reserved for Phase 2: when measured pressure appears (not hypothetically)**" — in other words, **a deliberate, accepted decision, with its revisit trigger already pinned down**. Filing it would have created a ticket for a decided non-problem and pulled the work forward without meeting the condition the doc itself had set.

- When you find a "missing mechanism", before removing, adding, or filing anything, **read whether the absence is recorded in prose as a decision** (docstrings saying "Phase N reservation" / "out of scope" / explicit delegation to operations, design records, issue threads).
- If you find a decision, also check **whether its trigger condition is met**. Overriding a conditional deferral while its condition is unmet is overwriting someone else's decision.

## Check 3 — Extrapolation doesn't die in review; it dies in the process that uses it

Real case: the claim "40 files contain this pattern − 7 handled = **the remaining 33 are broken**" **passed review by all three people** — implementer, lead, and independent verifier. What killed it was not review but **work**: the person migrating those files had no choice but to open them, and `grep -c 'import <target>'` = **0** (the population had been defined wrong — containing the pattern ≠ depending on the target). A check that takes two greps never ran until the point of use.

> **Review tests a claim's plausibility. Use forcibly opens the claim's referent — the thing it points at. A plausible extrapolation is strong against review and weak against use.**

- Before propagating a numeric claim ("M of N are broken"), take **the smallest step that uses it** — opening a single member of the population is enough to break it.
- **Dramatization is amplification, not inspection** (the claim was forwarded with "you are underestimating this finding" attached — that was the amplification).
- The correct form when you cannot claim a number: "The 6 directly inspected match. The remaining 27 are uninspected. **Therefore I claim no number.**"

## Check 4 — A conceded argument returns under a new name

Real case: the claim "inline comments appear in the same diff, so staleness is visible" was refuted by measurement, accepted, and retracted. **Two turns later**, a **new condition** was submitted: "(the other format) will decay over time, so we should add more inline" — as was pointed out, "decays" is true of inline and of the other format alike; **the argument that lost on drift resistance had been resubmitted relabeled as decay resistance**.

- Self-check whenever you add a condition or requirement: **state in one word the property this condition is protecting, and check whether it is the same property as a claim you conceded earlier in this thread**.
- **"True in general, but inert for the comparison" is the classic disguise.** A property true of both options cannot justify choosing either one. What supports a comparison is **a property only one side has**.
- Check whether the answer already exists as a **detection condition** — before adding a condition to "prevent" decay, a requirement to "make it detectable" already existed.

## Check 5 — Form gives you a candidate. Only the reason gives you the class.

> **A fix is structural ⟺ you can say why the defect exists, and what you removed is that reason. If you removed only the form, it is an instance fix.**

The operational test is "**can you name instance N+1?**" — you can name N+1 only when you know why N happened. And there are two kinds of "form wearing the face of a reason":

1. **A reason at the wrong altitude is still form.** "The reason is that `name == "X"` is in the condition" is a paraphrase of the code line. The real reason ("audit coverage was tied to the caller's choice of surface, not to the event") **survives a rewrite of the code**. Distrust any "reason" that names a specific identifier or line number — suspect form in disguise.
2. **An unfalsifiable reason closes nothing.** "Because nobody thought about it" is an impotent sentence wearing a reason's face. **A usable reason predicts where else the class appears** — a reason that cannot predict the next instance is not a reason.

- Before fixing, write one sentence: "why does this defect exist?" If you can't, you are still seeing only the instance. "I cannot state the reason" is a legitimate report (building a gate without the reason freezes the target in its open state).
- When tempted to reuse a mechanism, ask not "is the form the same?" but "**does the reason transfer?**". Recall itself is not forbidden — only adoption without verification is. And when rejecting, make explicit that you are rejecting the failure to transfer, not the having-thought-of-it (otherwise the candidates that ought to be verified stop surfacing at all).

## Check 6 — "Not urgent" and "not decided" are different axes

> **"Not urgent" answers "when"; "not decided" answers "whether". Mix them, and something undecided quietly acquires the status of "decided, just deferred".**

Why this is fatal: **nobody reopens what looks decided**. If it is "not decided", someone will come and decide it; "decided but not urgent" makes neglect look legitimate. In the real case, this blend dropped an arc's structural work into the gap between "closed issues" and "decision tickets" — filed nowhere.

- The remedy is always a single clause: "**Decided: we do X. Not urgent.**" or "**Undecided: blocked pending the ___ call.**" Only the blend is invalid; either pure form is legitimate.
- When you read someone else's "not urgent", ask which axis it is. If they cannot answer, it is the blend.
- The moment you close a unit of work (an arc) is the most reliable point at which this distinction can be forced — settle every leftover as either "filed" or "explicitly discarded", and do not accept "in the next arc".

## Check 7 — A plausible causal story is not, by itself, evidence

> **An explanation of the mechanism that fits a symptom "plausibly" is not evidence that the mechanism is what actually happened.** The smoother an explanation sounds, the more tempting it is to skip verifying it — but smoothness is not a substitute for verification.

Real case: a test failed while verifying a change. A causal story assembled itself immediately ("this change made the base background unreachable to the theming mechanism, so some alpha-derived colors stop resolving") — and the symptom fit that story **perfectly** (one color resolved, the other stayed at its default). Reporting it as "a design collision between two of the owner's past decisions" and asking for arbitration over a trade-off that did not exist was one step away. What stopped it was actually running `git diff origin/main -- <file> | grep '^-'` and reading the deletions: the implementation was an index-range text removal (a revert that replaces everything between two markers), and it had **swept up an unrelated CSS rule and deleted it along the way**. The failing test was failing not because of the change under test, but because **that rule had simply been deleted**.

- **Before explaining a failure as caused by the change under test, look at what is actually in the diff**: run `git diff origin/main --stat`, then read the deletions (`grep '^-'`). If anything besides your intended change is present, suspect that first.
- This check is cheap, and it prevents an entire class of self-inflicted failures that would otherwise get explained away with a good theory. A story's smoothness is not a reason to move faster — it is **the signal to verify the premise**.
- When undoing an experiment, prefer `git checkout -- <file>` or a WIP commit over an index-range text removal — a surgical edit can delete unrelated things sitting between its markers.

## Checklist

- [ ] Before writing "cannot": can you name the observed layer and the one beneath it? Did you count every dispatch path?
- [ ] Before acting on an absence: is the absence recorded in prose as a decision? Is its trigger condition met?
- [ ] Before propagating a numeric claim: did you actually open one member of the population?
- [ ] Before adding a condition: is it a relabeling of a claim you conceded? Is your basis a property true of both options?
- [ ] Before fixing: can you write "why does this exist?" in one sentence? Does that reason predict N+1?
- [ ] Before writing "not urgent": did you attach which axis it is — decided or undecided?
- [ ] Before blaming a failure on the change under test: did you actually read the diff's deletions? Are you concluding from a plausible story alone?

## Sources (measured during reyn development)

Check 1: #3010/#3011 (the three consecutive false "cannot" claims about markitdown), #3036 (false unreachability from a gate declaration — the fifth occurrence), #3340 (false "not wired" from textual absence — the sixth). Check 2: #3334 (a deliberate Phase 2 reservation nearly misclassified as a "gap"). Check 3: #3024 ("33" passed three reviewers, killed by two greps during migration work). Check 4: #3082 (drift resistance resubmitted as decay resistance). Check 5: the why-chain arc of 2026-07-29 (the conclusion flipped four times in one day), #3457/#3458 (the correct rejection of a mechanism reuse whose reason did not transfer). Check 6: #3411/#3447 (structural work lost in the gap the blend created). Check 7: the #3326 test incident (an index-range text removal swept up an unrelated CSS rule; a plausible causal story pointed at the wrong mechanism). The common foundation of all seven checks (observation, inference, refutation, and full-count verification) is codified in reyn's operating conventions (CLAUDE.md) as a five-question checklist referenced directly from that document.

## Related

- [Census vs structure](census-vs-structure.md) — "one observation is a census with N=1"; the foundation for Checks 1 and 3
- [Measuring the wrong target](measurement-target.md) — the MEASURED / READ / INFERRED labels
- [Liveness is decided by the producer](liveness-is-producer.md) — "record of intent vs record of behavior" (Check 2 is its absence variant)
- [Claims without context](cross-context-claims.md) — labels for arguments that cross role boundaries
- [Fix-class review](fix-class-review.md) — Check 5 applied on the review side
