---
name: strip-falsification
description: The four axes of strip-falsification — a report saying "I confirmed RED" is not evidence until you know how many sites were stripped, what exactly was broken, whether the strip actually landed, and whether killing the whole mechanism goes RED
tags: [verification, testing]
sources:
  - feedback_strip_falsify_all_sibling_guard_sites_not_just_one
  - feedback_witness_must_assert_a_value_a_dead_mechanism_cannot_produce
  - feedback_bound_test_must_flip_under_strip
---

# The Discipline of Strip-Falsification — Four Axes That Turn a RED Report into Evidence

## What strip-falsification is

**Strip-falsification (strip-falsify)** is a procedure for checking whether a test really protects the mechanism it claims to protect. The idea is simple: **temporarily remove (strip) the mechanism that is supposedly protected, and confirm that the test fails (goes RED)**. If the test stays green (GREEN), it is not protecting the mechanism.

Start with a minimal example. Suppose there is a function that rejects uploaded files larger than 1MB:

```python
MAX_BYTES = 1_000_000

def save_upload(data: bytes) -> str:
    if len(data) > MAX_BYTES:          # <- this is the guard we want to protect
        raise ValueError("too large")
    return store(data)
```

Against this, the following test looks reasonable at first glance but **does not protect the guard**:

```python
def test_save_upload():
    assert save_upload(b"hello") == "stored"   # only checks that a small file goes through
```

The way to find out is strip-falsification. Delete the two lines starting at `if len(data) > MAX_BYTES:` and run the tests. **Everything stays green.** In other words, if someone accidentally deletes this guard in the future, CI will tell you nothing. A test that actually protects it looks like this:

```python
def test_save_upload_rejects_large_file():
    with pytest.raises(ValueError):
        save_upload(b"x" * (MAX_BYTES + 1))    # if the guard is removed, this line raises nothing and the test goes RED
```

So far this is textbook material. The problem is that when you run this procedure in practice — especially in an environment where AI coding agents write both the implementation and the tests — **the report "I stripped it and confirmed RED" is itself repeatedly broken**. The ways it breaks can be organized along four axes.

## Axis 1 — How many did you strip? (coverage of sibling sites)

When the same guard is applied at **N sites**, implementers tend to strip only one representative site and report "RED confirmed."

Example: suppose the 1MB check above appears in three places — "via the web form," "via the API," and "via batch ingestion." The tests only exercise the web-form path. Stripping the API-side check then stays **green** (green-on-strip). If a future edit deletes the API-side guard, nobody notices.

- **Enumerate every application site of the guard** (grep for the helper function, the field, the relevant lines). Then confirm that **stripping each site goes RED**.
- The criterion is not "the mechanism is witnessed" but "**every site that depends on the mechanism is witnessed**." One RED plus "the code exists" for the remaining N−1 sites is incomplete.

> **Terminology**: a witness is a test that goes RED when the property in question is broken. "There is a witness" = "breaking it will be detected."

## Axis 2 — What did you break? (minimal breakage)

"I reverted to the original code and it went RED" **breaks multiple properties at once**, so it isolates nothing.

Example: suppose a PR (a) added escaping for file names and (b) deleted an old validation that rejected paths containing `__`. The implementer reports "reverting the whole PR → RED." But that RED might come from the validation that the revert *restored* in (b) throwing its exception again. **It is not evidence that the test reacted to the property the PR actually wanted to protect (collision avoidance via escaping).**

- The criterion is: "**break the single property to be protected, by itself alone, and see RED**." In this example, construct the **most dangerous intermediate state** a future edit is actually likely to produce — "remove only the escaping, leave the validation deletion in place" — and observe RED there.
- When delegating, name the target of breakage explicitly. Not "strip the fix" but "**remove only the escaping; keep everything else**." The closer the strip is to "undoing the PR," the weaker its evidential power.

## Axis 3 — Did the strip land? If it stayed green, did you pin down why?

Axes 1 and 2 implicitly assume that the strip itself succeeded. That assumption itself often breaks. There is a real case where it broke three times in a row within a single verification:

1. A sed replacement was supposed to be applied, but **the pattern did not match, so the replacement count was 0** (applied=0) — no experiment happened at all.
2. On retry, a **syntax error** was introduced, producing RED — but this RED means "**the experimental apparatus was broken**," not "the mechanism was broken."
3. On the third attempt the strip finally landed — and this time the result was **green**.

And reading the green in step 3 as "this guard was unnecessary" is premature. **Green has at least three possible meanings**:

1. The guard really is unnecessary.
2. **It is subsumed by another guard.** Example: downstream of an expression like `effective = min(a, b)`, a check for `b <= 0` is always contained within a check for `effective <= 0`, and an input that makes it go RED on its own may be **impossible to construct in principle**. In that case, **write it down in the code**: "coverage by subsumption, not a missing witness." If you don't, the next person will come to fix a witness gap that does not exist.
3. **The strip did not land / something else broke** (the experiment failed — same as 1. and 2. above).

- When delegating, do not stop at "confirm RED." Also write: **"confirm the replacement was actually applied (show the diff or the replacement count)" and "if it stays green, report it — green is not a conclusion, it is the next question."** Do not let the report stop at "it was green, so apparently it has no effect."

## Axis 4 — Does killing the whole mechanism go RED?

**Even when axes 1–3 are all satisfied, the mechanism as a whole may still be protecting nothing.** Individual strips show "this line has an effect"; they do not show "the mechanism is doing its job."

Real case (generalized): a speedup mechanism that saves computation results to a checkpoint and recomputes only the delta on the next run. The implementer stripped five sites individually, verified the replacement counts, and reported RED for each (axes 1–3 cleared). Yet when the reviewer **killed the entire mechanism with `if False`, all seven tests stayed green**.

Why? What the tests asserted was "the final computed result is correct" — and that value is **the same in a world where the checkpoint is dead and everything falls back to full recomputation**. The discriminant is:

> **A witness is vacuous unless it asserts a value that a dead mechanism cannot produce.**

Tests that assert the result of a fallback (full recomputation, default values, a recovery path) are structurally trapped this way. Any arm that asserts the fallback side must be paired with an arm that asserts "**a value that appears only when the mechanism is alive**" (e.g. that the recomputation count is 0, or a cache-hit counter).

- For designs that "**keep working even when broken**" — recovery / fallback / cache / fast paths — always perform a whole-mechanism kill (turn it into a no-op, `if False`) and confirm RED.
- "RED under individual reverts" is not a substitute for "RED under whole-mechanism kill." The case above is exactly one where every individual strip passed while the whole-mechanism kill stayed green.

## Corollary — bound / cap / threshold tests must flip

For changes that introduce an upper limit, a truncation, or a threshold, writing a happy-path test often means **the same assert passes even in an implementation with the bound removed**.

Example: suppose you introduce a read cap — "read only the first 4KB of the version file."

```python
# Bad test: token at the start + 8MB of junk after it
write_file("3.12.7\n" + "x" * 8_000_000)
assert resolve_version() == "3.12.7"   # an unbounded read also finds the leading token -> green in both implementations
```

The correct construction **pushes the token outside the cap so the assert flips between the two implementations**:

```python
# Good test: 5000 spaces push the token past the 4KB boundary
write_file(" " * 5000 + "3.12.7\n")
assert resolve_version() is None       # bounded: token not visible, falls back -> green
                                       # unbounded: finds "3.12.7" -> RED
```

Before pushing, actually remove the cap (e.g. replace the cap constant with a huge value, effectively disabling it) and reproduce the RED.

## Checklist (when delegating and when reviewing)

- [ ] Did you enumerate the guard's application sites? Does stripping each site go RED? (axis 1)
- [ ] Did you break "only" the property to be protected? Did you construct the most dangerous intermediate state? (axis 2)
- [ ] Did the strip land (replacement count, diff)? If green, did you identify which of the three categories it is? (axis 3)
- [ ] Does a whole-mechanism kill go RED? Does every arm asserting the fallback have a paired arm? (axis 4)
- [ ] For bound-type changes, does the assert flip between the two implementations?

## Sources (measured during reyn development)

The rules in this document were extracted from real verification failures in the multi-agent development of reyn, an AI agent runtime whose development is done almost entirely by a fleet of coding agents. Axis 1: #2900 (a reset line was green-on-strip) and #2903 (two of three stampers were unwitnessed). Axis 2: #3159 (a broad revert could not prove a reaction to injectivity). Axis 3: #3189 (applied=0 → syntax-error RED → green by subsumption, three in a row). Axis 4: #3195 (all four arms green and every individual strip cleared, yet the whole-mechanism kill also stayed green — an implementation that silently reset the budget cap was nearly merged). Bound flip: #2825 (4KB read cap).

## Related

- [Integrity of the strip instrument](strip-instrument-integrity.md) — if the strip's anchor or the execution environment is broken, the conclusions of these four axes break with it
- [How vacuous gates are born](vacuous-gates.md)
- [Wiring tests vs mechanism tests](wiring-vs-mechanism.md)
