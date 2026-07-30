# CATALOG — 全ドキュメント索引と出典対応表

このファイルは、各ドキュメントとソース(reyn 開発のクロスセッション記憶 pin)の対応を管理する。
reyn 側の pin は今後も増え続けるため、この表が定期同期の基準になる: 新しい pin が Tier 1〜3(下記の3分類)に該当するなら、既存 doc への追記か新規 doc を判断し、ここに行を足す。

言語規約: `.md` = 英語、`.ja.md` = 日本語。日本語を先に書き、英語版を後追いで同期する。

## docs/verification/(Tier 1 — AI 生成物の検証パターン)

体系マップ: [検証ナレッジの体系](docs/verification/index.ja.md)(全 doc のパイプライン配置と横断4原理。ソース pin なしのメタ文書)
役割マップ: [coder / tester / reviewer 分離運用の実務](docs/verification/roles.ja.md)(全体系の役割軸への射影 + verdict(レビュー判定)フローの pin 4件を統合)

| ドキュメント | ソース pin | 主な出典 issue |
|---|---|---|
| [strip 反証の規律](docs/verification/strip-falsification.ja.md) | `feedback_strip_falsify_all_sibling_guard_sites_not_just_one` / `feedback_witness_must_assert_a_value_a_dead_mechanism_cannot_produce` / `feedback_bound_test_must_flip_under_strip` | #2900 #2903 #3159 #3189 #3195 #2825 |
| [strip 計測器自体の健全性](docs/verification/strip-instrument-integrity.ja.md) | `feedback_strip_anchor_must_be_unique_or_it_kills_the_wrong_call_site` / `feedback_verify_venv_identity_at_measurement_time_not_by_central_audit` | #3310 #3341 #3363 #3370 #3389 |
| [配線テストと機構テスト](docs/verification/wiring-vs-mechanism.ja.md) | `feedback_wiring_test_strip_production_callsite_not_mechanism` | #2788 #2801 #2802 #3383 #3385 |
| [空虚なゲートの生まれ方](docs/verification/vacuous-gates.ja.md) | `feedback_gate_vacuity_hides_in_terminal_state_only_assertions` / `feedback_containment_gate_must_cover_both_axes_and_children` | #3288 #3299 #3358 #3363 #3370 #3311 #3337 #3341 |
| [検証環境の構造的盲目](docs/verification/environment-blindness.ja.md) | `feedback_verification_environment_structurally_blind` | #2975 #2962 #2965 #2952 #2973 #2982 #2978 #2981 |
| [census と structure](docs/verification/census-vs-structure.ja.md) | `feedback_census_vs_structure_definition_and_checked_premises` / `feedback_perf_fix_needs_fixclass_question_not_correctness_frame`(census 部分) | #2945 #2949 #2962 #2963 #2961 #2960 #2958 |
| [列挙の規律](docs/verification/enumeration-discipline.ja.md) | `feedback_census_vs_structure_definition_and_checked_premises`(§4)/ `feedback_measured_but_the_target_was_off_my_four_instances`(検査対象の面の定義の部分) | #2958 #2965 #2981 #2951 #3429 #3463 |
| [fix-class レビュー](docs/verification/fix-class-review.ja.md) | `feedback_perf_fix_needs_fixclass_question_not_correctness_frame` | #2937 #2938 #2948 #2945 |
| [生死は producer で判定する](docs/verification/liveness-is-producer.ja.md) | `feedback_liveness_is_producer_not_reader` | 2026-07-04 の機能語彙一掃作業, #3357 #3410 #3432 #3433 #3437 |
| [測定対象のずれ](docs/verification/measurement-target.ja.md) | `feedback_measured_but_the_target_was_off_my_four_instances` | #3437 #3433 #3411 #3458 #3459 #3461 #3429 #3463 |
| [fixture の出自を証明する](docs/verification/fixture-provenance.ja.md) | `feedback_prove_replay_fixture_was_rerecorded_by_pairing_with_old_code` | #3190 #3189 #3195 #3183 |
| [一括修正 PR のレビュー](docs/verification/sweep-reviews.ja.md) | `feedback_sweep_pr_review_the_untouched_decision_is_invisible_in_the_diff` | #3186 |
| [共有ヘルパーの意味論拡大](docs/verification/shared-helper-widening.ja.md) | `feedback_shared_accessor_must_not_outopinion_least_opinionated_caller` | #2947 |
| [監査は内容を照合する](docs/verification/audit-content-match.ja.md) | `feedback_audit_status_not_existence` / `feedback_drift_audit_content_match_not_line_existence` / `feedback_doc_mirror_claim_needs_impl_verify` | #1522 |
| [修正は生きた経路で証明する](docs/verification/fix-verification-live-path.ja.md) | `feedback_verify_dispatched_bug_is_active_before_fixing` / `feedback_verify_fix_through_declaration_consumer_not_gate_alone` / `feedback_covet_falsify_owner_actual_path_not_test_covered_sibling` | #2413 #2788 #2786 |
| [検証ラン自体の生存確認](docs/verification/verification-run-liveness.ja.md) | `feedback_track_background_verification_hang_not_slow` | #2259 |
| [文脈を持たない主張](docs/verification/cross-context-claims.ja.md) | `feedback_hedge_impl_dependent_design_steers` / `feedback_subagent_audit_verdict_needs_owner_context_regrounding` / `feedback_completion_report_must_match_owner_intended_architecture` | #2248 #2246 #1765 |
| [やり残しは欠陥の発見を遅らせる](docs/verification/incomplete-work.ja.md) | `feedback_incomplete_work_delays_defect_discovery` | #1505 #1317 |
| [正常系の外を踏む](docs/verification/beyond-happy-path.ja.md) | `feedback_review_error_runtime_paths_not_just_happy_structural` / `feedback_live_verify_read_whole_frame_not_just_feature` | #2195 #2238 |
| [復旧の源は破壊を生き残るか](docs/verification/recovery-truncation.ja.md) | `feedback_recovery_source_must_survive_truncation_review_gate` | #2259 #2260 |
| [緑を読む技術](docs/verification/green-reading.ja.md) | `feedback_green_is_not_evidence_it_ran_skip_is_green` / `feedback_verify_identity_of_measured_code_before_reading_green_or_red` / `feedback_req_resp_plus_one_swallowed_exception` | #2994 #3019 #3031 #2980 #1413 |
| [テストの偽物は本物の形に合わせる](docs/verification/test-doubles.ja.md) | `feedback_test_fake_inventing_a_field_makes_a_dead_gate_look_tested` / `feedback_envelope_detection_test_real_payload_shape` / `feedback_envelope_shape_fix_verify_fixture_matches_live_producer` / `feedback_enforcement_test_real_resolver_not_none` / `feedback_fake_backend_unit_misses_real_integration` / `feedback_roundtrip_test_nondefault_value` / `feedback_test_claim_must_match_test_content` | #3037 #1439 #1214 #1215 #1356 #1363 #1142 #1146 #1297 |
| [削除の検証](docs/verification/removal-verification.ja.md) | `feedback_falsify_removal_dead_premise_all_producers` / `feedback_refalsify_own_evidence_readers_and_live_dead_names_before_removal` / `feedback_import_green_not_runtime_green_decouple_consumers` / `feedback_verify_delete_target_absent_not_just_keep_present` / `feedback_removal_docsync_kept_concept_ne_kept_symbol` | #2104 #2151 #2434 |
| [役割分離運用の実務](docs/verification/roles.ja.md) | `feedback_reviewer_speculation_arrives_as_instruction_and_outranks_measurement` / `feedback_gate_merge_on_covet_verdict_not_batch_read_and_merge` / `feedback_wait_for_updated_verdict_before_merge_after_raising_finding` / `feedback_covet_note_gates_author_final` | #3437 #3471 #2774 #2826 #1603 |
| [完全性掃討の実務](docs/verification/completeness-sweeps.ja.md) | `feedback_fix_class_completeness_sweep` / `feedback_fix_one_of_n_parallel_paths_sweep_all_siblings` / `feedback_completeness_for_gate_routing_fixes` / `feedback_completeness_sweep_semantic_not_single_mechanism_grep` / `feedback_invariant_coverage_exhaustive_enumeration` / `feedback_registry_enumeration_covers_the_tool_axis_not_the_seam_axis` / `feedback_clean_break_completeness_full_repo_grep_not_src_tests` / `feedback_package_move_completeness_three_ref_classes` / `feedback_git_grep_underreports_use_plain_grep_for_audit` / `feedback_lsp_cold_start_findreferences_silent_underreport` | #1925 #2394 #2397 #1387 #1389 #1954 #1533 #3383 #3376 #2850 #2848 #1700 #1706 #1724 #1953 |
| [論拠の衛生](docs/verification/argument-hygiene.ja.md) | `feedback_cannot_claims_require_tracing_the_mechanism` / `feedback_absence_in_code_cannot_tell_forgotten_from_decided` / `feedback_extrapolation_dies_on_use_not_on_review` / `feedback_a_conceded_argument_can_return_under_a_new_name` / `feedback_form_gives_the_instance_only_the_reason_gives_the_class` / `feedback_not_urgent_and_not_decided_are_different_axes` | #3010 #3011 #3036 #3340 #3334 #3024 #3082 #3411 #3447 |

fixture-provenance には `feedback_verify_failure_claims_by_observation_not_inference`(#1800 #2059 #2060)を追記統合。

ソースの consolidator: `feedback_verification_blind_spot_family`(16 pin → 13 doc、2026-07-30)および `feedback_verification_discipline_family`(15 pin → 新規 7 doc + fixture-provenance 追記、2026-07-30)。いずれも memory-curator dir(pin を整理する担当セッションのディレクトリ)。第3弾(2026-07-30)は family 化されていない単発 pin 15 件 → 3 doc(緑を読む技術 / テストの偽物 / 削除の検証)。第4弾(2026-07-30)は 16 pin → 2 doc(完全性掃討の実務 / 論拠の衛生)。**これで Tier1 のコア(計 66 pin → 25 doc + 体系マップ + 役割マップ)は完了**(表中の roles は役割マップとして別掲)。

## docs/git-github/(Tier 2 — エージェント駆動 git/CI 運用)

地図: [git/GitHub 運用の罠](docs/git-github/index.ja.md)(6 doc の構造。メタ文書)

| ドキュメント | ソース pin | 主な出典 issue |
|---|---|---|
| [closing keyword は字面で発火する](docs/git-github/closing-keywords.ja.md) | `feedback_closing_keyword_in_backticks_is_not_parsed` / `feedback_github_closing_keyword_matches_literal_ignores_context` / `feedback_enumerate_part_of_prs_before_authorizing_a_closing_keyword` | #2951 #2990 #3003 #3006 #3187 #3432 #3462 #3043 #3368 #3015 |
| [ローカルは黙って古くなる](docs/git-github/stale-local.ja.md) | `feedback_gh_merge_leaves_local_tree_stale_sync_before_local_grep` / `feedback_sync_local_main_before_sequential_worktree_dispatch` / `feedback_local_checkout_branch_instability` / `feedback_multiref_fetchhead_resolves_to_first_ref_not_branch` / `feedback_line_numbers_are_not_identifiers_across_a_moving_main` | #3385 #3149 #2817 #2919 #1069 #2818 #3082 #3213 |
| [並行エージェントと git](docs/git-github/worktree-parallel.ja.md) | `feedback_parallel_coders_shared_central_file_hazard` / `feedback_same_role_concurrent_session_branch_divergence` / `feedback_explicit_git_add_not_blanket_in_artifact_heavy_session` / `feedback_verify_pushed_tree_matches_working_tree_before_pr` / `feedback_falsify_restore_never_git_checkout_uncommitted` / `feedback_rename_refactor_i001_full_scope_ruff` | #2681 #2838 #1502 #2187 #1685 #1687 |
| [マージは状態で確認する](docs/git-github/merge-gates.ja.md) | `feedback_confirm_merge_before_merged_ack` / `feedback_verify_merge_state_not_impl_complete_claim` / `feedback_kill_automerge_poll_before_reversing_merge_decision` / `feedback_merge_order_signature_conflict_sweep_after_landing` / `feedback_dont_over_park_ready_prs_merge_promptly` / `feedback_outstanding_item_must_live_where_the_merge_gate_reads` | #2043 #2447 #3000 #2928 #3121 #2840 #3349 |
| [issue は劣化する](docs/git-github/issue-lifecycle.ja.md) | `feedback_crosscheck_merged_prs_for_stale_done_issues` / `feedback_crosscheck_merged_prs_before_explaining_dispatching_arc` / `feedback_dispatch_brief_must_reflect_issue_comment_thread_not_just_body` / `feedback_issue_close_requires_condition_verification_record` / `feedback_track_deferred_work_before_close` / `feedback_arc_closure_remainder_must_be_filed_or_explicitly_dropped_in_the_closing_comment` / `feedback_open_issue_progress_continuous_update` / `feedback_open_ticket_count_is_maintenance_cost_prefer_do_over_file` / `feedback_investigate_before_filing_issue` / `feedback_over_cautious_issue_creation` / `feedback_unreproducible_bug_close` / `feedback_task_tracker_id_vs_github_issue_number` | #1406 #1206 #2940 #1115 #2597 #1010 |
| [docs-only PR の罠](docs/git-github/docs-prs.ja.md) | `feedback_docs_only_pr_can_break_impl_doc_mirror_test` / `feedback_mermaid_render_check_on_doc_pr` / `feedback_docs_restructure_followup_completeness_gate` / `feedback_no_history_refs_in_user_docs` | #1566 #1568 #3039 #1256 #1257 #2046 |

Tier2 第1弾(2026-07-30): 36 pin → 6 doc + 地図。**skills/ 第1弾**: [closing-keyword-check](skills/closing-keyword-check/skill.ja.md)(実行可能な checklist)。

## docs/orchestration/(Tier 3 — マルチエージェント運用)— 未着手

候補 pin(約14件): `background_pytest_poll_stall` / `review_menu_before_standby` / `peer_silent_stall_detection` / `shared_venv_worktree_identity`(一部は Tier 1 側で言及済み)ほか。

## skills/ — 未着手

実務性の高いものから checklist / skill 化する(Tier 2 の closing-keyword・merge-gate 系が有力)。

## 出典の記法について

- 本文中の `#NNNN` は、母体プロジェクト reyn(AI エージェントランタイム。ほぼ全開発をコーディングエージェントの fleet が実施)の PR / issue 番号。出典の透明性のため削除しない。
- pin 名は reyn 側セッションの memory ディレクトリ(`~/.claude/projects/-…-reyn-dev-<session>/memory/`)のファイル名。
