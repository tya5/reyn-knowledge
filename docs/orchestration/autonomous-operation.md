---
name: autonomous-operation
description: The discipline of autonomous operation — no blocking questions; reject "tired, next session" unless it comes with a number (judge by actual context usage; compaction continues automatically); review every item before standby; honest idling instead of fabricated work; reread the original goal at every milestone; don't stop right after dispatching
tags: [orchestration, multi-agent, autonomy]
sources:
  - feedback_no_blocking_askuser_in_autonomous_mode
  - feedback_no_fatigue_reflex_defer
  - feedback_no_fatigue_defer_harness_autocontinues
  - feedback_review_menu_before_standby
  - feedback_empty_settled_bucket_standby
  - feedback_milestone_goal_recheck_against_ticket
  - feedback_parallel_work_after_dispatch
---

# The Discipline of Autonomous Operation — the Art of Not Stopping, and the License to Stop

## Premise

Failures of autonomous operation (autonomous mode = an agent advancing work without step-by-step human approval) split into two symmetric kinds: **stopping where you should not** (blocking questions, the fatigue reflex, halting right after dispatching) and **skipping the checks where you should stop** (drifting from the goal, leaving loose ends unattended). This document records the failures measured on both sides, and the discipline that answers them. Throughout, a "turn" means one unit of agent response — the batch of work done before the next input arrives.

## 1. While running autonomously, never ask a blocking question

Measured: during overnight autonomous operation while the owner slept, the agent posed a three-option **synchronous question** (a question mechanism that halts all processing until a human answers) → the owner was asleep → **the entire night of autonomous runtime stopped**, resuming only when the owner woke up. The irony is that a sufficient instruction — "proceed with anything that doesn't need my intervention" — had already been given, and all the material needed to decide was on hand. The question was unnecessary.

- When in doubt, classify it yourself: "does this need the owner's intervention?"
  - **No intervention needed → proceed**: reviews, merges, documentation updates, correcting a premise, implementation whose scope is already settled.
  - **Intervention needed → non-blocking note**: prioritizing new features; design decisions that change the product's direction. Leave them on an issue (task ticket) or in a message as "options awaiting a decision," and **without stopping**, move on to other work that needs no intervention.
- "Is that all?" / "There's more, right?" are **signals to continue**, not cues to ask which option to pick.
- A synchronous question is permitted only during hours the owner is actually conversing, and only for a genuine either-or.

## 2. Don't say "tired — next session" without a number — compaction continues automatically

Midway through a long autonomous run, a deferral proposal surfaces reflexively — "this session is long, so let's do the rest (especially the hard part) in a fresh context" — from yourself, and from the party you delegated to. This is wrong on two counts:

1. **Technically**: when the context (conversation history) grows long, the execution environment automatically summarizes it and **keeps working** (context compaction — automatic summarization of a long conversation history). "Waiting for a fresh context" is a constraint that does not exist; stopping buys nothing — it just stops. (That said, on some model tiers and environments, sessions have been measured to stop at the compaction boundary without auto-resuming, or to ride to 100% usage and become unable to continue ([Detecting Stalled Agents](stall-detection.md) §3). That is exactly why the threshold management below is still needed — "it's automatic" does not mean "leave it alone.")
2. **As judgment**: "fatigue," "quality," "the session is long" are not numbers. The only admissible input is the **actual context usage rate**. Measured: when "it's major surgery, so a fresh context" was proposed, actual usage was **about 30% of the 1M (one million) token context limit** (a token is the unit of context volume). The owner pushed back twice ("you're giving up too early"; "show evidence, or prevent the recurrence").

The rule cuts **both ways**:

- **Deferring below roughly 60–70% is overcaution** — keep going. If the other party's "checkpoint (a pause at a clean stopping point) / fresh context / hard part with a fresh head" carries **no number, first make them produce the actual usage rate (or check it yourself); a deferral that remains numberless does not pass — instruct them to continue**. Measured: the lead (the coordinator who oversees multiple agent sessions) who had recorded this very lesson endorsed a numberless deferral themselves and was called out by the owner (recording a lesson and applying it are different operations).
- **Riding all the way to 100% is non-management** — measured: a session reached 100% usage, became unable to continue, and required manual human recovery. Near the threshold, add your own summary to the history or do a clean handoff.

Caution is expressed through verification (tests, frequent pushes), not through stopping. As long as work is in flight, the only legitimate reason to stop mid-way is an **external hard blocker** — something that truly cannot advance without a human decision. (Honest standby when the work has genuinely run out is §4 — that is completion, not stopping.)

## 3. Before standby, review every item on the list

Before entering standby (doing nothing until the next event), go through the current work list **item by item** and confirm each one is: (a) completed; (b) legitimately stopped, waiting on someone else; or (c) explicitly recorded as deferred. **Do not enter standby with items left hanging** — a delegate you asked who hasn't produced a PR (pull request — a proposed change), a PR awaiting review, a PR awaiting merge, an issue whose closing conditions are all met, a milestone that should be reported. The owner hands over the list and goes to bed. Whether that list keeps working as a live TODO is the responsibility of whoever received it.

## 4. Distinguish honest idling from fabricated work

If the review shows there truly is no settled, mechanical work left, **honest idling is the correct answer**. Do not "manufacture" work by grepping the code for TODO comments ("do this later" notes) — a TODO existing and a fix being a settled decision are different things, and it wastes expensive compute. The procedure: (1) proactively ask the lead for the next priority → (2) check the actual remaining work (open issues/PRs) → (3) if settled work exists, autonomously run the single highest-value item; if none exists, report that conclusion and stand by.

This does not contradict the "don't stop" of §1 and §6. Those describe the duty to keep going **when settled work exists**; this covers the case where, after the checking procedure, there **really is none**. Fabrication is the anti-pattern.

## 5. At every milestone, reread the original goal

A long autonomous run thins out the memory of the goal (all the more after context compaction). The countermeasure is cheap and concrete: at each milestone (between work items, before a merge, when resuming interrupted work), spend 30 seconds rereading **the originals — the ticket body and the design documents — not the in-context summary**, and confirm the current work still matches the original intent. Check not only the work content but also the **tracking state** (issue open/closed, scope, splits).

Measured: a reread performed right after this instruction paid off immediately — it uncovered that the parent issue had been **auto-closed early** when an intermediate child PR merged (with the core goal still unfinished), and the issue was reopened. Side lesson: child PRs hanging off a parent issue should say "part of #X" rather than use a closing keyword (notation that auto-closes an issue on merge) — only the final PR closes the parent ([Closing Keywords Fire on Surface Text](../git-github/closing-keywords.md)).

## 6. Don't stop right after dispatching

Dispatching a request into the background and ending the turn with nothing but a one-line acknowledgment is an anti-pattern (measured: it happened four times in a row, and the rule was established by the owner's "we were in autonomous mode — why are you stopped?"; since then it has been checked mechanically by a stop hook, a process that runs automatically at the end of each turn). In the turn right after dispatching, complete **at least two productive pieces of parallel work** before closing — vetting the next candidate, inspecting existing artifacts, updating records, and so on.

Just before closing the turn, ask yourself: (1) is the background task running? (2) is there productive follow-up work left? (3) is there an inspection that could run in parallel?

## Checklist

- [ ] About to ask a synchronous question while autonomous: did you classify intervention-needed yourself? Would a non-blocking note suffice?
- [ ] Before writing "defer to next time": did you check the actual context-usage number?
- [ ] The other party proposed a deferral: if it carried no number, did you reject it and instruct them to continue?
- [ ] Usage crossed the threshold (about 60–70%): did you proactively add a summary or hand off?
- [ ] Before standby: did you classify the work list item by item into completed / legitimately stopped / explicitly deferred?
- [ ] When no settled work exists: did you idle honestly after the checking procedure (ask for priorities → check remaining work), without fabricating?
- [ ] At a milestone: did you reread the original ticket? Did you also check the tracking state (issue open/closed, scope)?
- [ ] Right after dispatching: did you make sure not to end the turn on a one-line acknowledgment?

## Sources (measured during reyn development)

Blocking question: night of 2026-06-13/14 (a three-option synchronous question halted overnight autonomy entirely; owner: "you were stopped because you used ask-user"). Fatigue reflex: 2026-05-30 (actual usage ~30% of 1M at the moment of the deferral proposal; owner pushed back twice) + 2026-06-05 ("compaction runs automatically, so no intervention is needed — if you judge compaction is near, just add a summary") + 2026-06-28 #2259 PR-2b (the very person who had recorded this lesson endorsed a numberless deferral → owner called it out; 26 minutes of self-imposed stop). The unmanaged side: 2026-05-31 (usage reached 100%, unable to continue, manual recovery). Pre-standby review: 2026-07-18 (direct instruction before the owner went to bed). Honest idling: 2026-06-14, after the #1599 merge ("trawling TODOs to pose as work doesn't meet the 'settled' requirement; not manufacturing it was good discipline"). Goal rereading: 2026-06-28 (a reread right after the instruction found parent issue #2259 closed early — the merge of child PR #2264 had closed it). Right after dispatching: 2026-05-30 (four in a row; mechanized via the stop hook).

## Related

- [Detecting Stalled Agents](stall-detection.md) — how to spot others' stalls (this document is about not stalling yourself)
- [The Discipline of Dispatch Briefs](dispatch-briefs.md) — writing requests that don't leave the other party stopped
- [Issues Decay](../git-github/issue-lifecycle.md) — verifying and maintaining tracking state
- [Closing Keywords Fire on Surface Text](../git-github/closing-keywords.md) — the mechanism behind early parent-issue closes
