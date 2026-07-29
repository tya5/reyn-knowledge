---
name: incomplete-work
description: Incomplete work (partial completion, silent deferral) delays defect discovery — audits scope the full population, holes found get filed in the same session, and remainders become a visible checklist
tags: [verification, process, completeness]
sources:
  - feedback_incomplete_work_delays_defect_discovery
---

# Incomplete Work Delays Defect Discovery

## The shape of the problem

What is actually wrong with partially finished work, or work silently deferred? "We'll finish it eventually, so isn't it the same?" — No. **Incomplete work creates a state that *looks* working, which stretches out the time until a defect surfaces (discovery latency).**

A real case: documentation was written first, describing the ideal behavior (a "4-step confirmation flow"), and was left in place while the implementation never caught up. The actual behavior was a plain rejection. This divergence went unnoticed **for about a week, until someone happened to hit it in everyday use**. Neither the documentation nor the implementation looks broken on its own — only the divergence is the defect, and the divergence stayed invisible for as long as it was left alone.

This is kin to "unchanged files are indistinguishable between 'inspected' and 'never looked at'" from [Reviewing sweep PRs](sweep-reviews.md): **from the outside, incomplete work makes 'not done yet' indistinguishable from 'done'.**

## The discipline

### 1. The scope of an audit is the full population

Forbid "audit only the high-priority subset and drop the broad sweep." Doing the priority items first as a matter of ordering is fine, but **the broad sweep must also happen**. An audit that does not look at the full population is the same truncation as in [The discipline of enumeration](enumeration-discipline.md), performed at the process level.

### 2. Holes you discover get filed in the same session

Forbid "I'll open an issue later." When a gap opens between discovery and filing, the filing very likely evaporates — and a discovery that was never filed is **as good as nonexistent until the next person happens to hit it by accident**. Likewise, whenever you find a deferred item that has no tracking issue, file one on the spot.

### 3. If you cannot finish, leave a visible checklist of what is done and what remains

Not finishing everything within a session is normal and not the problem. The problem is the **silent drop** — afterwards, nobody can trace how far things got and what is still open. Leave a checklist that enumerates completed items / remaining items — a form whose structure makes completeness visible.

### 4. Fixes must enumerate every symmetric surface

If you fix one instance of a validation gate, **grep for every gate of the same kind and check them for the same hole**. A partial fix of the "fixed the file side, left the shell side alone" variety means that from the moment of the fix, the remaining siblings are shielded by the false impression that they were "apparently fixed." This is the implementation-side counterpart of the duty in [Fix-class review](fix-class-review.md).

### 5. Build redundancy into the filing path

Completeness of discovery and availability of the filing path are separate problems. So that a malfunctioning issue tracker (expired credentials, etc.) does not become a single point of failure for "track everything," **agree in advance that if the discoverer cannot file, someone else files on their behalf**.

## How to apply

- [ ] Does the audit/sweep plan include a full-population broad sweep (not a plan that ends with just the priority items)?
- [ ] Did you file every gap you found in this session, now?
- [ ] If ending with work unfinished, did you leave a completed/remaining checklist somewhere visible?
- [ ] Did you grep-enumerate the symmetric surfaces of what you fixed (sibling gates, parallel implementations)?
- [ ] Is there an agreed fallback path for when the filing path is blocked?

## Sources (measured during reyn development)

#1505/#1317: documentation-first ideal description left with the implementation lagging behind; invisible for about a week until hit by accident in everyday use. Owner's strict order (2026-06-12): "Leave nothing unfinished. Last time you left things unfinished on your own, discovery was delayed."

## Related

- [Reviewing sweep PRs](sweep-reviews.md) — the invisibility of "what was not done" (review side)
- [The discipline of enumeration](enumeration-discipline.md) — defining the full population and forbidding truncation
- [Fix-class review](fix-class-review.md) — full enumeration of same-class defects (review side)
- [Claims without context](cross-context-claims.md) — the duty to declare "landed a simplified version" up front
