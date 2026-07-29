# CATALOG — Document Index and Source Mapping

This file tracks the mapping between each document and its sources (cross-session memory pins from reyn development).
reyn's pins keep growing, so this table is the basis for periodic sync: when a new pin falls under Tier 1–3, decide between extending an existing doc or adding a new one, and add a row here.

Language convention: `.md` = English, `.ja.md` = Japanese. Japanese is written first; English versions are synced from it.

## docs/verification/ (Tier 1 — verifying AI-produced artifacts)

Map: [The system of verification knowledge](docs/verification/index.md) (pipeline placement of the 13 docs + 4 cross-cutting principles; meta-document with no source pins)

| Document | Source pins | Main source issues |
|---|---|---|
| [The discipline of strip-falsification](docs/verification/strip-falsification.md) | `feedback_strip_falsify_all_sibling_guard_sites_not_just_one` / `feedback_witness_must_assert_a_value_a_dead_mechanism_cannot_produce` / `feedback_bound_test_must_flip_under_strip` | #2900 #2903 #3159 #3189 #3195 #2825 |
| [Integrity of the strip instrument](docs/verification/strip-instrument-integrity.md) | `feedback_strip_anchor_must_be_unique_or_it_kills_the_wrong_call_site` / `feedback_verify_venv_identity_at_measurement_time_not_by_central_audit` | #3310 #3341 #3363 #3370 #3389 |
| [Wiring tests vs mechanism tests](docs/verification/wiring-vs-mechanism.md) | `feedback_wiring_test_strip_production_callsite_not_mechanism` | #2788 #2801 #2802 #3383 #3385 |
| [How vacuous gates are born](docs/verification/vacuous-gates.md) | `feedback_gate_vacuity_hides_in_terminal_state_only_assertions` / `feedback_containment_gate_must_cover_both_axes_and_children` | #3288 #3299 #3358 #3363 #3370 #3311 #3337 #3341 |
| [Structural blindness of the verification environment](docs/verification/environment-blindness.md) | `feedback_verification_environment_structurally_blind` | #2975 #2962 #2965 #2952 #2973 #2982 #2978 #2981 |
| [Census vs structure](docs/verification/census-vs-structure.md) | `feedback_census_vs_structure_definition_and_checked_premises` / `feedback_perf_fix_needs_fixclass_question_not_correctness_frame` (census part) | #2945 #2949 #2962 #2963 #2961 #2960 #2958 |
| [The discipline of enumeration](docs/verification/enumeration-discipline.md) | `feedback_census_vs_structure_definition_and_checked_premises` (§4) / `feedback_measured_but_the_target_was_off_my_four_instances` (surface-definition part) / `feedback_perf_fix_needs_fixclass_question_not_correctness_frame` (grep-authority part) | #2958 #2965 #2981 #2951 #3429 #3463 |
| [Fix-class review](docs/verification/fix-class-review.md) | `feedback_perf_fix_needs_fixclass_question_not_correctness_frame` | #2937 #2938 #2948 #2945 |
| [Liveness is decided by the producer](docs/verification/liveness-is-producer.md) | `feedback_liveness_is_producer_not_reader` | 2026-07-04 skill sweep, #3357 #3410 #3432 #3433 #3437 |
| [Measuring the wrong target](docs/verification/measurement-target.md) | `feedback_measured_but_the_target_was_off_my_four_instances` | #3437 #3433 #3411 #3458 #3459 #3461 #3429 #3463 |
| [Proving fixture provenance](docs/verification/fixture-provenance.md) | `feedback_prove_replay_fixture_was_rerecorded_by_pairing_with_old_code` | #3190 #3189 #3195 #3183 |
| [Reviewing sweep PRs](docs/verification/sweep-reviews.md) | `feedback_sweep_pr_review_the_untouched_decision_is_invisible_in_the_diff` | #3186 |
| [Shared-helper widening](docs/verification/shared-helper-widening.md) | `feedback_shared_accessor_must_not_outopinion_least_opinionated_caller` | #2947 |

Source consolidator: `feedback_verification_blind_spot_family` (memory-curator dir). The family's 16 pins were consolidated into the 13 docs above (as of 2026-07-30).

## docs/git-github/ (Tier 2 — agent-driven git/CI operation hazards) — not started

Candidate pins (from the inventory memo, ~28): `closing_keyword_in_backticks` / `gh_merge_leaves_local_tree_stale` / `line_numbers_not_identifiers` / `merge_order_signature_conflict` / `outstanding_item_must_live_where_gate_reads`, among others.

## docs/orchestration/ (Tier 3 — multi-agent fleet operations) — not started

Candidate pins (~14): `background_pytest_poll_stall` / `review_menu_before_standby` / `peer_silent_stall_detection` / `shared_venv_worktree_identity` (partly covered under Tier 1), among others.

## skills/ — not started

The most operational items become checklists / skills first (Tier 2's closing-keyword and merge-gate families are the leading candidates).

## Notes on provenance

- `#NNNN` in the text refers to PR/issue numbers of the parent project reyn (an AI agent runtime developed almost entirely by a fleet of coding agents). They are kept for provenance and never deleted.
- Pin names are file names in reyn-side sessions' memory directories (`~/.claude/projects/-…-reyn-dev-<session>/memory/`).
