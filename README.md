# Machine-Checked Gates for an Autonomous Agent Loop

**Read this as a page:** https://loop-engineering-site.vercel.app

I run an autonomous Claude Code `/loop` agent — a background loop that plans and executes its own next task without me sitting in the chat. This is what it took to make that loop trustworthy enough to actually leave unattended.

Every failure class below was already forbidden in prose — in the prompt, in `CLAUDE.md`, in the docs. All four recurred anyway. The fix wasn't a better sentence. It was replacing the model's self-report — "I checked, it's fine" — with a machine-checked gate: code that runs, and can't be argued with.

## Failure taxonomy

These four recur across roughly 15 recorded corrections. Ordered worst-first.

1. **Parking (most frequent)** — stalling on reversible, already-authorized work while declaring "awaiting your judgment."
2. **Misdiagnosing** — blaming context length for API and rate-limit errors. The real cause was request-volume throttling, recorded across four separate memos.
3. **Idling** — sleeping long between cycles with nothing to actually wait for.
4. **Overreaching** — welding custom autonomy machinery onto a native command that was already mine, built-in, and didn't need it.

## The inversion

- **Before**: the model decided when to stop. Any plausible-sounding sentence could end a work session — including all four failure patterns above.
- **After**: a machine admissibility gate decides. Continuing requires passing the gate; only the gate can sanction a stop. No phrasing escapes it.

What the gate actually checks — and everything built around it — is the architecture below.

## The loop, as one turn

Every cycle runs the same five stages. The gate sits in the middle: nothing executes without passing it, and nothing stops without its sanction.

1. **Plan** — a fresh planner subagent writes `tasks.json` from the mission + live signals.
2. **Gate** — each task passes 13 mechanical checks, or it's gated / stops.
3. **Execute** — admitted tasks run as a parallel batch (claim, finish, abort).
4. **Verify** — done tasks' verifiers are re-run later; "done" decays, drift is flagged.
5. **Replan** — budget-bounded: a new planner is spawned and the turn repeats.

The only legitimate way the loop ends: the gate admits zero tasks — and even that stop is refused unless a `no_regret` attempt is on record.

## The gate, exactly

The real admission sequence, in order. A task is admitted only if it clears all thirteen; the first one it fails decides where it goes. None of it is a judgment call — each row is a line of code.

| # | The task is asked… | …or it goes to |
|---|---|---|
| 1 | Did a reviewer reject it with evidence? | gated |
| 2 | Are its fields the right type — `verify` a string, not a list? | gated |
| 3 | Does its `id` collide with a `done` record? | gated |
| 4 | Does `why_now` link to the mission (≥10 chars)? | gated |
| 5 | Is there a runnable `verify` (not "tbd"/"manual")? | gated |
| 6 | Is that verifier non-vacuous — not `true` or a bare `echo`? | gated |
| 7 | Is it self-doable without a human? | Tom's queue |
| 8 | Is it free of stop/danger keywords? | Tom's verdict |
| 9 | Is it real work, not polish untied to a blocking gap? | gated |
| 10 | Discovery task? Then it must cite a live signal (≥20 chars). | gated |
| 11 | Discovery cap: is it the only open exploratory task? | gated |
| 12 | If `branch_independent`, does it name what it's independent of? | gated |
| 13 | Is its lane free — not owned by another session? | reclassify |
| ✓ | all thirteen clear | **ADMIT** |

## Architecture: seven layers, one gate at the center

Each layer was added because a failure class above found the gap in the layer before it.

### 1. Planning
- Plan-of-record file, schema-versioned, written by a fresh planner subagent — not the tired executor.
- Truth-reviewer sidecar checks the planner's claimed evidence.

### 2. Admission (the gate)
- Mechanically checks each proposed task for mission linkage.
- Requires an attached runnable verifier.
- Requires self-doability.
- Enforces a discovery cap — max 1 open exploratory task.
- Requires a documented no-regret decomposition attempt.

### 3. Stopping
- Only the gate issues a stop sanction.
- Killswitch ladder — throttle, pause, stop on machine-measured triggers.
- Flee-detector — an LLM judge flagging the agent parking on doable work.

### 4. Verification
- Every done task's verifier is re-run later ("survival"); done decays.
- Vacuous verifiers rejected — a bare `true` or a bare `echo` doesn't count.
- Verifier-drift detection catches edit-the-check.
- Repeat-failure breaker stops loops on the same normalized failure signature.

### 5. Execution
- Parallel batch executor — claim, finish, abort — with atomic claim and digest re-check.

### 6. Observation
- Append-only decision log.
- Crash-replayable event journal.
- Metrics.
- Control-plane attestation — a blessed hash of the machinery, so drift in the machinery itself is detected.

### 7. Multi-session safety
- Per-session queues with file locking.
- Lane registry so two sessions don't fight over the same work.
- Lost-wake detector.

## Inside the planning layer

The gate can only reject bad tasks — it can't invent good ones. That falls to the planning layer, and it's where "the agent finds its own work" either stays honest or turns into busywork.

- **A fresh reader, not the tired executor.** The plan is written by a planner subagent spawned with a clean context — not the same run that just executed. The executor has a stake in its own last decision; a fresh reader doesn't. It reads four things: the mission's done-bar (what "finished" means), the decision log (what's already been tried), the current task queue (done, pending, or gated), and the live signals — error logs, the decision journal, metrics.
- **Two ways it recognizes work.** *Directed* — a sub-goal of the done-bar that isn't met yet. *Discovery* — an anomaly in the live signals: a failure that keeps recurring, a gap nothing measures. Discovery is the one most likely to become invented busywork, so it's fenced: a discovery task must cite an actual observed signal (a real log line or journal entry, not a hunch), and only one may be open at a time.
- **"Nothing to do" has to be earned.** Before the loop may declare it's out of work, it records a no-regret attempt: a decomposition it tried, or a concrete reason none exists. A "nothing left" claim without that record is rejected. Stopping is cheap to claim and expensive to check, so the claim carries the proof.
- **Every proposed task is a record, not a sentence.** Each task is a structured record: `title`, `why_now` (the mission link), `verify` (a command that actually runs), `self_doable`, `mutates` (the files it writes), `origin` (`directed` or `discovery`), and — for discovery — `signal`, the cited evidence. That structure is what lets the gate check it mechanically instead of trusting prose.

So: can an agent find its own next task? Yes — but only work it can point at in the logs, and only after it has tried and failed to decompose the alternative. Recognition is grounded in what's observed, not in what sounds plausible.

## The one verified metric

North star: **survived work per million tokens**. First measurement: **2.68**.

Not a benchmark, nothing to compare it to — I finally had a number instead of a feeling.

## What I threw away

My first diagnosis was wrong. I concluded the loop was dying of task starvation — running out of work to do — and built a backlog miner to fix it: ROI scoring, idempotency, the works. I hardened it.

Then the real root cause surfaced: planning lived outside the loop. The loop died the moment it consumed the one hand-made plan I'd fed it — not because it ran out of things to mine.

I cut the miner entirely. **It survives only as an unused standalone tool.**

## The closing irony

The gate exists to make every stop mechanically answerable. It broke that promise on itself.

Whenever a planner wrote the `verify` field as a list instead of a string, the gate threw an uncaught Python `AttributeError`. The runner kept only the first line of a merged stdout+stderr stream, so the decision log recorded **28 stops across 6 sessions** whose entire recorded reason was the string:

```
Traceback (most recent call last):
```

An unanswerable stop, emitted by the machine whose one job is answerable stops.

**Fail-closed**: it crashed instead of continuing. Nothing was corrupted, nothing was silently admitted. It was just mute.

**The obvious fix is worse**: coerce the field to a string before `.strip()` and `["a","b"]` becomes `"['a', 'b']"` — a string with real length, which clears both the length check and the vacuous-verifier check, admitting a task whose verifier can never run. A fail-closed crash becomes a silent fail-open admit.

Correct fix: a wrong-typed field gets gated, with a message naming the fix — not coerced, not admitted.

## Transferable rules

1. Every rule must name its enforcement layer at birth. A rule that lives only in prose is a tracked defect, not a rule.
2. Stop must be the default. Continuation has to be earned through a machine check.
3. A verifier that cannot fail verifies nothing. Reject vacuous verifiers explicitly.
4. Re-run old verifiers. "Done" decays.
5. Watch for edits to the check itself.
6. When hardening a gate, prefer failing closed to a coercion that silently passes.

---

Single HTML file. No build step, no dependencies, no external requests.
