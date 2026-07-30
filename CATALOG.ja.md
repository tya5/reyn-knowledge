# CATALOG — 全ドキュメント索引と出典対応表

このファイルは、各ドキュメントとソース(reyn 開発のクロスセッション記憶 pin)の対応を管理する。
reyn 側の pin は今後も増え続けるため、この表が定期同期の基準になる: 新しい pin が Tier 1〜3 に該当するなら、既存 doc への追記か新規 doc を判断し、ここに行を足す。

言語規約: `.md` = 英語、`.ja.md` = 日本語。日本語を先に書き、英語版を後追いで同期する。

## docs/verification/(Tier 1 — AI 生成物の検証パターン)

体系マップ: [検証ナレッジの体系](docs/verification/index.ja.md)(13 doc のパイプライン配置と横断4原理。ソース pin なしのメタ文書)

| ドキュメント | ソース pin | 主な出典 issue |
|---|---|---|
| [strip 反証の規律](docs/verification/strip-falsification.ja.md) | `feedback_strip_falsify_all_sibling_guard_sites_not_just_one` / `feedback_witness_must_assert_a_value_a_dead_mechanism_cannot_produce` / `feedback_bound_test_must_flip_under_strip` | #2900 #2903 #3159 #3189 #3195 #2825 |
| [strip 計測器自体の健全性](docs/verification/strip-instrument-integrity.ja.md) | `feedback_strip_anchor_must_be_unique_or_it_kills_the_wrong_call_site` / `feedback_verify_venv_identity_at_measurement_time_not_by_central_audit` | #3310 #3341 #3363 #3370 #3389 |
| [配線テストと機構テスト](docs/verification/wiring-vs-mechanism.ja.md) | `feedback_wiring_test_strip_production_callsite_not_mechanism` | #2788 #2801 #2802 #3383 #3385 |
| [空虚なゲートの生まれ方](docs/verification/vacuous-gates.ja.md) | `feedback_gate_vacuity_hides_in_terminal_state_only_assertions` / `feedback_containment_gate_must_cover_both_axes_and_children` | #3288 #3299 #3358 #3363 #3370 #3311 #3337 #3341 |
| [検証環境の構造的盲目](docs/verification/environment-blindness.ja.md) | `feedback_verification_environment_structurally_blind` | #2975 #2962 #2965 #2952 #2973 #2982 #2978 #2981 |
| [census と structure](docs/verification/census-vs-structure.ja.md) | `feedback_census_vs_structure_definition_and_checked_premises` / `feedback_perf_fix_needs_fixclass_question_not_correctness_frame`(census 部分) | #2945 #2949 #2962 #2963 #2961 #2960 #2958 |
| [列挙の規律](docs/verification/enumeration-discipline.ja.md) | `feedback_census_vs_structure_definition_and_checked_premises`(§4)/ `feedback_measured_but_the_target_was_off_my_four_instances`(面の定義部分)/ `feedback_perf_fix_needs_fixclass_question_not_correctness_frame`(grep 権威部分) | #2958 #2965 #2981 #2951 #3429 #3463 |
| [fix-class レビュー](docs/verification/fix-class-review.ja.md) | `feedback_perf_fix_needs_fixclass_question_not_correctness_frame` | #2937 #2938 #2948 #2945 |
| [生死は producer で判定する](docs/verification/liveness-is-producer.ja.md) | `feedback_liveness_is_producer_not_reader` | 2026-07-04 skill 掃討, #3357 #3410 #3432 #3433 #3437 |
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

fixture-provenance には `feedback_verify_failure_claims_by_observation_not_inference`(#1800 #2059 #2060)を追記統合。

ソースの consolidator: `feedback_verification_blind_spot_family`(16 pin → 13 doc、2026-07-30)および `feedback_verification_discipline_family`(15 pin → 新規 7 doc + fixture-provenance 追記、2026-07-30)。いずれも memory-curator dir。第3弾(2026-07-30)は family 化されていない単発 pin 15 件 → 3 doc(緑を読む技術 / テストの偽物 / 削除の検証)。

## docs/git-github/(Tier 2 — エージェント駆動 git/CI 運用)— 未着手

候補 pin(棚卸しメモより、約28件): `closing_keyword_in_backticks` / `gh_merge_leaves_local_tree_stale` / `line_numbers_not_identifiers` / `merge_order_signature_conflict` / `outstanding_item_must_live_where_gate_reads` ほか。

## docs/orchestration/(Tier 3 — マルチエージェント運用)— 未着手

候補 pin(約14件): `background_pytest_poll_stall` / `review_menu_before_standby` / `peer_silent_stall_detection` / `shared_venv_worktree_identity`(一部は Tier 1 側で言及済み)ほか。

## skills/ — 未着手

実務性の高いものから checklist / skill 化する(Tier 2 の closing-keyword・merge-gate 系が有力)。

## 出典の記法について

- 本文中の `#NNNN` は、母体プロジェクト reyn(AI エージェントランタイム。ほぼ全開発をコーディングエージェントの fleet が実施)の PR / issue 番号。出典の透明性のため削除しない。
- pin 名は reyn 側セッションの memory ディレクトリ(`~/.claude/projects/-…-reyn-dev-<session>/memory/`)のファイル名。
