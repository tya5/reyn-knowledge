# CATALOG — Document Index and Source Mapping

This file tracks the mapping between each document and its sources (cross-session memory pins from reyn development).
reyn's pins keep growing, so this table is the basis for periodic sync: when a new pin falls under Tier 1–3, decide between extending an existing doc or adding a new one, and add a row here.

Language convention: `.md` = English, `.ja.md` = Japanese. Japanese is written first; English versions are synced from it.

## docs/verification/ (Tier 1 — verifying AI-produced artifacts)

Map: [The system of verification knowledge](docs/verification/index.md) (pipeline placement of all docs + 4 cross-cutting principles; meta-document with no source pins)
Roles: [Operating with separate coder / tester / reviewer agents](docs/verification/roles.md) (the whole system reprojected onto the role axis + 4 verdict-flow pins)

| Document | Source pins | Main source issues |
|---|---|---|
| [The discipline of strip-falsification](docs/verification/strip-falsification.md) | `feedback_strip_falsify_all_sibling_guard_sites_not_just_one` / `feedback_witness_must_assert_a_value_a_dead_mechanism_cannot_produce` / `feedback_bound_test_must_flip_under_strip` | #2900 #2903 #3159 #3189 #3195 #2825 |
| [Integrity of the strip instrument](docs/verification/strip-instrument-integrity.md) | `feedback_strip_anchor_must_be_unique_or_it_kills_the_wrong_call_site` / `feedback_verify_venv_identity_at_measurement_time_not_by_central_audit` | #3310 #3341 #3363 #3370 #3389 |
| [Wiring tests vs mechanism tests](docs/verification/wiring-vs-mechanism.md) | `feedback_wiring_test_strip_production_callsite_not_mechanism` | #2788 #2801 #2802 #3383 #3385 |
| [How vacuous gates are born](docs/verification/vacuous-gates.md) | `feedback_gate_vacuity_hides_in_terminal_state_only_assertions` / `feedback_containment_gate_must_cover_both_axes_and_children` | #3288 #3299 #3358 #3363 #3370 #3311 #3337 #3341 |
| [Structural blindness of the verification environment](docs/verification/environment-blindness.md) | `feedback_verification_environment_structurally_blind` | #2975 #2962 #2965 #2952 #2973 #2982 #2978 #2981 |
| [Census vs structure](docs/verification/census-vs-structure.md) | `feedback_census_vs_structure_definition_and_checked_premises` / `feedback_perf_fix_needs_fixclass_question_not_correctness_frame` (census part) | #2945 #2949 #2962 #2963 #2961 #2960 #2958 |
| [The discipline of enumeration](docs/verification/enumeration-discipline.md) | `feedback_census_vs_structure_definition_and_checked_premises` (§4) / `feedback_measured_but_the_target_was_off_my_four_instances` (surface-definition part) | #2958 #2965 #2981 #2951 #3429 #3463 |
| [Fix-class review](docs/verification/fix-class-review.md) | `feedback_perf_fix_needs_fixclass_question_not_correctness_frame` | #2937 #2938 #2948 #2945 |
| [Liveness is decided by the producer](docs/verification/liveness-is-producer.md) | `feedback_liveness_is_producer_not_reader` | 2026-07-04 skill sweep, #3357 #3410 #3432 #3433 #3437 |
| [Measuring the wrong target](docs/verification/measurement-target.md) | `feedback_measured_but_the_target_was_off_my_four_instances` | #3437 #3433 #3411 #3458 #3459 #3461 #3429 #3463 |
| [Proving fixture provenance](docs/verification/fixture-provenance.md) | `feedback_prove_replay_fixture_was_rerecorded_by_pairing_with_old_code` | #3190 #3189 #3195 #3183 |
| [Reviewing sweep PRs](docs/verification/sweep-reviews.md) | `feedback_sweep_pr_review_the_untouched_decision_is_invisible_in_the_diff` | #3186 |
| [Shared-helper widening](docs/verification/shared-helper-widening.md) | `feedback_shared_accessor_must_not_outopinion_least_opinionated_caller` | #2947 |
| [Audits must match content](docs/verification/audit-content-match.md) | `feedback_audit_status_not_existence` / `feedback_drift_audit_content_match_not_line_existence` / `feedback_doc_mirror_claim_needs_impl_verify` | #1522 |
| [Prove fixes on the live path](docs/verification/fix-verification-live-path.md) | `feedback_verify_dispatched_bug_is_active_before_fixing` / `feedback_verify_fix_through_declaration_consumer_not_gate_alone` / `feedback_covet_falsify_owner_actual_path_not_test_covered_sibling` | #2413 #2788 #2786 |
| [Liveness of the verification run](docs/verification/verification-run-liveness.md) | `feedback_track_background_verification_hang_not_slow` | #2259 |
| [Claims without context](docs/verification/cross-context-claims.md) | `feedback_hedge_impl_dependent_design_steers` / `feedback_subagent_audit_verdict_needs_owner_context_regrounding` / `feedback_completion_report_must_match_owner_intended_architecture` | #2248 #2246 #1765 |
| [Incomplete work delays discovery](docs/verification/incomplete-work.md) | `feedback_incomplete_work_delays_defect_discovery` | #1505 #1317 |
| [Beyond the happy path](docs/verification/beyond-happy-path.md) | `feedback_review_error_runtime_paths_not_just_happy_structural` / `feedback_live_verify_read_whole_frame_not_just_feature` | #2195 #2238 |
| [Recovery must survive truncation](docs/verification/recovery-truncation.md) | `feedback_recovery_source_must_survive_truncation_review_gate` | #2259 #2260 |
| [Reading green](docs/verification/green-reading.md) | `feedback_green_is_not_evidence_it_ran_skip_is_green` / `feedback_verify_identity_of_measured_code_before_reading_green_or_red` / `feedback_req_resp_plus_one_swallowed_exception` | #2994 #3019 #3031 #2980 #1413 |
| [Test doubles must match the real shape](docs/verification/test-doubles.md) | `feedback_test_fake_inventing_a_field_makes_a_dead_gate_look_tested` / `feedback_envelope_detection_test_real_payload_shape` / `feedback_envelope_shape_fix_verify_fixture_matches_live_producer` / `feedback_enforcement_test_real_resolver_not_none` / `feedback_fake_backend_unit_misses_real_integration` / `feedback_roundtrip_test_nondefault_value` / `feedback_test_claim_must_match_test_content` | #3037 #1439 #1214 #1215 #1356 #1363 #1142 #1146 #1297 |
| [Verifying removals](docs/verification/removal-verification.md) | `feedback_falsify_removal_dead_premise_all_producers` / `feedback_refalsify_own_evidence_readers_and_live_dead_names_before_removal` / `feedback_import_green_not_runtime_green_decouple_consumers` / `feedback_verify_delete_target_absent_not_just_keep_present` / `feedback_removal_docsync_kept_concept_ne_kept_symbol` | #2104 #2151 #2434 |
| [Operating with separate coder / tester / reviewer agents](docs/verification/roles.md) | `feedback_reviewer_speculation_arrives_as_instruction_and_outranks_measurement` / `feedback_gate_merge_on_covet_verdict_not_batch_read_and_merge` / `feedback_wait_for_updated_verdict_before_merge_after_raising_finding` / `feedback_covet_note_gates_author_final` | #3437 #3471 #2774 #2826 #1603 |
| [Completeness sweeps in practice](docs/verification/completeness-sweeps.md) | `feedback_fix_class_completeness_sweep` / `feedback_fix_one_of_n_parallel_paths_sweep_all_siblings` / `feedback_completeness_for_gate_routing_fixes` / `feedback_completeness_sweep_semantic_not_single_mechanism_grep` / `feedback_invariant_coverage_exhaustive_enumeration` / `feedback_registry_enumeration_covers_the_tool_axis_not_the_seam_axis` / `feedback_clean_break_completeness_full_repo_grep_not_src_tests` / `feedback_package_move_completeness_three_ref_classes` / `feedback_git_grep_underreports_use_plain_grep_for_audit` / `feedback_lsp_cold_start_findreferences_silent_underreport` | #1925 #2394 #2397 #1387 #1389 #1954 #1533 #3383 #3376 #2850 #2848 #1700 #1706 #1724 #1953 |
| [Argument hygiene](docs/verification/argument-hygiene.md) | `feedback_cannot_claims_require_tracing_the_mechanism` / `feedback_absence_in_code_cannot_tell_forgotten_from_decided` / `feedback_extrapolation_dies_on_use_not_on_review` / `feedback_a_conceded_argument_can_return_under_a_new_name` / `feedback_form_gives_the_instance_only_the_reason_gives_the_class` / `feedback_not_urgent_and_not_decided_are_different_axes` | #3010 #3011 #3036 #3340 #3334 #3024 #3082 #3411 #3447 |

fixture-provenance additionally consolidates `feedback_verify_failure_claims_by_observation_not_inference` (#1800 #2059 #2060).

Source consolidators: `feedback_verification_blind_spot_family` (16 pins → 13 docs, 2026-07-30) and `feedback_verification_discipline_family` (15 pins → 7 new docs + the fixture-provenance extension, 2026-07-30). Both in the memory-curator dir. Batch 3 (2026-07-30): 15 unconsolidated single pins → 3 docs (reading green / test doubles / verifying removals) + 4 verdict-flow pins → the roles guide. Batch 4 (2026-07-30): 16 pins → 2 docs (completeness sweeps in practice / argument hygiene). **This completes the Tier 1 core: 66 pins → 25 docs + the system map + the roles guide.**

## docs/git-github/ (Tier 2 — agent-driven git/CI operation hazards)

Map: [The map of git/GitHub hazards](docs/git-github/index.md) (structure of the 6 docs; meta-document)

| Document | Source pins | Main source issues |
|---|---|---|
| [Closing keywords fire on surface text](docs/git-github/closing-keywords.md) | `feedback_closing_keyword_in_backticks_is_not_parsed` / `feedback_github_closing_keyword_matches_literal_ignores_context` / `feedback_enumerate_part_of_prs_before_authorizing_a_closing_keyword` | #2951 #2990 #3003 #3006 #3187 #3432 #3462 #3043 #3368 #3015 |
| [Your local tree goes stale silently](docs/git-github/stale-local.md) | `feedback_gh_merge_leaves_local_tree_stale_sync_before_local_grep` / `feedback_sync_local_main_before_sequential_worktree_dispatch` / `feedback_local_checkout_branch_instability` / `feedback_multiref_fetchhead_resolves_to_first_ref_not_branch` / `feedback_line_numbers_are_not_identifiers_across_a_moving_main` | #3385 #3149 #2817 #2919 #1069 #2818 #3082 #3213 |
| [Parallel agents and git](docs/git-github/worktree-parallel.md) | `feedback_parallel_coders_shared_central_file_hazard` / `feedback_same_role_concurrent_session_branch_divergence` / `feedback_explicit_git_add_not_blanket_in_artifact_heavy_session` / `feedback_verify_pushed_tree_matches_working_tree_before_pr` / `feedback_falsify_restore_never_git_checkout_uncommitted` / `feedback_rename_refactor_i001_full_scope_ruff` | #2681 #2838 #1502 #2187 #1685 #1687 |
| [Confirm merges by state](docs/git-github/merge-gates.md) | `feedback_confirm_merge_before_merged_ack` / `feedback_verify_merge_state_not_impl_complete_claim` / `feedback_kill_automerge_poll_before_reversing_merge_decision` / `feedback_merge_order_signature_conflict_sweep_after_landing` / `feedback_dont_over_park_ready_prs_merge_promptly` / `feedback_outstanding_item_must_live_where_the_merge_gate_reads` | #2043 #2447 #3000 #2928 #3121 #2840 #3349 |
| [Issues decay](docs/git-github/issue-lifecycle.md) | `feedback_crosscheck_merged_prs_for_stale_done_issues` / `feedback_crosscheck_merged_prs_before_explaining_dispatching_arc` / `feedback_dispatch_brief_must_reflect_issue_comment_thread_not_just_body` / `feedback_issue_close_requires_condition_verification_record` / `feedback_track_deferred_work_before_close` / `feedback_arc_closure_remainder_must_be_filed_or_explicitly_dropped_in_the_closing_comment` / `feedback_open_issue_progress_continuous_update` / `feedback_open_ticket_count_is_maintenance_cost_prefer_do_over_file` / `feedback_investigate_before_filing_issue` / `feedback_over_cautious_issue_creation` / `feedback_unreproducible_bug_close` / `feedback_task_tracker_id_vs_github_issue_number` | #1406 #1206 #2940 #1115 #2597 #1010 |
| [The traps in docs-only PRs](docs/git-github/docs-prs.md) | `feedback_docs_only_pr_can_break_impl_doc_mirror_test` / `feedback_mermaid_render_check_on_doc_pr` / `feedback_docs_restructure_followup_completeness_gate` / `feedback_no_history_refs_in_user_docs` | #1566 #1568 #3039 #1256 #1257 #2046 |

Tier 2 batch 1 (2026-07-30): 36 pins → 6 docs + map. **First skill**: [closing-keyword-check](skills/closing-keyword-check/skill.md) (executable checklist).

## docs/orchestration/ (Tier 3 — multi-agent fleet operations) — not started

Candidate pins (~14): `background_pytest_poll_stall` / `review_menu_before_standby` / `peer_silent_stall_detection` / `shared_venv_worktree_identity` (partly covered under Tier 1), among others.

## skills/ — not started

The most operational items become checklists / skills first (Tier 2's closing-keyword and merge-gate families are the leading candidates).

## Notes on provenance

- `#NNNN` in the text refers to PR/issue numbers of the parent project reyn (an AI agent runtime developed almost entirely by a fleet of coding agents). They are kept for provenance and never deleted.
- Pin names are file names in reyn-side sessions' memory directories (`~/.claude/projects/-…-reyn-dev-<session>/memory/`).
