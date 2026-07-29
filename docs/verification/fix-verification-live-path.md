---
name: fix-verification-live-path
description: Verify fixes through the live path — the diagnosis named in the request, an isolated probe of a single gate, and the implementer's tests are all proxies, and proxies can even invert the verdict
tags: [verification, bugfix, review]
sources:
  - feedback_verify_dispatched_bug_is_active_before_fixing
  - feedback_verify_fix_through_declaration_consumer_not_gate_alone
  - feedback_covet_falsify_owner_actual_path_not_test_covered_sibling
---

# Prove Fixes on the Live Path — Three Proxy Traps

## The shape of the problem

When verifying a bug fix, there are three **proxies** you will be tempted to use in place of the real thing: (1) the diagnosis written in the request, (2) an isolated check of the fixed spot itself, and (3) the implementer's tests being green. None of the three substitutes for verification through the live path — the path the failure actually flows through. Proxies do not merely **get the answer wrong; they can invert it**.

## Trap 1 — Is that bug still alive?

Even when a bug-fix request arrives with a confident-sounding root cause attached, **confirm the bug is still ACTIVE before implementing anything**. A past fix of the same class may already have closed the whole class.

Real case: an implementer received a request describing "a class of crashes where a cached client gets closed by a different task," and traced the live path. Findings: execution was **strictly serial** (nothing creates concurrent tasks); there was exactly one call site that closes the client, and it already carried a **comment from a past same-class fix** ("close in the same task that opened it"). The crash the owner had actually hit was **somewhere else** (already handled by a separate fix). In other words, the request's premise — "an active bug" — was false, and **implementing the fix would have meant refactoring correct code, creating an opportunity to introduce a new bug**.

- (1) **Trace the live failure path** (not the diagnosis in the request). (2) Grep for past fixes of the same class (issue-numbered comments near the seams are the telltale). (3) Ask for concrete reproduction steps and the owner's symptoms, and check whether they match **this spot**. (4) If no active path is found, say so and **propose putting the work on hold** — "fixing an active bug" and "preventively hardening correct code" are separate decisions, and the latter is the owner's priority call.

## Trap 2 — An isolated gate probe can invert the verdict

When verifying a fix that adds a declaration or configuration entry, actually run the flow through **the declaration and every layer that consumes it**. Probing only the function at the fix site, in isolation, can produce **the opposite conclusion**.

Real case: to check whether a write-permission declaration added to a manifest took effect, the verifier called the runtime gate function in isolation → it denied the write → they nearly concluded "this fix is inert (has no effect)." In reality, in this system **the runtime gate by design never reads the declaration**. The declaration's true consumer is a different layer — a **startup-time precheck** — which routes declared paths into pre-approval; once the approval is persisted, writes go through. Run it end to end (declaration → startup check → approval → write) and the fix **was working correctly**. The isolated-gate proxy inverted a working fix into "inert" and was about to send back an unwarranted rejection.

- **Grep and enumerate all consumers** of a declaration or config field (not just the obvious gate), and run the real flow through each of them.
- **Match the verification environment to the production shape** (working directory, base for relative-path resolution, and so on). An isolated probe with the wrong shape emits false denials.

## Trap 3 — The implementer's tests may only walk a sibling path

When independently verifying (co-vetting) a fix that touches multiple call sites or branch paths, identify **the path the owner actually hit** yourself and falsify on it. Even if the implementer's tests are green, they may be driving **a different sibling path**.

Real case: the implementer's tests drove the interactive-prompt path (path A) and were green. The owner's actual failure occurred on **the default fullscreen path (path B)** — same mechanism, different call site. The verifier reproduced the exact shape of path B and confirmed correct behavior with the fix in place, and **RED with the fix reversed**. That was the decisive, load-bearing check that filled the blind spot in the implementer's tests.

- Enumerate the call sites the fix changed, and **trace the routing to determine which one the owner's reproduction flows through** (do not guess).
- Reproduce the exact shape of that path and strip-falsify it — good behavior with the fix in place, plus RED with the fix removed (see [The discipline of strip-falsification](strip-falsification.md)). Echoing the implementer's claims back is not verification.
- If the implementer's test **hardcodes the post-fix value itself**, it is a mechanism test and will not catch a future regression where the value disappears from the real call site — point it out ([Wiring tests vs mechanism tests](wiring-vs-mechanism.md)).
- Before concluding "zero results" about evidence (persisted files, etc.), **make sure you are looking in the right place**. In the real case, the storage layout used month-based subdirectories, and a search at the shallow level reported a false "zero results."

## How to apply

- [ ] Before implementing a fix: did you trace the live failure path? Did you grep for past same-class fixes? Do the owner's symptoms match this spot?
- [ ] Declaration/config fixes: did you enumerate all consumers and run end to end in the production shape?
- [ ] Co-vet: did you identify the owner's real path and strip-falsify on that path?
- [ ] Before concluding "zero results / inert / has no effect": did you confirm the place, layer, and environment you are looking at are the right ones?

## Sources (measured during reyn development)

Trap 1: MCP client fix B (tracing the live path revealed "already fixed by a same-class fix"; a hold was proposed and the owner and lead agreed). Trap 2: #2413 (an isolated gate probe nearly produced a false "inert" verdict; the declaration's true consumer was the startup check). Trap 3: #2788/#2786 (the implementer's tests drove the interactive path, the owner's failure was on the default path; the verifier's path-B reproduction plus reversal-RED was praised as "a model load-bearing check." The month-directory false zero is from the same case).

## Related

- [The discipline of strip-falsification](strip-falsification.md) — the general procedure for reversal RED
- [Wiring tests vs mechanism tests](wiring-vs-mechanism.md) — distinguishing mechanism tests from wiring tests
- [Structural blindness of the verification environment](environment-blindness.md) — "where does this check differ from the production shape?"
