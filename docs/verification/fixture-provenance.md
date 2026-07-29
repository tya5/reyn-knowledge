---
name: fixture-provenance
description: The claim "I re-recorded the fixtures" can be proven by pairing them with the pre-change code and observing RED — measure the correspondence, not the claim. Claims of "this failure is pre-existing" also have a settlement protocol
tags: [verification, testing, fixtures, llm]
sources:
  - feedback_prove_replay_fixture_was_rerecorded_by_pairing_with_old_code
  - feedback_verify_failure_claims_by_observation_not_inference
---

# Proving Fixture Provenance — Verify "I Re-Recorded It" Without Trusting the Claim

## Background — record/replay testing

Tests for applications that call an LLM commonly use the **record/replay approach**: record a real API response once, save it as a fixture, and replay the recording in CI. It is deterministic, fast, and incurs no API cost.

With this setup, any change that affects the request payload (tool descriptions, the system prompt, schemas) means the **fixtures must be re-recorded**. And then a claim arrives from the implementer: "I re-recorded the fixtures."

## The question — can the claim be verified from the artifacts alone?

Can you distinguish "a fixture re-recorded from the real thing" from "a fixture hand-written so the tests pass," using only the artifacts? Staring at the fixture's contents, the only reference point appears to be "replay goes green." And **green does not distinguish "recorded correctly" from "wrote values that happen to pass."**

## The answer — pair with the pre-change code and look for RED

If the replay matching key includes a hash of the request contents (which is how record/replay implementations are normally built), then **the fixture's key is uniquely determined by the actual code at recording time**. So set up a two-way experiment:

| Experiment | Expected |
|---|---|
| ① New code + new fixtures (the PR as-is) | passed |
| ② **Old code + new fixtures** (a deliberate mismatch) | **failed** |

**The RED in ② is the guarantee.** If the fixtures had been passing on values unrelated to the new code, swapping the code back to the old version should leave them passing. Breaking when paired with the old code = these fixtures correspond specifically to the new code = they were recorded from the real thing. If someone hand-wrote "values that happen to pass," there is no guarantee that ② goes RED — the same experiment serves as the discriminator.

> **Measure the dependence on what was supposedly re-recorded, not the claim "I re-recorded it."**

Where [The discipline of strip-falsification](strip-falsification.md) demonstrates **load-bearing** via "break the mechanism and see RED," this demonstrates **correspondence** via "**age the counterpart and see RED**." It is the same tool applied a different way.

## Make hand-editing structurally impossible

The re-recording procedure itself can also be put into a form that needs no proof:

> **Delete the fixture files → run the normal tests in record mode → regenerate from real payloads. Never open and edit them by hand.**

This makes hand-editing **structurally impossible**. The general form of the prescription is: "**eliminate the very path by which a plausible-but-wrong artifact can be constructed; do not rely on someone noticing the fake**" — the same shape as the prescription to replace hand-built test doubles with real objects (when they are cheap to construct).

Incidentally, the worst workaround is "**the fixtures fail, so revert the changed description text**." That is an inversion that throws away the very value of the PR in order to make the tests pass; in the real case it was not taken — the fixtures were re-recorded and the text fix was preserved.

## "This failure is pre-existing" claims also have a settlement protocol

A claim that often arrives alongside fixture problems: "this test failure reproduces on main too — it is a pre-existing issue unrelated to my change." Do not accept this one as a bare claim either.

First, there are **three** categories of "pre-existing failure," not two: (1) caused by the PR, (2) environment-specific, (3) **execution-order-dependent** (in a repository where randomized test ordering is enabled and the seed is not pinned, which tests fail changes from run to run). A failure count that changes on every run is a symptom of (3), not a "regression."

The settlement procedure:

- **Require the claim to include the seed.** Run **both main and the PR** with the identical seed.
- **Have them report only the set difference P−M (tests that fail only on the PR).** "Fails on both" is settled as outside this PR's responsibility, so it is not needed as decision input. "N tests failed, but they fail on main too" decides nothing by itself — the information you need has the form "**zero tests fail only on the PR**."
- **Make CI (a fixed environment) the court of last resort.** Environment-specific failures such as a corrupted local venv do not appear in CI. Local red × CI green is strong evidence of "environment-specific."
- Even when you are the one adjudicating, do not use "green when run in isolation" as grounds. Do the comparison with **the same procedure on a separate tree** (check out main and run the same full suite).

Generalization: **before reading green/red, confirm "did it run with the same order and the same seed?" Even if the measured target is the same, a different order is a different experiment.**

### Do not co-sign by inference — only observation settles a failure

**Co-signing** a peer's "pre-existing failure" claim by inference falls into the same hole. Real case: presented with a peer's report that "the 4 failures are pre-existing," a reviewer confirmed by grep that an old deprecated pattern **existed** in that test file and signed off "agreed, pre-existing" — without ever **running the suite to observe** the failures. A later clean-environment run showed **everything passes; main had been green the whole time**. Moreover, a second verifier's own "4 failures" were **a different four**, which they too retracted as contamination of their local environment (a stale editable-install remnant and a leftover `.pyc` directory causing wrong module resolution).

- **CI green (a clean, fixed environment) is the court of last resort for whether a failure is real.** A local full-suite run carries the runner's contamination and cannot settle the question alone.
- **Different verifiers reporting different failure sets is a strong tell of environment contamination**, not evidence of a regression.
- A grep showing that a plausible cause **exists** in the code is **inference, not observation** (the failure-side twin of [Audits must match content](audit-content-match.md)).
- **Do not file a tracking issue until the failure reproduces in a clean environment or is confirmed in CI** (false issues spend other people's time).

## Checklist

- [ ] For a PR that changes text reaching the LLM, did you state the need for fixture re-recording **at delegation time**?
- [ ] Does the "I re-recorded it" report come with a measured "old code + new fixtures goes RED"?
- [ ] Was the re-recording done via "delete → regenerate in record mode" (is any hand-editing path left open)?
- [ ] Does a "pre-existing failure" claim come with the seed and results on both main and the PR (the set difference P−M)?

## Sources (measured during reyn development)

Experiment ② and its general form: #3190 (tool-description change → replay key change; the architect measured failed with old `io.py` + new fixtures). Settled as PR-caused with CI as evidence: #3189/#3190. The third, order-dependent category and the P−M protocol: #3195 (confirmed P−M = 0 every time across 3 seeds × 2 independent worktrees). Eliminating the hand-edit path, and the isomorphic fake→real prescription: #3183. Co-signing by inference and per-environment phantom failures (two verifiers reported two different "4 failures"; clean env all green): #1800/#2059/#2060.

## Related

- [The discipline of strip-falsification](strip-falsification.md) — proof of load-bearing; this document is proof of correspondence
- [Integrity of the strip instrument](strip-instrument-integrity.md) — the identity of "what did you measure"
- [Measuring the wrong target](measurement-target.md) — the general form of "did you measure the same target under the same conditions"
