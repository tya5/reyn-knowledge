---
name: shared-helper-widening
description: Folding N inline checks into one shared helper silently widens semantics when the helper is more opinionated than the "least-opinionated caller" — no one asserts the negative, so tests never catch it
tags: [verification, refactoring, review]
sources:
  - feedback_shared_accessor_must_not_outopinion_least_opinionated_caller
---

# Shared-Helper Widening — Consolidate to the Most Cautious Caller

## The shape of the problem

A refactoring staple is "fold N inline checks into one shared helper" (building exception tuples, config lookups, predicate builders). The rule to observe when doing so:

> **The helper must not be more opinionated than the least-opinionated caller.** Otherwise, the semantics of a caller that wanted the narrow set silently **widen**.

## A real case

Two callers were using the exception classes of a lazily imported HTTP library:

- Caller A (retry decision): wants to catch **both** of `(ConnectError, ReadTimeout)`
- Caller B (timeout classification): wants to catch **only `ReadTimeout`**

If you now design it as "the shared helper returns the tuple `(ConnectError, ReadTimeout)`," the diff is minimal and it lands with **every test still green**. But from that moment on, at caller B, **`ConnectError` (a connection failure) is classified as a "timeout"** and routed into timeout-specific policy (automatic extension, interactive confirmation). The semantics widened silently.

In this example, caller B — the one that wants only the narrow set — is the **least-opinionated caller**. The helper must not be more opinionated than B (must not insist on catching both); that is what the title means.

Why does it survive review?

> **Because no one asserts the negative.** A test saying "ConnectError is **not** a timeout" does not normally exist. The green suite is evidence of nothing — **the missing test is the problem itself**.

## The shape of the design

The fix actually adopted has the helper **return the components separately and let each caller select its own subset**:

```python
def _get_httpx_exc_types():
    import httpx
    return httpx.ConnectError, httpx.ReadTimeout

# Caller A (retry decision): uses both
connect_err, read_timeout = _get_httpx_exc_types()
retryable = isinstance(e, (connect_err, read_timeout))

# Caller B (timeout classification): selects the narrow side itself
_, read_timeout = _get_httpx_exc_types()
is_timeout = isinstance(e, read_timeout)
```

The return value has the same tuple shape as the rejected design — what differs is the **contract**: the helper only lays out the raw materials, and each caller unpacks and selects its own subset.

## How to apply

- **Read every call site before designing the helper.** Fit its shape to "the union of what the callers need to distinguish," not "the union of what the callers consume."
- When reviewing a PR of this kind, ask about each folded site: **"Did this site previously distinguish something that the helper now paints over?"**
- **Write the reasoning in the helper's docstring, not the commit message.** The docstring is what the next person — the one who comes to add a third caller — will read.

## Sources (measured during reyn development)

#2947 (consolidation of the lazy httpx import; under the pre-built shared-tuple proposal, ConnectError would have been treated as a timeout with every test staying green).

## Related

- [Fix-class review](fix-class-review.md) — the danger of consolidation always lives in the call sites it flattens
- [Census vs structure](census-vs-structure.md) — the census premise "this is how today's callers use it"
- [How vacuous gates are born](vacuous-gates.md) — the same family of hole: the negative assert does not exist
