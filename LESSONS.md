# Lessons — failure modes of the loop itself

Accumulated across UseLeSs, nanoMoR, and saccade. These are failures of the *method*, distinct from your project's technical failure modes — those belong in your own `CLAUDE.md`.

**Append to this file when the loop fails you.** It is what makes the next project cheaper.

---

## What consistently works

**Background processes with completion notification.** Launch, set a Monitor, keep working. The loop's throughput is bounded by how much thinking happens during runs, not by how fast the runs are.

**Machine-parseable output.** One grep, no interpretation. Every hour spent on the log-line contract before writing code pays back several times.

**The KEEP/DISCARD/interpretation format.** It forces a verdict. Without it experiments blur into an undifferentiated sense that things are roughly working, which is the state in which people run the same failed configuration twice.

**Control experiments first.** UseLeSs burned time on MiniGrid Memory because the MLP baseline was never calibrated. When the treatment finally ran there was nothing to compare it against.

**Smoke tests.** A 500-iteration run that crashes at iteration 200 on a trivial bug costs the whole window. Five iterations first, every time. At tier 3 the smoke test also checks that diagnostics are non-degenerate, not just that nothing raised.

**Minimal changes, one variable.** Testing curvature regularization and `history_len` together produced a result attributable to neither.

**Cheap computations that can invalidate expensive ones, run first.** saccade's power gate costs inference only and can eliminate half the project in week two. Generalize the move: find the cheapest thing that could make the expensive thing pointless, and do it before the expensive thing.

---

## What consistently fails

**Blocking `sleep`.** `sleep 600 && check` burns context to no purpose and leaves the agent unable to work while it waits. There is always a better use of that window.

**Skipping the smoke test.** Under deadline pressure, always.

**Trusting single-seed results.** On UseLeSs, expanding a 20-episode eval to 100 reversed the ordering of two arms. A single-seed result is a hypothesis, not a finding.

**Over-tuning hyperparameters when the problem is architectural.** Sweeping learning rate for three experiments when the actual defect was a missing observation wrapper. The tell: small consistent gains from unrelated knobs, and no single change producing the effect size you predicted.

**Choosing the environment or benchmark without calibrating it.** UseLeSs spent four sessions on environments that did not require the capability under test. Before running the experiment, verify the instrument can *see* the effect: does the benchmark have headroom, does the baseline fail in the way you expect, would a known-good method score higher?

**Chasing a published number.** Published results routinely come from internal forks, tuned harnesses, and undocumented configurations. Reproducing them can be impossible with public tooling. Measure your own baseline, gate against that, and expect to land below the paper.

**Aggregate scores hiding categorical failure.** A compression setting can post a survivable average while failing *every* stack-trace item — which was the specific capability being bought. Look at actual samples, not only the number. Break the score down by category before believing it.

---

## Failures specific to running an agent as the researcher

**A fresh session skips step 1.** It does not know what it does not know, and the log is long. Fix: make reading it one command (`make brief`) and put that command as step 1 of the loop in `CLAUDE.md`. Making the right thing short beats instructing.

**The agent decides whether a number is good.** Especially after several experiments trending toward a result it would like to have. Fix: gates with thresholds declared before the run, and a non-zero exit code. Take the judgment call away.

**Monitors that look configured and never fire.** Missing `--line-buffered`; grep block-buffers in the pipe. Also: watching a pattern the code never prints, and watching a log the run writes elsewhere. Fix: the Makefile prints the exact command at launch, and setup step 7 verifies parsing against a synthetic log.

**Shell `&` instead of the harness's background flag.** The job runs, and is invisible — no completion notification, no task entry, no way to check on it except by guessing at PIDs.

**Stale `CLAUDE.md`.** It says commands exist that don't, or that nothing is built when things are. An agent trusts it over the filesystem and confidently does the wrong thing. Fix: update it in the same commit that invalidates it, and prefer "inspection targets work now; run targets fail with a pointer until their entry point exists" over an undifferentiated "not built yet."

**Settled decisions re-litigated.** A new session proposes the architecture you rejected in week one, plausibly, because the rejection was never written down. Fix: a Settled table with the reason, and the framing that changing an entry is a protocol revision rather than an experiment.

**Scope creep as "one small addition."** Each candidate is individually reasonable. Collectively they are why the result is perpetually four weeks out. Fix: name them in an explicit out-of-scope list with one sentence each on why they don't get you there faster.

**Discarding the code and the record together.** The reason `hypothesis_log.md` is gitignored. Verify `git check-ignore -v hypothesis_log.md` on day one, before the first DISCARD rather than during it.

**Reporting the last number rather than the right one.** `grep FINAL_RESULT | tail -1` on a log with two runs appended returns the second. Use per-experiment log paths (`$(LOGDIR)/$(PROJECT)-$(EXP).log`) and never reuse an `EXP` name.

---

## Domain-specific patterns worth carrying

**Generative models.** Expensive metrics (FID and friends) do not belong in the inner loop — use a proxy for iteration and the full metric for validation. Save sample grids: metrics improve while outputs get worse more often than anyone expects. Early training is seed-sensitive; do not read much into differences before the model has settled.

**MoE / routing architectures.** Loss cannot see routing collapse. Instrument entropy, load balance, and dead-token rate separately, per layer, and always log the quantity that distinguishes collapse from healthy specialization.

**Agentic / long-horizon evaluation.** Rollouts dominate the budget, so seed count must be derived rather than chosen. Behavioral metrics have wide variance and a third-party harness can compress a metric's dynamic range to nothing before you measure anything — check that the instrument works before trusting a null result from it.
