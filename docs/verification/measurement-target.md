---
name: measurement-target
description: "Measured, but the target was off" is more dangerous than "didn't measure" — numbers come out, so you feel you have verified. Before measuring, write one sentence: does this measurement cover what the decision needs?
tags: [verification, measurement, review]
sources:
  - feedback_measured_but_the_target_was_off_my_four_instances
---

# Measuring the Wrong Target — Producing Numbers Is Not Answering the Question

## The shape of the problem

**You can notice the "didn't measure" error yourself. You cannot notice the "measured, but the target was off" error — numbers come out, so you feel you have verified.**

There is a record of an experienced review agent committing this class of error **four times in one day**. In all four cases the measurement itself was executed correctly and produced numbers. What was off was the gap between **the target of the measurement** and **what the decision needed**:

| What was supposedly measured | The actual mismatch |
|---|---|
| Why a feature was not being implemented | **Guessed without measuring**, and handed the guess to the other party dressed up as an instruction. The other party had already measured |
| Confirming a documentation fix | Checked by **line number**. Another change had shifted the lines |
| What a branch changed | Used a **two-dot diff** (`git diff main branch`). Changes someone else had already landed on main appear as "deletions in this branch." The correct form is the **three-dot** diff (`git diff main...branch`) |
| The number of orphaned index entries | Scanned only by **filename convention** (`*_family.md`) and missed real entries that did not follow the convention (reported 73; the actual count was 31) |

The first step of the countermeasure is simple:

> **Before measuring, write one sentence: "Does this measurement cover what the decision needs?"** If it does not, measuring decides nothing — you spend time and gain only the feeling of having verified.

## Measuring the current state vs. measuring the target state

A more sophisticated form of target mismatch: for a decision of the form "should we choose option A or option B," you end up measuring **how things currently are**.

Real case: for the design decision "which is better — the current format that returns errors as a structured dict, or a new format that returns them as plain text," a measurement of "how does the LLM **currently read** the error dict" was proposed. The reason it was rejected is the essential point:

> That measurement measures the **current state**, but the decision is about the **target state**. "How is it read today" tells you what you would lose by changing, but not which target is better.

The replacement is to **measure the harm**: is the model actually making mistakes because of the current format? That is **decisive whichever way it comes out**, and only then does it function as decision input.

## When measurement "collides" with measurement — first ask: did we measure the same target?

When two people's measurements disagree, there is a question to ask before ruling on which one is right:

> **Did the two of them measure the same target?**

Real case: an implementer requested a correction — "my own measurement contradicts the verdict you gave me." One side had measured the test **on the main branch** (flaky, reproducing only one run in three); the other had measured **on their own branch** (failing deterministically). **Both were right** — that test's recording key depends on the payload, and the branch's change (a system-prompt change) necessarily changes the key. The error was handing over the observation from main **in assertive form, as a claim about the other person's tree**: "it's noise, unrelated to your change."

> **Relay rule: when you hand a measurement result to someone else, always attach "which tree and which commit it was measured on." A measurement you cannot annotate that way must not be handed over.**

Moreover, in this incident the redone re-measurement was **itself** stale (after merging on the remote, local sync was neglected and the measurement ran on a tree four commits behind). Which is to say: you can step on target mismatch twice within the same exchange.

## A coarse predicate fires for every member of the set

The same class shows up in monitoring and automation. Real case: a watcher meant to detect test-suite completion fired repeatedly about the same target. The predicate was "**any process involving that tree's venv**," so every time the linter ran, it fired "suite completed." What was being measured was not "the suite running" but "something in that tree." The fix is to make the predicate 1:1 with the target (track a specific pid plus process name, and fire exactly once when the pid disappears — a vanished pid drops out of the set, so duplicates are structurally impossible).

## If you measured by identifier, cross-check one item by content

Three of the four cases were caused by **using an identifier as a proxy for the target** (line numbers, two-dot vs. three-dot diffs, filename patterns). Identifiers drift when the target moves. If you measured by identifier, **cross-check at least one item by content** (is that sentence really on that line; is that file really of that kind?).

Note that errors appear not only on the low side but also on the **high side** (73 reported vs. 31 actual). Over-reporting feels harmless, but it **spends other people's time**.

## Attribute it to procedure, not judgment

To close, one point that helps a measurement culture take root. When an implementer refused a mechanical rename and discovered a doubly stale state, the reviewer praised it as "good judgment." The implementer's correction:

> I looked at the shape first, too. I thought "a substitution will do," ran the verification function, got False back, and only then learned it was doubly stale. **If I had not measured, I would have written the careless fix as well. It was not that my judgment was good — what made the difference was that a measuring step was interposed.**

Writing "that person has good judgment" does not reproduce. Writing "**when that step is interposed, careless fixes get stopped**" does reproduce. When praising in review, likewise, name the mechanism, not the judgment.

## Checklist

- [ ] One sentence before measuring: does this measurement cover what the decision needs?
- [ ] If the decision is about a target state, are you measuring the harm rather than the current state?
- [ ] When handing a measurement to someone, did you attach the tree and the commit? Do you have grounds for the assertive form?
- [ ] Did you tag your own claim as MEASURED / READ / INFERRED (are you handing over INFERRED dressed up as MEASURED)?
- [ ] Is the predicate 1:1 with the target? If you measured by identifier, did you cross-check one item by content?

## Sources (measured during reyn development)

Four in one day: #3437 (INFERRED dressed up as an instruction), #3433 (line numbers / two-dot diff), the memory-index audit (convention scan, 73→31). Target state and measuring harm: #3411. Two-party collision: #3458/#3459 (PR #3461, main and branch as different targets), plus the simultaneous stale local (#3433 family; local sync after gh merge is covered under Tier 2, git-github, planned). Coarse predicate: suite watcher (2026-07-29). Attribution to procedure: self-reports by the agents responsible for #3429/#3463.

## Related

- [Integrity of the strip instrument](strip-instrument-integrity.md) — the most upstream target mismatch: "which code did you measure?"
- [The discipline of enumeration](enumeration-discipline.md) — a mis-defined surface is the enumeration version of target mismatch
- [Liveness is decided by the producer](liveness-is-producer.md) — the MEASURED / READ / INFERRED distinction
