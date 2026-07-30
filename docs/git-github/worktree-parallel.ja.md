---
name: worktree-parallel
description: worktree はファイルシステムを分けるが、git の状態(stash・refs・共有 checkout)は分けない — 並行エージェントの fan-out・stash 禁止・コミット後の木の検証・リネーム後の linter まで、並行作業の git 規律
tags: [git-github, git, multi-agent]
sources:
  - feedback_parallel_coders_shared_central_file_hazard
  - feedback_line_numbers_are_not_identifiers_across_a_moving_main
  - feedback_same_role_concurrent_session_branch_divergence
  - feedback_explicit_git_add_not_blanket_in_artifact_heavy_session
  - feedback_verify_pushed_tree_matches_working_tree_before_pr
  - feedback_falsify_restore_never_git_checkout_uncommitted
  - feedback_rename_refactor_i001_full_scope_ruff
---

# 並行エージェントと git — worktree が分けるもの・分けないもの

## 前提

複数のコーディングエージェントを同じリポジトリで並行に走らせる運用では、各エージェントに **worktree**(同一リポジトリの並行チェックアウト。`git worktree add` で作る)を与えるのが定石である。worktree は**作業ディレクトリを分離する** — しかし **git の内部状態の多くは全 worktree で共有**される(`.git` 本体は1つ)。この「分かれているようで分かれていない」境界が、並行運用の事故の源になる。

> **「隔離 worktree にいるから安全」は、ファイルシステムについては真、git の状態(stash・refs・共有 checkout)については偽。**

## 1. fan-out の前に「共有の中心ファイル」を見る

並列化が正しいのは、各ユニットが**互いに素なファイル**を触るときである。全ユニットが同じ中心ファイル(レジストリ、台帳、enum、ディスパッチ表)を編集する作業を3並列で撒いた実例では、得られたのは (1) その1ファイルでのマージ衝突と、(2) 誰かが `git stash` を使った瞬間の**worktree 横断の stash 混線**だけだった。

- fan-out(複数エージェントへの一斉分配)の前に: **各ユニットが同じ中心ファイルを編集しないか**を確認する。編集するなら、(a) 直列化(A がマージ → B が rebase → …)、(b) **中心ファイルは1人が所有**(他のエージェントは中心ファイル以外だけを編集)、(c) 中心ファイルに1回しか触れない構造、のどれかにする。
- どの構成でも**マージは直列**(rebase の鎖)にする — 中心ファイルへの編集が、前の結果の上に載るように。

## 2. stash は使わない — スタックはリポジトリ全体で1本

`git stash` のスタックは**全 worktree 共有で1本**である。実測: 隔離 worktree で `git stash`(退避対象なし)→ `git stash pop` が、**別セッションが積んだ古い退避内容を自分のツリーに展開**した。

- **stash を使わない。** 一時退避は自分のブランチに閉じる WIP commit で行う。
- stash 一覧が非空でも**自分のものと仮定しない**。他人の stash は pop も drop もしない。

関連する復元の罠: 検証のために1行 strip(検証のための一時的なコード無効化)した後の復元に **`git checkout <file>` / `git restore <file>` を使わない** — これらは**そのファイルの内容を HEAD まで戻す**ので、strip した1行だけでなく、**そのファイルの未コミット差分の全体を黙って消す**。レビュー中の大きな未コミット diff の上で1点だけ検証する場面が最も危険。strip の前に `cp <file> /tmp/<file>.bak`、復元は cp 戻しか、その1行だけの逆編集で行う。

## 3. 最初の git コマンドは、共有 checkout に落ちることがある

worktree を割り当てられたエージェントの**最初の** `git checkout -b` が、worktree ではなく**共有のプライマリ checkout**(他のセッションが読む main の作業ツリー)で走った実例が、連続する2エージェントで起きた。指示に「worktree で作業せよ」と書いてあっても、**最初のコマンドは作業ディレクトリが worktree に向く前に実行されうる**。

- 依頼に書く: 「**最初の git コマンドの前に `git rev-parse --show-toplevel` を実行し、worktree のパスであることを確認せよ**」。
- 実例の2件は**実装エージェント自身が commit 前に気づいて復旧**した。そのうえで受け入れ側も**各エージェントの作業完了ごとに共有 checkout の `git status` を確認**し、両件の清浄を独立に裏づけた — 信頼しつつ検証する(trust-but-verify)。

## 4. 同名ロールの並行セッション — push reject は一次証拠

同じロール名のセッションが2つ同時に走り、調整メッセージが**片方にしか届かない**ことがある。セッション間メッセージの自分の受信箱が空でも「最新の指示を全部受け取っている」とは限らない。実例では、方向転換(HOLD(一時停止指示)+ 方針変更)が別セッションにだけ届き、古い前提で実装して push → **non-fast-forward reject**(相手が先に push している)で初めて衝突に気づいた。

- **push reject は「他セッションがこのブランチを触っている」の一次証拠**。即 `git fetch` + `git log HEAD..origin/<branch>` で相手の commit を読む。**絶対に force-push しない**。
- 相手の commit がより新しい権威(HOLD・方針転換)を明記していれば、**自分の未 push commit を破棄して収束**する(`git reset --hard origin/<branch>`)。未 push の commit は誰にも影響しておらず、reflog にも残るので、この破棄は安全である。
- 収束後は、**自分がどんな古い前提で動き、何を破棄したかを透明に報告**し、以前の「これから修正版を出します」等の宣言を撤回する。

## 5. コミットの衛生 — add の仕方より「コミット後の木の検証」が本体

コミット段階には**逆向きの罠が2つ**あり、どちらか一方の対策だけでは他方を踏む:

- **罠A(混入)**: 長いセッションは未追跡の作業ファイル(検証用バックアップ、プローブ出力、実験スクリプト)を溜める。そこで `git add -A` を使うと全部掃き込む — 実測: 4ファイルのつもりが**46ファイル・+8019行**を巻き込み、マージ阻止された。
- **罠B(欠落)**: 明示パスの `git add <p1> <p2> <p3>` は、**パスが1つでも実在しないと原子的に中断し、何も stage しない**。続く commit は事前に stage 済みのものだけを積む — 新規ファイルが黙って落ち、**ローカルのテストは(working tree に対して走るので)緑のまま**、コードの無い PR が出る。

∴ 統一規則は「add の流儀」ではなく**事後検証**である:

```bash
git status --porcelain          # 空 = working tree と HEAD が一致(テストが HEAD(= これから push する内容)を測っている保証)
git ls-tree -r HEAD --name-only | grep <新ファイル>     # 新規物が木に入った
git show HEAD:<file> | grep <新シンボル>                # 配線が commit に入った
git diff --stat origin/main...HEAD                      # 期待のファイル集合と一致
gh pr view <N> --json files                             # push 後の権威(変更ファイル名の一覧。changedFiles は件数のみ。ローカル比較は木が古いと誤る)
```

作業ファイル(artifact)が多いセッションでは明示 add(+ `git diff --cached --name-only` の照合)を使い、検証用バックアップ(`.bak` 等)はセッション末に消す。

## 6. リネーム後の linter — テストのファイル群が噛みつく

パッケージ/モジュールのリネームは import 文の**アルファベット順の位置**を変えるため、import 整列の linter(isort / ruff I001)が**書き換えた全 importer** — しばしば数百のテストファイル — で発火する。ローカルの linter 実行が `src/` 限定だったり rebase 前だったりすると「clean」と誤信し、CI で赤になる(同じ日に2件の実測)。

- リネーム系リファクタの後は、**最終の rebase 済み HEAD 上で** `ruff check src tests --fix`(CI と同じ全スコープ)を回してから push する。
- レビュー側: リネーム PR の CI 赤は、まず tests/ の I001 を疑う。

## チェックリスト

- [ ] fan-out 前: 全ユニットが触る共有中心ファイルは無いか。マージは直列か
- [ ] stash を使っていないか。strip の復元に `git checkout <file>` を使っていないか
- [ ] (依頼に)最初の git コマンド前の worktree 確認を入れたか。(受け入れ側)共有 checkout の status を確認したか
- [ ] push reject に force で応じていないか。相手の commit を読んで収束したか
- [ ] コミット後: porcelain 空・ls-tree・show・diff --stat・files で木を検証したか
- [ ] リネーム後: 全スコープの linter を rebase 後 HEAD で回したか

## 出典(reyn 開発での実測)

中心ファイル fan-out と stash 混線: #2681 アーク(3並列で衝突+refs/stash clobber)・空 stash 後の pop が他人の WIP を展開した事例(2026-07-28。出典 pin 内の参照番号は内部作業項目のもので、GitHub issue 番号ではない)。共有 checkout への初手: 0061 アーク(連続2エージェント、commit 前に自己捕捉)。同名ロール並行: #2838(HOLD が別セッションへ、reject で検知し収束)。混入: #1502(46ファイル+8019行、`git reset --soft` で再構成)。欠落: pathspec 中断による部分 commit(レビュアの事前読みが捕捉)。I001: #1685/#1687(同日2件)。

## 関連

- [ローカルは黙って古くなる](stale-local.ja.md) — stale base・共有ツリー・行番号の時間ずれ
- [strip 計測器自体の健全性](../verification/strip-instrument-integrity.ja.md) — 共有 venv という同型の「分かれていない」面
- [完全性掃討の実務](../verification/completeness-sweeps.ja.md) — リネームの参照3クラス
