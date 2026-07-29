---
name: environment-blindness
description: A gate can witness an assumption only when its environment differs from the environment the assumption speaks about — "the test is green" and "the environment can turn the test red" are separate claims, and only the latter deserves belief
tags: [verification, environment, ci]
sources:
  - feedback_verification_environment_structurally_blind
---

# Structural Blindness of the Verification Environment — "Green" vs "Capable of Turning Red"

## The central proposition

> **A gate can witness an assumption only when the gate's execution environment differs from the environment the assumption is talking about.**

It sounds abstract, but a concrete example makes it simple.

Example: suppose we ship the assumption "on the user's machine, typing `python3` launches our interpreter." Can an ordinary pytest job verify this? **No.** The pytest job **tries to verify the assumption while simultaneously satisfying it itself** (the CI runner has the correct python3 on PATH). The assumption and the test environment are the same fact, so falsification is impossible in principle.

A gate that can witness this **deliberately constructs an environment shaped like production**: install the actually-built wheel into a throwaway venv, and invoke it **without activating the venv and without putting it on PATH**. This environment differs at exactly the point where the development environment differs from production, so if the assumption is broken, the gate is able to turn red.

The one line to remember:

> **"The test is green" and "the environment has the capability to turn the test red" are separate claims, and only the latter is worth believing.**

## Real case — three blind spots in one change (each for a different reason)

In a change to a sandbox mechanism (seccomp / Landlock), **the same changeset contained three defects**, each invisible to the verification environment for a **different reason**:

| Defect | Why the environment was structurally blind |
|---|---|
| `stat`/`lstat` missing from the syscall allowlist | The dev machine's CPU architecture (aarch64) **has no `stat` syscall at all** (only the `*at` family). This defect fires only on x86_64 with older glibc (the major LTS distros) |
| A file-truncate hole | The verification host's kernel was new (ABI 4); the hole exists only on **ABI 1–2 kernels** (= the mainstream installed base out in the world) |
| A test loading the real irreversible filter into pytest's own process | The dev machine (macOS) lacks the library, so `is_available()` is False and **the hazard cannot fire** |

The same night, the same shape elsewhere: a dogfood scenario not running in CI → a reference to a deleted feature sat unseen for **13 days**. CI not installing an optional dependency → an entire module skipped while CI stayed green. Dead wiring → the self-destruct bug downstream (a configuration that rejects execve right before exec) never fired "because it was unreachable."

> **A dead mechanism hides defects that will fire the moment it comes back to life.**

## The fix side has the same traps

**Trap 1: a structural fix that "makes the guard impossible to forget" kills the only real verification.** As a countermeasure to the third defect above, a proposal — "a fixture auto-applied to every test that forbids loading the real filter" — was considered and **rejected**: that fixture would silently neutralize the upcoming CI job whose very purpose is to load the real filter. Before "structuring the guard," check **whether there is a verification that the guard must not kill**.

**Trap 2: a reviewer's "better fix" reproduces the very defect class being fixed.** The suggestion "instead of removing the syscall from the allowlist, put it under the management of the higher-level mechanism" leaves the hole open on old kernels, because that management feature **exists only on new kernels** — and on the proposer's (new) verification host it looks closed. Proposals, too, must be examined along the axis of "in which environments does this claim hold?"

## The third form — an accurate description standing in for a mechanism

Environment blindness has a form that holds even when not a single lie is present:

> **Even an accurate description is no gate if no environment enforces it. "We wrote it in the docs" is a paraphrase of "no environment witnesses this."**

Real case: in one review, a reviewer correctly rejected option C — "merely **warn** about a dangerous configuration overlap" — with "a warning is not a defense." Hours later, the same reviewer **nearly merged a different PR containing the same content in the form of a documentation warning**. The wording was accurate; there was not one lie in it. Yet since the answer to "which environment can witness this assumption?" was "none — it was only written down," it was exactly the option C that had been rejected.

A docs-only guard is legitimate only when it is explicitly marked as a **placeholder with a due date** ("doc-only interim until #N lands").

## An honest "unverified" does real work

The syscall defect above was found because the implementer wrote, in plain text in the report, "**unverified on x86_64**." Starting from that one line, the reviewer was able to reach the concrete omission (`stat`/`lstat`). Vague confidence hides holes; **an honest "unverified on X" is load-bearing**.

## How to apply

- Before trusting a green suite, ask: **"What must be true of this environment for this test to have the capability to fail?"** If the environment itself supplies the fact under test, that green is worthless.
- When shipping an assumption, **name the point where production differs from the development environment, construct an environment that differs at that point**, and place the gate there.
- Require implementation reports to state, in plain text, "verified in which environments, unverified in which."
- When you see a docs-only guard, ask "which environment witnesses this?" If there is no answer, it is not a defense.

## Sources (measured during reyn development)

The three-blind-spot changeset: #2975 (seccomp allowlist / Landlock ABI / hazard unable to fire on macOS). Same shape: #2965 (dogfood not in CI, 13 days undetected), #2952 (near-miss of everything skipped due to an uninstalled extra), #2962 (seccomp never once loaded in production). Gate-construction examples: #2973 (wheel reachability gate), #2982 (x86_64 sandbox CI job). Reproduction of option C: #2978 → #2981.

## Related

- [Census vs structure](census-vs-structure.md) — this document is the environment version of "a check protects only the domain it sampled"
- [How vacuous gates are born](vacuous-gates.md)
- [Liveness is decided by the producer](liveness-is-producer.md)
