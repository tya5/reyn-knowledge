---
name: audit-content-match
description: Audits match content, not existence — a symbol's existence is not evidence that it runs, a matching line number is not evidence of the claim, and documentation statements are verified against the implementation before being mirrored
tags: [verification, audit, documentation]
sources:
  - feedback_audit_status_not_existence
  - feedback_drift_audit_content_match_not_line_existence
  - feedback_doc_mirror_claim_needs_impl_verify
---

# Audits Must Match Content — 'It Exists' Is Neither 'It Works' Nor 'It Says That'

## The shape of the problem

Feature inventories, audits of drift between documentation and implementation, mirroring text from one document into another — these tasks share a common failure: **treating "I found something that matches" as evidence that "the claim is true."** Checking for existence is cheap and tempting, but what an audit actually needs to verify is not existence — it is **content**: does it behave that way, and does it actually say that there?

## Shape 1 — A symbol's existence is not evidence that it runs

In a feature-inventory audit, there is a real case where "external channel integration (Slack, etc.)" was classified as "implemented." The evidence looked sufficient: a forwarding module **exists**, a routing class **exists**, even a configuration block **exists**. But the source said, in plain text: "**Wiring is intentionally NOT included in this PR; it will come in a future PR.**" All of it was scaffolding — skeleton code put in place ahead of a future implementation.

> **The existence of a symbol (class, config, file, CLI entry) does not mean the feature is wired up and working. Scaffolding normally ships before wiring.**

An existence grep takes one command; confirming operational status (STATUS) requires reading the implementation. That asymmetric cost is what degrades audits into existence checks. And listing scaffolding as a production feature produces **exactly the drift — documentation running ahead of the implementation — that the audit was supposed to prevent**.

**The three-part STATUS pass:**

1. Grep near the symbol for stub markers: `future` / `Phase 2` / `NotImplemented` / `TODO` / `NOT included`
2. Confirm there is a substantive (non-stub) implementation body
3. Confirm there is a **live call site or integration point** (the same axis as [Liveness is decided by the producer](liveness-is-producer.md))

The verdict is not a binary "implemented / not implemented." Classify into "in production / experimental / behind an option / conceptual (excluded from the list)," and surface borderline cases **as review decisions** rather than silently including or dropping them.

### Corollary — counting is not inspecting

In the same audit, `ls -d */ | wc -l` returned 15, which was reported as "15 skills." Inspecting each directory showed that three of them were cache-only empty directories; the real count was 12. **Extrapolating from a count is no substitute for inspecting each item** ([The discipline of enumeration](enumeration-discipline.md)).

## Shape 2 — A matching line number is not evidence of the claim

When auditing a statement of the form "`file.py:291` has this behavior" (a file:line citation), do not stop at confirming **that something is on that line**. Read far enough to confirm **that the content of that line actually embodies the claimed meaning**.

Real case: while auditing the claim "relative path resolution is at `file.py:291`," a grep for `file.py` across the repository returned a result **truncated to the first two hits**; line 291 of one of those files **did contain** `def glob(...)`, so the auditor confirmed it as "matches — correct." In fact there were a third and a fourth `file.py` elsewhere, and the claim pointed at one of those. The glob definition at line 291 of the first file was **merely a construct of the same kind that happened to be there**.

- For ambiguous file names, **enumerate all candidates before** judging (read `git ls-tree -r --name-only | grep '/file\.py$'` to the end; do not cut it off with `head` — layer 1 of [The discipline of enumeration](enumeration-discipline.md)).
- Treat "a construct of the same kind exists on that line" with suspicion, recording your reason for rejecting or accepting it. The test is whether that line implements **the specific behavior that was claimed**.
- If the other party disputes your audit finding with primary evidence, **re-take the reading against the currently canonical version** before defending your result.

## Shape 3 — Verify documentation statements against the implementation before mirroring them

Before **mirroring** an implementation-dependent claim written in a conceptual document into another document ("X is isolated," "this happens atomically," "unique per scope," "runs exactly once"), verify it against the primary source — the implementation. Conceptual documents sometimes overclaim, and copying without verification **propagates and amplifies the error**.

- Mirroring any sentence involving isolation, atomicity, idempotency, scope, count, or ordering comes with a grep of the relevant implementation.
- The review axis is not "is the documentation (internally) consistent" but "**does the documentation match the implementation**."
- When the text and the implementation disagree, **treat the implementation as canonical and fix the documentation** (do not trust the documentation and bend the implementation to it).

This discipline **closes the entrance** to the chain in [Census vs structure](census-vs-structure.md) where "false statements are fossils of incomplete enumeration": unverified mirroring is precisely the path by which the fossils multiply.

## How to apply

- [ ] For each "found it → recorded as implemented" step, did you run the three-part STATUS pass (stub-marker grep / substantive body / live call site)?
- [ ] Before reporting a count, did you inspect each item individually?
- [ ] For file:line citations, did you read whether the **content** of the line embodies the specific claimed behavior? Did you enumerate all candidates with the same file name?
- [ ] If a doc-to-doc mirror includes implementation-dependent claims, did you grep the implementation before mirroring?

## Sources (measured during reyn development)

Shape 1: #1522 (feature-inventory audit; the owner caught "wiring intentionally not included" in the source before landing. The same PR also had the 15-directories → 12-real extrapolation). Shape 2: a file:line audit where grep head-truncation led to confirming line 291 of the wrong `file.py` as a "match"; a peer corrected it with primary evidence. Shape 3: the implementation-verification discipline for doc-to-doc mirroring (2026-07).

## Related

- [Liveness is decided by the producer](liveness-is-producer.md) — the general form of "records of intent vs records of behavior"; this document is its audit variant
- [The discipline of enumeration](enumeration-discipline.md) — enumerate all candidates, no `head`, no extrapolating from counts
- [Census vs structure](census-vs-structure.md) — the chain by which unverified statements become the next reader's premise
