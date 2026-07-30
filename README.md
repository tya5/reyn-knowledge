# reyn-knowledge — Verification Craft for AI Coding Agent Fleets

A knowledge base of **verification failure patterns — and their countermeasures — specific to developing and reviewing software with fleets of AI coding agents** (Claude Code and similar).

## What is this?

This material comes from the development of [reyn](https://github.com/tya5/reyn), an AI agent runtime. Nearly all of reyn's implementation, review, and merge decisions are performed by a fleet of coding agents. Along the way, hundreds of "verification failures and their fixes" accumulated as cross-session memory entries.

Most of that memory turned out to be knowledge you will not find in software engineering textbooks — **field records of the ways agents over-trust the tests, gates, and fixtures they themselves wrote**. For example:

- An agent reports "I stripped the mechanism and confirmed RED" — but the strip never landed
- 8,540 tests and CI stayed green while the mechanism had never once run in production
- Needing to distinguish "I re-recorded the fixture" from "I hand-wrote values that pass"

This repository generalizes those records: the reyn-specific context is peeled away so the lessons apply to any development that uses AI coding agents. It is written as educational material — each document includes concrete code examples that a CS student can follow.

## Structure

```
docs/
├── verification/   # Tier 1: verifying AI-produced artifacts (strip-falsification, vacuous gates, census vs structure, …)
├── git-github/     # Tier 2: agent-driven git/CI operation hazards (closing keywords, stale local trees, merge gates, …)
└── orchestration/  # Tier 3: multi-agent fleet operations (planned)
skills/             # executable checklists (first: closing-keyword-check)
CATALOG.md          # index of all documents + source mapping
```

Start with [the map of the verification knowledge](docs/verification/index.md) — it lays out how the 25 documents form a pipeline of "what to doubt, in what order, between a completion report and the merge," plus the four principles that cut across all of them. For the now-standard setup with separate coder / tester / reviewer agents, [the roles guide](docs/verification/roles.md) reprojects the whole system into per-role obligations (the coder's report contract, the tester's falsification menu, the reviewer's independent verification, the merge gates). For standalone reading, [The discipline of strip-falsification](docs/verification/strip-falsification.md) and [Structural blindness of the verification environment](docs/verification/environment-blindness.md) are representative. For the git/GitHub operations side, start from [the Tier 2 map](docs/git-github/index.md).

## Languages

- `*.md` — English
- `*.ja.md` — Japanese (written first; the English versions are synced from them)

## Minimal glossary

| Term | Meaning |
|---|---|
| RED / GREEN | test failure / test success |
| gate | a test or automated check placed to make a specific regression impossible |
| witness | a test that turns RED when a given property breaks; "witnessed" = "breaking it gets detected" |
| strip-falsify | temporarily disable a mechanism and confirm the suite turns RED |
| vacuous | green even on a build where the defect it guards against is present |
| co-vet | independent second verification of an implementer's claims by another agent or human |
| pin | one entry of reyn's cross-session memory (the source material of this repo) |

## Editing flow (source pin → doc)

1. Read the source pin; separate the **invariant lesson** from the **reyn-specific incidents**
2. Replace project-specific vocabulary with general vocabulary (session names → roles, internal mechanism names → generic mechanisms)
3. Write the invariant lesson as the main text, with **concrete toy-code examples a CS student can follow**
4. Keep the reyn incidents in a "Sources" section, issue numbers included (provenance is never deleted)
5. Cross-link related docs and add a pin → doc row to `CATALOG.md`

reyn's pins keep growing, so this repository is not a one-time export but a **periodically synced** distillation. The unit of sync is the CATALOG mapping table.

## Provenance

`#NNNN` in the text refers to reyn PR/issue numbers. Every example is generalized from real development logs.
