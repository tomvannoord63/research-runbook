# research-runbook

Setup instructions for running a coding agent as a research loop: it forms a hypothesis, makes a minimal change, launches a run, reads a gate, records the verdict, and iterates — without a human in the seat for every step.

The loop shape comes from Karpathy's autoresearch work. Everything on top of it was added across three projects with wildly different iteration costs, and **the additions are not decoration** — each exists because something specific broke, and what broke was almost always a consequence of one iteration getting more expensive. `LINEAGE.md` traces that, and marks where the borrowed framing ends and my own claims begin.

## The thesis

**The machinery a research loop needs is a function of what one iteration costs.**

| One iteration costs | What the loop needs |
|---|---|
| Minutes | A metric and a log. Karpathy's shape works as-is. |
| Hours | Persistent memory across sessions, background execution + monitoring, machine-parseable output. |
| Hours **and dollars** | Gates with pre-declared thresholds, derived seed counts, a frozen-config list, cheap proxies for iteration. |

Building tier-3 machinery for a tier-1 problem is procedural theater. Running a tier-3 problem on tier-1 machinery is how you burn a week of GPU rental and discover the baseline moved.

**Find your tier first.** It is the first question in `INTAKE.md` and it determines everything else.

## Files

| File | What it's for |
|---|---|
| `SETUP.md` | The process, in order. Start here when spinning up a project. |
| `LINEAGE.md` | Where each piece came from and what broke to cause it. Read once. |
| `CONVENTIONS.md` | The log-line contract — `FINAL_RESULT:`, `GATE:`, `*_DIAG:`. The load-bearing piece. |
| `INTAKE.md` | Questions to answer before writing code. Have the agent answer them with you. |
| `LESSONS.md` | Failure modes of the loop itself, accumulated. Distinct from your project's failure modes. |
| `templates/` | `CLAUDE.md`, `hypothesis_log.md`, `Makefile`, `gitignore` — copy and fill. |

## Quickstart

```bash
KIT=/path/to/research-runbook      # wherever you cloned this
cd /path/to/your-new-project && git init

cp  $KIT/templates/CLAUDE.md          CLAUDE.md
cp  $KIT/templates/hypothesis_log.md  hypothesis_log.md
cp  $KIT/templates/Makefile           Makefile
cat $KIT/templates/gitignore       >> .gitignore

git check-ignore -v hypothesis_log.md    # must print a rule. if empty, stop and fix
```

Then work through `SETUP.md`. The whole of it is doable before any training code exists — and should be, because step 7 is verifying the harness against a synthetic log, which is much easier when there is no real run to confuse it.

## The invariants

Everything else is negotiable. These four are what make it a loop rather than a series of runs:

1. **`hypothesis_log.md` is gitignored.** It has to survive the `git restore` that discards a failed experiment. This single line is the reason the loop has memory.
2. **Every run prints machine-parseable results.** The agent greps; it does not interpret prose it wrote earlier.
3. **Every verdict is KEEP or DISCARD, written down before moving on.** Including — especially — the failures. `Ruled Out` is what stops session 9 from re-running session 3's dead end.
4. **Never block on `sleep`.** Background the run, set a Monitor, and use the wait to plan the next experiment.

## Provenance

- **Karpathy — autoresearch.** The loop shape and the agent-as-researcher framing. Link in `LINEAGE.md`.
- **UseLeSs** — hierarchical RL, 17+ experiments over 6 sessions. First version of the persistent log, the `FINAL_RESULT:` convention, and the background-process pattern. *Private.*
- **[nanoMoR](https://github.com/tomvannoord63/nanoMoR)** — Mixture-of-Recursions on a nanochat base. `EXP=`-named runs, domain diagnostic prefixes, Monitor integration. *Public — the generation-1 examples can be checked against it.*
- **saccade** — vision grafting onto a 33B MoE. Gates with thresholds, derived seed counts, the freeze list, cheap-proxy discipline. *Private.*

Methodology only. No experimental results from any of these appear in this repo.

## License

MIT. See `LICENSE`.
