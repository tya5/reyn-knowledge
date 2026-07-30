---
name: git-github-index
description: Tier 2 の地図 — エージェント駆動の git/GitHub 運用に固有の罠6領域。自動化(closing keyword・ポーラー・hook)は字面と状態で動き、並行性(worktree・並行 PR・stale local)は「分かれているようで分かれていない」面で壊れる
tags: [git-github, index, overview]
---

# git/GitHub 運用の罠 — Tier 2 の地図

[Tier 1(検証)](../verification/index.ja.md)が「エージェントの完了報告を疑う順番」なら、Tier 2(本ナレッジ集の第2層 = git/GitHub 運用)は**その周辺のインフラ** — git・GitHub・CI という共有機械が、エージェント運用特有の使い方(サーバ側マージの連打、worktree の並行作業、自動マージ、issue 駆動のタスク管理)でどう壊れるかの記録である。

通底する2つの構造:

1. **自動化は字面と状態で動く**。closing keyword(`Fixes #N` 等、PR マージ時に issue を自動クローズさせる記法)のパーサは文意を読まず、マージポーラーは dispatch 時の決定を実行し続け、hook はコマンド文字列に反応する。人間の意図は入力にならない。
2. **並行性は「分かれているようで分かれていない」面で壊れる**。worktree(1リポジトリで複数の作業ツリーを持つ機能)は stash を分けず、ローカル checkout はサーバ側マージに追随せず、並行 PR は各々緑のまま合成で壊れる。

## ドキュメント

| ドキュメント | 一言で |
|---|---|
| [closing keyword は字面で発火する](closing-keywords.ja.md) | バッククォート内は issue を閉じず、否定文の生キーワードは閉じてしまう。検証は GitHub 側の parser の出力で。閉じる前に兄弟 PR を列挙 |
| [ローカルは黙って古くなる](stale-local.ja.md) | サーバ側マージは checkout を動かさない。stale grep の偽指摘・stale base の静かな mass revert・行番号の時間ずれ |
| [並行エージェントと git](worktree-parallel.ja.md) | worktree が分けるもの・分けないもの。1つの中心ファイルに変更が集中する一斉分配・stash 禁止・コミット後の木の検証 |
| [マージは状態で確認する](merge-gates.ja.md) | hook の偽「merged」・ポーラーの reopen+merge・残件は gate が読む面に・並行 PR のシグネチャ衝突 |
| [issue は劣化する](issue-lifecycle.ja.md) | body はスナップショット、解決はスレッドと merged PR 側。クローズの検証記録・残件の決着・起票の較正 |
| [docs-only PR の罠](docs-prs.ja.md) | doc をパースするテスト・mermaid の描画・renderer 別 slug・リンクの列挙証明・内部履歴 ref 禁止 |

## 実行可能な checklist(skill = エージェントがそのまま実行できる手順書)

- [closing-keyword-check](../../skills/closing-keyword-check/skill.ja.md) — PR を開く/マージする前の closing keyword 総点検(本領域で最も事故が多い)

## 読む順

初見なら [closing keyword](closing-keywords.ja.md) → [マージは状態で確認する](merge-gates.ja.md) → [ローカルは黙って古くなる](stale-local.ja.md)。並行運用の設計者は [並行エージェントと git](worktree-parallel.ja.md) を先に。
