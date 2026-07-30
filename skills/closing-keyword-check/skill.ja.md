---
name: closing-keyword-check
description: PR を開く前・マージする前・マージした後の closing keyword 総点検。字面スキャン → parser 出力の両方向確認 → 閉じる資格の列挙 → 閉じるべきかの意味判断 → マージ後の実測。出典は docs/git-github/closing-keywords
---

# closing-keyword-check — PR の issue 自動クローズを事故らせない手順

**使いどき**: issue 番号に言及する PR を開くとき / `Closes #N` を依頼文に書くとき / マージするとき。

## Step 1 — 字面スキャン(本文と commit メッセージの両方)

```bash
# PR 本文と、この branch の全 commit メッセージを対象にする
{ gh pr view <PR> --json body -q .body; git log --format=%B origin/main..HEAD; } \
  | grep -inE '(close[sd]?|fix(e[sd])?|resolve[sd]?)([^#]{0,3})#[0-9]+'
```

各ヒットを判定する:

- **閉じる意図の行**: バッククォートの**外**にあるか(中にあると閉じない)。
- **閉じる意図の無い行**: 否定文(「does NOT close #N」)・説明文・所有格の偶然(`fix #N's ...`)でも**発火する**。キーワードを使わない言い換えに書き直す(「#N はこの PR の後も open のまま」)。**squash マージは commit メッセージを連結する**ので、本文だけ直しても足りない。

## Step 2 — parser の出力で両方向を確認

```bash
gh pr view <PR> --json closingIssuesReferences
```

- 閉じるべき issue が**入っているか**(入っていなければ Step 1 の false negative)。
- 閉じてはいけない issue が**入っていないか**(入っていれば false positive)。
- 「書いた/書いてない」は検査ではない。**この出力だけが答え**である。

## Step 3 — 閉じる資格の列挙(`Closes #N` を書く前)

```bash
gh pr list --state all --search "<N> in:body"
```

- `part of #N` を持つ **open の PR がゼロ**のときだけ `Closes #N`。残っていれば `part of #N` にする。
- 理由: 閉じた事実そのものが残件の存在を隠す(closed issue は棚卸しから消える)。

## Step 4 — 閉じるべきかの意味判断(機械化できない一点)

- **issue のタイトルを読む**。それはこの PR が直したものを記述しているか?
- 依頼文の中で `Closes #N` と「X には触るな」が矛盾していないか(#N のタイトルが X なら矛盾している)。

## Step 5 — マージ後の実測

```bash
gh issue view <N> --json state   # 言及した各 issue について
```

- 閉じるべきものが closed になったか、閉じてはいけないものが open のままか、**両方向**を見る。
- 予期しないクローズがあれば、timeline の `commit_id` が**非 null** なら commit メッセージ経路が原因。reopen し、原因(commit SHA)を issue に記録する。

## 背景

各 Step の出典事故(2日間の誤クローズ、ルール説明文による発火、指示文経由の伝播ほか)は [closing keyword は字面で発火する](../../docs/git-github/closing-keywords.ja.md) を参照。
