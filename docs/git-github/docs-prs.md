---
name: docs-prs
description: Docs-only PRs have build surfaces too — tests that parse the doc, mermaid rendering, anchors that differ per renderer, proving link completeness by enumeration, and the ban on internal-history refs in user-facing docs
tags: [git-github, docs, ci]
sources:
  - feedback_docs_only_pr_can_break_impl_doc_mirror_test
  - feedback_mermaid_render_check_on_doc_pr
  - feedback_docs_restructure_followup_completeness_gate
  - feedback_no_history_refs_in_user_docs
---

# The Traps in Docs-Only PRs — Documents Have Build Surfaces Too

"It only touches `.md`, so it's safe" is a false shortcut. Documentation has four **build surfaces that can break**, just like code: the tests that parse it, the renderer that draws it, the links that point at it, and the audience that reads it.

## 1. pytest can break even on a docs-only PR

In a repository with **tests that parse documentation content and assert on it** (implementation↔doc mirror tests), editing an `.md` file is directly a change to a test's input. Measured: on a PR that removed lines from a feature-list doc, there existed a test calling `parse_feature_map()` and asserting on the count and structure (it passed — a fact known only because the tests were **run rather than assumed**).

- Before pushing a docs-only PR: look for **tests that read that doc** with `grep -rln <doc-filename> tests/`, and run any hits.
- The reverse direction holds too: a doc that is required to stay in sync with constants on the code side means a doc edit can break the code side's sync guard.

## 2. mermaid is verified by rendering — a clean diff can still be broken

In mermaid (a diagram notation that can be written inside Markdown), the characters `()` `[]` `{}` inside node text are interpreted as **node-shape syntax**. Measured: the parentheses in `checkout(seq)` written into a mindmap node collided with the syntax, and the **entire diagram silently failed to render** (in the diff it looks like an ordinary text line; the owner found it).

- For any PR containing mermaid blocks, grep the node text for brackets and special characters, and where possible **actually render it** to confirm (GitHub preview / mermaid.live). "The diff is clean" does not mean "the render is correct."

## 3. Anchor slugs differ per renderer

From the same heading, GitHub and mkdocs (a static site generator) can produce **different anchor strings**. Measured: on a heading containing an em-dash (—), GitHub turned the two spaces' worth into two hyphens while mkdocs collapsed consecutive hyphens into one — **matching either slug breaks the other**.

- The permanent fix is not double-checking but **removing the source**: **don't write em-dashes, or any characters that produce consecutive spaces, in headings** (that alone makes the two agree).
- And "the docs build is green" does not say **green with respect to which renderer** (the docs edition of [Reading green](../verification/green-reading.md)) — in the measured case the strict build was green only because mkdocs happened to be the side that matched.

## 4. Prove link completeness by enumeration — a green build is not enough

The completeness proof for a link-fix or table-of-contents restructuring PR is **manual exhaustive enumeration**. Two reasons: (a) the strict-mode build (`mkdocs build --strict`) is sometimes **not part of the PR's CI**; (b) even when it is, there can be **directories outside its scope**, and "zero warnings under strict" says nothing about those.

- Link-fix PRs: **enumerate every old-style reference by kind** (`grep -rhoE '<old-pattern>' docs/ | sort | uniq -c`) and confirm the new location of each target one by one. **Only when every kind is covered is it complete.** Measured: this enumeration caught a PR that fixed only 2 of 3 kinds (it was sent back in a single round with the complete set of the remainder attached — not in piecemeal back-and-forth).
- ToC PRs: reconcile the page counts manually — N removed / N added / 0 dropped.
- This is the docs edition of [Completeness sweeps in practice](../verification/completeness-sweeps.md): per-pattern fixes miss their siblings.

## 5. Don't write internal-history references into user-facing documents

In the body of a reference or guide aimed at operators and users, never write development-internal history references (proposal numbers, PR numbers, design-record numbers, issue numbers, internal symbol names) **inline**. To a user that is noise (owner's verdict: "from the user's point of view, it's just noise"), and tracing history is the job of the PR body and commit message side.

- Document bodies describe **current behavior in the user's vocabulary**. Not "ADR-0031 retired" but "it is no longer loaded and a warning is shown."
- Placement table: user-facing doc body ❌ / a single "See also" link at the end △ / PR body & commit message ✅ / code comments ✅.
- Scope calibration: references & user guides = strip / concept explainers = judge per reference (design-rationale history may stay) / developer-facing deep dives = out of scope / **external vendor references (upstream issues etc. that a user can actually look up) = keep**.
- Review gate: inspect a doc PR's added lines with `grep -iE "FP-[0-9]|PR-[A-Z0-9]|ADR-[0-9]|#[0-9]{3}"`.

## Checklist

- [ ] Did you search for, and run, the tests that parse the doc?
- [ ] Is there any character in mermaid node text that collides with the syntax? Did you confirm the render?
- [ ] Are the headings free of em-dashes and characters that produce consecutive spaces?
- [ ] Did you prove link/ToC completeness by exhaustive enumeration, not by a green build?
- [ ] Are the user-facing bodies free of internal-history refs (judged by the scope table)?

## Sources (measured during reyn development)

Doc-parsing test: Control IR arc PR-6 (feature-map line removal × `test_fp0036_coverage.py`). mermaid: #1566→#1568 (the parentheses in `checkout(seq)`, found by the owner). Slug divergence: #3039 (em-dash, resolved by replacing it with a colon; including the impact calibration that mkdocs is the primary surface but the GitHub view of a public repo is a real secondary one). Link enumeration: #1256 (manual 15/15/0 reconciliation), #1257 (enumeration caught a fix covering 2 of 3 kinds). History refs: owner directive 2026-05-30, #2046 (33 internal refs stripped / 2 vendor refs kept / concepts excluded).

## Related

- [Completeness sweeps in practice](../verification/completeness-sweeps.md) — the general form of proving completeness by enumeration
- [Reading green](../verification/green-reading.md) — "green with respect to which surface?"
- [Audits must match content](../verification/audit-content-match.md) — verifying against the implementation before transcribing into docs
