---
name: orchestration-index
description: The map of Tier 3 — the five areas of operating a fleet of concurrent agent sessions: stall detection, relay verification, dispatch-brief discipline, autonomous-operation discipline, and communication channels with the cost of waiting. The cross-cutting structures — "silence and hearsay are ambiguous" and "the brief and the channel determine behavior"
tags: [orchestration, index, overview]
---

# The Map of Multi-Agent Operation Hazards — Tier 3

If [Tier 1 (verification)](../verification/index.md) is about how to doubt a single artifact, and [the Tier 2 map](../git-github/index.md) covers the hazards of shared infrastructure (git/GitHub), then Tier 3 — the third layer of this knowledge collection — is about **the operational problems of running multiple agent sessions concurrently (a fleet)**: who is stalled, whose claims may be believed, how to word a request so it is carried out as intended, and where to talk and how to wait so that cost does not leak away.

Two structures cut across everything here:

1. **Silence and hearsay are ambiguous.** From the outside, an agent's silence cannot be told apart — "quietly working," "waiting for a response," and "genuinely stopped" all look the same — and a claim that arrives through a hub looks like fact but is, in substance, hearsay. ∴ Judge by measurement, not by inference (the three-point measurement, a single curl, machine declarations).
2. **The brief and the channel determine the system's behavior.** An agent upholds only what the brief states, at the altitude the brief states it, and the properties of a channel (short-lived vs persistent, synchronous vs event-driven) translate directly into failure modes and cost structure. ∴ What you design is not just the agent itself, but the wording of the dispatch brief and the choice of communication channel.

## Documents

| Document | In one line |
|---|---|
| [Detecting Stalled Agents](stall-detection.md) | Silence is ambiguous. The three-point measurement for the most frequent stall shape (waiting on a dead background job), cross-checking against known durations, separating machine declarations from inference, fan-out/fan-in (distributing requests and collecting results) as a pair |
| [Verifying Before You Relay](relay-verification.md) | Claims that cross a hub are hearsay. Verify infrastructure state directly at minimal cost; carry a claim together with the command that was run and its scope; numbers only as copied output; rule out confounders |
| [The Discipline of Dispatch Briefs](dispatch-briefs.md) | Name the invariant instead of "copy X"; user-visible changes require documentation; state verification obligations as concrete behaviors of the content; write GO (permission to start) as a decision |
| [The Discipline of Autonomous Operation](autonomous-operation.md) | Don't ask blocking questions; reject the fatigue reflex with numbers; full-count review before idling; never fabricate work; re-read the original goal; don't stop right after dispatching |
| [Communication Channels and the Cost of Waiting](channels-and-cost.md) | Separate channels by purpose; pin contracts in issues; wait event-driven; run long work in the background; if you wrote "I will post it," post it in the same turn |

## Reading order

If you are about to build a fleet: [The Discipline of Dispatch Briefs](dispatch-briefs.md) → [Communication Channels and the Cost of Waiting](channels-and-cost.md) (the things to decide before you build the machinery). If you already operate one and your daily pain is "agents stall, falsehoods get relayed": [Detecting Stalled Agents](stall-detection.md) → [Verifying Before You Relay](relay-verification.md). The self-discipline to hand to the agents themselves is [The Discipline of Autonomous Operation](autonomous-operation.md).

## Relationship to Tier 1 and Tier 2

Most Tier 3 documents are the "operational version" of a Tier 1 verification principle: the three-point measurement of stall detection applies [Liveness Is Decided by the Producer](../verification/liveness-is-producer.md), relay verification applies [Claims Without Context](../verification/cross-context-claims.md), the verification obligations in dispatch briefs apply [Audits Must Match Content](../verification/audit-content-match.md), and the "recording and applying are different actions" of autonomous operation applies cross-cutting principle 4 ("procedure, not attention") of [The System of Verification Knowledge](../verification/index.md) — each carried into the multi-agent setting. For the full picture of per-role obligations, see [Operating with Separate Coder / Tester / Reviewer Agents](../verification/roles.md).
