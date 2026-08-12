# Lineage — how this was derived

Three generations. Each one kept the loop shape and added machinery, and in every case the machinery was a response to one iteration getting more expensive. Read this once so you know which parts you can drop.

---

## Generation 0 — the loop shape

Karpathy's autoresearch work is where I took the framing of a coding agent as the one running the experiments (see References). The shape:

```
hypothesis → minimal change → run → read the number → decide → repeat
```

An agent proposes a change, runs an eval, reads a metric, and iterates.

What makes that work unmodified is a **short eval window** — call it minutes. At that cadence the loop is essentially a REPL. You need no memory across sessions, because the session outlives the experiment. You need no gates, because you are watching. You need no cost discipline, because there is no cost.

**Everything below is a response to that window getting longer.** If a full iteration genuinely takes minutes, stop reading and use generation 0. The rest of this document is overhead you have not yet earned.

---

## Generation 1 — UseLeSs and nanoMoR

Eval window: **minutes to hours.** Local machine, MPS or a single consumer GPU. Free but slow. 17+ experiments over 6 sessions on UseLeSs; nanoMoR's smoke test alone ran ~80 minutes.

Six things broke.

### Sessions end before the research does

Context windows fill; sessions restart. The agent would re-derive conclusions it had already reached, and worse, re-run experiments it had already ruled out.

→ **`hypothesis_log.md`**, and critically, **gitignored**. The KEEP/DISCARD workflow reverts failed experiments with `git restore`. A tracked log would be reverted along with the code that produced it — deleting the record of *why* the code was thrown away, which is the only durable output of a failed experiment. The gitignore is the entire trick. It is the difference between a loop with memory and a loop without.

### The agent could not reliably read its own output

Asking an agent to scan 4,000 lines of training log and extract the final validation loss works until it doesn't, and when it doesn't it silently reports the wrong number.

→ **`FINAL_RESULT: <value>` as the last line of every run.** One grep, no interpretation. This is the ancestor of everything in `CONVENTIONS.md`.

### Blocking on a run destroyed the context window

`sleep 600 && check results` burns context to no purpose and leaves the agent unable to do anything useful while it waits.

→ **Background execution plus a Monitor on the log.** In Claude Code specifically this means the Bash tool with `run_in_background: true` — *not* a shell `&`, which detaches the process into invisibility. Then the agent plans the next experiment while the current one runs.

### Single-seed results lied

Repeatedly. On UseLeSs, expanding a 20-episode eval to 100 episodes reversed the ordering of two arms.

→ **Multi-seed validation, report mean ± std.** A result is real only if it holds across seeds. Generation 2 replaces the "just run 3 seeds" heuristic with something derived — see below.

### The headline metric hid the actual failure

nanoMoR's validation loss looked fine while the router had collapsed: every token exiting at recursion step 1, no routing happening at all. Loss alone could not see it.

→ **Domain diagnostic lines under their own prefix** (`ROUTING_DIAG:`), logged every run, with documented healthy ranges. The generalization: *identify the mechanism your headline metric cannot observe, and instrument it separately.*

### Two variables moved at once

Testing curvature regularization and `history_len` in the same run produced a result that could not be attributed to either.

→ **One variable per experiment.** Boring, and violated constantly under time pressure.

### What generation 1 looks like

Four files. `CLAUDE.md` (tracked), `hypothesis_log.md` (ignored), `Makefile`, `.gitignore`. A `FINAL_RESULT:` line and one diagnostic prefix. `EXP=`-named runs writing to `/tmp/<project>-$(EXP).log`. This tier is sufficient for a large fraction of research projects and is where most should stop.

---

## Generation 2 — saccade

Eval window: **hours to a day, on a rented GPU billing by the hour.** A single arm can occupy an H100 for a day. The loop shape is unchanged; every failure mode from generation 1 still applies. What changed is that **being wrong now costs money**, and the errors that cost the most are the ones you do not notice.

Seven additions.

### The agent was deciding whether numbers were good

"Validation loss 3.14 — that's an improvement" is a judgment call, and an agent makes it inconsistently across sessions, especially when it has been working toward a result and would like one.

→ **`GATE:` lines that carry the value, the threshold, and the verdict**, with the threshold declared *before* the run. `make gates-exp` exits non-zero when any gate fails. The agent no longer decides whether a number passed; it reports what the gate said. This is the single highest-value addition in generation 2 and it is nearly free.

**A gate that isn't printed with its threshold and verdict isn't a gate.** It's a hope.

### Casual multi-seed became unaffordable

Generation 1's "run 3 seeds and eyeball it" assumes seeds are cheap. When one seed is a day of rented GPU, seed count has to be justified rather than chosen.

→ **Derive it.** Measure the coefficient of variation across seeds on the baseline arm, then:

```
n ≈ 16 · (CV / 0.10)²    per arm      # to detect a 10% shift at 80% power
```

CV 5% → n ≈ 4, affordable. CV 10% → n ≈ 16, not. **Metrics whose required n exceeds your budget get dropped**, and if too few survive, the question they were supposed to answer cannot be answered — abandon it before spending anything on training.

This inverts the usual order. Instead of running the experiment and discovering it was underpowered, you compute the power first, using inference only, and let it kill the question in week two. On saccade this check can eliminate half the project before a single training step. **It is the highest-value computation in that protocol.** Generalize it: *find the cheapest computation that could invalidate your expensive one, and do it first.*

### Baselines drifted silently

Every conclusion in a controlled study is a delta measured inside a fixed setup. Change the harness version, the sampling temperature, or the precision, and every previously measured baseline is quietly invalid — with no error, no warning, and numbers that still look plausible.

→ **An explicit "Things the Agent Must NOT Change" list** in `CLAUDE.md`: harness and version, sampling parameters, precision, serving configuration, the fixed probe set, benchmark items once measured, the measured baseline numbers themselves.

Prose is the weak version of this and everyone knows it. The strong version is a committed config hash and a `make freeze-check` that refuses to launch on a mismatch. Write the prose list on day one; mechanize it as soon as the configs exist.

**Never compare a baseline at one precision against a treatment at another.** Quantization can itself move behavioral metrics, which makes it a confound in exactly the study most likely to use it.

### A number without its control kept getting reported

A benchmark score means nothing without the score achievable by cheating — priors, question phrasing, guessing. On saccade that control is the "blind" baseline: the same items scored with the image withheld.

→ **The log line carries the control alongside the treatment**, in the same line, always:

```python
print(f"CODESHOT: res={res} ratio={ratio} score={s:.4f} blind={b:.4f} tokens_per_shot={n}")
```

Not in a separate line, not in a separate file, not "we measured it once in March." Same line, every time. If they can drift apart, they will.

### Fresh sessions skipped step 1

Step 1 of the loop is "read `hypothesis_log.md`." It is the easiest step to skip and the most expensive to skip, and a fresh session skips it by default because it does not know what it does not know.

→ **`make brief`** — prints Current Best, Ruled Out, and Open Questions in one command. Making the right thing one word long is more effective than instructing an agent to do it.

### Monitors that looked configured and never fired

```bash
tail -f log | grep -E "FINAL_RESULT|GATE:"          # silent, possibly forever
tail -f log | grep -E --line-buffered "FINAL..."     # correct
```

Without `--line-buffered`, grep block-buffers inside the pipe and emits nothing until 4KB accumulates. The Monitor appears configured. It fires hours late or not at all. On a rented GPU that is the difference between catching a crash at the first checkpoint and discovering it after an hour of billing.

→ **`--line-buffered` documented as mandatory, with the reason**, and the exact command printed by the Makefile at launch so nobody retypes it from memory.

Also learned: **choose the watch pattern to preserve signal.** saccade's per-layer routing diagnostics emit one line per MoE layer per checkpoint — including them in the Monitor pattern would bury the crashes. They are deliberately excluded, and routing health surfaces as a single `GATE:` line instead.

### Scope crept, disguised as small additions

Every abandoned direction arrives looking like one small addition. Individually each is reasonable; collectively they are why the interesting result is always four weeks away.

→ **Two tables in `CLAUDE.md`.** *Settled* — decisions not to re-litigate, with the reason, where changing one is a protocol revision rather than an experiment. *Out of scope* — the tempting adjacent work, named explicitly, with a sentence on why it doesn't get you to the result faster.

Naming them costs nothing and converts a recurring negotiation into a lookup.

### One more that generalizes: rank your questions

saccade has a primary question that is guaranteed to produce a usable result and a secondary question that is more novel and might produce nothing. They are explicitly **not** co-equal, the work is sequenced primary-first, and the doc states outright that if the secondary dies at the power gate the project is still a success.

Deciding this in advance is what makes an early negative result a finding rather than a crisis. Pair it with a **minimum publishable unit** — the smallest complete artifact worth shipping — so you always know how far you are from something.

---

## What generalizes vs. what was project-specific

| Piece | Reusable? |
|---|---|
| Gitignored `hypothesis_log.md` | Always. The single most important line. |
| `FINAL_RESULT:` / machine-parseable output | Always |
| Background run + Monitor, never `sleep` | Always |
| KEEP → commit, DISCARD → `git restore` | Always |
| One variable per experiment | Always |
| Baseline/control before treatment | Always |
| `make brief` | Always. Costs ten minutes. |
| `GATE:` lines with pre-declared thresholds | Whenever a run costs enough that you'd rather not re-run it |
| Freeze list | Whenever conclusions are deltas rather than absolute numbers |
| Derived seed count (power gate) | Whenever seeds are expensive. Otherwise just run 3. |
| Cheap-proxy-for-iteration discipline | Whenever full eval ≫ proxy eval in cost |
| Settled / out-of-scope tables | Whenever the project runs longer than a few weeks |
| Domain diagnostic prefix | Always — but the *contents* are entirely project-specific |
| Specific thresholds, metrics, arms | Never. These are the project. |

The last row is worth dwelling on. It is tempting to copy saccade's routing diagnostics or nanoMoR's expected-performance table into a new project because they look like structure. They are not structure; they are findings. Copy the *shape* — "identify the mechanism your headline metric cannot see, give it a prefix, document its healthy range" — and derive the contents fresh.

---

## References

- **Karpathy — autoresearch.** https://github.com/karpathy/autoresearch — origin of the generation-0 framing: a coding agent running the hypothesis loop itself. The characterization of iteration cost as the governing variable is mine, not a claim about that work.
- **nanoMoR** — https://github.com/tomvannoord63/nanoMoR. Mixture-of-Recursions on a nanochat base. Public; the generation-1 examples can be checked against it.
- **UseLeSs** — hierarchical RL, 17+ experiments over 6 sessions. Not public; cited for provenance.
- **saccade** — vision grafting onto a 33B MoE. Not public at time of writing; cited for provenance. Methodology only — no results appear here, because as of this writing there are none.
