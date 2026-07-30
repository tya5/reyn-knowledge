---
name: docs-prs
description: docs-only PR にもビルド面がある — doc をパースするテスト、mermaid の描画、renderer ごとに違う anchor、リンク完全性の列挙証明、利用者向け文書への内部履歴 ref 禁止
tags: [git-github, docs, ci]
sources:
  - feedback_docs_only_pr_can_break_impl_doc_mirror_test
  - feedback_mermaid_render_check_on_doc_pr
  - feedback_docs_restructure_followup_completeness_gate
  - feedback_no_history_refs_in_user_docs
---

# docs-only PR の罠 — 文書もビルド面を持つ

「`.md` しか触っていないから安全」は誤った近道である。ドキュメントには、コードと同じように**壊れうるビルド面**が4つある: それをパースするテスト、それを描画するレンダラ、それを指すリンク、それを読む対象読者。

## 1. docs-only でも pytest は割れうる

ドキュメントの内容を**パースして assert するテスト**(実装↔文書のミラーテスト)を持つリポジトリでは、`.md` の編集がそのままテストの入力の変更になる。実測: 機能一覧の doc から行を削る PR で、`parse_feature_map()` を呼んで件数と構造を assert するテストが存在した(通ったのは、**仮定せず実行したから**確認できた事実である)。

- docs-only PR を push する前に: `grep -rln <docファイル名> tests/` で**その doc を読むテスト**を探し、ヒットしたら回す。
- 逆向きも成立する: コード側の定数と同期が要求されている doc は、doc 編集がコード側の同期ガードを割る。

## 2. mermaid は「描画」を検証する — diff がきれいでも壊れる

mermaid(Markdown 内に書ける図の記法)の node テキストに含まれる `()` `[]` `{}` は **node 形状の構文**として解釈される。実測: mindmap の node に書いた `checkout(seq)` の括弧が構文と衝突し、**図全体が silent に描画エラー**になった(diff 上はただのテキスト行に見える。発見したのはオーナー)。

- mermaid ブロックを含む PR は、node テキストの括弧・特殊文字を grep し、可能なら実際に**描画して**確認する(GitHub のプレビュー / mermaid.live)。「diff がきれい」は「描画が正しい」を意味しない。

## 3. anchor の slug は renderer ごとに違う

同じ見出しから GitHub と mkdocs(静的サイトジェネレータ)が**別の anchor 文字列**を作ることがある。実測: em-dash(—)を含む見出しで、GitHub は空白2つ分をハイフン2つに、mkdocs は連続ハイフンを1つに畳んだ — **どちらの slug に合わせても、もう片方が壊れる**。

- 恒久策は検算ではなく**発生源の除去**: **見出しに em-dash や連続空白を生む記号を書かない**(それだけで両者が一致する)。
- そして「docs のビルドが緑」は「**どの renderer についての緑か**」を語らない([緑を読む技術](../verification/green-reading.ja.md)の doc 版) — 実測では strict ビルドが緑だったのは、mkdocs が「一致する側」だったからにすぎない。

## 4. リンク完全性は「列挙」で証明する — ビルドの緑では足りない

リンク切れ修正・目次再編の PR の完全性証明は、**手動の全数列挙**である。理由は2つ: (a) 厳格モードのビルド(`mkdocs build --strict`)が **PR の CI に入っていない**ことがある、(b) 入っていても**検査対象外のディレクトリ**があり、「strict で警告ゼロ」はそこについて何も言わない。

- リンク修正 PR: 古い形式の参照を**種類ごとに全列挙**(`grep -rhoE '<旧パターン>' docs/ | sort | uniq -c`)し、各参照先の新しい場所を1つずつ確認。**全種類が覆われて初めて完全**。実測: 3種類中2種類だけ直した PR を、この列挙が捕まえた(残余の完全な集合を添えて1回で差し戻し — 小出しの往復にしない)。
- 目次 PR: 削除 N / 追加 N / 脱落 0 のページ数照合を手動で。
- これは[完全性掃討の実務](../verification/completeness-sweeps.ja.md)の doc 版である: パターン単位の修正は同類を取りこぼす。

## 5. 利用者向けの文書に、内部履歴の参照を書かない

運用者・利用者向けの reference / guide の本文に、開発内部の履歴参照(提案番号・PR 番号・設計記録番号・issue 番号・内部シンボル名)を**インラインで書かない**。利用者にとってそれはノイズであり(オーナー評: 「ユーザからすればノイズでしょ」)、履歴の追跡は PR body / commit message の側の仕事である。

- 文書本文は**現在の挙動を利用者の語彙で**記述する。「ADR-0031 を廃止」ではなく「もう読み込まれず、警告を表示する」。
- 置き場所の表: 利用者向け doc 本文 ❌ / 末尾の "See also" リンク1本 △ / PR body・commit message ✅ / コードコメント ✅。
- スコープの較正: reference・利用者ガイド = 除去対象 / 概念解説 = 参照ごとに判断(設計理由の履歴は残しうる) / 開発者向け深掘り = 対象外 / **外部ベンダーの参照(利用者が実際に調べられる upstream issue 等)= 残す**。
- レビューゲート: doc PR の追加行を `grep -iE "FP-[0-9]|PR-[A-Z0-9]|ADR-[0-9]|#[0-9]{3}"` で検査。

## チェックリスト

- [ ] その doc をパースするテストを探して回したか
- [ ] mermaid の node テキストに構文衝突する文字は無いか。描画を確認したか
- [ ] 見出しに em-dash・連続空白を生む記号を使っていないか
- [ ] リンク・目次の完全性を、ビルドの緑ではなく全数列挙で証明したか
- [ ] 利用者向け本文に内部履歴 ref を書いていないか(スコープ表で判断したか)

## 出典(reyn 開発での実測)

doc パーステスト: Control IR アーク PR-6(feature-map の行削除 × `test_fp0036_coverage.py`)。mermaid: #1566→#1568(`checkout(seq)` の括弧、オーナー発見)。slug 差: #3039(em-dash、colon 置換で解消。mkdocs が primary surface だが public repo の GitHub view も実在の secondary、という影響較正込み)。リンク列挙: #1256(15/15/0 の手動照合)・#1257(3種中2種の修正を列挙が捕捉)。履歴 ref: 2026-05-30 オーナー指示・#2046(内部 33 件 strip / vendor 2 件 keep / concepts 除外)。

## 関連

- [完全性掃討の実務](../verification/completeness-sweeps.ja.md) — 列挙による完全性証明の一般形
- [緑を読む技術](../verification/green-reading.ja.md) — 「緑はどの面についての緑か」
- [監査は内容を照合する](../verification/audit-content-match.ja.md) — doc 転記前の実装裏取り
