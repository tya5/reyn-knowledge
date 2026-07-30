---
name: merge-gates
description: 「merged」は PR の state を見てから言う — hook の発火・コマンドを打った事実・CI の緑はどれも証拠にならない。自動マージポーラーは古い決定を実行し続け、並行 PR は各々緑のまま main を壊す
tags: [git-github, merge, automation, ci]
sources:
  - feedback_confirm_merge_before_merged_ack
  - feedback_verify_merge_state_not_impl_complete_claim
  - feedback_kill_automerge_poll_before_reversing_merge_decision
  - feedback_merge_order_signature_conflict_sweep_after_landing
  - feedback_dont_over_park_ready_prs_merge_promptly
  - feedback_outstanding_item_must_live_where_the_merge_gate_reads
---

# マージは状態で確認する — hook・ポーラー・並行 PR の罠

## 中心命題

> **「merged」と報告してよいのは、`gh pr view <N> --json state,mergedAt` が `MERGED` + 非 null を返した後だけ。** マージコマンドを打った事実も、自動化フックの「merged」通知も、CI の緑も、マージの証拠にならない。

これは[緑を読む技術](../verification/green-reading.ja.md)の「観測物は自分の指示対象を名乗らない」の git 版である。以下、この命題が破られた形を実測順に。

## 1. 文字列検知の hook は「merged」と嘘をつく

コマンド実行を監視する自動化(hook)が「`gh pr merge` という**文字列**」に反応して「PR #N merged」と通知する構成では、**マージが失敗しても通知は発火する**。実測で3回: draft のままの PR(「still a draft」で失敗)、シェルの構文エラーで中断、branch が古くて拒否 — いずれも hook は「merged」と主張し、そのまま人に伝えれば誤報になる。

- 条件付きマージ(`if [緑]; then gh pr merge; fi`)と「merged ✅」の報告を**同じターンで並列に送らない**。マージ実行 → state 確認 → 報告、の順を守る。どうしても同時なら「緑になり次第マージします(未完了)」という**未完了の言い方**にする。
- 他者の「全部 merge 済み」報告も同じゲートを通す: 伝聞を伝播する前に `state==MERGED` を自分で確認する(open のまま残った PR が「全完了」報告に混ざっていた実測がある)。

## 2. 厳格なブランチ保護の下では、マージ失敗が「常態」になる

「base ブランチが最新であること」を必須にする設定(strict status checks)では、**1本マージするたびに他の open PR が全部 BEHIND になる**。その状態の `gh pr merge` は失敗する — そして上記の hook は「merged」と発火する。

- 連続マージの既定コスト: 1本 land するごとに、次の PR を `gh pr update-branch <N>` → CI 再実行を待つ → `state,mergedAt` を確認。**「さっき緑だった」は、もうマージ可能性を意味しない。**

## 3. 自動マージポーラーは「dispatch 時の決定」を実行し続ける

「CI が緑になったらマージする」バックグラウンドのポーリングは便利だが、**dispatch 時点の決定を焼き込んだ fire-and-forget** である。決定が変わっても、ポーラーは知らない。

実測の最悪例: オーナーがその PR の方針を却下したので `gh pr close` した — が、ポーラーは生きていた。1分後に CI が緑になり、ポーラーの `gh pr merge` が **closed の PR を reopen して、却下済みの変更を main にマージした**(revert する羽目になった)。**close はポーラーを無効化しない**。

- **マージの決定が反転した瞬間(却下・保留・要修正)、close より先にポーラーを殺す。**
- ポーラー自身にガードを入れる: マージ直前に `state==OPEN` を再確認する。
- レースの疑いがあれば forensic は `mergedAt > closedAt`(閉じた後にマージされた = ポーラーが勝った)。即 revert。

## 4. 残件は「gate が読む面」に置く

「CLEAR、ただし1点だけ直して」という条件付きの合格を受け取り、その1点を**チャットで**実装者に伝え、PR をマージポーラーの対象に入れたままにした実測: CI が緑になった瞬間、**修正本体だけがマージされ、条件の1点(テスト追加)は積み残された**。

> **自動化した gate は、自分が読む面しか見ない。「CLEAR ただし1点」は人間の記憶の中では条件付きだが、機械にとっては CLEAR である。**

- **残件を機械可読の状態にしてからポーラーを張る**。順序が逆だとこの事故になる。
- 置き場所は環境に依存する。実測では、全エージェントが同一の GitHub ユーザで認証する環境のため **change request(公式のブロック機構)が自 PR に使えず**、実際に使えたのは2つだけ: **PR 本文の Test plan に未チェックの `- [ ]` を残す**(作成者側)と、**draft に戻す**(`gh pr ready --undo`。レビュア側から一方的に打てる唯一の機械可読ブロック — `mergeStateStatus` が CLEAN でもマージ不可になる)。
- 複数 PR を1本のポーラーで回すときは、**条件付きの PR をループから外す**。

## 5. 並行 PR のシグネチャ衝突 — 各々緑のまま main が壊れる

共有シグネチャ(コンストラクタ引数・関数引数・プロトコルのメソッド)を変える PR と、**旧シグネチャの新しい呼び出しを追加する**兄弟 PR が同時に飛んでいると、**両方の CI は緑のまま、両方が main に載った瞬間に main が壊れる**。git のマージはテキストで行われる(重なりが無ければ clean)が、意味の契約は合成で破れる。**CI はこの衝突に構造的に盲目**である(先に分岐したブランチは、兄弟の新しい呼び出しを一度もコンパイルしない)。

- シグネチャを変える PR を land したら、**即座に main を grep して旧シグネチャの呼び出し残存を掃き、fresh な main でフルスイートを回す**。マージした PR のブランチ CI を信用しない。
- 実装者から「main の既存失敗です」と報告が来たら、**同期し直した main で検証し、生きた回帰として扱う**(ノイズ扱いにして迂回しない)。修正は独立の小 PR で(進行中のリファクタの純度を守る)。
- 直接コンストラクタを呼ぶ箇所(壊れる)と、上位 API 経由の箇所(API が内部で吸収し無傷)を区別して移行対象を絞る。

## 6. ready な PR を寝かせない

逆側の罠もある: 「他を優先して」という注意配分の指示を「この ready な PR に触るな」と解釈し、CI 緑+レビュー済みの PR を4時間寝かせた結果、後続 PR と衝突して stale 化した(オーナー: 「勝手に止めて stale を蓄積しないで」)。

- **緑+レビュー済みはすぐマージする**。マージは低コストで、寝かせることが conflict を蓄積する。
- 本物のゲート(未解決の指摘・保留中の検証・設計疑問)があるときだけ hold し、hold 中は定期的に rebase して鮮度を保つ。

## 補 — verdict の読み方

「verdict を読む・マージするを同じバッチでやらない」「指摘を出したら gate は更新後の verdict」「PASS+note は作者の final を待つ」の3規則は[役割分離運用の実務](../verification/roles.ja.md)のマージゲート節を参照(本ドキュメントはその機械側・状態側の補完)。

## チェックリスト

- [ ] 「merged」の報告は `state==MERGED` + `mergedAt` 非 null の確認後か
- [ ] hook・通知・コマンド実行の事実を merged の証拠にしていないか
- [ ] マージ決定が反転したとき、close より先にポーラーを殺したか。ポーラーに OPEN ガードはあるか
- [ ] 条件付き CLEAR を機械可読の状態(未チェック Test plan / draft)にしてからポーラーを張ったか
- [ ] シグネチャ変更を land した直後、main の grep とフルスイートを回したか
- [ ] 緑+レビュー済みの PR を、注意配分の指示を理由に寝かせていないか

## 出典(reyn 開発での実測)

hook 偽発火: #2043/#2051(条件 gate 不発なのに並列 ack)・#2447(draft PR で hook 3回誤発火)。strict 常態化: #3000 設定後の #3165→#3167。ポーラーの reopen+merge: #2928(オーナー却下の1分後)。残件の置き場: #3349(修正だけマージ、テスト積み残し。change request が同一ユーザ環境で使えない実測込み)。シグネチャ衝突: #3121 アーク(#3123×#3122、フルスイートを回した第三者が発見)。寝かせ過ぎ: #2840(4時間で #2841 と衝突)。

## 関連

- [役割分離運用の実務](../verification/roles.ja.md) — verdict フローの3規則(人間側)
- [closing keyword は字面で発火する](closing-keywords.ja.md) — マージが引き起こすもう1つの自動作用
- [ローカルは黙って古くなる](stale-local.ja.md) — マージ後のローカル同期
- [緑を読む技術](../verification/green-reading.ja.md) — 観測物は自分の指示対象を名乗らない
