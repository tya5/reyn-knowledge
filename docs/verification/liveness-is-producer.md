---
name: liveness-is-producer
description: Judge liveness by chasing the producer (the side that writes, the side that emits) — neither the existence of readers nor "declarations" in schemas, docs, constants, and names counts as evidence of life. Those are records of "what someone intended", not records of "what the system did"
tags: [verification, dead-code, refactoring]
sources:
  - feedback_liveness_is_producer_not_reader
---

# Liveness Is Decided by the Producer — Records of Intent vs Records of Behavior

## The shape of the problem

After a feature has been deleted or renamed, the codebase is left with piles of remnants bearing the feature's name. Code that walks directories, `elif kind == "X":` branches, deserializers, config keys. Are these alive, or dead?

The intuitive judgment tends to be: "**there is reader code**, so it's alive". That is the mistake.

Real case: in a post-deletion sweep, roughly 551 remnants were classified "structurally used, KEEP". Code that reads the event directory, code that walks the state directory, branches that handle `kind == "skill_done"` — all of it was code that gets called. One line from the owner — "**Why do you think it's alive? The feature doesn't exist**" — prompted a re-examination on the producer side: **zero code constructs and writes** that directory path. **Zero code emits** that event kind. All of it was dead.

> **The criterion for liveness is "does a live producer still write/emit that data?", not "does a reader exist?"**

When a feature is deleted, the producers vanish with it, but **the readers often survive** (walkers, branches, deserializers). Code that reads data that will never be produced again is **dead code**, not something to "keep structurally". Keeping it means (a) actual corpses remain, and (b) the old names the deletion was supposed to erase get preserved.

## How to check

- **Grep for the producer.** Who **writes** that file/path, who **emits** that event/kind, who **constructs** that dict — not the readers.
- "The reader is called from a live endpoint" is insufficient — the endpoint may simply be reading legacy data that is no longer produced.
- On-disk path fragments: look for **the place that constructs that directory at runtime**, not the example in a docstring.
- Event kinds: `elif kind == "X"` is dead if nothing emits `"X"`.
- Default suspicion: **after a feature deletion, readers/consumers bearing its name are "dead until a live producer is shown"** — not the reverse.

## DELETE or RENAME

Once liveness is settled, the next step is treatment. The criterion is sharply asymmetric:

- **Dead** (either end is missing / reading it has no effect on behavior) → **delete the whole mechanism** (code + tests + names). **Never rename dead code** — giving a corpse a new name is disguised technical debt, and because it never turns CI RED, it outlives a plain leftover in a worse way.
- **Alive** (produced, consumed, affects current behavior) + only the name is stale → **rename** (name only).
- **If uncertain, lean toward deletion.** Mistakenly deleting live code gets reported by CI immediately, in RED (recoverable). Mistakenly renaming dead code stays hidden indefinitely.

## The unified form — the family collapses to one question

This family of errors (judging by readers, judging by declarations, judging by names) all reduces to a single question:

> **"Is this a record of what someone intended, or a record of what the system did?"**

Schema declarations, statements in documentation, tuples of kind constants, the description attached to a declaration, and **the identifier (the name) itself** — all of these are the former. **Only producers, execution, and measurement record behavior.** If you derive a claim about behavior ("this event gets logged", "this mechanism is working") from a record of intent, the hole is real no matter how authoritative the artifact looks. The bridging check is mechanical: **is there a producer?**

### Declarations are the more dangerous disguise

A stronger disguise than "there's a reader, so it's alive" is "**there's a declaration, so it's alive**" — and it appeared in three substrates simultaneously in a single night:

| Substrate | Why it looked alive | Measured reality |
|---|---|---|
| schema | Listed in the audit-requirements table with required fields | **Zero emitters** |
| doc | Listed among the kind examples in public documentation | **Renamed; doesn't exist** |
| event log | Kind declared in the constants tuple, and the doc explicitly says "logged" | **Zero write paths in src or in tests** |

The third is the heaviest. The documentation **promised recovery behavior** — "replaying restores subscription state" — but **the thing to be replayed is never written**, so it cannot happen even in principle. This is not stale text; it is a **false guarantee** — an operator who believes it will read a "nothing is there" state as normal.

### Why names are the most dangerous member of the family

A description comes with a "sentence" that can be scrutinized (there is room to notice "this is a statement of intent"). **A name delivers the suggestion without any sentence.** Real case: from the name `Cancelled`, an association with the standard library's `CancelledError` nearly led to inferring "there is a convention violation" — measurement showed it was a domain exception of its own, and no violation existed.

### A description is a specification, not a report

There is a real case of reading the description attached to a declaration ("represents that a tool **was loaded** from search results") and concluding "the system does that". Measurement showed the opposite: the processing happens on the external server side, and the local system has zero corresponding branches. The clincher: **the commit message that added the declaration itself stated "no changes to the main code"** — the declaration had been deliberately created without ever writing the emitter.

> **A declaration's description states "what the declaration is for", not "what the system is doing". Without a producer, it is the specification of something unimplemented.**

### Mistaking the level of abstraction

There are disguises other than "dead": `read_file` was listed in an example list of event kinds, but it was not a kind — it was the **value** of a field inside `tool_executed` events. Not dead, but **not that species of thing at all**. Updating the examples does not fix this — build the mechanism that derives the enumeration from the primary source (the actual producers), and this category error surfaces automatically (you are forced to explain why `read_file` does not appear in the derived list).

## A gate can be satisfied from two directions

There are two ways to turn a "declared ⊆ emitted" gate green: **add the emitter**, or **remove the declaration**. Which one is right is determined by "should this feature live?" — and **the gate does not answer that**. Also, "the emitter cannot be added" (the observation point is external) and "nobody has added it" are distinct states, and they must be written up as such.

## How to apply

- Do not use "it is declared" as evidence. Registries, schemas, constant tuples, doc tables — none of these are producers. **Grep the writing side** (emit / append / write / construct).
- Label claims MEASURED (I measured it) / READ (an artifact of intent says so) / INFERRED (I inferred it). For READ, attach "what that source is evidence **of**" — collapsing READ into MEASURED is the recurring error.
- Classify liveness item by item. Do not bundle on the intuition "probably alive".

## Sources (measured during reyn development)

The reader-based misjudgment: the 2026-07-04 skill sweep (551 KEEPs → all dead under producer inspection; owner's callout). The three declaration substrates: #3357/#3410 (schema, zero emitters), #3432 (doc, renamed kind), #3433 (log kind, false recovery guarantee). Reading a description as a report: #3437 (notably, the mechanical defense against this error is the very gate that PR introduces — the review process itself became a witness to the gate's necessity). Suggestion by name: the `Cancelled` case (2026-07-29).

## Related

- [Wiring tests vs mechanism tests](wiring-vs-mechanism.md) — the sibling error: "the mechanism exists" ≠ "the mechanism is reached"
- [Census vs structure](census-vs-structure.md) — false descriptions are fossils of incomplete enumeration
- [How vacuous gates are born](vacuous-gates.md) — "describing a property is not witnessing it"
