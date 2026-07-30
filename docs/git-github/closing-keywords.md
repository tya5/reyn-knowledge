---
name: closing-keywords
description: GitHub closing keywords fire on surface text — inside backticks nothing closes, and a raw keyword inside a negation closes. Neither "I wrote it" nor "I didn't" is an inspection; only the parser's output answers. The six ways to step on them, and the duty to enumerate before closing
tags: [git-github, github, automation]
sources:
  - feedback_closing_keyword_in_backticks_is_not_parsed
  - feedback_github_closing_keyword_matches_literal_ignores_context
  - feedback_enumerate_part_of_prs_before_authorizing_a_closing_keyword
---

# Closing Keywords Fire on Surface Text — Six Ways to Step on Them

## Background — what closing keywords are

On GitHub, writing `Closes #123` / `Fixes #123` / `Resolves #123` in a PR (pull request) body or a commit message makes **issue #123 close automatically the moment the PR merges**. It is a convenient mechanism, but **the parser does not read the meaning of the sentence**. It runs on exactly two surface rules:

1. **It does not read inside code spans (text wrapped in backticks `` ` ``).**
2. **Outside a code span, it fires on a literal match regardless of context.**

These two rules mass-produce outcomes that are the exact opposite of human intent. In agent development, PR volume is high and auto-closing issues **is the de facto substance of task management**, so a misfire translates directly into an operational accident: a task silently disappears, or silently lingers.

## Central claim

> **"I wrote `Closes #N`" is not evidence that anything will close, and "I didn't write it" is not evidence that nothing will. The only thing that answers is the parser's output — `gh pr view <PR> --json closingIssuesReferences`.**

This has the same shape as "rc=0 doesn't say what ran" in [Reading green](../verification/green-reading.md): **what you wrote and what happens can disagree in either direction**. And checking costs one command — **runnable even against an open PR before merge**.

## The six ways to step on it (all observed)

| # | Shape | Observed case |
|---|---|---|
| 1 | **Inside backticks → does not close** (false negative) | A PR containing `` `Closes #2972` `` merged, and the issue stayed open. The merge itself looks perfectly normal, so nobody notices |
| 2 | **Raw keyword in a negation or explanation → closes** (false positive) | The heading "**this does NOT close #2940**" **closed #2940**. The very sentence written precisely to prevent a close performed one, and an owner-reported bug sat wrongly closed for **two days** without any of its three watchdogs noticing |
| 3 | **Sentences that talk about the keyword are read as declarations** | A sentence **explaining the rule** — "writing `Closes` here would auto-close #2827" — contained a raw `close #2827`, which actually closed #2827 |
| 4 | **The commit-message path** | The default body of a squash merge is **the concatenation of the individual commit messages**. A `Closes #1909` left in an intermediate commit landed in main's commit message and closed the issue — **even though `closingIssuesReferences` was empty before the merge** |
| 5 | **Instructions carry the trap** | A PR written **exactly as a reviewer instructed** — "state 'closed by #3410' in the body" — nearly closed #3410. Not just the person writing the PR: **whoever hands over the wording** can be the source |
| 6 | **Accidental word order** | `Applies the real fix #3457's temporary carve-out...` — nobody was talking about closing anything, but **English word order happened to produce `fix #N`** (the possessive `'s` is invisible to the parser) |

Why shape 1 happens so easily: many repositories have a house style of "wrap identifiers in backticks," so `Closes #N` gets **formatted to look the same as the surrounding literals** — it arises not from carelessness but as a side effect of style. Moreover, if a brief (task instruction) exhibits the broken form as an example, **the implementing agent copies it faithfully** (measured: a brief written ten minutes before the trap was discovered reproduced the same accident four hours later — a trap learned mid-flight is not closed until you **sweep back through the briefs already in flight**).

## Discipline

**Writing:**

- To close: write `Closes #N` as **plain text with no backticks**.
- To not close: **write without using the keyword**. ❌ "does not close #N", "closed by #N" → ✅ "**#N stays open after this PR**", "**completed by the PR on the #N side**".
- Scan the full text **as word order** for `fix` / `close` / `resolve` **immediately preceding** `#N` (shape 6). Commit messages are in scope too (shape 4).
- When instructing someone else on PR wording, **read that wording itself through the parser's eyes** (shape 5).

**Verification (both directions):**

- After opening a PR: `gh pr view <PR> --json closingIssuesReferences` — check both that what should close **is included** and that what must not close **is not included**.
- After merging: look at the target issue's actual `state`. **The pre-merge check is necessary but not sufficient** (shape 4's commit-message path slips past body inspection).
- Don't read meaning into a close event's `commit_id` / `actor` fields — measured: PR-body keyword-driven and manual closes both show `commit_id=null` with the same actor, so **null distinguishes nothing** (inspect via `gh api repos/<owner>/<repo>/issues/<N>/timeline`). A **non-null commit_id, however, is the mark of the commit-message path**.

## "Will it close" and "should it close" are separate questions

Verifying the parser answers "**will** this PR close #N", not "**should** it". Observed accident: a brief demanded both "`Closes #3043`" and "don't touch the 30-second timeout" — **#3043's title was that timeout bug**. The PR closed the issue while its own body said "out of scope," nearly erasing the trail of a bug that had been reproduced three times. The implementer's own retrospective:

> I ran `closingIssuesReferences` twice. Both times it answered "did the keyword parse?" — the real question was "**should** this PR close this issue?" **I was running a check that made me feel verified.**

- **Before closing, read the issue's title and confirm it describes what this PR fixed** (this is a judgment of meaning and cannot be mechanized — knowing that it cannot is the discipline).
- Recovery after a wrong close: if the title has drifted from reality, "**correct the title to what was actually fixed, and move the remainder to a new issue**" can be more correct than reopening (people scanning closed issues later read only the titles).

## Enumerate before closing — don't close on "probably the last PR"

When multiple PRs advance a single (umbrella) issue, before writing `Closes #N` **enumerate the open PRs that reference that issue**:

```bash
gh pr list --state all --search "<N> in:body"
```

Write `Closes #N` only when zero are open; otherwise `part of #N`. Observed: closing without enumerating left **a still-open sibling PR behind while the arc sat closed-before-complete for 40 minutes**.

> **Why you must not close on "this is probably the last one": the fact of being closed itself hides the remaining work.** A closed issue drops out of review and out of open-list triage, so the leftovers move to "a place nobody looks."

Writing sub-PRs: refer to the umbrella as `part of #N` / `toward #N`. **Even a future-tense explanation like "the follow-up PR-3 will closes #N" must not contain the raw keyword** (it fires via shapes 2 and 3).

## The mechanization record — with the lessons from building the gate

This trap was ultimately plugged with a CI gate (an automatic check for **contradiction between the author's declared intent and the parser's output**). The construction process itself carries lessons:

- **The check runs in three directions**: an intent-to-close declaration where the parser doesn't close (shape 1) / an intent-not-to-close declaration where the parser closes (shapes 2 and 3) / **nothing declared, yet it closes** (only the two declaration-based checks would leave a hole).
- **Escape-hatch design**: a proposed whole-body switch — "this PR only discusses #N" — was **rejected by review using the gate-introducing PR itself as the counterexample** (that PR contained both a genuine declaration and discussion, and a whole-body switch would disable checking of the genuine declaration too). What was adopted is a **per-issue-number** marker. Keep escape hatches narrow in vocabulary.
- **A mechanism that needs a bypass on its very first real case has failed its witness**: to get past the gate, the first version put **eleven invisible zero-width spaces** into the PR body — an undiscoverable bypass is exactly the defect class the gate was supposed to kill. It was treated as a merge blocker.
- **The gate reads only the body** → shape 4 (commit messages) remains. A green gate does not read as "the close was prevented" — **the path the gate plugged and the path that fired can be different**.

## Checklist

- [ ] To close: plain-text `Closes #N`. To not close: a rephrasing that uses no keyword
- [ ] Inspected the body and commit messages, **as word order**, for `fix`/`close`/`resolve` adjacent to `#N`?
- [ ] After opening the PR: checked `closingIssuesReferences` in both directions?
- [ ] Before `Closes #N`: enumerated the open `part of #N` PRs? (the fact of a close hides leftovers)
- [ ] Does the issue's **title** describe what this PR fixed? ("should it close" is a separate question)
- [ ] After merge: looked at the target issue's actual state?
- [ ] Learned a trap mid-flight: swept back through the briefs already sent?

## Sources (measured during reyn development)

Shape 1: #2990/#3006. Shape 2: #2951→#2940 (two days wrongly closed), #2264/#2273 (umbrella #2259 closed twice). Shape 3: #3003→#2827. Shape 4: #3187→#1909 (the non-null commit_id as evidence). Shape 5: #3432→#3410. Shape 6: #3462→#3457. "Should it close": #3043→#3048. Enumeration duty: #3368 (40-minute premature close). The gate: #3015 (check_pr_closing_intent, including the ZWSP incident, the escape-hatch design, and the demonstration of non-vacuity — it caught its own introducing PR twice); operational details in #3186 (the marker is not a free-text field / editing the body does not re-run the gate).

## Related

- [Reading green](../verification/green-reading.md) — the same root: the divergence between "written" and "happened"
- [Incomplete work delays discovery](../verification/incomplete-work.md) — settling leftovers at arc completion
- [How vacuous gates are born](../verification/vacuous-gates.md) — proving non-vacuity when building a gate
