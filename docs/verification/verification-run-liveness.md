---
name: verification-run-liveness
description: '"Slow" and "stuck" are different states — treat a long run with zero bytes of output and near-zero CPU as a hang hypothesis. Do not report progress on a run you have not read'
tags: [verification, testing, async]
sources:
  - feedback_track_background_verification_hang_not_slow
---

# Liveness of the Verification Run Itself — 'Slow' vs 'Stuck'

## The shape of the problem

Kick off the test suite in the background and work on something else — an everyday scene in agent development. That is exactly where this trap lives.

Real case: a full suite was launched in the background and, for **44 minutes with a 0-byte output file and near-zero CPU time**, the agent kept reporting "waiting on the suite — a few more minutes." When the owner asked "can that shell actually see its progress?" and someone checked, it turned out the run **was not slow — it was hung** (stopped permanently).

> **Elapsed time far beyond the known-normal duration + near-zero CPU = a "blocked" hypothesis, not "slow."**

(This suite's normal time is about 4.5 minutes. 44 minutes with 55 seconds of CPU is not the shape of "10x slower" — it is the shape of "executing nothing.")

**Do not report progress on a run you have not read.** You may say "almost done" only after confirming that the output is growing or that CPU is being consumed. This is the time-domain version of [Measuring the wrong target](measurement-target.md) — the act of "waiting" masquerades as evidence that "verification is making progress."

## Safety net — make hangs dump their stacks

The worst property of a hang is that it **silently eats wall-clock time**. For Python/pytest, keep a faulthandler timeout as standard equipment:

```bash
pytest -o faulthandler_timeout=45   # a test stuck for 45 seconds dumps all thread stacks and fails
```

This turns a hang from a "silent time thief" into a **failure that self-reports which test stopped and in which frame**. To hunt the culprit, bisect with `-x` (stop at the first failure); the innermost frame of the dump is the cause.

## The cause in the real case — the "immortal async worker" anti-pattern

The cause of this hang is instructive, so here it is verbatim. A worker task that continuously drains writes in the background was written like this:

```python
async def drainer(queue):
    while True:
        item = await queue.get()
        try:
            await write(item)
        except BaseException:   # ← this is the trap
            continue
```

In Python's `asyncio`, a request to stop a task (cancellation) is delivered into the task **as an exception** — `CancelledError`. `except BaseException` (or a bare `except:`) **swallows it**. If cancellation arrives in the middle of a write, it gets suppressed, the loop continues, and the task becomes **immortal**. Then the shutdown sequence of `asyncio.run` (cancel all tasks and wait for them) **waits forever** — that is what the 44 minutes of 0 bytes and 0 CPU actually were.

Prescription:

- **Catch only `Exception`** (route genuine failures to the caller).
- **Re-raise `CancelledError`** (after first settling any in-flight, unfinished response). The task then terminates as the stop request intended.
- Write a regression test for shutdown: run the shutdown sequence in a daemon thread (a separate thread that does not block process exit), and after `join(timeout=...)`, **assert the thread is no longer alive**. If this regresses, CI goes **RED immediately** instead of hanging.

## How to apply

- [ ] Do you have a baseline — "normal takes about this long" — for background verification runs?
- [ ] Before reporting progress, did you actually look at output growth and CPU usage?
- [ ] Did you arm long-running suites with a faulthandler timeout (or an equivalent self-reporting mechanism)?
- [ ] Did you audit the cancellation path of any newly added resident worker or drainer task (no `except BaseException` / bare `except:`)?

## Sources (measured during reyn development)

#2259 PR-2a (2026-06-28): 44 minutes, 0 bytes, 55 seconds of CPU kept being reported as "slow" until the owner's question exposed it. The cause was the drainer worker's `except BaseException` swallowing `CancelledError`, leading to shutdown waiting forever. The faulthandler dump was the primary data that pinpointed the offending line.

## Related

- [Measuring the wrong target](measurement-target.md) — the inverse of "a coarse predicate fires on every member of the set": reading a target that emits no signal as "in progress"
- [Integrity of the strip instrument](strip-instrument-integrity.md) — the adjacent axis of the same "observing the execution environment" concern: load and concurrency
