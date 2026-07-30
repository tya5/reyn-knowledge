---
name: green-reading
description: '"rc=0 / N passed" says only that everything that ran was green — it says nothing about what ran. Skips look green. Before citing green, establish whether the witness executed, which code the green is about, and whether the run really finished'
tags: [verification, testing, ci]
sources:
  - feedback_green_is_not_evidence_it_ran_skip_is_green
  - feedback_verify_identity_of_measured_code_before_reading_green_or_red
  - feedback_req_resp_plus_one_swallowed_exception
---

# Reading Green — What Ran, Which Code, and Did It Really Finish

## Core claim

> **"The tests are green" says only that everything that ran was green. It says nothing about what ran. A skip looks green.**

∴ `rc=0` (exit code 0) or "8686 passed" is **not evidence that any particular claim was witnessed**. What you must read is not the pass count but **the list of skips**.

## A real case — the party who said "your rc=0 is the refutation" was wrong

The claim under test: does this PR leave the docker integration unbroken? Two verifiers reported on the same PR (in verifier B's environment, the docker daemon was down):

- Verifier A: "**2 failed** (the docker integration tests)" — honestly adding "corroborated, but I have not produced a direct refutation"
- Verifier B: "**rc=0 / 8686 passed / 19 skipped**. I also could not independently refute it."

A reviewer who read this lectured B: "**your own rc=0 is exactly the refutation you say you could not produce** — zero failures on the same code, so the code is not the variable." B pushed back with an actual measurement using `pytest -rs` (the flag that prints skip reasons):

```
SKIPPED test_devcontainer_build_e2e.py:72 : no reachable Docker daemon
SKIPPED test_mount_e2e.py:121/131/150/172/181 : same as above
```

> **"My rc=0 does not mean 'the docker build passed' — it means 'the docker build never ran.'"**

A and B had gone through **different branches of the same tests** (A = the real-build branch, B = the skip branch). ∴ **B's green cannot refute A's red.** "The code is identical, therefore the code is not the variable" is true, but it does not entail "the build ran in B's environment" — the variable was whether the daemon was alive, and in B's environment the tests were **never executed**.

The sharpest part of this error: in **the very message** where the reviewer was lecturing the other party to "read your own evidence," the reviewer counted only the passes and **never once looked at the number 19**.

## Discipline

- Before citing a suite result as evidence for a claim, ask: **"Did the test that is supposed to witness that claim actually execute?"** Print skip reasons with `-rs`, or run that test by name in isolation.
- A discipline for whoever reports — and whoever commissions the report: **report "what was skipped" together with the pass count.** A bare `rc=0` is an unfalsifiable report; "19 skipped: <reason>" is a report that can support a judgment.
- **Two green environments are still zero witnesses if neither of them ran the branch in question.**

## Healthy skips — "dormant tripwires"

Not every skip is a blind spot. Two checks that identify a healthy skip:

1. **Is the skip reason specific (legible)?** Not "unsupported," but naming the exact thing that is missing ("this OS lacks mechanism X").
2. **Is the unreachability real?** Does the target genuinely not exist, or does the target exist while the test harness simply fails to reach it? (A real case of the latter: a test that bypassed the production entry point kept "passing" for **41 days**.)

A skip that satisfies both is a **dormant tripwire** — the moment the target mechanism appears in that environment, the skip turns into execution and goes RED if things are broken. **This is strictly better than a TODO comment**: it is a mechanism scheduled to fire, not prose that hopes to be read.

## Layer 2 — Which code is that green about?

(Everything up to this point is Layer 1: *what ran*.) The meaning of green/red presupposes that **what you measured is identical to what will run**. This premise breaks in both directions (there is a real case of two people falling off opposite sides on the same night):

- **False negative**: a virtual environment was out of sync, so an **old version** of the supposedly-fixed code was measured and misread as "still FAIL."
- **False positive**: a strip (temporarily disabling the fix) was performed by **monkeypatching the parent process**, but the mechanism by design always runs in a **fresh subprocess**, so the patch never landed. **Code with nothing disabled was being measured** — the experiment never happened, yet the "green despite the strip" result was on the table, one step away from the wrong conclusion (that the tests do not protect the fix). It was caught only because "green despite the strip" could not be explained; editing the real source and stripping again produced 5 REDs = the tests protected the fix all along.

> **Before reading green/red, first establish: "which code is this green/red about?"** Subprocess boundaries, virtual environments, PYTHONPATH, and import paths are all entrances to this hole.

(Breakage mode 3 in [Integrity of the strip instrument](strip-instrument-integrity.md) is the venv edition of this discipline.)

A corollary: **grep does not distinguish "tombstones in prose" from "live calls."** Confirming "this API is no longer called" is done with the AST (abstract syntax tree), not with the presence or absence of a string — the 41-day incident above was created precisely by conflating "the string is there" with "code runs there."

## Layer 3 — "It finished normally" is itself a claim to verify

Exit code 0 plus an empty stderr does not mean "ran to completion without exceptions." **An exception may have been swallowed along the way.** The diagnostic signature in a loop that calls an LLM makes this easy to spot:

> **Request-log count = response-log count + 1, and rc=0, and stderr is empty, and the agent stopped mid-task = an exception was swallowed on the loop path. This is not a normal termination.**

Why this shape: the request is logged **before** the API call, the response **after** it. Raise in between, and only the request remains. The isolation procedure is also simple: (1) temporarily add traceback output to the exception-swallowing handler (`except Exception`), (2) rerun to capture the actual exception, (3) remove the instrumentation after the fix.

Before attributing a short run or a mid-course stop to "model capability" or "a timeout," **count requests against responses.**

## Checklist

- [ ] Before citing a suite result: did the relevant witness test execute? (`-rs` / run it in isolation)
- [ ] Does the report include the list of skips with reasons? Are you treating a bare rc=0 as evidence?
- [ ] When you see a skip: is the reason specific, and is the unreachability real (dormant tripwire, or blind spot)?
- [ ] Before reading green/red: which code is this green/red about? (venv, subprocess, import path)
- [ ] Before believing "it finished normally": any sign of an exception swallowed mid-run? (req/resp count)

## Sources (measured during reyn development)

Skips are green: #2994 (rc=0 under a stopped docker daemon misread as a "refutation"; `-rs` revealed all 19 skips were docker). Telling healthy skips apart: #3019. The 41-day pass-through: #2980. Identity of the measured code: #3031 (the parent-process-patch false positive) and the unsynced venv the same night (the false negative). Swallowed exceptions: #187/#1413 (`response.choices[0]` raised IndexError on an empty response → the whole loop died silently → fixed with retry + an explicit error).

## Related

- [Structural blindness of the verification environment](environment-blindness.md) — this document is the same hole seen from the side that reads the reports
- [Integrity of the strip instrument](strip-instrument-integrity.md) — identity of the measured code (venv edition)
- [Liveness of the verification run](verification-run-liveness.md) — the adjacent hole of reading "not finished" as "in progress"
- [How vacuous gates are born](vacuous-gates.md)
