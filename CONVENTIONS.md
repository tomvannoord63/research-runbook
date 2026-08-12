# Log-line conventions

The contract between a run and the agent reading it. Everything in the Makefile is a grep against these, so decide them before writing code — changing a prefix later orphans every log you have.

## Rules

1. **One line, one prefix, `key=value` pairs.** No multi-line records, no JSON blobs, no tables. `grep | tail -1` has to be sufficient.
2. **Unbuffered.** `python -u`, or `PYTHONUNBUFFERED=1`. Otherwise the line exists but arrives an hour late, and any Monitor watching for it is useless.
3. **Fixed precision.** `:.4f` or `:.6f`. Scientific notation and varying width break naive parsing at the worst time.
4. **A number rides with its control.** Same line. Not a second line, not a separate file, not "we measured it in March."
5. **Print unconditionally.** A diagnostic behind `if verbose:` is a diagnostic you will not have for the run that mattered.

---

## `FINAL_RESULT:` — the headline

```python
print(f"FINAL_RESULT: {metric}={value:.6f}")
```

Last line of the run. One per run. This is the number that answers "did it work."

---

## `GATE:` — the pre-declared decision

```python
print(f"GATE: {gate_id} value={v:.4f} threshold={t:.4f} verdict={'PASS' if ok else 'FAIL'}")
```

**The threshold is declared before the run, not chosen after it.** That is the whole point: it removes the agent (and you) from the position of deciding whether a number is good while invested in the answer.

Requirements:

- Emit `value` **and** `threshold` **and** `verdict`. A bare PASS/FAIL is unauditable — six months later you cannot tell whether it passed by a hair.
- `gate_id` is stable across runs. It is the join key across the whole project.
- `make gates-exp` **exits non-zero if any gate fails.** Verify this in setup step 7. A gate that prints FAIL and exits 0 looks like enforcement and is not.

> **A gate that isn't printed with its threshold and verdict isn't a gate. It's a hope.**

---

## `<DOMAIN>_DIAG:` — the mechanism your headline can't see

```python
print(f"ROUTING_DIAG: layer={i} h_img={hi:.4f} h_text={ht:.4f} ratio={hi/ht:.4f} overlap={ov:.4f}")
```

Every project needs at least one. The design question is not "what else can I log" but:

> **What mechanism can my headline metric not observe?**

Worked examples:

- nanoMoR: validation loss looked healthy while the router had collapsed — every token exiting at recursion step 1, no routing happening. `ROUTING_DIAG:` carried entropy, dead-token ratio, and per-depth load balance.
- saccade: a benchmark score cannot see whether image tokens are crowding into the experts text needs. Same prefix, different contents.

Three rules that generalize:

**Document the healthy range, including the ranges that look bad but aren't.** The general shape to watch for: a diagnostic whose *low* reading means failure in one regime and healthy specialization in another. A single scalar cannot separate those, and the difference is usually "stop and fix this" versus "this is the interesting result." If your diagnostic has a reading that is easy to misread as a failure, say so at the point of definition — the agent that reads it at 2am will not re-derive it.

**Log the disambiguating quantity even when it feels redundant.** The corollary of the above: when one number has two incompatible interpretations, a second number resolves it, and the one you skipped is invariably the one that turns out to have been load-bearing.

**Keep it out of the Monitor pattern if it's high-volume.** One line per layer per checkpoint will bury a stack trace. Surface its health as a single `GATE:` line and leave the raw lines for `make diag-exp`.

---

## `COST:` — optional, tier 3

```python
print(f"COST: gpu_hours={h:.2f} usd={d:.2f} arm={arm}")
```

If you are renting, log it. Accumulated in `hypothesis_log.md`, it turns "can we afford the next phase" from a guess into a lookup, and it prices the cheap-proxy discipline honestly.

---

## Choosing the Monitor pattern

```bash
tail -f $(LOG) | grep -E --line-buffered "FINAL_RESULT|GATE:|Error|Traceback|CUDA out of memory"
```

**`--line-buffered` is mandatory.** Without it grep block-buffers inside the pipe and emits nothing until ~4KB accumulates. The Monitor looks configured. It fires hours late or not at all.

Pattern design is a signal-to-noise problem, not a completeness problem:

- **Always include:** the headline, gates, and the crash strings for your stack (`Error`, `Traceback`, OOM, whatever your scheduler prints on preemption).
- **Usually exclude:** high-volume diagnostics. Their health belongs in a `GATE:` line.
- **Include a heartbeat** if runs are long enough that silence is ambiguous — a periodic step/loss line so "nothing since 14:00" distinguishes "still running" from "hung."

Put the pattern in the Makefile as `WATCH_PATTERN`, print it at launch, and never retype it from memory.

---

## Anti-patterns

| Don't | Why |
|---|---|
| Log the score without its control | The most common way to report a non-result as a result |
| Add a prefix mid-project | Orphans every prior log; the comparison you wanted is gone |
| Vary field order or names across runs | Breaks `grep`, breaks later analysis, breaks silently |
| Print gates only on success | You need the FAIL line most |
| Compute the diagnostic only at the end | You want the trajectory; collapse mid-run is invisible from the endpoint |
| Log to stderr only | `2>&1 | tee` catches it, but a Monitor on a redirect may not |
