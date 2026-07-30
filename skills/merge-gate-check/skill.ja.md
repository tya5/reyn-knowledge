---
name: merge-gate-check
description: マージ前後の状態検査手順。merged は state で確認 → 自動マージ/ポーラーの停止 → 残件は gate が読む面へ → 連続マージ後の合成検査 → ローカル同期。出典は docs/git-github/merge-gates ほか
---

# merge-gate-check — 「merged」を状態で確認し、マージの副作用を閉じる手順

**使いどき**: PR をマージするとき / 「merged しました」の報告を受けたとき / 複数の PR を連続でマージするとき。

前提: 「merged」という**報告・表示・hook(イベント発火時に自動実行される処理)の出力は、どれもマージの証拠ではない**。証拠はサーバ側の状態だけである。

## Step 1 — merged は state で確認する

```bash
gh pr view <PR> --json state,mergedAt,mergeCommit
```

- `state: MERGED` + `mergedAt` 非 null を見て初めて merged と扱う。hook・CI 表示・相手の報告は偽「merged」の実測がある。
- 「実装完了」報告と「merged」は別物 — 完了 claim を受けても、この1発を打ってから ack する。

## Step 2 — 自動マージとポーラーを先に止める(方針を反転する前に)

```bash
gh pr view <PR> --json autoMergeRequest        # GitHub の自動マージが仕掛かっていないか
gh pr merge --disable-auto <PR>                # 反転するならまず解除
pgrep -fl <ポーラー名>                          # 自前のマージポーラーは gh では止まらない —
kill <PID>                                     # プロセスを特定して kill する
```

- マージ方針を撤回・反転するときは、**dispatch 済みの自動化(auto-merge・マージポーラー)を先に殺す**。実測: 閉じたはずの PR をポーラーが reopen+merge した。
- 自動化は「dispatch 時の決定」を実行し続ける — 人間の翻意は入力にならない。
- 恒久対策: ポーラー側に「マージ実行の直前に PR の state==OPEN を再確認するガード」を入れる(reopen+merge 事故の再発防止)。

## Step 3 — 残件は gate が読む面に置く

- マージ後に残る作業(follow-up)は、**マージ gate が実際に読む場所**(open issue)に置く(gate = マージ判断者や自動チェックが close の前に必ず見る一覧のこと。open issue はその一覧に出るが、PR 本文の checkbox・会話ログ・自分の記憶は誰の一覧にも出ない)。
- `Closes #N` を含む PR なら、先に [closing-keyword-check](../closing-keyword-check/skill.ja.md) を通す。

## Step 4 — 連続マージ後は合成を検査する

```bash
git fetch origin main
git log --oneline -<N> origin/main             # 今日 landing した PR 群
# 合成後の main で1回、フルの検査を回す(各 PR が緑でも合成は別物)
```

- 並行 PR は**各々緑のまま合成で壊れる**(実測: シグネチャ変更の衝突)。landing 直後の main で、PR 単位でなく**合成結果**に対して suite を1回回す。
- 特に: 同じ中心ファイルに触った PR 群を続けてマージした直後。

## Step 5 — マージ後にローカルを同期する

```bash
git fetch origin && git status                 # ローカルの遅れを見える化
git pull --ff-only                             # 作業前に追随
```

- サーバ側マージは**ローカルの checkout を動かさない**。stale なローカルでの grep は偽の指摘を生み、stale な base から切った branch は他人の変更の巻き戻しを静かに含む。マージしたら、次の作業の前に必ず同期する。

## 背景

各 Step の出典事故(hook の偽「merged」、ポーラーの reopen+merge、checkbox に置いた残件の消失、シグネチャ衝突、stale base ほか)は [マージは状態で確認する](../../docs/git-github/merge-gates.ja.md)・[ローカルは黙って古くなる](../../docs/git-github/stale-local.ja.md)・[並行エージェントと git](../../docs/git-github/worktree-parallel.ja.md) を参照。
