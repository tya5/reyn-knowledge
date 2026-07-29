---
name: wiring-vs-mechanism
description: Mechanism tests cannot notice the deletion of wiring — "the mechanism is correct" and "production reaches that mechanism" are different claims and need separate asserts
tags: [verification, testing]
sources:
  - feedback_wiring_test_strip_production_callsite_not_mechanism
---

# Wiring Tests vs Mechanism Tests — "It Works" and "It's Connected" Are Different Claims

## The shape of the problem

When adding a new feature, the code consists broadly of two parts.

1. **Mechanism**: the function or class that does the work itself
2. **Wiring**: the place where production code actually calls that mechanism (the call-site)

And **it is extremely common for tests to verify only the mechanism and forget to verify the wiring**. Toy example:

```python
# Mechanism: install an error handler
def install_error_handler(app):
    app.set_exception_handler(log_and_recover)

# Wiring: the production startup path (main.py)
def start():
    app = App()
    install_error_handler(app)   # <- this one line is the "wiring"
    app.run()
```

The bad test gets written like this:

```python
def test_error_handler_installed():
    app = App()
    install_error_handler(app)            # the test calls the mechanism itself
    assert app.exception_handler is not None
```

This test proves that `install_error_handler` works correctly (the mechanism). But **delete the `install_error_handler(app)` line from `start()` and it stays green**. In other words, the wiring that delivers the feature to production is protected by nothing. If a future refactoring removes that one line, the feature dies silently and the bug this PR was supposed to close simply comes back.

The way to check is an application of [strip-falsification](strip-falsification.md): **make the strip target the production call-site, not the mechanism**. If deleting the wiring line leaves the suite green, it is a mechanism test, not a wiring test.

This shape is not rare. In practice, **all three PRs reviewed in a single night had exactly this defect**. Respectively: (a) the test hardcoded arguments in its own driver, (b) the test called an internal function directly, (c) the test hand-assembled state and bypassed the real entry point (the session layer) — different in form, but in every case the test never traveled "the path from the production entry point to the mechanism."

Note that the implementer's claim "it was RED before the fix" is often not a lie. **What was RED was the symptom (the absence of the mechanism), not the wiring.** When you hear "RED before fix," ask "RED when stripping what?"

## "A correct mechanism exists" and "every path goes through it" are different claims

The general form of this defect is:

> **Whenever you recommend building a mechanism, or report having built one, always demand "correctness of the mechanism" and "reachability of the mechanism" as two separate asserts.**

Real case (generalized): a design proposal said "consolidate the responsibility of deep-copying objects at external egress into a single dedicated helper, and have the gate inspect the helper's output with an `is` comparison." This proposal protects only "a helper with the correct contract exists." **What is needed is the separate claim "every egress point goes through that helper"** — and no number of additional unit tests for the helper protects that claim by even a millimeter.

There is a record of the same reviewer making this exact conflation **twice on the same day**. An error that occurs twice is not a problem of individual attentiveness but a **missing procedure (a missing delegation template)** — the correct remedy is to build "assert the mechanism's correctness and reachability separately" into the standard delegation wording.

## Reachability has a time axis — the fifth site that sprouts tomorrow

Even if you witness with tests that "all four of today's egress points go through the helper," **the fifth egress point added tomorrow has no witness, and the absence of a witness is silent** (a shallow copy slips in and lies dormant, asymptomatic until something breaks).

The remedy is to additionally require that "new sites cannot bypass the helper" via a **syntax-level gate (an AST check or similar)**. If what you search for is "the shape of an expression" rather than "intent," it can be made a syntactic gate, and false positives can be confined to the single site of the helper itself. However:

- **Require the gate's uncaught range to be spelled out in its docstring.** A gate whose false positives include legitimate sites will eventually be suppressed (disabled), and **a suppressed gate protects nothing**.

## Checklist

For the implementer:

- [ ] Does the test reach the mechanism from production's actual entry point (a real session, a real runner, real dispatch)? Is it substituting direct calls to internal functions or hand-assembled state?
- [ ] Is the target of strip-falsification "the production call-site" (not the mechanism)?
- [ ] Are both asserts present — correctness of the mechanism and reachability?

For the reviewer (co-vet = independent second verification):

- [ ] Did you refuse to take "RED before fix" at face value and confirm **what was stripped**?
- [ ] Did you strip the production call-site yourself? If it stays green, it is "a mechanism test with unguarded wiring" → require one real-path test before merge (that is usually all it takes)
- [ ] Did you consider whether the shape calls for a structural gate (syntax check) against future site additions?

## Sources (measured during reyn development)

reyn is an AI agent runtime whose development is done almost entirely by a fleet of coding agents. Three in one night: #2788 (test hardcoded arguments in its own driver), #2801 (called an internal function directly; stripping the production-side call stayed green), #2802 (hand-assembled state, bypassing the session layer). All three were discovered by an independent second verifier stripping the production call-site. Separating correctness and reachability: #3383/#3385 (design advice on deep-copy helper consolidation, where the reviewer fell into the same hole twice in one day, leading to the remedy of building it into the delegation template).

## Related

- [The discipline of strip-falsification](strip-falsification.md)
- [Liveness is decided by the producer](liveness-is-producer.md) — a kindred error: "there is a reader" or "there is a declaration" is not evidence of reachability
- [How vacuous gates are born](vacuous-gates.md)
