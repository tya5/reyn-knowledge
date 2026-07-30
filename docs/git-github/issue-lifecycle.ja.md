---
name: issue-lifecycle
description: issue は劣化する — 本文は起票時のスナップショットで、解決はスレッドと merged PR 側に溜まる。dispatch 前の照合、クローズの検証記録、残件の決着(「次アークで」は第三の状態)、起票の較正(両方向)まで
tags: [git-github, issues, process]
sources:
  - feedback_crosscheck_merged_prs_for_stale_done_issues
  - feedback_crosscheck_merged_prs_before_explaining_dispatching_arc
  - feedback_dispatch_brief_must_reflect_issue_comment_thread_not_just_body
  - feedback_issue_close_requires_condition_verification_record
  - feedback_track_deferred_work_before_close
  - feedback_arc_closure_remainder_must_be_filed_or_explicitly_dropped_in_the_closing_comment
  - feedback_open_issue_progress_continuous_update
  - feedback_open_ticket_count_is_maintenance_cost_prefer_do_over_file
  - feedback_investigate_before_filing_issue
  - feedback_over_cautious_issue_creation
  - feedback_unreproducible_bug_close
  - feedback_task_tracker_id_vs_github_issue_number
---

# issue は劣化する — 本文・スレッド・クローズ・残件の規律

## 前提

エージェント運用では issue tracker が**タスク管理の実体**になる(誰が何をやるかを issue の open/close が決める)。そして issue の情報は**時間とともに決まった形で劣化する**: **本文(body)は起票時点のスナップショット**であり、その後の計測・真の原因・直した PR は**コメントスレッドと merged PR の側**に蓄積される。この非対称を忘れると、「もう直っている問題を再実装する」「完了済みのアークを『これから着手』と説明する」という無駄が量産される。

## 1. dispatch(実装者への作業割り振り)・解説の前 — body だけを読まない

**stale-done(直っているのに open)**: merged PR が `fix(N):` や `Fixes #N` をタイトルや本文に含んでいても、[書式によっては auto-close しない](closing-keywords.ja.md)ため、**解決済みの issue が open のまま残る**。実測: open 16件を body だけで精査して「全部有効」と結論 → 実は2件が実装済みで、片方は**再実装の dispatch 寸前**だった。

- open issue の整理・dispatch の前に: `gh pr list --state merged --search "<N> in:title,body"` で **merged PR と突き合わせる**。

**stale-premise(前提が覆っている)**: 複数 issue のアークをオーナーに解説し、GO(着手承認)を得て dispatch した後で、**アークの中核が既に別 PR 群で landed 済み**と判明した実測がある(不要な PR を1本 merge してしまった)。GO は「本当に未着手の作業」に対してだけ意味を持つ。

- アークを解説・dispatch する前に: **各 issue の核心前提を1文で言語化**し、merged PR 履歴と現在の main で**まだ真か**を照合する。

**body とスレッドの食い違い**: 依頼(brief)を body の原因分析から書いたら、実装者が**スレッド**を読んで発見した — (1) body の仮説は後の実測で覆っていた、(2) 真のホットパスは別、(3) **それは既に merged PR で直っていた**。実装者はその場しのぎの修正(band-aid)を正しく拒否した。

- 依頼を書く前に `gh issue view <n> --comments` で**スレッド全体**を読む。body とスレッドが食い違ったら、**新しい一次データであるスレッドが勝つ**。

## 2. クローズの規律 — closed issue は自己文書化していなければならない

issue は監査の単位である。closed issue は「**なぜ done なのか**」をそれ自体で語れなければならない:

- **クローズ記録**: 最終状態レポート + クローズ条件の検証記録(条件のチェックリスト + 各 ✓ + 証拠の参照)を **issue 本体に**残す。
- **auto-close の盲点**: `Closes #N` によるマージ連動クローズでは、**検証記録が PR 側にしか残らない**。マージ後に issue へ検証コメントを追記する(怠ると、issue だけを見る人には「勝手に閉じた」に見える)。
- **クローズ直前に、原起票の全文を再読**して全スコープ(全サブ要求)が満たされたか照合する。部分完了なら閉じずに注記を追記。
- **クローズの種類で承認が分岐**: 実装+検証済みの proper-close は自走(承認を待たずに実行)してよい(過剰な承認伺いは逆に禁止)。**実装せずに畳む** close(「測ったが効果なし」「スコープ外にする」)は**オーナーの事前許可**が要る — それは作業の中止という判断だからである。

## 3. 残件の決着 — 「次アークで」は第三の状態

アーク(一連の作業のまとまり)や umbrella issue(複数 PR を束ねる親 issue)を閉じるとき:

> **残件は、closing comment の中で「起票済み(#番号を書く)」か「明示的に捨てた(理由を書く)」のどちらかに決着させる。「次アークで」「優先度が付いたら起票」は第三の状態であり、それが劣化そのものである。**

同じ issue に**自然実験**が残っている: 同じ日に同じアークで閉じた2つの残件のうち、closing comment に**書かれた**方は今日まで辿れ、**書かれなかった**方は「出所不明のタスク項目」に劣化して中身が誰の記憶にも残らなかった。

- 「後で起票する」と書くだけの deferred は**事実上チケット無し** = 不可視化。閉じると同時に実起票し、closed 側に forward-pointer、新 issue 側に back-ref。
- **起票前の verify-before-file**: その残件が**現在の main で実在するか**を一次証拠で確認する(grep のヒット先は「既に除去済み」と知らせる通知文だった — それを未対応の残件と誤読して起票しかけた実測がある)。
- 設計記録(ADR)の deferred 節は **grep 可能な受け皿**として有効 — 粒度不明のまま抱えるより、そこを指す方が強い。
- 自分のタスクリストに**チケットの無い項目**を見つけたら赤信号: 起票するか捨てるかを今決める。放置は「なぜか終わらない項目」として無限に残る。

## 4. 進捗は landing のたびに反映する

「後でまとめて更新」は、まとめる時に漏れる(同じ見落としを1セッションで3回繰り返した実測)。

- PR が issue を部分/完全に解決して merge されたら、**その場で** issue に「landed: 〜 / remaining: 〜」を書く。完全なら close。
- 基盤的な変更(コンポーネント削除・大リファクタ)が landed したら、**それを参照する全 open issue を横断チェック**し、obviated(前提消滅)を即 annotate/close する。
- マルチセッションのアークでは、**umbrella issue に単一の STATUS コメント**(✅完了 / 🔄進行中 / 📋残り)を維持し、PR が landed するたびに追記する。そして**状況を聞かれたら、記憶からではなくその記録を読んでから答える** — 完了済みのアークを作業記憶から「未着手」と2度誤答した実測が、この規律の由来である。

## 5. 起票の較正 — 両方向に誤る

**過少検証(推測で起票しない)**: issue は他の全セッションが真に受ける、外部から監査される場である。起票前に: 症状を**直接観測**したか(ログ行・コミット・ジョブ ID を引用) / 再現コマンドはあるか / **意図された挙動でないこと**を除外したか / スコープは1件に収まっているか / 初見の読者が直し方と検証法を理解できるか。

**過剰保有(open は保守コスト)**: open チケットは毎回の棚卸しで再読・再トリアージされる**実在のコスト**である。

- **file より do**: ブロッカーが無く決着済みの follow-up は、チケット化せず**今閉じ切る**。起票するのは、本当に先送りが必要(GO 待ち・設計待ち・外部条件)で追跡に値するものだけ。「いつか欲しいかも」は起票しない。
- 定期的に merged PR と open issue を突き合わせ、stale-done を検証記録つきで閉じる。**誤クローズは取りこぼしより悪い**ので、完了が不確かなら open のまま。

**過剰エスカレーション(何でも承認待ちにしない)**: 既存パターンの N+1 適用・進行中の作業群の自然な次の一歩は、自走の範囲である(実測でのオーナー評: 「この程度のことで承認待ちはぬるい」)。エスカレートすべきは、大きなアーキテクチャ変更・新方針の確立・完全に新しい軸・ユーザー向け UX の大変更・コスト超過・セキュリティ姿勢の変更・作者間の設計衝突。

**再現しないバグは深追いしない**: 1度観測されたが N≥3 回の再試行・現 HEAD で再現しないものは、「ghost bug、再観測されたら再開」と明記して close する(これは§2の「実装せずに畳む close」の一種にあたる。実例の運用では「再現しないものは深追い不要で close してよい、今後も」という包括の事前許可が出ていた — 同種の包括許可が無い環境では§2に従い個別に確認する)。部分的に再現するなら flaky として別軸で追跡する。ghost の追跡は複数人のサイクルを溶かす。

## 6. ID の衝突 — 内部トラッカと GitHub 番号

セッション内部のタスク管理 ID(#187 等)と GitHub の issue/PR 番号は**同じ形で別物**を指す。受け手は #N を GitHub と読む(自然な解釈)。実測: 内部 ID で仕事を振り、受け手が verify して「その GitHub 番号は無関係の merged PR」と発見した。

- **内部トラッカの項目はトピック名で言い、#N は GitHub にだけ使う。** 受け手側も、#N が GitHub に解決するかを確認してから動く。

## チェックリスト

- [ ] dispatch 前: merged PR との突き合わせ・前提を1文で言語化して照合・スレッド全読をしたか
- [ ] close 時: 検証記録を issue 本体に残したか。原起票の全文と照合したか。実装せず畳む close に許可を得たか
- [ ] 残件: 起票済み(#番号)か明示的破棄(理由)に決着したか。「次アークで」を書いていないか
- [ ] landing のたびに issue へ反映したか。status 質問には記録を読んでから答えたか
- [ ] 起票前: 一次証拠・再現・意図的挙動の除外。起票せず今閉じ切れないか。エスカレーションは7種に該当するか
- [ ] #N は GitHub にだけ使ったか

## 出典(reyn 開発での実測)

stale-done: #1406/#1407・#1206/#1291。stale-premise: #187 capstone(不要 PR #1435 を merge)。スレッド優先: #2940(body の仮説は覆済み・修正済み、実装者が拒否)。クローズ記録: 2026-06-19 オーナー指示 + 5 issue の遡及 backfill、content-cancel 違反 #1791。残件: #1115(未起票 deferred → #1199/#1200 を遡及起票)・#2597(自然実験: 書かれた残件と書かれなかった残件)・false-positive 起票を止めた verify-before-file(2026-06-01)。進捗反映: #1375/#1397/#1401(3連続 miss)・Control IR アークの2度の誤答(2026-07-04 オーナー再強調)。較正: 2026-07-10 オーナー4連メッセージ(open 積み上がり)・#1010(「ぬるい」)・ghost bug close 指示(2026-05-28)。ID 衝突: 2026-06-06。

## 関連

- [closing keyword は字面で発火する](closing-keywords.ja.md) — auto-close の機構そのもの
- [やり残しは欠陥の発見を遅らせる](../verification/incomplete-work.ja.md) — 残件可視化の一般原理(本ドキュメントはその issue 面)
- [論拠の衛生](../verification/argument-hygiene.ja.md) — 「急がない」と「決めていない」は別の軸
- [監査は内容を照合する](../verification/audit-content-match.ja.md) — 記録と実体の照合の一般形
