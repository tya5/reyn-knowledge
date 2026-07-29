---
name: beyond-happy-path
description: A review that checks only the happy path and structural correctness waves through error paths, invalid runtime inputs, and everything else on the screen — explicitly ask "what happens on failure", "what if a weak model writes a strange value", and "what should be on screen but isn't"
tags: [verification, review, ui]
sources:
  - feedback_review_error_runtime_paths_not_just_happy_structural
  - feedback_live_verify_read_whole_frame_not_just_feature
---

# Beyond the Happy Path — Error Paths, Runtime Inputs, and the Whole Frame

## The shape of the problem

The default attention of a PR review goes to "does the happy path work correctly" and "is the structure correct (no dangling references, naming issues, leftovers)." Outside that lie two classic blind spots. **Both have measured cases that slipped past unit tests and static review alike and were only caught in real use (live / dogfood).**

## Blind spot 1 — Error paths and runtime inputs

Three questions a review must ask explicitly:

1. **What does the failure path of this change look like?**
2. **What happens if invalid, stale, or unexpected input arrives at runtime?** — In AI-agent environments especially, "**what if a weak model writes a strange value**" is a real input class
3. **If this is a UI that owns the terminal/screen, what leaks onto it on error?**

Real case 1 (runtime input): in a status-vocabulary rename (completed → done), the full-population audit of readers and comparers was done. What was missed was **the constraint on the writing side** — the status argument of the update operation exposed to the model remained an **unconstrained string**. At runtime, a weak model (unaware of the rename) wrote the old vocabulary "completed", validation threw an exception, and it surfaced as data corruption.

> **When auditing consumers for a vocabulary/format change, include not just the read/compare side but also the input/write-constraint side.**

Real case 2 (error path): in a switch from a full-screen UI to an inline UI, the deletion-completeness audit (dangling references) was done. What was missed was the behavioral fact that **the inline UI also owns the terminal** — on error, logger output leaked into the UI and corrupted the screen. The end-to-end happy-path tests were green, and nobody had walked the error path.

## Blind spot 2 — Everything on the screen except the feature

When verifying or reviewing a live UI check (screenshot, terminal capture), attention is **sucked toward the feature under verification**. But the capture shows **the entire state of the screen**. Peripheral regressions — a line that should appear but doesn't, a misaligned line, a stale display — **sit there in plain sight, unobserved**.

Real case: an inline UI **never rendered the user's own input on screen** (only the agent's replies and tool lines were drawn). This bug **was visible** in the verifier's own capture (not a single `>` user line anywhere in the conversation). The verifier checked the feature under verification (the rewind selection menu), the reviewer reviewed the feature's correctness, and **neither of them looked at the missing surrounding frame**. It was caught by the owner actually using it.

> **Verify captures in two passes: (1) Is the feature under verification correct? (2) Scan the entire frame and explicitly ask, "What should be on this screen but isn't?"**

"Did my feature work?" and "Is this screen in a complete, consistent state?" are different questions, and only the former fires naturally. Does the conversation read as a coherent thread (are both user lines and agent lines present)? Are the expected lines, markers, and displays all there?

## Live / dogfood is complementary, not redundant

Every case above was "unit tests + static review = green, discovered in real use." Verification through actual use has **a different axis** from feature-scoped verification (the whole experience, error paths, real data). Do not read end-to-end happy-path green as "verification complete."

## How to apply

- [ ] Did the review ask the three questions: what is the failure path? what about invalid runtime input (including weak models)? what leaks onto the screen on error?
- [ ] Did the consumer audit for a vocabulary/format change include write-side constraints (input validation)?
- [ ] Did you look at UI captures in two passes (the feature + a whole-frame hunt for "what's missing")?
- [ ] Did you plan real-use verification (live / dogfood) as a complement to unit + static review, not a substitute?

## Sources (measured during reyn development)

Blind spot 1: the write-side constraint gap in the status-vocabulary rename (at runtime a weak model wrote the old vocabulary and caused corruption); #2195 (log leakage on error after the UI switch). Blind spot 2: #2238 (missing echo of user input — verifier and reviewer both missed it in the same capture; discovered in the owner's real use). In every case unit + static was green; caught live.

## Related

- [How vacuous gates are born](vacuous-gates.md) — the danger of "X does not happen" asserts (green merely because the error path is dead)
- [Structural blindness of the verification environment](environment-blindness.md) — the general form of "can this environment turn the test red?"
- [Reviewing sweep PRs](sweep-reviews.md) — the review-side version of "what the diff doesn't show." This document is about "what the frame *does* show but nobody looks at"
