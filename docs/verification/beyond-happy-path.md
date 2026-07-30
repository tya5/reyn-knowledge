---
name: beyond-happy-path
description: A review that checks only the happy path and structural correctness waves through error paths, invalid runtime inputs, everything else on the screen, and appearance itself — explicitly ask "what happens on failure", "what if a weak model writes a strange value", "what should be on screen but isn't", and "am I conflating a mechanism witness with an appearance witness"
tags: [verification, review, ui]
sources:
  - feedback_review_error_runtime_paths_not_just_happy_structural
  - feedback_live_verify_read_whole_frame_not_just_feature
  - feedback_visual_appearance_is_an_owner_gate_axis_i_was_not_applying
  - feedback_ansi_survival_is_not_a_witness_for_looks_good
  - feedback_verify_via_read_when_terminal_corrupt
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

## Blind spot 3 — Mistaking a mechanism witness for an appearance witness

Verifying a UI change on real hardware has one more, deeper trap. A witness (a check that fails if the property under test breaks) that "the mechanism reached the screen" and a witness that "the result looks right" are **separate claims**, and the first does not imply the second.

Real case: a search hit and the keyboard-cursor row were highlighted with reverse video (an ANSI decoration that inverts foreground and background) because it survives the component's style compositing, whereas a background color does not. The real-terminal capture confirmed exactly that — the ANSI escape code was present on the row — **and the result was still wrong**: inverting a light color on a dark palette painted a near-white band across the whole dark theme, and the owner, actually using it, judged it "turns white and feels off — it wrecks a design that used to be polished."

The verifier's own analysis: whether the ANSI code survives on the row, and whether focus stays put, are **witnesses of function**, not **witnesses of appearance**. When a report says "verified on real hardware," what the reviewer must check is **what, exactly, was witnessed**. This gives a discriminator:

> **Is this witness evidence that the mechanism fired, or evidence of what a person receives?** Data taking the right path, an ANSI code being present, focus not moving — these are the former. Whether a label is actually rendered, how a band actually looks, whether an element disappears correctly — these are the latter. **A UI change that passes only on the former will always be missing the latter.**

Only a human actually looking at the screen (here, the owner) can judge appearance; from a position that can only see the mechanism (code review, parsing an automated capture), the quality of appearance is unknowable in principle. The correct posture is not to try to render that judgment yourself, but to surface it for the person who can.

- **Treat visual changes as gated approvals, on par with functional changes**: leave the visual item **unchecked** in the pre-approval checklist and let the person who will actually see it decide.
- **When you receive "verified on real hardware," ask what was verified**: ANSI survival and unmoved focus are function checks, not appearance checks.
- **Require a before/after comparison**: for a visual change, ask for before/after screenshots or a request to check it on real hardware.
- **A companion trap — check not just "does it correctly appear" but "does it correctly disappear"**: after introducing a decoration, an element can keep sitting on screen after it no longer applies (lingering as visual noise). A visual change spawns a new appearance question with every fix, so it is **never done until you actually show the state after the change**.

Separately: raw terminal output (the display) is sometimes not trustworthy (mixed-in color codes, interleaved lines, broken widths). When that happens, do not trust grep or the on-screen display as-is — confirm the actual content by **writing it to a file and reading that**, or by **emitting structured data (e.g. JSON) and parsing it** — a grounded check that bypasses display-layer noise.

## Live / dogfood is complementary, not redundant

Every case above was "unit tests + static review = green, discovered in real use." Verification through actual use has **a different axis** from feature-scoped verification (the whole experience, error paths, real data). Do not read end-to-end happy-path green as "verification complete."

## How to apply

- [ ] Did the review ask the three questions: what is the failure path? what about invalid runtime input (including weak models)? what leaks onto the screen on error?
- [ ] Did the consumer audit for a vocabulary/format change include write-side constraints (input validation)?
- [ ] Did you look at UI captures in two passes (the feature + a whole-frame hunt for "what's missing")?
- [ ] Before letting "verified on real hardware" close out a visual change, did you check what was witnessed (mechanism or appearance)? Did you leave the visual item unchecked in the approval gate?
- [ ] When terminal output was untrustworthy, did you switch from raw grep/display to a file write-and-read or structured-data check?
- [ ] Did you plan real-use verification (live / dogfood) as a complement to unit + static review, not a substitute?

## Sources (measured during reyn development)

Blind spot 1: the write-side constraint gap in the status-vocabulary rename (at runtime a weak model wrote the old vocabulary and caused corruption); #2195 (log leakage on error after the UI switch). Blind spot 2: #2238 (missing echo of user input — verifier and reviewer both missed it in the same capture; discovered in the owner's real use). Blind spot 3: #3488/#3489 (reverse-video ANSI survival was verified but the appearance was wrong; the owner caught it on real hardware → #3490 → the approval-gate rule was written down) + #3302 (a data-path witness missed that a label was never rendered — the same discriminator, a different instance) + terminal-output reliability (2026-05-31; when the display is corrupted, verify by writing to a file and reading it). In every case unit + static was green; caught live.

## Related

- [How vacuous gates are born](vacuous-gates.md) — the danger of "X does not happen" asserts (green merely because the error path is dead)
- [Structural blindness of the verification environment](environment-blindness.md) — the general form of "can this environment turn the test red?"
- [Reviewing sweep PRs](sweep-reviews.md) — the review-side version of "what the diff doesn't show." This document is about "what the frame *does* show but nobody looks at"
