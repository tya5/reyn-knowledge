---
name: closing-keyword-check
description: Full closing-keyword inspection before opening a PR, before merging, and after merging. Surface-text scan → both-direction check of the parser output → enumerating eligibility to close → the semantic judgment of whether to close → post-merge measurement. Sources in docs/git-github/closing-keywords
---

# closing-keyword-check — Keep PR Auto-Close from Misfiring

**When to use**: when opening a PR that mentions an issue number / when writing `Closes #N` in a dispatch brief / when merging.

## Step 1 — Surface-text scan (both the body and the commit messages)

```bash
# Target the PR body plus every commit message on this branch
{ gh pr view <PR> --json body -q .body; git log --format=%B origin/main..HEAD; } \
  | grep -inE '(close[sd]?|fix(e[sd])?|resolve[sd]?)([^#]{0,3})#[0-9]+'
```

Judge each hit:

- **Lines intended to close**: is it **outside** backticks (inside them, it won't close)?
- **Lines not intended to close**: negations ("does NOT close #N"), explanatory prose, and accidental possessives (`fix #N's ...`) **still fire**. Rewrite them to avoid the keyword ("#N stays open after this PR"). **Squash merges concatenate commit messages**, so fixing only the body is not enough.

## Step 2 — Confirm both directions with the parser's output

```bash
gh pr view <PR> --json closingIssuesReferences
```

- Are the issues that should close **in the list** (if not, Step 1 produced a false negative)?
- Are the issues that must not close **absent from the list** (if present, a false positive)?
- "I wrote it / I didn't write it" is not an inspection. **This output alone is the answer.**

## Step 3 — Enumerate eligibility to close (before writing `Closes #N`)

```bash
gh pr list --state all --search "<N> in:body"
```

- Write `Closes #N` only when there are **zero open PRs** carrying `part of #N`. If any remain, use `part of #N`.
- Reason: the fact of closing itself hides the existence of remaining work (closed issues vanish from inventory passes).

## Step 4 — The semantic judgment of whether to close (the one point that can't be mechanized)

- **Read the issue's title.** Does it describe what this PR fixed?
- Within the dispatch brief, do `Closes #N` and "don't touch X" contradict each other (if #N's title is X, they do)?

## Step 5 — Post-merge measurement

```bash
gh issue view <N> --json state   # for each issue mentioned
```

- Look in **both directions**: did what should close become closed, and did what must not close stay open?
- If there is an unexpected close: a **non-null** `commit_id` on the timeline means the commit-message path was the cause. Reopen, and record the cause (the commit SHA) on the issue.

## Background

For the source incidents behind each step (the wrongful close that stood for two days, firing from a rule-explanation sentence, propagation through a dispatch brief, and more), see [Closing keywords fire on surface text](../../docs/git-github/closing-keywords.md).
