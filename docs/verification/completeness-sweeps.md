---
name: completeness-sweeps
description: Enumeration techniques that close a class, in practice — sibling-site sweeps, parallel paths and the single seam, coverage built from meaning, invariant tables, the registry axis vs the seam axis, how far grep reaches, the reference classes of a package move, and underreporting by the search tools themselves
tags: [verification, completeness, refactoring, tooling]
sources:
  - feedback_fix_class_completeness_sweep
  - feedback_fix_one_of_n_parallel_paths_sweep_all_siblings
  - feedback_completeness_for_gate_routing_fixes
  - feedback_completeness_sweep_semantic_not_single_mechanism_grep
  - feedback_invariant_coverage_exhaustive_enumeration
  - feedback_registry_enumeration_covers_the_tool_axis_not_the_seam_axis
  - feedback_clean_break_completeness_full_repo_grep_not_src_tests
  - feedback_package_move_completeness_three_ref_classes
  - feedback_git_grep_underreports_use_plain_grep_for_audit
  - feedback_lsp_cold_start_findreferences_silent_underreport
---

# Completeness Sweeps in Practice — Enumeration Techniques That Close a Class

[The discipline of enumeration](enumeration-discipline.md) covered how completeness claims break (truncation). This document is the practical side — a collection of techniques for **closing an entire class once you have fixed one defect in it**. If [Fix-class review](fix-class-review.md) supplies the reviewer's questions, this is the implementer's toolbox.

## 1. A resource-class fix immediately triggers a sibling-site sweep

When you fix one site of an unbounded-input / resource-exhaustion / DoS-class bug, **sweep the sibling sites of the same vulnerability class at the same time**. Real case: a fix that put a cap on an unbounded HTTP response body left behind siblings in the same "unbounded input" class (another HTTP façade, incoming WebSocket frames, subprocess stdout) — they were only discovered as a cluster in a dedicated sweep later.

- Include a **sibling-sweep results table** in the fix PR (grep for the same pattern → classify each site as fixed / not applicable / ticket filed).
- Two especially dangerous shapes: **a façade named "safe" that is unbounded** (the name promises safety), and **a flag that was designed but never wired up** (a `truncated` field exists yet is always False).
- Fix with a uniform mechanism plus a single-source limit value (do not duplicate the literal at each site).

## 2. Parallel paths — a duplicated decision creates debt that ages on one side only

Fix a bug in one of **N parallel paths** that share the same concern, and the rest silently stay stale. Real case: the fix "store the raw payload, not the wrapper around it" went into path A (dict-based) and shipped → users kept hitting the same bug on path B (string-based). The root cause of the divergence: **the decision ("which part is the raw payload") existed in duplicate on both paths**.

- **Parallel-path sweep**: after fixing path A, grep for handlers of the same kind (offloaders, dispatchers, serializers) and check each path.
- **Unify the decision, not the mechanism.** If two paths make the same decision, the decision belongs in one shared helper — a duplicated decision is divergence debt.
- A fix that adds a branch to gating/routing has two correct shapes: (a) **enumerate all parallel methods and add it to each** (plus a symmetry test), or (b) **consolidate the shared logic into a single seam** — per-site checking becomes unnecessary, so omissions are structurally impossible (completeness-by-construction). Prefer (b) when possible.
- But some divergence is **essential** (e.g. the outputs land in different places). Decide how far to unify by tracing where each output lands, and **share only the accidental divergence**. Over-unification becomes a different debt: mode-switching complexity.
- Run the disproof of a parallel-path fix **on the path that had been left behind** (the side you fixed is guaranteed green already).

## 3. Build coverage from meaning — not from a grep of one mechanism's callers

Real case of running the sweep "hide X on every surface that enumerates or displays it" as a grep for callers of `list_names()`: the CLI's list command alone enumerated via **a different mechanism (walking the directory directly)**, never appeared in the grep, and got left behind (it surfaced in real use as a behavioral contradiction — delegation says "not found" yet the listing shows it).

- Build coverage from the **semantic requirement** ("every surface that enumerates agents"). A grep of one API's callers is not a proof of coverage.
- Minimum two moves: grep the callers of the canonical seam, plus **a second search that assumes another mechanism exists** (or walk a list of surface kinds one by one).
- As a detector for what a static sweep missed, real use (live/dogfood) works — surfaces invisible to grep still show up as behavioral contradictions.

## 4. The proof of "no X after B" is a table

For invariants of the form "X does not happen after barrier B" ("no log appends after quiescence", "no events after shutdown"), the proof is not "B waits for the obvious set". It is a table: **enumerate every source that can produce X, and show that each source is gated on B**.

Real case: a quiescence primitive waited for "the obvious set of running work", but a coverage proof — grep over every task-spawn site — found **three appendable tasks that were not gated** (a timer, intervention dispatch, and untracked fire-and-forget tasks). All are races that fire only rarely, so review alone would never catch them.

- Procedure: (1) grep all task spawns and event sources in scope, (2) for each, "can it produce X?", (3) for producers, "is it gated on B?", (4) **make the table `source → produces X? → gated?` the deliverable** (so blanks are visible), (5) one test per type of X-producing source.
- **Untracked fire-and-forget tasks are the classic hole** (there is no handle to join). The fix is usually "track them in a joinable set".

## 5. The registry axis and the seam axis — decide which axis you are covering before you enumerate

Enumerating from a registry covers only **the axis that registry counts**. Defects often live **where values cross a boundary (a seam)**, and seams appear in no registry.

Real case: the gate for the defect "every tool definition escapes as a shallow copy, and an external library's in-place mutation permanently corrupts the canonical schema" **enumerated all three of its arms from the live registry** and claimed "tools registered in the future are covered from day one". **Complete on the tool axis, partial on the seam axis** — an unfixed leak site remained in another file, and the very tool whose breakage had prompted the issue was still being corrupted through it.

- **Enumerate seams from the value side**: grep the repo for the **shape of the projection expression** by which the canonical object escapes (`dict(x.parameters)`). Counting tool names will never find it.
- **If N arms all go through the same single call, that is not N seams — it is N samples of one seam.**
- For a fix-class where the same expression shape occurs at two or more sites, do not fix the sites individually — **make the projection single** (one helper owns the responsibility, and every site calls it). But **an assert on the helper's contract is not a substitute for a per-seam witness** (see "correctness and reach are separate asserts" in [Wiring tests vs mechanism tests](wiring-vs-mechanism.md)).
- Second shape: **"I converted one site to derive the value" ≠ "the value is hardcoded nowhere"**. After deriving, grep across tests/ **from the literal side of the value** (the concrete value such as `"category"`, not the concept name). In the real case, the moment one cell was registered, three **other** tests that had hardcoded that cell as an "unregistered example" went RED.
- Corollary: take a test's **negative examples from outside the space the system grows into** ("things that can never enter the set", not "things not yet in the set"). If you must use a negative example with an expiry, state the expiry.

## 6. How far grep reaches — src/tests alone is not a clean break

The completeness gate for fully removing a dependency or feature (clean break) is not "grep over `src/ tests/` = 0" but **whole-repo grep minus a historical allowlist** (ADRs — architecture decision records — and decision documents that legitimately record that the thing existed). Real case: after the src/tests-scoped grep passed, three leftovers were found: **the lockfile** (removing the extra from pyproject without re-locking leaves the dependency reinstallable), **sample configs** (`*.example`), and **a dead rule in `.gitignore`**.

- Include in the sweep: lockfiles (re-lock + check in the same PR), sample configs, ignore files, CI config, packaging globs, docs.
- **A rename is a clean break of the old name.** The sweep must include **LLM-visible text** (other tools' descriptions, prompts, menus) — an old tool name inside a description is not prose; it is **a live call site that makes the model invoke a tool that no longer exists**. The same goes for normative files (the guidance agents read before working).

## 7. Package moves — references come in three classes

When moving a package (`X/` → `group/X/`), there are three classes of references to sweep, and a grep for imports catches only the first:

1. **Dotted imports** (multi-line parenthesized blocks escape a single-line grep)
2. **Dotted string literals** (packaging globs, literals in AST gates, dynamic import strings)
3. **File-path reads composed from segments** (shapes like `_SRC / "web"` — invisible to both greps)

Furthermore: the scope is **every import origin across src + tests + scripts + auxiliary directories**. Include **absolute self-imports inside the moved package**. If the package name **collides with a runtime data-directory name**, do **not** rewrite the runtime-side literals (that creates the inverse defect). For moves that deepen nesting, grep for **parent-directory walks from `__file__`** — if tests inject the path, CI **hides** the breakage, so demand a test that exercises the default path without injection. The durable form is path resolution **anchored at the package root**, not at the module itself.

Symbol renames add four points of their own: **pre-grep the new name for collisions** (silently rebinding over an existing symbol is a quiet mis-bind that CI cannot detect), **replace only via code-context patterns** (bare word replacement corrupts prose), **record the sibling-symbol counts before and after** (proof the change was surgical), and **treat the full suite as the final net** (a bare symbol inside a code string destined for a subprocess matches no pattern).

## 8. The search tools themselves underreport

Completeness claims ride on the health of your search tools. Two measured cases:

- **`git grep` can silently underreport in tests/** (measured: it returned only 2 of 9 files). For deletion / zero-reference claims, treat **plain `grep -rn`** as authoritative. When they disagree, plain grep wins.
- **LSP findReferences has two silent failure modes**: (a) the first call right after a cold start returns only the definition (correct from the second call on); (b) **after out-of-process file changes — git checkout, edits by external tools — the index stays stale and returns a false zero against the correct current line** — and the zero is stable, so retrying cannot expose it. Any safety-relevant "remaining callers = 0" claim must be cross-checked against plain grep, and **record the tree state (commit) at which you measured**.

## Checklist

- [ ] Resource-class fix: did the PR include the sibling-sweep results table (fixed / not applicable / filed)?
- [ ] Parallel paths: did you consolidate the decision into a single helper? Was the disproof run on the left-behind path?
- [ ] Was coverage built from meaning? Did you search for a second mechanism?
- [ ] Is the invariant a `source → produces? → gated?` table? Did you track the fire-and-forget tasks?
- [ ] Did you declare the enumeration axis (registry axis / seam axis)? Did you grep seams from the value's shape?
- [ ] Is the clean break whole-repo minus the historical allowlist? Did you include lockfiles, sample configs, and LLM-facing text?
- [ ] Move/rename: three reference classes × every import origin? Did you offset search-tool underreporting with plain grep?

## Sources (measured during reyn development)

Sibling sweeps: #1925 (the cluster of missed HTTP caps). Parallel paths: #2394/#2397 (the two offloaders), #1387/#1389 (missed second read gate → consolidation into a shared helper). Coverage from meaning: #1954 (the separate iterdir mechanism). Invariant tables: #1533 (three ungated appending tasks). Seam axis: #3383 (the shallow-copy leak site), #3376 (three literals found after derivation). Clean break: #2850 (lockfile/example/gitignore), #2848 (old tool names in descriptions). Package moves: #1700/#1706/#1707/#1711/#1714/#1719/#1724. Tool underreporting: #1953 (git grep 2/9), the two LSP modes (2026-07-15, including the retraction-then-reconfirmation history).

## Related

- [The discipline of enumeration](enumeration-discipline.md) — the mechanism side: how completeness breaks (truncation)
- [Fix-class review](fix-class-review.md) — the reviewer's side of the same discipline
- [Verifying removals](removal-verification.md) — completeness in the special case of deletion
- [Wiring tests vs mechanism tests](wiring-vs-mechanism.md) — helper contract ≠ per-seam witness
