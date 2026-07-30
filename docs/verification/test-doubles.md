---
name: test-doubles
description: Test fakes (fakes, fixtures, None, default values, proxy checks) manufacture apparent coverage unless they match the real shape — the double and the test share the same assumptions, so they keep agreeing with each other while missing reality
tags: [verification, testing, fixtures]
sources:
  - feedback_test_fake_inventing_a_field_makes_a_dead_gate_look_tested
  - feedback_envelope_detection_test_real_payload_shape
  - feedback_envelope_shape_fix_verify_fixture_matches_live_producer
  - feedback_enforcement_test_real_resolver_not_none
  - feedback_fake_backend_unit_misses_real_integration
  - feedback_roundtrip_test_nondefault_value
  - feedback_test_claim_must_match_test_content
---

# Test Doubles Must Match the Real Shape — Six Ways to Manufacture Fake Coverage

## Core claim

A test double (fake, stub, fixture, stand-in value) **shares the assumptions of whoever wrote the test**. So a test built on a double goes green even when the assumptions are wrong, because **the double and the test agree with each other**. Only "the real shape" can bring reality into a test — the real type, the envelope the real producer builds, the real resolver, the real backend, a non-default value, the real mechanism.

Below: the six standard ways apparent coverage gets manufactured, and the prescription for each.

## Shape 1 — The fake invents a field that does not exist in production

Starting with the most severe real case. A permission gate was written as:

```python
resolver = getattr(rs, "permission_resolver", None)   # rs is the shared-state object
if resolver is not None:
    await resolver.require_file_write(...)             # ← never fires
```

The actual type of `rs` (a plain dataclass) **has no field named `permission_resolver`**. The third argument of `getattr` silently returned None, and the gate was **dead on every path**. Yet nobody noticed, because the test stub looked like this:

```python
class _RS:
    def __init__(self, factory, resolver=None):
        self.permission_resolver = resolver   # ← "invents" a field that does not exist in production
```

> **The only thing that ever supplied what the handler reads was the test fake. ∴ The gate looked "tested" and could never fire in production.**

Three lessons:

- **The letter of a "no mocks" policy does not stop hand-written fakes.** The policy that named `MagicMock` was being followed. What was violated was its rationale ("fakes silently rot the real contract"). **The meaning of the ban is not "don't use MagicMock" but "don't fake collaborators."**
- **Fake callables and fake data objects break differently.** A fake function fails **loudly** on a signature mismatch. A fake data object silently carries fields absent from the real type, and `getattr(obj, "x", default)` **never raises and returns the default forever**.
- **If the real thing is cheap to construct, don't write a fake.** The real type in this incident was a plain dataclass; there was never a reason not to construct the real one.

Detection is hard, too: **you cannot grep for what is missing** (it is not written, so there is nothing to search for). What works is a structural cross-check — the diff between "every attribute name production reads off that object (enumerated via the AST)" and "the field set of the real type." And **first run that check against a version of the code known to contain the defect, and confirm it detects it, before** relying on it (the checking-tool edition of [The discipline of strip-falsification](strip-falsification.md)).

## Shape 2 — The fixture's envelope differs from the real producer's shape

When testing code that detects or parses values inside a boundary-crossing payload (tool execution results, inter-process messages, API responses), hand-writing the fixture **from the model in your head** drops you into this trap.

Real case: a predicate deciding "is this execution result an error?" looked at the top-level `status`. The test was green with the hand-written fixture `{"status": "error"}`. But the real dispatcher **wraps** every result as `{"status": "ok", "data": <real result>}`, so execution errors are **nested** under `data.status` and the top level is always `ok`. **The predicate was a no-op against all 58 real data items.** The fix stayed perfectly consistent with its own tests while detecting zero production cases — a **closed loop**.

- Do not type fixtures by hand; **copy them from real traces or event logs**, or **read the producer's code** to pin down the shape.
- On the review side: for fixes that handle envelopes, confirm that **the fixture matches the real producer's shape** by grepping the producer side. "The fix passes its tests" is meaningful only under the premise that the fixture is right.
- In systems that wrap everything in a uniform wrapper, the raw status is always nested — detection that looks only at the top level is almost certainly wrong.

## Shape 3 — None or a permissive default lets everything through the check

A test meant to protect enforcement was **passing None as the resolver**. On that path, the rule was "no resolver → permit" — a permissive, fail-open default — meaning **things are permitted even when the declaration is missing**. The test was green regardless of whether the declaration existed, and could not verify the very thing to protect: "no declaration → deny."

- Permission/enforcement tests must construct and pass **a real (non-None) resolver**.
- Then falsify on top of that: only after confirming that **deleting the declaration makes the test fail** is it a test that protects the declaration.

## Shape 4 — A fake backend cannot verify the real backend's own assembly

A unit test that injects a recording fake backend verifies **the caller's logic**. But bugs in **the real backend's assembly** (the container-execution argument list, the `-i` flag, stdin forwarding) are caught only by an integration test that drives the real thing. Real case: units on the fake were green while the real `docker exec` lacked `-i` and stdin was silently discarded.

- For backend-integration PRs, do not accept "units green = integration confirmed." Require **one e2e run against the real backend** and **an assert on the assembled argument list itself**.

## Shape 5 — Round-trip tests at default values pass trivially

In a `set → reload → get` round-trip test, writing **the same value as the field's default** means the test **passes even if the write is a no-op** (the reload returns the default regardless of the write). Real case: round-tripping every setting at "its own default" scored 113/113 — and hiding among them was **a disconnected field the loader never reads at all**.

- Round-trip / identity tests must always use **a value different from the default** (+1 for integers, negate for booleans, append for strings).

## Shape 6 — The test's claim and its body diverge (proxy checks)

A test whose docstring claims to "verify the loader/resolver/runtime" while its body is nothing but `yaml.safe_load` + asserts on a dict's shape. It **never calls the real mechanism**, so it stays green **even when the snippet shape shown in the docs is wrong** — in the real case, the wrong shape passed merge while pinned in place by the test, and fired in production. Note this happens even with no mocks involved.

- Inspection procedure: read each test docstring's **claim** → check whether the body **actually calls** that mechanism → on divergence, either rewrite the body to call the real thing, or downgrade the claim to match reality ("checks the snippet exists").
- Extension: do not accept "faithfully reproduces X"-type claims on an aggregate-level falsification test alone. **Cross-check each component 1:1 against its real counterpart** (aggregate falsification proves only that "the aggregate logic runs").

## Checklist

- [ ] Do the attributes the fake supplies exist on the real type? Are you writing a fake when the real thing is cheap to build? (Shape 1)
- [ ] Is the fixture derived from a real trace, or cross-checked against the producer's code? Did you check for nesting? (Shape 2)
- [ ] Did you pass a real resolver to enforcement tests, and confirm they fail when the declaration is deleted? (Shape 3)
- [ ] Did you run the backend integration once for real? Did you assert the assembled argument list? (Shape 4)
- [ ] Do round-trip tests use values different from the defaults? (Shape 5)
- [ ] Does each test's claim match what its body actually calls? Did you check "reproduces" claims component by component? (Shape 6)

## Sources (measured during reyn development)

Shape 1: #3037 (gate, audit, and recovery — all three faces dead at once; includes the AST-vs-field-set diff check and the proof of that check's own soundness). Shape 2: #1439 Fix#2 (a no-op fix missing all 58 items passed review; the real-trace replay gate caught it). Shape 3: #1214→#1215 (a None-resolver test hid a missing declaration). Shape 4: #1356→#1363 (the missing `-i`). Shape 5: #1142→#1145/#1146 (113/113 by round-tripping every setting at its default). Shape 6: PR-N11→N12 (claiming "loader verification" with YAML parsing only) and #1297 (co-signing "faithful reproduction" on aggregate falsification alone).

## Related

- [How vacuous gates are born](vacuous-gates.md) — fakery on the gate side rather than the test side
- [Proving fixture provenance](fixture-provenance.md) — matching recorded fixtures to reality
- [Census vs structure](census-vs-structure.md) — the fake edition of "verification protects only the domain you sampled"
- [Structural blindness of the verification environment](environment-blindness.md) — a fake is the smallest unit of "an environment shaped like your assumptions"
