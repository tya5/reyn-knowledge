---
name: vacuous-gates
description: The typical forms of born-vacuous gates — terminal-state-only asserts, positive controls on a different path, existence checks, prose-only properties. The fix is "intermediate cross-section + same delivery path + measured RED on a defective build"
tags: [verification, testing]
sources:
  - feedback_gate_vacuity_hides_in_terminal_state_only_assertions
  - feedback_containment_gate_must_cover_both_axes_and_children
---

# How Vacuous Gates Are Born — Tests That Stay Green on Broken Builds

A **gate** is a test or automated check put in place to ensure a specific regression can never happen again. A **vacuous** gate is one that stays green even on a build where the very defect it is supposed to guard against is actually present.

A vacuous gate is **worse** than no gate at all: as long as CI keeps returning green, nobody re-inspects that surface. Worse still, vacuous gates are often **born-vacuous** — they have never once detected the defect, yet they keep accumulating a track record of green.

The following typical forms of creeping vacuity were extracted from real development records (Forms 1–3 struck three times in a row on a single working day; the other forms come from separate incidents).

## Form 1 — Asserting only the terminal state

Example: in streaming delivery, we want to guarantee that "when multiple deltas arrive, the screen shows one entry containing the full text."

```python
def test_stream_renders_one_entry():
    send_deltas(["Hel", "lo, ", "world"])
    send_completion_frame()
    assert len(view.entries) == 1
    assert view.entries[0].text == "Hello, world"   # only looks at the terminal state after completion
```

This test **passes even if zero deltas arrive**, because the completion frame itself produces "one entry, full text." In other words, the essential path — "deltas arrive and get rendered" — can be dead and the test still goes green.

**How to close it: assert an intermediate cross-section.** Look at the "in-progress state" before the end:

```python
    send_deltas(["Hel", "lo, "])
    assert view.entries[0].text == "Hello, "        # ← partial text must be visible mid-stream
    send_deltas(["world"]); send_completion_frame()
    assert view.entries[0].text == "Hello, world"
```

Once you look at the intermediate cross-section, the "nothing happened" case automatically goes RED, and **the reachability witness is built into the gate itself**.

## Form 2 — The positive control goes through a different path

A **positive control** is a test construction that "feeds in a case where the signal should appear, to confirm the detector actually reacts." The trap: **if the positive control goes through a different path than the thing under verification, it only proves the neighboring path is alive.**

Example: the goal was to verify that events arrive via `push_event()`, but the positive control inserted directly into the internal queue with `_queue.put()`. With that setup, the control succeeds even if `push_event()` is dead.

**How to close it: route the positive control through the exact same delivery path as the thing under verification.** (If you add the intermediate cross-section from Form 1, a separate positive control often becomes unnecessary in the first place.)

## Form 3 — Existence checks and weak RED

Asserting "the constant is defined" or "the module imports" **witnesses nothing about the property that the value actually makes it into the output**. A test that only goes RED on ImportError can barely detect the death of the mechanism.

**How to close it (common to all forms): actually build a defective version and measure that the gate goes RED before shipping.** That is the only proof that a gate is not born vacuous. This is an application of [The discipline of strip-falsification](strip-falsification.md).

## Form 4 — The prose claims two properties, the tests cover only one

Docstrings, PR bodies, and comments frequently claim more than the tests do.

Real case 1 (generalized): the docstring says "this key is **borrowed**, not taken," while the tests cover only the "borrow" side — zero tests for the "return" side.

Real case 2: the reserved-key table `RESERVED_KEYS` carried the claim "no new keybinding can silently take a reserved key," while the gate only checked "these two keys of this feature don't collide." A change that later takes a reserved key left **all 27 tests green**.

> **Describing a property is not witnessing it. Writing it down is an act that makes it findable, not an act that makes it true.**

The question that actually worked as a detector:

> **Is this named table the "subject" of an assert inside the tests, or merely "material" used to build some other set?**

`RESERVED_KEYS` was **read** inside the tests, so it looked wired up. But nothing was asserted **about** it. Two corollaries:

- **The uncovered property tends to be the more general one.** "Any binding vs any reserved key" is broader than "these two keys vs everything." You write the broad one in prose and assert the narrow one.
- **The table itself must also be non-vacuous.** An empty `RESERVED_KEYS` makes `not (reserved & live)` trivially true. Add one line: `assert reserved`.

## Form 5 — Geometry (containment) gates have axes and hierarchy

A geometric claim like "nothing overflows the screen" cannot be expressed as a single inequality. There is a record of the same hole slipping through four times, in different shapes, across three incidents:

| What the gate was looking at | The path that slipped through |
|---|---|
| `height > 0` and `display == True` | Both stayed True while the element was **pushed off-screen** (y = −8). "Exists" is not "visible" |
| Only the **upper bound** of containment (`y + h <= H`) | A **negative y** sailed through. Both ends are needed (`y >= 0` and `y + h <= H`) plus a squashed lower bound |
| The width of the outer region | The consumption happened **inside**; the outer width never changed. **Wrong surface measured** |
| Parent only, horizontal axis only | The parent fits but a **child element** is off-screen. x is correct yet the element is clipped invisible in the **vertical** direction |

The rule:

> **Take containment down to "both axes × both ends × the actual child elements you want visible."** The parent fitting does not mean the children are visible, and fitting on one axis does not mean being non-invisible on the other.

- Before choosing the surface to assert, identify **where the consumption happens** (the outer region or the internal layout).
- **Parametrize over at least 2 sizes**, and always include the dimensions where breakage was actually measured. A single-size containment gate claims nothing beyond "it fits at that width."
- **When a strip comes back green, suspect the gate first.** It is more likely you measured the wrong surface, axis, or depth than that the implementation happened to be correct.
- Exception: for gates on "derived values" the strip expectation can be **inverted** (if the label width is derived from the longest label, then shortening a label and seeing no breakage is the correct outcome = green is the expected result of that strip). **Never count a green-is-expected strip as evidence of non-vacuity.**

## Do not rely on weak detectors — divergence between siblings is luck

A discovery episode from practice: out of 5 strips, exactly 1 came back green; the implementer flagged that as anomalous, chased it down, and found a real hole. But this detection **only works when siblings diverge**. If the whole family is unreachable, everything goes green and fails silently. The thing to generalize is not anomaly detection but a **reachability assertion** — have the gate itself carry "was this leg really executed?"

## Three questions before writing or reviewing a gate

1. Are you looking only at the terminal state? Is there an **intermediate cross-section**?
2. Does the positive control go through the **same path**?
3. Did you **measure RED on a defective version**?

Asserts of the form "X does not happen" are especially dangerous: they go green when the production path for X is merely dead (same root as [Liveness is decided by the producer](liveness-is-producer.md)).

## Sources (measured during reyn development)

Forms 1–3: #3288/#3299 (three in a row in one session: layout, reachability witness, terminal-only assert). Form 4: #3358 (borrow/return asymmetry), #3363 (RESERVED_KEYS, 27 passed on the takeover strip). Weak-detector self-report: #3370. Form 5: #3311 → #3337 → #3341 (the same hole three times; in one of them a green strip was read as "well, it passed, good enough" and a second path of the same shape was shipped as-is).

## Related

- [The discipline of strip-falsification](strip-falsification.md) — the general procedure for "measure RED on a defective version"
- [Wiring tests vs mechanism tests](wiring-vs-mechanism.md)
- [Structural blindness of the verification environment](environment-blindness.md) — the form where the gate's "environment" itself cannot surface the defect
