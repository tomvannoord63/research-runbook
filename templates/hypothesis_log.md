# Hypothesis Log — [PROJECT]

This file is **NOT tracked by git**. It accumulates across all experiments and survives the
`git restore` that discards a failed one. Read it before every experiment (`make brief`).
Update it after every experiment.

Verify it is actually ignored: `git check-ignore -v hypothesis_log.md`

<!-- The three sections below Current Best are what `make brief` prints. The awk range
     stops at the first `---`, so keep those separators in place. -->

---

## Current Best

> **Nothing established.** No baseline has been measured. Every number below is a
> placeholder, and no treatment result means anything until they are filled in.

```
Control / blind baseline:        NOT MEASURED
Self-measured baseline:          NOT MEASURED
Current best:                    NOT MEASURED
Chosen [main axis]:              UNDECIDED
```

Reference only — **do not gate against these**: [published numbers, with the reason they
are context rather than targets].

---

## Session 1 — [theme] (YYYY-MM-DD)

### Experiment N — YYYY-MM-DD
**Phase**:
**Hypothesis**: [specific predicted direction — not "try X" but "X will move Y in
direction Z because W"]
**Change**: [exact config/code change, one variable]
**Result**: [paste the machine-parseable lines verbatim]
**Gates**: [gate_id: PASS|FAIL for every gate this run touched]
**Verdict**: KEEP / DISCARD
**Interpretation**: [why did this happen? this is the part that outlasts the number]
**Next**: [what this implies for the next run]

---

## Ruled Out

*Tested and conclusively dead, with the reason. This section is what stops a later session
re-running a dead end — it only works if failures are written down as carefully as
successes.*

*Be specific. "9x at 720p, score 0.31 vs blind 0.28 — fails every stack-trace item while
the aggregate looks survivable" beats "9x was bad." The interpretation is what transfers;
the number will be superseded.*

---

## Open Questions

*Things worth investigating, not yet tested. Carry the unanswered items from INTAKE.md in
here on day one. Mark answered ones with strikethrough and the answer rather than deleting
them — "we already checked that" is itself a finding.*

1.
2.
