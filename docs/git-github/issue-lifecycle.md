---
name: issue-lifecycle
description: Issues decay — the body is a snapshot from filing time, and resolution accumulates in the thread and in merged PRs. Covers pre-dispatch cross-checking, verification records at close, settling remainders ("next arc" is a third state), and filing calibration (in both directions)
tags: [git-github, issues, process]
sources:
  - feedback_crosscheck_merged_prs_for_stale_done_issues
  - feedback_crosscheck_merged_prs_before_explaining_dispatching_arc
  - feedback_dispatch_brief_must_reflect_issue_comment_thread_not_just_body
  - feedback_issue_close_requires_condition_verification_record
  - feedback_track_deferred_work_before_close
  - feedback_arc_closure_remainder_must_be_filed_or_explicitly_dropped_in_the_closing_comment
  - feedback_open_issue_progress_continuous_update
  - feedback_open_ticket_count_is_maintenance_cost_prefer_do_over_file
  - feedback_investigate_before_filing_issue
  - feedback_over_cautious_issue_creation
  - feedback_unreproducible_bug_close
  - feedback_task_tracker_id_vs_github_issue_number
---

# Issues Decay — Disciplines for Bodies, Threads, Closure, and Remainders

## Premise

In agent-driven operation, the issue tracker becomes **the substance of task management** (the open/close state of issues decides who does what). And the information in an issue **decays over time in a fixed pattern**: **the body is a snapshot from the moment of filing**, while the later measurements, the true root cause, and the PR that fixed it accumulate **on the side of the comment thread and the merged PRs**. Forget this asymmetry and you mass-produce waste: re-implementing problems that are already fixed, or explaining a completed arc as "about to start."

## 1. Before dispatch or briefing — don't read only the body

**stale-done (fixed but still open)**: even when a merged PR names `fix(N):` or `Fixes #N`, [depending on the wording it does not auto-close](closing-keywords.md), so **resolved issues stay open**. Measured: a review of 16 open issues based on their bodies alone concluded "all still valid" — in fact 2 were already implemented, and one of them was **moments away from being dispatched for re-implementation**.

- Before triaging or dispatching open issues: **cross-check against merged PRs** with `gh pr list --state merged --search "<N> in:title,body"`.

**stale-premise (the premise has been overturned)**: there is a measured case where, after briefing the owner on an arc of multiple issues, getting a GO, and dispatching, it turned out that **the core of the arc had already landed in a separate set of PRs** (one unnecessary PR got merged). A GO only means something for work that is genuinely not yet started.

- Before briefing or dispatching an arc: **state each issue's core premise in one sentence** and check against the merged-PR history and current main whether it **is still true**.

**Body vs. thread divergence**: a dispatch brief was written from the body's root-cause analysis, and the implementer, reading **the thread**, discovered: (1) the body's hypothesis had been overturned by later measurements, (2) the real hot path was elsewhere, (3) **it had already been fixed by a merged PR**. The implementer correctly refused the band-aid.

- Before writing a brief, read **the entire thread** with `gh issue view <n> --comments`. When the body and the thread disagree, **the thread — the newer primary data — wins**.

## 2. Closure discipline — a closed issue must be self-documenting

An issue is a unit of audit. A closed issue must be able to say **why it is done** on its own:

- **Closure record**: leave a final-state report plus a verification record for the closure conditions (a checklist of the conditions + a ✓ for each + references to the evidence) **on the issue itself**.
- **The auto-close blind spot**: with a merge-linked close via `Closes #N`, **the verification record exists only on the PR side**. Add a verification comment to the issue after the merge (skip it, and anyone looking only at the issue sees "it just closed by itself").
- **Immediately before closing, re-read the full original filing** and check that the entire scope (every sub-request) is satisfied. If completion is partial, annotate instead of closing.
- **Approval branches on the kind of close**: a proper-close of implemented-and-verified work may proceed autonomously (excessive asking-for-approval is itself prohibited). A close that **folds the issue without implementing** ("measured it, no effect", "moving it out of scope") requires **the owner's prior permission** — because it is a decision to abandon the work.

## 3. Settling remainders — "next arc" is a third state

When closing an arc (a coherent batch of work) or an umbrella issue (a parent issue bundling multiple PRs):

> **In the closing comment, settle every remainder as either "filed (write the #number)" or "explicitly dropped (write the reason)". "In the next arc" and "will file it once it gets a priority" are a third state — and that third state is the decay itself.**

A **natural experiment** remains on one such issue: of two remainders closed on the same day in the same arc, the one that **was written** into the closing comment can still be traced today, while the one that **wasn't** decayed into a "task item of unknown origin" whose content survived in no one's memory.

- A deferred item that merely says "will file later" is **effectively ticketless** = invisible. File the real issue at the moment of closing, with a forward-pointer on the closed side and a back-ref on the new issue.
- **Verify-before-file**: confirm with primary evidence that the remainder **actually exists on current main** (there is a measured near-miss of almost filing an issue after misreading a grep hit on a notice announcing "already removed" as an unaddressed remainder).
- A deferred section in a design record (ADR) works as a **greppable catch-basin** — pointing there is stronger than carrying items of unknown granularity.
- Finding a **ticketless item** on your own task list is a red flag: decide now to file it or drop it. Left alone, it persists indefinitely as "the item that somehow never finishes."

## 4. Reflect progress at every landing

"I'll batch-update later" leaks at batching time (measured: the same oversight repeated three times in a single session).

- When a PR that partially or fully resolves an issue gets merged, write "landed: … / remaining: …" on the issue **right then**. If complete, close it.
- When a foundational change lands (a component removal, a large refactor), **sweep every open issue that references it** and immediately annotate/close the obviated ones (premise gone).
- In a multi-session arc, maintain **a single STATUS comment on the umbrella issue** (✅ done / 🔄 in progress / 📋 remaining) and append at every PR landing. And **when asked for status, read that record before answering — not memory**: this discipline originates from twice mis-answering, from working memory, that a completed arc was "not started."

## 5. Filing calibration — errors go in both directions

**Under-verification (don't file on speculation)**: an issue is an external audit surface that every other session takes at face value. Before filing: did you **directly observe** the symptom (quote the log line, commit, job ID)? / do you have a reproduction command? / have you ruled out that it is **intended behavior**? / does the scope fit in a single issue? / can a first-time reader understand how to fix it and how to verify it?

**Over-retention (open tickets are maintenance cost)**: open tickets are a **real cost**, re-read and re-triaged at every inventory pass.

- **Do over file**: a follow-up with no blocker and a settled resolution should be **finished now**, not ticketed. File only what genuinely must be deferred (waiting on a GO, on design, on external conditions) and deserves tracking. Never file "might want this someday."
- Periodically cross-check merged PRs against open issues and close the stale-done ones with verification records. **A wrong close is worse than a missed one** — if completion is uncertain, leave it open.

**Over-escalation (don't put everything on approval hold)**: the N+1th application of an existing pattern, or the natural next step of work already in flight, is within autonomous range (measured owner verdict: "waiting for approval on something at this level is soft"). What should escalate: major architecture changes, establishing new policy, entirely new axes, major user-facing UX changes, cost overruns, security-posture changes, design conflicts between authors.

**Don't chase unreproducible bugs**: something observed once that does not reproduce after N≥3 retries on the current HEAD gets closed with an explicit note — "ghost bug; reopen if observed again" (if it partially reproduces, track it separately as flaky). Chasing ghosts melts multiple people's cycles.

## 6. ID collisions — internal trackers vs. GitHub numbers

Session-internal task-management IDs (#187 and the like) and GitHub issue/PR numbers point to **different things in the same shape**. A recipient reads #N as GitHub (the natural interpretation). Measured: work was assigned by an internal ID, and the recipient verified and discovered "that GitHub number is an unrelated merged PR."

- **Refer to internal-tracker items by topic name; use #N only for GitHub.** On the receiving side too, confirm that #N resolves on GitHub before acting.

## Checklist

- [ ] Before dispatch: did you cross-check against merged PRs, state each premise in one sentence and verify it, and read the full thread?
- [ ] At close: did you leave a verification record on the issue itself? Did you check against the full original filing? Did you get permission for a close that folds work without implementing?
- [ ] Remainders: settled as filed (#number) or explicitly dropped (reason)? Did you write "in the next arc" anywhere?
- [ ] Did you reflect every landing onto the issue? Did you read the record before answering a status question?
- [ ] Before filing: primary evidence, reproduction, intended behavior ruled out. Could you finish it now instead of filing? Does the escalation fall under one of the seven kinds?
- [ ] Did you use #N only for GitHub?

## Sources (measured during reyn development)

stale-done: #1406/#1407 and #1206/#1291. stale-premise: the arc-187 (internal label) capstone (unnecessary PR #1435 merged). Thread over body: #2940 (the body's hypothesis was already overturned and fixed; the implementer refused). Closure records: owner directive 2026-06-19 + retroactive backfill of 5 issues; content-cancel violation #1791. Remainders: #1115 (unfiled deferred → #1199/#1200 filed retroactively), #2597 (the natural experiment: the written vs. the unwritten remainder), and the verify-before-file that stopped a false-positive filing (2026-06-01). Progress reflection: #1375/#1397/#1401 (3 consecutive misses), the two wrong answers on the Control IR arc (owner re-emphasis 2026-07-04). Calibration: the owner's four-message run of 2026-07-10 (open-ticket pileup), #1010 ("soft"), the ghost-bug close directive (2026-05-28). ID collision: 2026-06-06.

## Related

- [Closing keywords fire on surface text](closing-keywords.md) — the auto-close mechanism itself
- [Incomplete work delays discovery](../verification/incomplete-work.md) — the general principle of making remainders visible (this document is its issue-side face)
- [Argument hygiene](../verification/argument-hygiene.md) — "not urgent" and "not decided" are different axes
- [Audits must match content](../verification/audit-content-match.md) — the general form of checking records against reality
