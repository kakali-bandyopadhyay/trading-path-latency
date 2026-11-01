# trading-path-latency

C++ market-data → features → inference path, with the latency budget pre-registered
before optimisation and every measurement documented with its caveats.

**Status: pre-registration stage (tag `v0.1-prereg`). No implementation yet.**

## What this is

A measurement, not a demo. The question is what the compute path from a decoded
market-data message to a model prediction actually costs, and which stage of it dominates.
The budget in [`prereg/PREREGISTRATION.md`](prereg/PREREGISTRATION.md) was fixed before any
code was written, so that the result cannot be arranged after the fact to match the number.

Two things are deliberately excluded from the claim, and the write-up will say so early
rather than late:

- **This is service time, not tick-to-trade.** No NIC, no kernel ingest, no bypass. The
  path begins at a decoded record in memory.
- **No economic requirement fixes the budget.** The model is a weekly-rebalanced signal;
  nothing in it is latency-sensitive. The budget exists because a pre-registered threshold
  is falsifiable, and a vibe is not.

## The design decision that constrains the other repository

A GBM has a native evaluator of roughly 200 lines — flat node array, pointer walk, float
compare. Serving it in C++ is honest, and so an end-to-end claim about the path is honest.
A sequence model would have forced a Python bridge into the middle of the measurement,
at which point "end-to-end path" would include someone else's runtime and the number would
belong to them.

So the model class was fixed at artefact-1 design time, not after profiling here. There is a
companion repository, [`tsmom-panel-study`](https://github.com/kakali-bandyopadhyay/tsmom-panel-study),
whose model selection is bounded by this constraint.

## What is in here

| path | contents |
|------|----------|
| `prereg/PREREGISTRATION.md` | the frozen protocol: budget, arrival process, message mix, statistics, kill criteria, pre-committed readings of every outcome |
| `prereg/DEVIATIONS.md` | the deviation journal — every departure from the protocol, with what it does to the claim |
| `src/` | the implementation (not yet written; v1 is frozen by the pre-registration) |
| `results/` | measurement outputs, all runs kept, including excluded ones |
| `writeup/` | the report |

## Method, in one line

Profile first, decide second. The omissions in §10 of the pre-registration — no lock-free
queue, no SIMD, no custom allocator — are each recorded with the production condition that
would make them necessary. Skipping a technique because the contention is identically zero
in a single-threaded replay loop is judgement. Building the whole menu is compliance.

## Licence

MIT. See [LICENSE](LICENSE).
