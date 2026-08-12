# Setup — spinning up a research-runbook project

Work through this in order. Steps 1–7 are all doable **before any training code exists**, and should be. Step 7 in particular is much easier to get right when there is no real run around to confuse a broken parser with a broken experiment.

Budget: half a day for steps 1–7 on a project you already understand.

---

## Step 1 — Establish the tier

What does one iteration cost, in wall-clock and in dollars?

| Tier | Iteration | Build |
|---|---|---|
| 1 | Minutes, free | Loop shape + a metric. Skip to step 8. |
| 2 | Hours, free | Everything except gates, freeze list, power gate. |
| 3 | Hours, billed | All of it. |

Answer honestly. A tier-1 project dressed in tier-3 machinery is procedural theater, and the agent will start treating the ceremony as the work. A tier-3 project on tier-1 machinery is how you spend a week of rental discovering the baseline moved in session 2.

Also answer now, because the rest of the setup depends on it:

- **What is the sub-60-second quick test?** The command that proves nothing is broken. If you cannot name one, that is the first thing to build.
- **What is the cheap proxy for the expensive eval?** You will iterate against the proxy and spend the expensive eval only on things that already look real.

---

## Step 2 — Answer the intake questions

`INTAKE.md`. Do it *with* the agent — the point is not the answers, it's the shared model. The four that most often go unanswered and most often hurt:

- What is the **control** for each metric? (Not the baseline arm — the control. What score does cheating get?)
- What is the **minimum publishable unit**?
- Which questions are **primary vs secondary**, and is the project still a success if the secondary dies?
- What is the **git workflow** for KEEP/DISCARD?

Write the answers into `CLAUDE.md` as you go. That is what `CLAUDE.md` is.

---

## Step 3 — Create the four files

```bash
cp .../templates/CLAUDE.md          CLAUDE.md
cp .../templates/hypothesis_log.md  hypothesis_log.md
cp .../templates/Makefile           Makefile
cat .../templates/gitignore      >> .gitignore
```

Then immediately verify the one line that matters:

```bash
git check-ignore -v hypothesis_log.md    # must print the .gitignore rule
```

If that comes back empty, the log is tracked, and the first `git restore` on a DISCARD will delete the record of why you discarded it. **Check this before anything else.**

Set `PROJECT` at the top of the Makefile. Everything else has a working default.

---

## Step 4 — Define the log-line contract

Before writing code, decide what every run prints. See `CONVENTIONS.md` for the full contract; the short version:

```python
print(f"FINAL_RESULT: {metric}={value:.6f}")
print(f"GATE: {gate_id} value={v:.4f} threshold={t:.4f} verdict={'PASS' if ok else 'FAIL'}")
print(f"{DOMAIN}_DIAG: layer={i} key={v:.4f} ...")
```

Two decisions here that are hard to change later:

**Which diagnostic prefix, and what goes in it.** Ask: *what mechanism can my headline metric not observe?* nanoMoR's val_loss could not see router collapse. saccade's benchmark score could not see whether image tokens were crowding into text experts. That mechanism gets its own prefix, logged every run, with documented healthy ranges — including the ranges that look bad but aren't. Getting this wrong means running twenty experiments blind to the thing that was actually happening.

**Which controls ride along in the same line.** If a number is meaningless without its control, put them in one line. Not two lines, not a separate file. They will drift apart otherwise.

---

## Step 5 — Write the freeze list

*(Tier 2–3. Skip at tier 1.)*

In `CLAUDE.md`, under "Things the Agent Must NOT Change," list everything whose modification silently invalidates a previously measured baseline. Typical members:

- Harness / eval framework and its version
- Sampling parameters — every one, with values
- Precision (**never** compare a baseline at one precision to a treatment at another)
- Serving configuration, if batching affects results
- Any fixed probe or item set
- Benchmark items, once scored
- The measured baseline numbers themselves
- `hypothesis_log.md` — append only

For each, one clause on *what breaks*. "Frozen at Phase 0. All conclusions are deltas measured inside one harness" is enforceable by a reader. "Don't change the harness" is not.

Prose is the weak version. Mechanize it — a committed config hash plus a `make freeze-check` that refuses to launch on mismatch — as soon as the configs exist. Silent invalidation should become a loud failure.

---

## Step 6 — Wire the loop into the Makefile

The template ships the targets. What you supply is the entry points at the top:

```make
TRAIN ?= train.py
EVAL  ?= evaluate.py
```

Before those files exist, the run targets fail with a pointer to your spec instead of a traceback. That is deliberate: the Makefile goes live unchanged the day the code lands, and until then the inspection targets are already useful.

Targets, and what each is for:

| Target | Loop step |
|---|---|
| `make brief` | 1 — Current Best / Ruled Out / Open Questions |
| `make smoke EXP=<n>` | 4 — cheap sanity run, always before `exp` |
| `make exp EXP=<n>` | 5 — the run |
| `make monitor-cmd EXP=<n>` | 5 — reprint the Monitor command |
| `make gates-exp EXP=<n>` | 7 — every gate + verdict, non-zero exit on any FAIL |
| `make status-exp` / `result-exp` / `diag-exp` | 7 — inspection |
| `make ps` / `kill-exp` / `logs` | housekeeping |

---

## Step 7 — Verify the harness against a synthetic log

**Do this before the first real run.** Write a fake log containing every line format you documented in step 4, then confirm every inspection target parses it:

```bash
cat > /tmp/myproject-fake.log <<'EOF'
FINAL_RESULT: val_loss=3.141592
GATE: baseline_repro value=80.1000 threshold=78.7000 verdict=PASS
GATE: routing_health value=0.6200 threshold=0.7000 verdict=FAIL
MYDOMAIN_DIAG: layer=0 h_img=2.1 h_text=2.9 ratio=0.72
EOF

make result-exp EXP=fake
make diag-exp   EXP=fake
make gates-exp  EXP=fake ; echo "exit=$?"     # must print "1 passed, 1 failed", exit non-zero
```

(`make` exits 2 when a recipe fails, not 1. Test for non-zero, not for a specific code.)

Every parsing bug you find here is one you would otherwise have found at the end of an expensive run, at the exact moment you were least equipped to tell "the parser is broken" from "the experiment failed."

Check the exit code specifically. A `gates-exp` that prints FAIL and exits 0 is worse than no gate — it looks like enforcement and isn't.

---

## Step 8 — Run the loop

```
1. make brief                              — what is established, what is ruled out
2. Formulate ONE hypothesis, with a predicted direction
3. Minimal change: config over code, code over architecture
4. make smoke                              — no crash AND diagnostics non-degenerate
5. Launch in background; set the Monitor
6. While waiting: plan the next experiment or analyze the last one. Never sleep.
7. Collect. make gates-exp
8. Append to hypothesis_log.md
9. KEEP → commit. DISCARD → git restore
10. Repeat
```

**Step 5, mechanically** (Claude Code):

- Bash tool with `run_in_background: true`. **Not** a shell `&` — a backgrounded shell job is invisible to the harness and you lose completion notification.
- Then set a Monitor on the log. `make exp` prints the exact command; `make monitor-cmd EXP=<name>` reprints it.
- `--line-buffered` is not optional. Without it grep block-buffers in the pipe and the Monitor stays silent — configured, and never firing.

**Step 6 is where the loop earns its name.** Waiting is the budget you spend on thinking. Analyze the previous result properly, draft the next hypothesis, look at actual samples rather than only aggregate scores.

**Step 9 is not optional and is the step people drop.** The log is gitignored precisely so `git restore` can throw away a failed experiment's code while the record of why survives. Skip the write-up and you have thrown away the only durable output of a failed run.

---

## Step 9 — Maintain

- **`Ruled Out` is as important as `Current Best`.** It is what stops session 9 from re-running session 3's dead end, and it only works if failures get written with the same specificity as successes. "9x at 720p, score 0.31 vs blind 0.28 — fails every stack-trace item while the aggregate looks survivable" beats "9x was bad."
- **Record the interpretation, not just the number.** The number is superseded by the next run. The interpretation is what compounds.
- **Update `CLAUDE.md` when a decision settles.** Move it from Open to Settled with the reason. An undocumented settled decision gets re-litigated by the next session, which is pure waste.
- **Append to `LESSONS.md` in this repo when the loop itself fails you.** That is the file that makes the next project cheaper.
