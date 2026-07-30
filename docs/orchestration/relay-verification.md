---
name: relay-verification
description: Relay verification — a claim arriving through the hub looks like a fact but is hearsay in substance. Infrastructure state is the cheapest direct check there is; carry claims as literal command + scope; numbers are copies of output only; rule out confounds before relaying a finding
tags: [orchestration, multi-agent, communication]
sources:
  - feedback_probe_infra_state_before_propagating_peer_claim
  - feedback_peer_broker_articulate_verify_obligation
  - feedback_broker_numbers_file_copy_only
  - feedback_confound_exclusion_before_propagating_peer_finding
---

# Verifying Before You Relay — Measure Hearsay Yourself Before Carrying It

## Premise

In fleet operation, claims flow in through a message hub — a relay that carries messages between sessions. A claim that arrives through the hub **looks like a fact but is, in substance, hearsay**. And relaying amplifies: the moment the lead carries it to the owner, the hearsay **wears the face of a first-hand report** (the receiver trusts the lead's report as already verified). This document takes the labeling discipline of [Claims Without Context](../verification/cross-context-claims.md) and specializes it to the act of relaying.

## 1. Infrastructure state is the cheapest claim in the world to verify

Measured case: reports from two sessions that "the LLM proxy is down" were relayed to the owner across multiple turns **without checking even once**, and real work (live verification, A/B measurement) was halted on that "outage." When the owner asked "do you actually have confirmation that it's down?", a single `curl` settled it — **it had been alive the whole time** (every endpoint HTTP 200, process running; the peer had overgeneralized a transient slowdown into "down").

- Whether a service is alive, its port, its health, its process: **one shot of curl / lsof / ps** settles it. At that price, there is no excuse for skipping verification.
- A peer's "X is down" is a **hypothesis to verify**, not a fact to relay.
- If you have no choice but to carry it unverified, label it: "**peer reports X; unverified on our side**" — do not launder it into a plain declarative sentence.
- Capture the raw failure: connection refused / merely slow / 5xx are **different failures**, and the word "down" crushes that distinction.

## 2. Carry claims as literal command + scope — shorthand is a source of propagation accidents

Shorthand claims like "audit clean," "CI green," or "trigger condition met" have caused a measured propagation accident across a three-session chain: the originator had only run a narrow-scope check (`--check private-state`), but the report to the hub was written at broad scope (`--strict tests/` exit 0), the relayer forwarded it unverified, and **only when the receiver ran the literal command directly did 18 errors surface**.

- **As sender**: write the **command and scope you actually ran, verbatim**. "`--check private-state tests/` → 0 errors" is defensible; "audit clean" is an accident source.
- **As receiver**: if the claim **gates your next action**, run the quoted literal command yourself before moving.
- **When a scope mismatch is discovered**: **broadcast a correction to everyone** who saw the original claim. Do not fix it quietly and move on (other receivers may be applying the same inference right now).

## 3. Numbers are copies of output only — no hand-typing, no memory, no recomputation

Numbers placed in a status report (PR counts, merge state, tallies) must be **copies of the output of a command run on the spot** — nothing else. Typing them by hand, reproducing them from memory, and recomputing them in your head are forbidden. In particular, **alarms (CI FAIL and the like) must never be raised without quoting the output line** — a fabricated regression alarm that propagated through the fleet is the measured incident this discipline originates from.

## 4. Before relaying a finding, rule out confounds

Before escalating a peer's live-verification or field-test "finding" to the owner, confirm that **confounds in the test setup (a confound is a co-occurring factor that distorts the result) have been ruled out**.

Measured case: a report that "12/12 empty stalls on unknown actions = structural blocker" was carried to the owner as-is, as a "deep structural defect." In reality, the test had been built from **fake components plus tasks that made no sense**, which had **amplified a real but partial defect (about 67% on real tasks) into total failure (100%)**. There is a symmetric trap on the other side, once the confound comes to light: overcorrecting to "pure artifact (no defect at all)" — the true value was in between.

- Before escalating: (a) is the finding based on **a genuine, coherent task** (a real schema plus an achievable objective); (b) are you **holding both hypotheses — "blocker" and "pure artifact" — until primary data from real tasks settles it**; (c) does the escalation message **state the status of confound exclusion explicitly**?

## Checklist

- [ ] Infrastructure-state claim: did you measure it yourself with one shot (curl/lsof/ps)? If unverified, did you attach the "unverified" label?
- [ ] When sending a claim: did you write the literal command + scope? Did you avoid shorthand?
- [ ] When a claim gates your action: did you run the quoted command yourself?
- [ ] When a scope mismatch surfaces: did you broadcast a correction to all receivers?
- [ ] Numbers and alarms: are they copies (quotes) of command output? No hand-typing, memory, or recomputation?
- [ ] Before relaying a finding: are confounds ruled out? Are you holding both hypotheses? Did you state the exclusion status explicitly?

## Sources (measured during reyn development)

Infrastructure: the LLM-proxy "down" relay incident (in fact all 200s; surfaced by a single question from the owner). Literal command: the three-session chain of 2026-05-28 (a narrow-scope run propagated as a broad-scope claim; 18 errors on the receiver's direct run) plus the next day's "gone from the backlog = merged" misreport (it was actually closed) and its broadcast correction. Number copies: the propagation of a fabricated regression alarm (the v9 field test). Confounds: #187 Stage A (fake components plus meaningless tasks amplified 67% into 100%; settled at an intermediate value on real tasks).

## Related

- [Claims Without Context](../verification/cross-context-claims.md) — the general form of the claim-labeling discipline
- [Measuring the Wrong Target](../verification/measurement-target.md) — the relay rule for measurements (attach the tree and commit)
- [Argument Hygiene](../verification/argument-hygiene.md) — check 3, "use a numeric claim once before propagating it"
- [Reading Green](../verification/green-reading.md) — what the shorthand "CI green" loses
