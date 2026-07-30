---
name: stall-detection
description: How to find stalled agents — silence is ambiguous (quiet work, waiting on the LLM, waiting on compaction, waiting on a dead job, genuinely stopped). The most frequent stall shape and its three-point measurement, cross-checking against known durations, separating machine declarations from inference, fan-out and fan-in as a pair
tags: [orchestration, multi-agent, monitoring]
sources:
  - feedback_coder_background_pytest_poll_stall_pattern
  - feedback_running_task_overrun_known_duration_is_hung_not_slow
  - feedback_sonnet_peer_stalls_at_compaction
  - feedback_peer_silent_local_work_invisible_to_watchers
  - feedback_repeated_status_text_requires_active_flag_check
  - feedback_stall_idle_session_declared_not_broker_inferred
  - feedback_verify_peer_progress_not_assume_dispatch
---

# Detecting Stalled Agents — Silence, Self-Reports, and Machine Declarations

## Premise

When you run multiple agent sessions concurrently (a fleet), the central operational problem is that **both "stalled but looks like it's working" and "working but looks stalled" happen routinely**. An agent's silence is ambiguous, and the outside view cannot distinguish the cases: quietly working / waiting for an LLM response / waiting on compaction (summarizing a long conversation history) / **waiting on a dead background job** / genuinely stopped. This document is a discipline of the stall shapes that were actually measured, and of how to tell each one apart.

## 1. The most frequent stall shape — waiting on a dead background job

**The pattern** (measured: six occurrences in a single night): an implementation agent launches the full test suite **in the background** and **ends its turn waiting for the completion notification** → the background job dies silently (or the completion notification never arrives) → the agent stays "waiting" forever, **stalled indefinitely with the work uncommitted and no PR opened**.

**Detection is a simultaneous measurement of three points** (never a guess):

1. Among the open PRs, the work in question is **absent**
2. In that agent's worktree, uncommitted diffs are **present**, and new commits are **absent**
3. In the process table (`ps`), the test process it is supposedly waiting on is **absent**

If all three line up, a dead-job stall is confirmed. **If even one of the test processes it is supposedly waiting on is still alive, the run is legitimately in progress** — do not rush the agent (in a measured case, not rushing the one-process-alive case turned out to be correct).

```bash
# Example of the three-point measurement (<agent> = the agent, <worktree> = its working tree)
gh pr list --author <agent>            # 1. is the work absent from the open PRs?
git -C <worktree> status --porcelain   # 2. uncommitted diffs present?
git -C <worktree> log --oneline -3     #    any new commits?
ps aux | grep pytest                   # 3. is the awaited process alive?
```

> **"The notification hasn't arrived" is not evidence of "it's still running."** The notification is an event on the side of the producer (the process that generates the notification); if the producer dies, the notification dies with it (the operational version of [Liveness Is Decided by the Producer](../verification/liveness-is-producer.md)).
> Addendum: **CPU usage is not a liveness indicator for the agent itself** (most of its time is spent waiting for LLM responses, using almost no CPU). Watch the agent via its declarations, pushes, and messages — and watch **its child processes via ps**. They are separate indicators.

**The root cause was often an instruction that was impossible to execute.** Writing "run the tests in the foreground" into every brief did not stop the stalls, and here is why: the full suite's runtime **exceeded the shell's default execution timeout**, so running it as instructed was always cut off — the agent had no option but to escape into the background. Therefore, prevention is a three-item set written into the brief:

1. Foreground execution **with an explicit timeout** (tell the agent that under the default value, finishing is physically impossible)
2. **Commit the work before running the suite** (commit-first) — then even if the agent stalls, the work is recoverable, which demotes the stall from a "catastrophe" to "an event resolved with a single nudge"
3. **Do not end the turn; read the entire output** (ending the turn with the job in the background means "nobody is left to read it" — in a measured case, an agent that re-read the run in the foreground discovered two real failures. Had it stayed in the background, both would have shipped)

Note the measured coda: **the very person who recorded this prevention forgot, ten days later, to put two of these items (the explicit-timeout foreground run and the commit-before-suite) into a brief and reproduced the same stall three times** — "having recorded the countermeasure" and "applying the countermeasure" are different actions (the dispatch-side instance of cross-cutting principle 4, "procedure, not attention," in [The System of Verification Knowledge](../verification/index.md)).

## 2. Cross-check "currently running" self-reports against the known duration

An agent's self-report of "it's running right now" is to be believed only after checking it against **the known duration of that task**. Measured: a suite that normally took 3–4 minutes (the known value at the time) had been "running" for **44 minutes** — that is not "slow," it is a **hang** (it was in fact a bug that waited forever during shutdown handling). Note that the normal value itself drifts over time (a record of the same suite at another point puts it at about 4.5 minutes) — cross-check by **ratio**, not by absolute value. The relayer confirmed only that the run was **alive**, never that it was **progressing**, and reported to the owner "it's running, so it's on track."

- A still-running that exceeds the known duration by **several times to 10x** is to be treated as hung, not slow ([Liveness of the Verification Run Itself](../verification/verification-run-liveness.md) is the same discipline on the run's side).
- Do **not relay "running, therefore fine" as is**.

## 3. Attributing silence — what to suspect before "stuck"

- **Waiting on compaction**: on some model tiers, sessions stop at a compaction boundary and do not resume automatically. When a long autonomous job goes silent, **suspect a compaction wait first**, before hypothesizing that the code is stuck (the typical combination: shown as active on the message hub — the inter-session message relay — zero output, large work size). Measured: 10 hours of silence were misattributed to "stuck on the implementation." Prevention: **cut dispatches to a granularity that does not straddle compaction boundaries** (one PR per axis rather than one multi-axis PR).
- **Quiet work without pushes**: when a session that was pushing actively until moments ago goes silent, the first hypothesis is "**absorbed in work without pushing**," not "dead" — measured: a peer whose 25 minutes of silence nearly got escalated as a suspected restart was in fact busy finding and fixing a race bug. Send a nudge (a light check-in) first, and give it time to reply.
- **The invisibility of local-only commits**: measured case where the peer had kept implementing in roughly 20-minute increments, but the commits stayed local and unpushed, so from the outside it looked like "42 minutes of silence." Prevention is not detection but **a request made at dispatch time**: "incremental push at every milestone plus a one-line status," "open a draft PR early for heavy work" — this makes the false-stall misreading itself disappear.

## 4. Separate machine declarations from inference — for stall/idle, the session's self-declaration is the single source of truth

> **The watcher must not infer "no post for N minutes = idle."** From the outside, "just quiet during a long, correct computation," "genuinely stopped," and "waiting on a human" cannot be distinguished — only the session knows its own internal state. ∴ idle/active is **declared mechanically by the session itself via a stop hook (a handler that runs automatically when a turn ends)**, and the watcher subscribes to that declaration.

Operational points (all from measurement):

- **Both edges are required**: an idle declaration on stop alone is a one-way trap — "said idle once, looks idle forever." Pair it with an active declaration at the start of work (in the turn-start hook).
- **Machine axis and semantic axis in separate fields**: if the machine declaration (active: bool) and the LLM-written semantic status string ("waiting on CI," etc.) ride in one field, one overwrites the other. The bool is the deterministic authority; the string is an optional hint.
- **A repeated identical status string is a staleness signal, not evidence of activity** (status strings do not auto-update — a disconnected session keeps displaying its last words). After seeing the same string a few times, **check the active flag directly**. Measured: dozens of idle events were suppressed on the grounds of an identical string; a direct check showed active=false.
- **Escalation-for-verification is not the forbidden inference**: the watcher detects idle mechanically and escalates to the lead, and **the lead verifies** (deciding "done vs stalled" is the job of the party escalated to — the three-point measurement of §1 is exactly the procedure the lead runs at this verify step). Do not conflate "the watcher must not declare a stall" with "the watcher must not escalate candidates."
- **A "fire only when everyone is idle" gate misses an individual session's mid-work idle** (measured: a session that pushed only 1 of 5 stages and then disconnected went unnoticed for 70 minutes). If the stop-time declaration says "mid-work (WIP continuing)," alert on it even alone; "waiting on a dependency (idle waiting for X)" is legitimate and suppressed — have the declaration text distinguish these two.

## 5. Do not assume dispatch = motion — fan-out and fan-in are a pair

Fan-out (distributing requests to several peers) and fan-in (collecting and confirming their results) are a pair — a dispatch is not finished until it is collected. "I sent the request, therefore it's moving" is merely an assumption. Measured: a silent stall after a dispatch ack (acknowledgment of receipt) kept being reported as "presumably implementing," and the owner was forced into the role of stall detector three to four times within a single session.

- Right after a dispatch, **confirm pickup (a signal of starting the work)**. An ack alone does not mean "moving."
- After a stretch of silence, **measure the PR / the remote branch / the last-activity time yourself** before reporting status. Write the report not as "presumably implementing" but as a verified fact: "**last activity HH:MM, no PR yet = progress unverified**."
- **STOP per task**: when gating one task, do not stop that peer's **other unblocked tasks** (measured: a session-level STOP idled an entire peer that held independent tasks). Allow complete idleness only when every task is blocked/done.
- **One species of false stall is your own outgoing message**: to a plan-first peer (one that waits for approval before acting), a GO written in a "we recommend" tone makes the peer wait — correctly. Before declaring a stall, suspect whether your own last message was soft.

## Checklist

- [ ] On seeing a report that ends with "waiting on pytest": did you measure the three points — PR, worktree, ps?
- [ ] Did the brief include "foreground with an explicit timeout," "commit before the suite," and "don't end the turn; read the entire output"?
- [ ] Did you cross-check the "currently running" self-report against the known duration?
- [ ] Attribution of silence: did you consider a compaction wait, work without pushes, and local-only commits before "stuck in the code"?
- [ ] Did you judge the stall by machine declaration (the active flag), not by inference — and avoid treating a repeated status string as evidence?
- [ ] Did you run the post-dispatch fan-in (pickup confirmation, periodic measurement, reports as verified facts)?

## Sources (measured during reyn development)

Dead-job waits: six in one night (internal work label P5 plus #3083 #3086 #3090 #3091 #3093, six occurrences in total) → identification of the impossible instruction → three recurrences ten days later from the same omission in the brief (2026-07-28), and two real failures discovered by re-reading in the foreground. Duration cross-check: #2259 PR-2a (a 44-minute hang relayed as "running, so on track"). Compaction wait: #1495 (10 hours of silence misattributed to a stuck implementation; the owner pointed it out). Quiet work: 2026-06-28 (25 minutes of silence → actually fixing a race bug). Local-only commits: 2026-06-02 (a 42-minute false stall). Machine declarations: owner directive of 2026-06-03 (the declaration as single source of truth, both edges, separate fields); #2840/#2846 (active=false missed because of a repeated status string); 2026-06-13/14 (the two blind spots in the escalation design). Fan-in: 2026-05-31 through 06-02 (the same shape three to four times within one session; the owner became the detector); #2296 (a false stall caused by a soft GO).

## Related

- [Liveness of the Verification Run Itself](../verification/verification-run-liveness.md) — "slow" vs "stalled" on the run's side
- [Liveness Is Decided by the Producer](../verification/liveness-is-producer.md) — the principle of judging liveness by facts on the producer's side, not the observer's records (its context is verifying deleted code, but the principle is the same shape)
- [Operating with Separate Coder / Tester / Reviewer Agents](../verification/roles.md) — the full picture of the dispatch side's obligations
