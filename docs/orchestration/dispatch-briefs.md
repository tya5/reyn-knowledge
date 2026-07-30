---
name: dispatch-briefs
description: The discipline of dispatch briefs — name the invariant instead of saying "copy X" (three tiers of safety), make documentation updates a required item for user-visible changes, write verification duties as concrete actions on content rather than labels, and write GO as a decision rather than a recommendation
tags: [orchestration, multi-agent, dispatch]
sources:
  - feedback_mirror_brief_must_name_the_invariant
  - feedback_dispatch_brief_must_require_doc_for_userfacing
  - feedback_sub_agent_dispatch_assertion_shape_verify
  - feedback_sub_agent_primary_evidence_cross_reference
  - feedback_go_decision_not_recommendation_for_planfirst_peer
  - feedback_missing_doc_for_new_feature_is_undetectable_by_stale_checks
---

# The Discipline of Dispatch Briefs — Agents Do Only What Is Written, at the Altitude It Is Written

## Premise

A dispatch brief — the instruction text you hand over when delegating work to an agent — is close to a program. The receiver executes what is written and does not execute what is not written. More important still, **the "altitude" (level of abstraction) at which something is written determines the range over which it is protected.** This document is a record of measured incidents in which the wording of the brief itself created the accident, and the patterns that prevent them. Where [Operating with Separate Coder / Tester / Reviewer Agents](../verification/roles.md) gives the full picture of each role's obligations, this document is the dispatcher-side casebook.

## 1. "Copy X" is dangerous — name the invariant

First, the shape of the problem, using a bounded queue (a waiting line with a cap on how many items it can hold):

```python
# A: on overflow, drop the oldest item
if queue.full():
    queue.get_nowait()   # discard the oldest item to make room
queue.put_nowait(item)

# B: on overflow, drop the newest item
if not queue.full():
    queue.put_nowait(item)   # if full, give up on the new item instead
```

At the altitude of "bounded queue," A and B are **identical**. The difference appears only one level down, in "what gets dropped" — and that is the level where accidents happen. Where you want subscribers to receive the latest (a cap that lets a lagging reader catch up), A is correct. But use A where the queue holds **the work that is supposed to run next**, and it silently drops the job that was about to run. Both are "legitimate bounded queues," so things break **while the tests stay green**.

Measured case: a brief went out saying "copy the shape of the bounded queue in `bus.py`; do not invent a new design." `bus.py` was the wrong model (the issue ticket itself named the correct model, yet the dispatcher, having read the ticket, pointed at the wrong file). Copied as instructed, the result would have been "working, green, plausible" code that silently dropped the next job to run. It came to light because the implementing agent refused to copy (as described below, the brief happened to contain one defensive step — that is what made the refusal possible).

A brief has three tiers of safety:

| Shape of the brief | Safety |
|---|---|
| **Name the invariant (a property that must always hold)**: "Cap the backlog, but never drop the one item that runs next" | **Best** — survives even a wrong model filename |
| **Name a model + explicitly delegate identifying the invariant**: "Read X first, and you name what it preserves; decide whether to copy only after that" | **Safe** — the model guides exploration, and the delegation makes "refusing to copy" an expected move (not insubordination) |
| **Model only**: "Copy X" | **Dangerous** — the subject of this section |

What actually prevented the accident was not skepticism but a single step included in the brief itself: "First read `bus.py` and state **what** it limits (count? concurrency? rate?)" — that question dragged "what gets dropped" into view. Therefore: **always place "state what this model preserves" as a step before "copy."** The naming step is the defense; the copying step is not.

One more failure of the same shape: a design that said "there is an existing command that returns the list, so just call it" — true as far as "registered and callable." But **its output was a single string**, and the receiver's iteration could only consume a list. Do not stop checking at the altitude of "it exists, it can be called"; go one level down to "**can downstream consume the shape of the output?**" **Reachable ≠ usable.**

## 2. For user-visible changes, write "update the documentation" as a required item

Measured case: after running roughly 15 PRs (change proposals) on implementation-centric briefs, new configuration keys, hook points, and commands **accumulated undocumented** in the feature list and the configuration reference. CI (automated tests) kept passing — documentation is not a CI pass condition. It surfaced only when the owner said "look into documentation gaps." A decisive controlled contrast exists from the same period: the group of changes whose briefs did include documentation updates **had documentation**. Put it in and it gets done; leave it out and it does not — it is that simple.

- **In the brief**: for user-visible changes (new configuration keys, hook points, commands, event types), state the corresponding documentation update explicitly as a required item.
- **On the review side**: flag any "user-visible change" whose PR file list contains no docs/ (an implementation-only PR is a sign of documentation debt).
- **When fanning out many PRs in parallel**: a two-stage setup also works — minimal documentation in each PR, plus asking a docs owner for an exhaustive gap check at each milestone ([Completeness Sweeps in Practice](../verification/completeness-sweeps.md)).

**Why a "documentation staleness check" alone is not enough.** A mechanism that detects documentation rot (an audit that hunts for a mismatch between what's written and what's implemented) is, in principle, **shaped like falsification** — it checks "has this claim become a lie?" But a brand-new feature with **no documentation at all** has no existing claim to contradict. No grep, no reviewer keyword sweep, and no CI drift gate can catch it, because **absence never shows up in a diff**.

Measured case: an entire conversation-pane feature arc (five merged PRs) added not a single line to the feature list or the reference docs. The author's own dispatch-brief checklist only asked "does this make some doc a lie?", and the honest answer was no every time — this was not a detection failure but **a case where the question itself was shaped to be blind to absence**. It surfaced through a docs owner's cross-cutting audit.

- **Always ask "which doc should this appear in" as a separate question from "does this make a doc a lie"**: for a brief that adds anything a user can see or invoke (a keybinding, a screen, a visible affordance, a command, a configuration key), raise this second question as its own required item.
- **Mind the write order**: a feature-list row is normally a link into a reference section that already exists, so **write the section first and add the list row second**. Doing it in reverse creates a dangling reference — a row with nowhere to link.
- **Inventory a multi-PR feature arc again at its end, not just per PR**: each individual PR can look complete in the sense of "not making any doc a lie," while the feature list as a whole is missing **every row for that entire feature**. The completeness of a single item and the completeness of the ledger (registry) are different checks.

## 3. Write verification duties as concrete actions — checking labels and checking content are different things

A request to "verify that ..." gets satisfied by the **cheapest check** the receiver can find. Two measured cases (the same review was split across several subagents — child agents spawned for auxiliary work — and these happened on two of them, the A and D of the Sources):

- (a) "Verify there are no violations of the test-grade notation" → the subagent looked only at the **labels** in docstrings (the descriptive text attached to a function), never at the content of the asserts (a test's check statements), and reported "no violations" → CI detected 6. Check statements like `assert len(result) == 3`, which pin down the output format, had been left in.
- (b) "Verify which domain this function belongs to" → the subagent **inferred from the test's directory name** (under tests/chat/, therefore the chat domain), did not read the content, and reported "no impact on other domains" → it was in fact touching another domain's data, and the change had to be reverted wholesale.

It is easy to call both of these receiver laziness, but the side that can prevent them is the dispatcher. **Write verification duties as concrete actions against the content, not the surface (labels, names, placement)**:

```
## Verification duties (a template to paste into a brief)
1. Do not stop at checking labels (docstring notation, etc.)
2. Enumerate the check statements themselves, directly, with grep:
   grep -rn "assert len(" tests/
   grep -rn 'assert.*==.*"' tests/     # comparisons against fixed strings
3. Read the target function's body directly and follow the chain of imports and calls
4. Surface inference of the form "the directory is X-ish, so it is the X domain" is forbidden
5. Include the primary evidence you used (grep output, names of the functions you read) in your report
```

This is the dispatcher-side counterpart of [Audits Must Match Content](../verification/audit-content-match.md) (that something exists and that its content matches are different things). Rather than counting on the receiver's attentiveness, the brief spells out "no surface inference" and "attach primary evidence" in writing.

## 4. Write GO as a "decision" — recommendations get waited on

Suppose a counterpart working plan-first (a policy of not starting implementation until approval is given) sends a question asking for your ruling. If you reply — meaning it as final — "(2) is KEEP (leave as is), **recommended**," the counterpart **correctly waits**: a "recommendation" is a leaning (the decision is still theirs to make), not a settled ruling, so the more disciplined the counterpart, the less it will start.

Measured case: the silence created by this ambiguity was misdiagnosed by the dispatcher as a stall, and the status-check ping — the message finally rewritten in assertive language — became the de facto permission to start. **The waiting side's discipline was correct; the ambiguity was on the dispatching side.**

- When issuing a final ruling, say in words that it is a decision: "**GO — settled on X** (a decision, not a recommendation)" (GO = the settled signal of permission to start).
- Reserve "recommended / on balance" for when you genuinely want the counterpart to decide.
- If a counterpart you believe you gave a GO falls silent, **reread your own outgoing message** before suspecting a stall (was the phrasing soft?). The counterpart of [Detecting Stalled Agents](stall-detection.md) §5.

## Checklist

- [ ] About to write "copy X": did you name the invariant to preserve? At minimum, did you delegate with "you name it"?
- [ ] Did you place "state what this model preserves" before the copying step?
- [ ] Did you take the check down to whether the model's output shape is usable downstream (reachable ≠ usable)?
- [ ] For a user-visible change, did the brief include the documentation update as a required item?
- [ ] Did the verification request spell out "content, not labels," "no surface inference," and "attach primary evidence"?
- [ ] Did you word a final ruling to a plan-first counterpart as a "decision"? When they fell silent, did you reread your own message first?

## Sources (measured during reyn development)

Invariant: #2620 (2026-07-16, the bus.py misnaming — the analysis by the implementing agent that refused to copy is the canonical form of this rule) + #3045 (2026-07-17, demonstration of the "delegate" tier: shown two candidate models, the agent correctly refused with "neither — a synthesis") + the same night's list-retrieval design (reachable ≠ usable). Documentation as a required item: 2026-07-05 (about 15 PRs on implementation-centric briefs accumulated holes in the feature list and reference, surfaced by an owner remark; contrasted with the group of changes whose briefs included documentation updates) + #3483/#3487/#3488/#3489/#3491 (a five-PR conversation-pane feature arc that shipped complete while leaving no trace in the feature list or reference docs, surfaced by a docs owner's cross-cutting audit). Verification duties: the PR-N3 review (subagent A: labels only, "no violations" → CI detected 6, rewritten in commit a26c3e9c / subagent D: surface inference from directories → fully reverted in commit 9516221f). GO: #2296 ("KEEP recommended" read as a leaning; the counterpart correctly waited, and the dispatcher misdiagnosed a stall).

## Related

- [Operating with Separate Coder / Tester / Reviewer Agents](../verification/roles.md) — the full picture of each role's obligations (this document is its dispatcher-side casebook)
- [Audits Must Match Content](../verification/audit-content-match.md) — the general form of label/content divergence
- [Detecting Stalled Agents](stall-detection.md) — the false stalls that soft GOs create
- [Completeness Sweeps in Practice](../verification/completeness-sweeps.md) — techniques for enumerating a class and closing it exhaustively (applicable to documentation checks as well)
