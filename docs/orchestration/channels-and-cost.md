---
name: channels-and-cost
description: Communication channels and the cost of waiting — split channels by purpose (ephemeral signals on the hub, decisions in issues, lessons in records); never pin a contract on the hub; wait event-driven and don't keep waking an idle LLM; run long work in the background; if you wrote "I will post it," post it in the same turn
tags: [orchestration, multi-agent, communication, cost]
sources:
  - feedback_inter_session_communication_paths
  - feedback_canonical_contract_in_issue_not_broker
  - feedback_polling_vs_llm_cost_tradeoff
  - feedback_long_running_commands_background_not_foreground
  - feedback_no_sync_launch_long_work_blocks_conversation
  - feedback_broker_post_articulate_action_gap
  - feedback_no_idle_loops_only_session_watcher
---

# Communication Channels and the Cost of Waiting — Where to Talk, How to Wait

## Premise

Running multiple agent sessions in parallel creates multiple communication channels: a **message hub** (a relay for short inter-session messages; delivery lags by tens of seconds, and messages vanish when a session restarts), **issues/PRs** (task tickets and change proposals; persistent, auditable after the fact), and **record files** (persisted lessons). Choose the wrong channel and important decisions vanish in an ephemeral queue, or exchanges of signals pile up noise and round-trip cost. The other axis is the **cost of waiting**: depending on how you make an agent wait, you either keep waking — and paying for — an LLM (language model) that has nothing to do, or wake it only when an event arrives.

## 1. Split channels by purpose

| Channel | Purpose | Persistence |
|---|---|---|
| Hub, one-to-one | Light signals ("PR is up," "received"), short-lived coordination | Lost on restart |
| Hub, broadcast to everyone | Announcing policy, mass-notifying a correction | Same as above |
| Issue + label | Awaiting a human decision, requesting approval | Persistent + auditable |
| PR comment | Review discussion | Persistent |
| Record file | Lessons, patterns | Persistent (local) |

Two principles. **Never seek a decision over an ephemeral channel** (it vanishes, and afterwards nobody can trace who decided what). **Never write short-lived status reports to a persistent channel** (they become noise). Also, chains of "received → thanks → you're welcome" are pure round-trip cost: outside of active coordination (requests, blockers, correction demands), the right signal is silence. The measured trigger was the lead's inbox accumulating **480 messages** — signals, status, and questions all mixed together.

## 2. Never pin a contract on the hub — write it in an issue and point people at it

Measured: an attempt to finalize a data-format contract (the interface specification both sides must agree on) between two implementation sessions — a writer side and a reader side — over the hub. The arbiter reversed themselves on two minor points and sent multiple "this is final" messages. Because the hub has roughly 30 seconds of delivery lag, each party responded to **the version that reached them last**; they kept crossing, each looking at a different version, and never converged. The hub cannot hold a contract — it is ephemeral, arrival order scatters, and it has no canonical copy (canonical = the one place everyone refers to as correct).

- Write the contract in **one place** — an issue/PR comment — and declare: "all contract messages on the hub (including my own) are void; discuss deltas in the issue." Return the hub to short-lived signals like "pushed, please review."
- **An arbiter does not reverse themselves**: hesitant re-decisions amplify divergence far more than a single firm decision. If you decide, decide on substance, once.
- **Ratify convergence**: if the two implementers independently arrive at the same sound design, don't push your own (over-kneaded) version — ratify what they reached. The goal is that one correct agreement exists, not that your version wins.
- Trigger: the moment you are about to send the **second** "final version" of the same decision to the hub, stop and write it in an issue.

## 3. Wait event-driven — don't keep waking an idle LLM

There are three ways to make an agent "wait for news," and their cost while waiting differs by orders of magnitude:

| Method | LLM cost while waiting | Where it fits |
|---|---|---|
| OS-level watcher process (wakes the agent only when something arrives) | **0** (just a light OS check) | Inbox watching (standing) |
| Fixed-interval self-wake (a timer wakes the agent periodically) | **Incurred on every wake** | Only when truly needed, at a sparse interval (60+ minutes) |
| Event-driven (triggered by the other party's post or report) | 0 | Everything that can be structured around the other party reporting |

A fixed interval is acceptable only when all three conditions hold: (1) no event-driven path exists; (2) the periodic check itself has operational value; (3) a sparse interval suffices. Measured anti-pattern: a 4-minute self-wake loop, whose LLM cost compounded to **7–10x**.

One stronger rule on top (a direct owner instruction): **while idle with no task, creating any self-triggering loop other than the inbox watcher is prohibited outright.** Waiting means the passive form — do nothing until an event wakes you; do not keep waking yourself with polling (checking at a fixed interval) or scheduled wakeups. Note that a mid-work "watch until CI finishes, then naturally stop" is not an idle loop — the prohibition targets self-restarting with no end.

## 4. Run long work in the background — don't block the conversation

Launching anything that won't finish in seconds (a full test run, spawning a child agent, a build) in the foreground (synchronous execution: nothing else can happen until it ends) means you cannot respond to the owner's messages or to the hub until it completes. To the owner this reads as a "freeze" — measured: an owner who perceived no response canceled the work. A role that must respond promptly to the conversation and the hub launches long work in the background and picks it up on the completion notification.

This, however, sits in **tension** with [Detecting Stalled Agents](stall-detection.md) §1 (waiting on a dead background job is the most frequent stall shape): background execution doesn't block the conversation, but a job that dies silently is a breeding ground for stalls. The reconciled pattern: (a) commit before launching (even if the job dies, the work is recoverable); (b) never read "no completion notification yet" as "still running" — measure with ps; (c) only the final verification run, whose full output must be read, goes in the foreground with an explicit timeout.

## 5. If you wrote "I will post it," post it in the same turn

You **wrote** "will notify the other party" or "acknowledge receipt and await completion" in a turn-end summary — but never actually posted. This separation of stating from doing quietly breaks automation. The other party cannot even wait for the message that was supposed to arrive (they don't know it's coming). Simply nothing happens, and it surfaces only when a human asks, "did you notify them?"

Measured, twice in a row: the owner discovered a missed notification and worked around it by hand (first case). **Immediately after that lesson was recorded**, another merge produced the same missed-notification shape (second case) — recording does not guarantee execution. The countermeasure is phrase detection:

- The moment you write "notify X," "convey receipt," or "request Y" in your own prose is the cue to perform the post **within the same turn**. Don't leave it in writing under a "planned" heading.
- A merge is not a single action but a compound one: performing the merge + confirming with the author + notifying whoever was waiting so they can start + reporting — all in the same turn.
- Just before closing the turn, ask yourself: "did I write any mention that requires a post, without actually posting it?"

## Checklist

- [ ] Is this communication a decision, a signal, or a lesson? Does the channel match the purpose?
- [ ] Are you trying to pin a decision or contract on an ephemeral channel? Did you write it in one place in an issue and point people at it?
- [ ] Are you about to send a second "final version" of the same decision? Did you ratify the implementers' convergence?
- [ ] Is the waiting setup event-driven? Are you planting a self-wake loop while idle?
- [ ] Before choosing a fixed interval, did you confirm all three conditions (no event-driven path, operational value, sparse suffices)?
- [ ] Are you blocking the conversation with a synchronous launch of long work? Are you following the pattern: background + commit first + measure with ps?
- [ ] Every post you wrote "I will notify" about — did you execute it in the same turn?

## Sources (measured during reyn development)

Channel separation: 2026-05-27 owner instruction "also look into optimizing the inter-session communication paths" (triggered by the 480 mixed messages in the lead's inbox). Contract: #1135 (2026-05-31, version crossings of a data-format contract; owner: "why not write it in the issue and have them refer to that?"). Cost of waiting: 2026-05-27 owner instruction "polling only when truly necessary" (13-minute and 30-minute fixed cycles reorganized into a watcher process + sparse intervals; the measured 7–10x cost of the 4-minute self-wake loop) + the outright ban on idle loops (2026-06-25 owner instruction). Synchronous launch: 2026-06-25 (owner canceled work over the unresponsiveness of a synchronous launch) + during a long foreground run, hub messages don't get through and the session looks stalled (the lead's broadcast to everyone). Posting: 2026-05-28 PR #993 (first missed notification, surfaced by the owner's question) → same day, #995/#998 (second case, right after the lesson was recorded).

## Related

- [Detecting Stalled Agents](stall-detection.md) — the price of background execution (waiting on dead jobs) and how to detect it
- [Verifying Before You Relay](relay-verification.md) — the discipline of what you put on a channel
- [The Discipline of Autonomous Operation](autonomous-operation.md) — don't stop right after dispatching (kin of the same-turn principle)
- [Issues Decay](../git-github/issue-lifecycle.md) — decay and maintenance on the persistent-channel side
