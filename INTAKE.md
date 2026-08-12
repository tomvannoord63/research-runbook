# Intake — questions to answer before writing code

Answer these *with* the agent, not for it. The value is the shared model, not the document. Answers go into `CLAUDE.md`; the ones you cannot answer become the first entries under Open Questions in `hypothesis_log.md`.

Marked **[$]** = added at tier 3, when iterations cost money. Skip at tier 1.

---

## Cost and cadence — answer first, everything depends on it

1. **How long is one training run?** Minutes, hours, or a day. This is the tier.
2. **How long is one evaluation?** Often the real binding constraint — it is what you do most.
3. **[$] What does one iteration cost in dollars?** If nonzero, you need gates and a `COST:` line.
4. **What is the sub-60-second quick test?** The command that proves nothing is broken. If you cannot name one, build it first.
5. **What is the cheap proxy for the expensive eval?** You will iterate against the proxy and spend the expensive eval only on things that already look real. Name both.
6. **What hardware is available, and for how long?** Local vs rented changes the whole discipline.

## The research question

7. **What is the primary question?** One sentence. If it takes a paragraph, it is more than one question.
8. **Is there a secondary question, and is it ranked below the primary?** They are never co-equal, whatever it feels like on day one.
9. **[$] What kills the secondary question, and is the project still a success if it dies?** Decide this in advance. It converts an early negative result from a crisis into a finding.
10. **What is the minimum publishable unit?** The smallest complete artifact worth shipping. You want to always know how far you are from something.
11. **What are the ablation axes, in priority order?** These are the knobs the loop turns.
12. **What prior work is closest?** Specifically: what did it settle that you should not re-derive?

## Metrics, baselines, controls

13. **What metric defines success?**
14. **What is the control for that metric?** *Not* the baseline arm — the control. What score does cheating get: guessing, priors, question phrasing alone, the input withheld? This is the most-skipped question in the list and it is what separates a result from a number.
15. **What is the baseline, and is it measured or cited?** Prefer measured. Published numbers frequently come from internal forks, tuned harnesses, or configurations nobody documented. **Gate against your own measurement**; treat the published number as context and expect to land below it.
16. **What mechanism can your headline metric not observe?** This defines your `<DOMAIN>_DIAG:` prefix. See `CONVENTIONS.md`.
17. **What are the healthy ranges for those diagnostics — including readings that look bad but aren't?**
18. **[$] How noisy is each metric across seeds?** Measure the coefficient of variation on the baseline arm, then `n ≈ 16·(CV/0.10)²` for a 10% shift at 80% power. Metrics needing more seeds than you can afford get dropped, before training rather than after.
19. **Are you near a floor or ceiling?** A benchmark your model already fails at 0.3 cannot show degradation. Pick instruments with headroom in the direction you expect to move.

## Setup and freezing

20. **What gets frozen, and when?** Harness and version, sampling parameters, precision, serving config, probe sets, benchmark items, measured baselines. See `SETUP.md` step 5.
21. **What is the data pipeline, and is the data licensed for what you intend?** "Available to download" is not "licensed to train on." Answer before building the pipeline, not after.
22. **What is the checkpoint strategy?** Frequency, retention, and storage ceiling. Never delete a checkpoint until the verdict is DISCARD — and know in advance what you do when the disk fills.
23. **What logging/tracking?** wandb, TensorBoard, CSV, none. The log-line contract is required regardless; this is in addition to it.

## Agent-specific

24. **What is the git workflow for KEEP/DISCARD?** Branch per experiment, or commit-on-KEEP and restore-on-DISCARD against main? Decide before the first DISCARD, not during it.
25. **What files must the agent NOT modify?** Data loading, eval harness, baseline implementations, frozen configs, `hypothesis_log.md` (append only).
26. **What is deferred, and until when?** Interfaces that will be needed later but not now. Write down the trigger condition — "not until the end of Phase 3, when a grafted checkpoint first needs rollouts" — so deferral is a decision rather than an accumulating debt.
27. **What is explicitly out of scope?** List the tempting adjacent work by name, each with one sentence on why it doesn't get you to the result faster. Every one of them arrives looking like a small addition.

---

## The four that most often go unanswered

Not because they are hard, but because they feel premature on day one and expensive on day sixty:

- **#14, the control.** Skipping it is the most common way to report a non-result as a result.
- **#10, the minimum publishable unit.** Without it, scope has no floor and every addition is defensible.
- **#9, what kills the secondary question.** Without it, the negative result arrives as a crisis instead of a finding.
- **#24, the git workflow.** Without it, the first DISCARD is improvised, and improvised reverts are how the persistent log gets deleted.
