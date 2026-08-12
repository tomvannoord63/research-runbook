# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

<!-- ============================================================================
     AUTORESEARCH TEMPLATE. Fill the [brackets], delete the HTML comments.

     Ordering principle: an agent reads top-down and acts on the first thing that
     looks actionable. Put orientation and constraints ABOVE detail. If a section
     does not change what an agent DOES, cut it.

     Sections marked [$] are tier 3 (iterations cost money). Delete at tier 1-2.
     ============================================================================ -->

## Development Environment

- **Python**: [version, how managed]
- **Package manager**: [uv / pip / conda]
- **Local machine**: [what runs here — and what explicitly does not]
- **Training**: [hardware, memory ceiling, rented or owned]

<!-- Say what the local machine CANNOT do. An agent that doesn't know the model
     won't fit will happily spend twenty minutes discovering it. -->

**Undecided — ask rather than invent:** [decisions not yet made that an agent would
otherwise silently guess at]

<!-- This line prevents a whole class of confident wrong answers. Keep it current;
     move items out as they settle. -->

## Project Status

[One paragraph. What exists, what is planned. Be specific about the boundary —
"pre-implementation, the spec lives in context/" or "training pipeline works,
evaluation is stubbed."]

Specs — read before writing code:
- `[path]` — [what it settles. Mark the source of truth explicitly.]

## Project Overview

[2-3 sentences: what the project does and what is actually novel about it. If the
novelty is not the obvious part, say which part it is not.]

### Questions, ranked — do not treat as co-equal

1. **Primary.** [The one guaranteed to produce a result, whatever the answer.]
2. **Secondary.** [Higher value, higher variance.]

[$] The secondary can die at [the gate that kills it]. If it does, [what the project
becomes] — and that is a success, not a failure. **Never let secondary-question work
block or reorder primary-question work.**

### Phases

| Phase | Content | Duration |
|---|---|---|
| 0 | [feasibility — no training] | |

### Minimum publishable unit

[The smallest complete artifact worth shipping, and roughly when it lands. Everything
after is upside.]

## Repository Structure

```
[tree — mark what is unbuilt]
```

### Key principle: separate what changes from what persists

| File | Git tracked? | Purpose |
|---|---|---|
| Code, configs | Yes | Reproducible experiments |
| `hypothesis_log.md` | **No** | Survives `git restore`, accumulates findings |
| `CLAUDE.md` | Yes | Agent instructions, updated as the project evolves |
| Logs, checkpoints | No | Ephemeral / too large |

## Commands

`make help` lists every target. `EXP=` names the run and determines the log path,
`$(LOGDIR)/[project]-$(EXP).log`.

```bash
make brief                     # Current Best / Ruled Out / Open Questions
make smoke EXP=<name>          # cheap sanity run — always before exp
make exp EXP=<name> WANDB=1    # full run
make monitor-cmd EXP=<name>    # reprint log path + Monitor command
make gates-exp EXP=<name>      # every gate + verdict; non-zero exit if any FAIL
make status-exp / result-exp / diag-exp EXP=<name>
make ps / kill-exp / logs
```

<!-- State plainly which targets work TODAY. A stale command list is worse than none:
     an agent trusts it over the filesystem. -->

### Agent workflow for launching runs

1. **`make brief`** — loop step 1. Do this before proposing an experiment, especially
   in a fresh session.
2. Launch with the **Bash tool, `run_in_background: true`** — this creates a task
   tracked in the Claude Code UI. Do **not** use shell `&`; a backgrounded shell job
   is invisible to Claude Code.
3. Set a persistent **Monitor** on the log. Every run target prints the exact command
   at launch; `make monitor-cmd EXP=<name>` reprints it:

   ```bash
   tail -f /tmp/[project]-<name>.log | grep -E --line-buffered "FINAL_RESULT|GATE:|Error|Traceback"
   ```

   **`--line-buffered` is not optional.** Without it grep block-buffers inside the pipe
   and the Monitor stays silent until the buffer flushes — monitoring that looks
   configured and never fires.
4. The Monitor should fire at each checkpoint and on any crash. **Never block on
   `sleep`** waiting for a run.
5. [$] The meter is running — confirm the first checkpoint looks sane rather than
   discovering a crash an hour later.
6. When the run ends: `make gates-exp`, append to `hypothesis_log.md`, **then** discuss
   the next experiment with the user.

## Architecture

<!-- The big picture that requires reading several files to reconstruct. Explain WHY
     things are structured as they are — an agent can read the code but cannot recover
     the reasoning. Where a choice was forced rather than preferred, say so; it stops
     an agent from "improving" it. -->

## Critical Metrics to Track

Every run logs machine-parseable lines. **A gate that isn't printed with its threshold
and verdict isn't a gate.**

```python
print(f"FINAL_RESULT: {metric}={value:.6f}")
print(f"GATE: {gate_id} value={v:.4f} threshold={t:.4f} verdict={'PASS' if ok else 'FAIL'}")
```

### [DOMAIN] diagnostics — `[DOMAIN]_DIAG:` prefix

<!-- The mechanism your headline metric cannot observe. See CONVENTIONS.md. -->

| Metric | Reads as |
|---|---|
| `[name]` | [interpretation, healthy range] |

<!-- CRITICAL: document readings that LOOK like failures but aren't, and the quantity
     that distinguishes them. This is where diagnostics most often get misread. -->

```python
print(f"[DOMAIN]_DIAG: [key]={v:.4f} ...")
```

**When to measure:** [before training / every N steps / final]

### [Benchmark] — `[BENCH]:` prefix

Always emit the control alongside the score. **A score without its control is
meaningless.**

```python
print(f"[BENCH]: score={s:.4f} control={c:.4f} ...")
```

## Autonomous Research Loop

1. Read `hypothesis_log.md` (`make brief`) — know what has been tried and ruled out.
2. Formulate **one** hypothesis with a specific predicted direction.
3. Make the minimal change. Prefer config over code, code over architecture.
4. `make smoke` — verify no crash *and* that diagnostics are non-degenerate.
5. Launch the full run in background; set a Monitor on the log.
6. While waiting: plan the next experiment or analyze prior results. **Never block on
   `sleep`** — it burns context for nothing.
7. Collect results. `make gates-exp`.
8. Append to `hypothesis_log.md`.
9. **KEEP → commit. DISCARD → `git restore`.** This is precisely why the log is
   gitignored: it survives the reset that throws the code away.
10. Repeat.

### Log entry format

```markdown
### Experiment N — YYYY-MM-DD
**Phase**:
**Hypothesis**: [specific predicted direction]
**Change**: [exact config/code change]
**Result**: [the machine-parseable lines]
**Gates**: [gate_id: PASS|FAIL for every gate this run touches]
**Verdict**: KEEP / DISCARD
**Interpretation**: [why did this happen?]
**Next**: [what this implies for the next run]
```

`hypothesis_log.md` also carries **Current Best**, **Ruled Out**, and **Open Questions**.
Keep them current — Ruled Out is what stops a later session re-running a dead end, and it
only works if failures get written down as carefully as successes. Be specific, and record
the *interpretation*, which outlasts the number.

### Practices that earn their place here

- **Baseline before treatment, always.** A treatment number without its control is not
  a result.
- **One variable per experiment.** Two knobs moved together cannot be attributed.
- **Proxy for iteration, full eval for validation.** [Name both.]
- **Look at actual samples**, not only the score. An aggregate can look survivable while
  every item in the category you care about fails.
- **Never delete a checkpoint until the verdict is DISCARD.**

### [$] Seeds and the power gate

Seed count is *derived*, not chosen. Measure the coefficient of variation across seeds on
the baseline arm, then:

```
n ≈ 16 · (CV / 0.10)²   per arm      # to detect a 10% shift at 80% power
```

CV 5% → n ≈ 4. CV 10% → n ≈ 16. **Threshold is n ≤ [N].** Metrics failing it get dropped.
**If fewer than [K] survive, [the question] lacks the power to be answered — abandon it
before any training.**

## Things the Agent Must NOT Change

Config drift silently invalidates every baseline. Once frozen, these are immutable —
changing one means re-measuring the baseline from scratch.

- **[Harness / framework and version.]** [What breaks.]
- **[Sampling parameters — list every value.]**
- **[Precision.]** Never compare a baseline at one precision to a treatment at another.
- **[Fixed probe / item sets.]**
- **[The measured baseline numbers.]**
- **`hypothesis_log.md`** — append only, never delete or rewrite entries.

<!-- Prose is the weak version. Mechanize it (committed config hash + `make freeze-check`)
     as soon as the configs exist. -->

## Decisions — Settled vs Open

### Settled — do not re-litigate; changing one is a protocol revision, not an experiment

| Decision | Choice | Why |
|---|---|---|
| | | |

### Open — the search space, in priority order

1. [The main axis]

## Known Failure Modes (Read Before Running)

<!-- Each: name, symptom in terms of a LOGGED metric, and response. If the response is
     "stop and report," say so — a documented stopping point is a contribution, and
     without it an agent will improvise something ambitious and expensive. -->

**[Name].** Symptom: [logged metric crosses X]. Fix: [response, escalating no further
than needed].

## Reference Points and Gates

Context only — **gate against the measured baseline, not these.**

| Quantity | Value |
|---|---|

| Gate | Threshold |
|---|---|

## Deferred Interfaces — Do Not Implement Yet

**[Thing].** Not until [explicit trigger condition]. Design for it; do not write it.

## Scope Discipline

Explicitly **out**, however interesting: [list].

The recurring failure mode is that each of these reads as *one small addition*. [The MPU]
is [N] weeks away and nothing on that list gets there faster.
