---
name: stale-local
description: ローカルは黙って古くなる — サーバ側マージはローカルの checkout を動かさない。stale な grep は偽の指摘を生み、stale な base からの branch は静かな mass revert になり、行番号は動く main を跨げない
tags: [git-github, git, measurement]
sources:
  - feedback_gh_merge_leaves_local_tree_stale_sync_before_local_grep
  - feedback_sync_local_main_before_sequential_worktree_dispatch
  - feedback_local_checkout_branch_instability
  - feedback_multiref_fetchhead_resolves_to_first_ref_not_branch
  - feedback_line_numbers_are_not_identifiers_across_a_moving_main
---

# ローカルは黙って古くなる — 「自分が見た木」と「効く木」は別

## 前提 — なぜエージェント運用で特に起きるか

`gh pr merge`(GitHub CLI でのサーバ側マージ)は **origin/main(リモートの正)を進めるが、手元の checkout には何もしない**。人間の開発では「マージしたら pull する」が自然に挟まるが、エージェントが1セッションで何本も PR をサーバ側マージしていくと、**ローカルの main は気づかないうちに5〜15コミット遅れる**。その古い木に対する grep・診断・行番号・分岐(branch cut)が、以下の事故を量産する。

これは[測定対象のずれ](../verification/measurement-target.ja.md)・[strip 計測器の健全性](../verification/strip-instrument-integrity.ja.md)の git 版である: **「自分が見た木」と「効く木」は別物**で、しかもズレは**時間とともに黙って開く**。

## 事故1 — stale な grep が、偽の指摘と偽の安心を生む

実例(偽の指摘): レビュアーが完全性の掃討(「同じクラスの未修正兄弟は他に無いか」)をローカルの木に grep で撃ち、ヒットした箇所を「未修正の兄弟」として修正要求に載せた。**そのシンボルは5コミット前に削除済み**だった。実装者が `git show origin/main:<path> | grep -c ...` → 0 で反証。健全な PR に偽の指摘を投げる形で、[strip 計測器](../verification/strip-instrument-integrity.ja.md)の「健全なゲートへの誤告発」と同じ損害である。

実例(偽の警報): 別の日、シグネチャ変更 PR のマージ後にローカル grep で「削除された引数を使う残存が10件」— しかし PR のフルスイートも CI も緑で、矛盾する。原因はローカル HEAD が **15コミット遅れ**で、その10件は移行済みだった。

> **完全性・「ここに在る/無い」系の主張の grep は、ローカルではなく `git show origin/main:<path>` か `git grep <パターン> origin/main` で撃つ。** 通常の探索なら stale でも実害は小さいが、「これで全部」「ここにも在る」という主張は、**木が古いだけで偽になる**。

付随: 挙動確認の `python -c "import mypkg; ..."` も**別の clone を解決していることがある**(`mypkg.__file__` で解決先を確認するまで信用しない — [strip 計測器](../verification/strip-instrument-integrity.ja.md)壊れ方3)。

## 事故2 — stale な base からの branch は「静かな mass revert」になる

段階的なアーク(フェーズ N+1 がフェーズ N のマージ済みコードに依存する)で、次のフェーズの作業ブランチを**ローカル main から切る**と、base に前フェーズが入っていない。結果は2段階に悪い:

1. **衝突するだけなら、まだ良い**(気づけるから)。
2. 本当の害は、**実装者の設計判断が腐る**こと。実例では、base に前フェーズが無かったため、実装者は「その機能はまだ存在しない」を前提にスコープを決めてしまった — 衝突の解決だけでは直らず、**stale な base が無効化した設計判断の見直し**まで必要だった。

さらに一般化される(worktree のディスパッチに限らない): **docs だけの PR でも起きる**。実例では、9コミット遅れの detached HEAD(どのブランチにも乗っていない checkout 状態)から切った「+111行のドキュメント PR」の diff が、実は **−789行 = 9コミット分の巻き戻し**を含んでいた。「+N docs」という見出しは additive(追加のみ)の証明にならない。

- 分岐前に同期: `git fetch origin main && git merge --ff-only origin/main`。実装側にも開始時の `git rebase origin/main` を依頼に含める。
- **マージ前の tripwire**: 小さいはずの PR に予期しない削除・大きな負の行数がある、または `git merge-base <branch> origin/main` が現在の origin/main と一致しない → stale base。マージせず rebase して diff を取り直す。

## 事故3 — checkout は、いつのまにか別のブランチにいる

実例: マージ後の `git pull origin main` が「divergent branches」で失敗。原因は、**自分の checkout がいつのまにか他者の feature ブランチの上にいた**(共有の作業ディレクトリで他エージェントが checkout していた)。

- `git pull` / `git reset` などローカル状態に依存する操作の前に、**`git branch --show-current` が main であることを確認**する。
- そもそも検証は**ブランチ非依存の参照を主軸**にする: `git fetch origin main` → `origin/main` 参照で読む。ローカルの checkout 状態と無関係に正しい。
- 根本対処は**置き場所**: エージェントは共有ツリーで作業せず、自分専用の clone / worktree(同一リポジトリの並行チェックアウト)を持つ。

## 事故4 — FETCH_HEAD は「最初にフェッチした ref」を指す

`git fetch origin main <branch>` のように**複数 ref を同時にフェッチ**した後の `git show FETCH_HEAD:<file>` は、**ブランチではなく最初の ref(通常 main)のファイルを読む**。

実例: ブランチに rename が入ったかをこの形で「検査」し、main の古い内容を見て「rename が載っていない」と実装者に誤った指摘を投げた(実際は最初の commit から入っていた)。**測定誤りの中でも最悪の形** — 古い内容が「見える」ので、確信を持って行動してしまい、しかも他者への不当な帰属を伴う。

- ブランチ内容の検査は**明示 ref**: `git show origin/<branch>:<file>`。複数 ref フェッチ後の FETCH_HEAD は使わない。
- 指摘に相手が異議を唱えたら、**両側を明示 ref で**取り直してから立場を維持する。誤りと分かったら、指摘と、それを伝播した先への**撤回を即座に**行う。

## 事故5 — 行番号は「動く main」を跨ぐ識別子ではない

長時間の分析(数百件の行番号つき分類)を続けている間に、共有ツリーが別の作業で pull され、対象ファイルが**+107行**動いた実例がある。分析データは全て古いスナップショット基準になっていた。救ったのは**適用直前の内容ベース再アンカー**(行番号の指す内容が期待と一致するかの照合)で、100件中87件の不一致が出て発覚 — 照合がなければそのまま適用してファイルを破壊していた。

- **測った瞬間と使う瞬間が離れる作業**(分析→一括適用、grep 結果の行番号保持、「N行目を strip」型の指示)は、**適用直前に内容で再アンカー**する。
- 照合は**一致率ではなく不一致の件数**を見る(数件のズレは一致率では埋もれる)。
- **他者が使う識別子**(行番号・シンボル位置・「N件ある」)を出すときは、ローカル HEAD ではなく `origin/<ref>` に対して測る — 「たまたま同一だった」は規律ではない。
- 比較の baseline が**自分の作業ツリーと矛盾している** diff は、退行ではなく同期問題を先に疑う。

### 付記 — worktree は git 状態の多くを共有する

worktree は**作業ディレクトリを分けるが、git の状態の多くは分けない**。実測: `git stash` のスタックは**全 worktree で1本**であり、隔離 worktree での `git stash pop` が**他セッションの古い退避内容を自分のツリーに展開**した(即復旧、被害ゼロ)。

- **stash を使わない**(一時退避は自分のブランチに閉じる WIP commit で)。stash 一覧が非空でも自分のものと仮定しない。
- 「隔離 worktree にいるから安全」は、**ファイルシステムについては真、git 状態(stash / refs / config)については偽**。

## チェックリスト

- [ ] 完全性・存在の主張の grep を `origin/main` 参照で撃ったか(ローカル木ではなく)
- [ ] branch を切る前・依存フェーズのディスパッチ前に、local main を origin に同期したか
- [ ] 小さいはずの PR に負の行数・予期しない削除は無いか。merge-base は現在の origin/main か
- [ ] ローカル状態依存の操作の前に、現在のブランチを確認したか
- [ ] ブランチ内容の検査に FETCH_HEAD を使っていないか(明示 ref か)
- [ ] 行番号・件数を人に渡すとき、origin ref で測ったか。適用直前に内容で再アンカーしたか

## 出典(reyn 開発での実測)

事故1: #3385(5コミット遅れで削除済みシンボルを誤指摘。実装者は「存在しないシンボルへの免除記載はそれ自体 drift」として免除記録も正しく拒否)・#3149(15コミット遅れの偽残存10件)。事故2: #2817(P4 が P3 抜きの base から分岐、設計判断まで汚染)・#2919(detached HEAD からの docs PR が −789行の revert を内包、self-caught)。事故3: #1069(checkout が他者の feature ブランチ上、共有ディレクトリ起因 — 専用 clone へ移行で解決)。事故4: #2818(FETCH_HEAD が main を解決、偽の指摘と不当な帰属)。事故5: #3082(+107行ドリフト、再アンカー 87/100 不一致で破壊回避、復旧 381/389)・#3213(worktree 横断の stash 混入)。

## 関連

- [測定対象のずれ](../verification/measurement-target.ja.md) — 「どの木で・どの commit で測ったか」の中継規則
- [strip 計測器自体の健全性](../verification/strip-instrument-integrity.ja.md) — 測定対象の同一性(venv 版)
- [削除の検証](../verification/removal-verification.ja.md) — 明示 fetch した tip に対する不在 assert
