---
name: strip-instrument-integrity
description: A strip is a measuring instrument, and if the instrument is broken no conclusion about the target's health can be drawn — non-unique anchors, duplicate declarations, measuring a different tree, and load; four failure modes and the self-check procedure
tags: [verification, testing, environment]
sources:
  - feedback_strip_anchor_must_be_unique_or_it_kills_the_wrong_call_site
  - feedback_verify_venv_identity_at_measurement_time_not_by_central_audit
---

# Integrity of the Strip Instrument — Four Ways the Measuring Tool Itself Breaks

[Strip-falsification](strip-falsification.md) is a measuring instrument for whether a gate is real. **If the instrument is broken, nothing you observe about the target yields a conclusion.** And the ways the instrument breaks follow fixed patterns, most of which err in the direction of "falsely accusing a healthy gate of being vacuous" (false positives). Errors in this direction are harder to discover than the false negatives that let vacuous gates slip through: the person told "your test has no effect" usually just goes ahead and "fixes" it.

## Failure mode 1 — the anchor is not unique (you killed the neighboring site)

When stripping mechanically, you usually delete the target code fragment (the anchor) via string replacement. If the anchor **occurs more than once in the file**, the wrong occurrence gets deleted.

```python
# session.py
def on_switch(self):
    self._running.clear()      # occurrence 1 (the switch handler)

def on_resolve(self):
    self._running.clear()      # occurrence 2 (<- this is the one we want to verify)
```

```python
src = src.replace("self._running.clear()", "pass", 1)   # deletes the first occurrence (occurrence 1)
```

You kill the occurrence that is not the verification target, the tests stay green → you report "the gate is vacuous" → in reality the gate was healthy and the strip simply missed. In the real case, a false accusation of this shape nearly got three healthy gates rewritten.

- **Count the anchor's occurrences before stripping.** If `count == 1` does not hold, target by line number or enclosing function name, or widen the anchor until it is unique.
- Checking that "something changed" (`assert src != before`) is **insufficient**. That proves "you changed something," not "you changed the site under verification."

## Failure mode 2 — the mechanism is declared in duplicate at multiple places

Even if the anchor is unique within the file, **another file may declare the same thing**.

Example: a UI component's height was declared in both `chrome.py` and `app.py`, and only the `app.py` side actually took effect. Stripping only the `chrome.py` side leaves the tests green → misread as "the gate has no effect." `count == 1` is a **within-file check**; it cannot see a second declaration in another file.

> **How to tell: if the result of the strip is not "a different kind of breakage" but "absolutely nothing changes," you killed the wrong target.** Before accusing a healthy gate, look for other places declaring the same thing.

Note that the shadowed side is, more often than not, "the side with the explanatory docstring." Editing it in the future does nothing, and the gate stays green either way, so this duplication is not caught by the gate.

## Failure mode 3 — you are measuring different code altogether (identity of the execution environment)

The most insidious form. In environments that use a shared Python venv with an editable install (`pip install -e .`), **the source tree the venv points to can be different from your own working tree**.

```
$ .venv/bin/python -c "import myapp; print(myapp.__file__)"
/repos/worktrees/agent-B/src/myapp/__init__.py    # <- not your tree!
```

Running tests in this state **measures someone else's tree, not your code**. The generating mechanism has also been identified: in an environment where multiple agents work in worktrees (parallel checkouts of the same repository), running `pip install -e .` inside a worktree while `VIRTUAL_ENV` still points at the parent's shared venv **re-points the parent venv's editable install to your own worktree**. In other words, it is not an accident — left alone, it is a mechanism guaranteed to recur.

Three points matter:

1. **GREEN reports are the more dangerous ones** (counter to intuition). Measuring a different tree means measuring "code you never changed," so the strip comes out green, producing the false accusation "the gate is vacuous." RED at least means something was measured (if it is RED even on the other tree). When the environment comes under suspicion, re-verify the GREEN reports first.
2. **Central auditing is impossible in principle.** Even if an auditor runs `python -c "import myapp"` in another session's directory, the python used is the one on the auditor's own PATH. Which python each session actually invokes is knowable only to that session. Therefore **the measurer must self-check at the moment of measurement** — there is no alternative:

   ```bash
   python -c "import myapp; assert myapp.__file__.startswith('$PWD/src'), myapp.__file__"
   ```

   Place this as the **zeroth** precondition check of the strip (before anchor uniqueness). The strip's conclusion depends entirely on "which code was measured."
3. **Read the resolution, not the declaration.** The contents of configuration files (`.pth` and the like) are "where it declares it points"; "where it actually resolves to" is measured with `myapp.__file__`. The gap between declaration and resolution is precisely the substance of this failure mode.

Prevention: in a worktree, **create an isolated venv first**, and never run `-e .` against the shared venv.

## Failure mode 4 — load and concurrency (when did you measure?)

In a concurrent-agent environment, there is a recurring error of **explaining a slow run or a timing flake by "the first thing visible in your own transcript."** In the real case, an agent published "my polling loop" as the cause of a failure — and only afterwards ran `ps`, which showed another session running the full test suite; the published cause was at best secondary.

- Before explaining a slow run / timing flake, run `ps` and `uptime`. "Probably interference" and "probably my fault" are equally evidence-free.
- **Report both green and red together with the load at the time of capture.** A green under load and a green in quiet conditions are different claims.
- "The first cause visible in your own transcript" is **the cheapest hypothesis, not the most probable one**. The essence of this error is failing to notice that you have only lined up the suspects you already knew about.

## When measurements disagree, settle it — don't average

If your strip result and someone else's strip result disagree, do not wave it through with "both have a point." Strip reports are the foundation of the entire verification discipline; if this wobbles, the credibility of every subsequent RED report drops. **Push until you identify why they differed** (the four failure modes above are the suspect list). Likewise, the party told "your strip missed" must not take it at face value — re-measure yourself. False accusations are stopped only by re-measurement.

## Checklist (before performing a strip)

- [ ] (0) Self-check the execution environment: `import myapp; assert myapp.__file__.startswith(your own tree)`
- [ ] (1) Is the anchor's occurrence count 1? If not, did you target by line number / function name?
- [ ] (2) After the strip, did the intended site actually change (inspect the diff)?
- [ ] (3) If the result is "nothing changes at all," did you look for other places declaring the same thing?
- [ ] (4) Did you attach the load / concurrency conditions at run time to the result (green/red)?

## Sources (measured during reyn development)

reyn is an AI agent runtime whose development is done almost entirely by a fleet of coding agents. Non-unique anchor: #3310 (killed the wrong one of two `clear()` occurrences, nearly falsely accusing three healthy gates of being vacuous). Duplicate declaration: #3341 (a height declared in two files; the inert side was stripped and stayed green). venv identity: #3363 (the shared venv pointed at another agent's worktree) and #3370 (identification of the re-pointing mechanism via `uv pip install -e .`, self-reported by the responsible agent). Load: #3389 (published own polling as the cause; it was actually another session's full suite).

## Related

- [The discipline of strip-falsification](strip-falsification.md) — the four axes that turn reports into evidence, on the premise that the instrument is healthy
- [Measuring the wrong target](measurement-target.md) — the broader class of "measured something, but not what the decision needed"
