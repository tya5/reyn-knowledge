---
name: stall-triage
description: A triage procedure for agents that look stalled. Your own last message → machine declaration → three-point measurement (PR/worktree/ps) → attribution → duration cross-check → report as verified facts. Source docs/orchestration/stall-detection
---

# stall-triage — Sorting out "it looks stalled" in five minutes

**When to use**: a peer you dispatched to has gone silent / a report is stuck at "waiting on tests" with no visible motion / the watcher escalated an idle candidate.

Premise: an agent's silence is ambiguous (quietly working / waiting on the LLM / waiting on context compaction / waiting on a dead background job / genuinely stopped). The checks below are ordered **cheapest first**.

## Step 1 — Re-read your own last message (the cheapest check)

- If the peer is plan-first (no implementation until approval): did you write your GO in the words of a decision? If it was in a "we recommend" tone, **the peer is waiting — correctly**. That is not a stall. Rewrite decisively and resend.
- Did you leave one of its questions unanswered? Did the brief contain an impossible instruction (e.g. a foreground run that cannot finish under the default timeout)?

## Step 2 — Check the machine declaration directly

- Look at the session's mechanical active/idle **declaration (a bool)**, not the LLM-written status string ("waiting on CI," etc.) — **a repeated identical string is a staleness signal, not evidence of activity**.
- If the declaration says active, keep open the possibility that it is just quiet during a long, correct computation. Go to Step 5.

## Step 3 — The three-point measurement (never guess)

```bash
gh pr list --author <agent>            # 1. is the work absent from the open PRs?
git -C <worktree> status --porcelain   # 2. uncommitted diffs present?
git -C <worktree> log --oneline -3     #    any new commits?
ps aux | grep <the awaited process>    # 3. is the process alive?
```

## Step 4 — Attribute the cause from the results

| Measurement | Attribution | Next move |
|---|---|---|
| No PR + uncommitted diffs + no process | **Waiting on a dead background job** (most frequent) | Nudge (a light check-in) and have it commit to recover the work |
| Even one process alive | Legitimately in progress | Don't rush it. Only cross-check the duration in Step 5 |
| No diffs, no process, active until recently | Absorbed without pushing, or waiting on compaction | Nudge first and give it time to reply. Restart is the last resort |
| Local commits exist but nothing pushed | False stall (merely invisible) | Put "push at every milestone + one-line status" into the next brief |

## Step 5 — Cross-check "currently running" against the known duration

- Still-running beyond **several times to 10x** the task's normal value is treated as hung, not slow.
- The normal value drifts over time (suites grow) — cross-check by **ratio**, not by absolute value.

## Step 6 — Report as verified facts

- Not "presumably implementing" but "**last activity HH:MM, no PR yet = progress unverified**."
- Never relay "running, so on track" as is (only after Step 5).

## Background

The source incidents for each step (six dead-job waits in one night, the 44-minute hang relayed as "on track," 10 hours of silence misattributed, and more) are in [Detecting Stalled Agents](../../docs/orchestration/stall-detection.md).
