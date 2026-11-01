# Pre-registration — `trading-path-latency`

**Version 0.1** · frozen 1 November 2026 · tagged `v0.1-prereg`
**Author:** Kakali Bandyopadhyay · **Licence:** MIT

This document fixes, before any code is written and before any measurement is taken,
what will be built, what will be timed, what the budget is, and how each possible
outcome will be read. It is committed as part of the repository's first commit and
will not be amended past tag `v0.1-prereg`. Anything that changes afterwards is
recorded in `prereg/DEVIATIONS.md` and travels with the result.

---

## 0. Scope, stated as narrowly as it actually is

**Timed:** the compute path from a decoded market-data message already resident in
memory, through feature state update, to a GBM prediction written to an output slot.
On a single pinned core, single-threaded.

**Not timed:** wire time, kernel ingress, NIC, DMA, no kernel bypass, no network stack,
no order egress. This is **service time**, not tick-to-trade. The distinction is the
difference between an engineering measurement and a marketing claim, and only the first
is on offer here.

Consequence to be carried into the write-up: a reader who wants "our latency" in the
sense a trading desk means it is not being given it, and the write-up will say so in
the first paragraph rather than in a footnote.

---

## 1. Claim, and its shape before results

The primary claim has two parts, and only the first is guaranteed to be reportable:

> (a) On the declared host and protocol, the single-message path
> (decode → features → GBM inference) completes within a budget fixed in advance:
> **p99 ≤ 10 µs**, **p50 ≤ 4 µs**, **p99.9 ≤ 25 µs**.
>
> (b) The budget is dominated by feature assembly, not tree traversal.

(b) is a **prediction under test**, not an assumption. If tree traversal dominates,
that is the finding and it is reported as a reversal (§8).

Deliberately absent: any claim that 10 µs matters for this strategy. Artefact 1's signal
is a 52-week trailing return rebalanced weekly. Nothing in that decision is latency-sensitive.
The budget exists because a pre-registered threshold is falsifiable and a vibe is not.

### 1.1 Why the budget numbers are what they are, and what makes them meaningful

No economic constraint forces these figures, and pretending otherwise would be the
dishonest part. They are an **engineering target chosen to be tight enough to force the
layout decisions that distinguish this from a naive implementation**: no allocation on
the hot path, no `std::map` / `std::string` keys, no virtual dispatch, flat contiguous
model arrays.

**The baseline this budget is measured against.** An unoptimised but reasonable C++
implementation — state in `std::map<std::string, std::vector<double>>`, every one of the
100 features recomputed from raw history on every message rather than updated
incrementally, model nodes heap-allocated and pointer-chased — is expected to land at
**p99 ≈ 50–150 µs**. The estimate is arithmetic, not a measurement: ~100 hash lookups and
~20,000 element visits per message for feature assembly, plus ~1,600 dependent loads
through 200 trees whose nodes are scattered across the heap.

Against that baseline, a **p99 budget of 10 µs demands roughly 5–15×**, which is a real
engineering requirement: it cannot be met by tidying the naive version, only by changing
the layout (incremental feature update, contiguous node array, no per-message allocation).
That is the intended band — hard enough to be informative, not rigged to fail.

**Pre-declared safeguard against a vacuous budget.** If the measured naive baseline comes
in below **30 µs p99**, the 5–15× premise is wrong and 10 µs proves little. In that case
the result is reported as *"budget met without forcing any layout decision"* — a null
result for the exercise — and any tightening of the budget belongs to v2 with its own
pre-registration. The budget is not adjusted after seeing the baseline.

---

## 2. The model class, fixed here because of a constraint in the other direction

The serving constraint acts **backwards** on model choice, and the choice was therefore
made at artefact-1 design time, not after profiling:

- A GBM (or a linear model) has a native evaluator of roughly 200 lines: a flat array of
  nodes, a pointer walk, a float compare. Serving it in C++ is honest, and an
  end-to-end claim about the path is therefore honest.
- A sequence model would require a Python bridge, a runtime, or an inference library,
  and the "end-to-end path" would then contain a foreign body whose cost is not
  attributable to this implementation. That converts a measurement into a benchmark
  of someone else's runtime.

**Frozen consequence:** the model class is GBM / linear. Artefact 1 must select within
that class. If artefact 1 later requires a model outside it, that is a deviation
affecting both repositories, recorded in both, and the end-to-end claim here is
re-scoped or withdrawn.

---

## 3. System under test (v1, frozen)

### 3.1 Input

A normalised message stream with four types, in the proportions of §6.2:

| type   | payload |
|--------|---------|
| `add`    | instrument id, side, price, quantity, order id |
| `modify` | order id, new price, new quantity |
| `delete` | order id |
| `trade`  | instrument id, price, quantity, aggressor side |

Records are fixed-width in the tested path. Decoding from the recorded/replayed buffer
**is part of the timed path**; the buffer is not pre-decoded into structs. Reading the
next record and validating it are inside the timestamps.

### 3.2 State

Per instrument: a fixed-capacity price-level book (64 levels per side), an order-id
hash map with open addressing and a fixed load factor, and a feature ring buffer.
All capacity is allocated once at start-up. **Zero allocations on the hot path** —
asserted by a test that intercepts `operator new` and fails the run if it is called
after warm-up.

### 3.3 Features — 100 columns

Twenty base features, each expanded over five lags `{0, 1, 2, 3, 8}` → 100 columns.
The lag expansion is deliberate: it is how real feature sets acquire width, and it makes
ring-buffer indexing, rather than the final arithmetic, the thing being measured.

Base features:

| # | feature | inputs |
|---|---------|--------|
| 1–4 | trailing mid-price return over 1, 5, 15, 60 updates | mid |
| 5–8 | EWMA of returns, half-lives 4, 16, 64, 256 updates | mid |
| 9–12 | realised volatility over 15, 60, 240, 960 updates | mid |
| 13–17 | bid-ask spread, normalised spread, mid-to-microprice, depth imbalance, trade imbalance | book |
| 18–20 | order-flow imbalance over 10, 50, 200 updates | book, orders |

Feature definitions are pinned in `src/features.md` and implemented once; the Python
reference and the C++ implementation must agree (see §9) or nothing here counts.

### 3.4 Model, and the two baselines

- **Model:** LightGBM GBM, 200 trees, `max_depth` 8, 100 features, `float32` leaves,
  trained on synthetic data from the generator in §6 with seed `20261101`. Exported to a
  documented text format (not the library's binary format) and loaded by the C++ side.
- **Evaluator:** flat array of `struct Node { int32 feature; float threshold; int32 left;
  int32 right; }` — 16 bytes, contiguous, no pointers; one walk per tree, leaf values summed.
  No early exit, no pruning, no vectorisation in v1 (§10).
- **Naive C++ baseline — the comparison that matters.** Same model, same message stream,
  built for the opposite design: `std::map<std::string, std::vector<double>>` state, all
  100 features recomputed from raw history per message, heap-allocated nodes. Compiled
  `-O2`. This is what §1.1's 50–150 µs estimate refers to, and it is what the §8
  justification rule is written against. If the optimised path cannot beat it by a wide
  margin, the layout work bought nothing.
- **Python baseline — context only, not a comparison.** The library scoring the same model
  on pre-built feature rows, timed under the same protocol. It exists to show the order of
  magnitude a bridge would have cost, and to make the argument in §2 concrete. It is *not*
  the justification threshold: beating a Python loop by 2× would be trivial and would prove
  nothing about the C++ layout.

### 3.5 Estimand — exactly what is inside the timestamps

```
t0 = clock read immediately after the message record is confirmed present in the buffer
t1 = clock read immediately after the prediction is written to the output slot
```

Excluded from both: buffer refill, thread wake-up, the timing harness's own accumulated
storage of samples (amortised by writing to a pre-allocated array of counters, never
printf inside the loop). The harness measures and reports its own clock overhead
(§5.5) so a reader can subtract it; it is **not** subtracted from the reported figures.

---

## 4. Budget (frozen, per single message, unloaded replay)

| stage | p50 | p99 | p99.9 |
|-------|-----|-----|-------|
| decode + book update | 1.0 µs | 2.0 µs | 5.0 µs |
| feature assembly (100 columns) | 0.8 µs | 2.0 µs | 6.0 µs |
| GBM inference (200 × depth 8) | 2.0 µs | 6.0 µs | 15.0 µs |
| **total path** | **4.0 µs** | **10.0 µs** | **25.0 µs** |

Stage budgets sum to less than the total on purpose: they are independent targets, not an
accounting identity. A stage may miss its own target while the total meets its own;
both facts get reported.

---

## 5. Measurement protocol (frozen)

### 5.1 Host

**Bare-metal Linux, Ubuntu 24.04 LTS, on the i7-1165G7 (4c/8t, 12 MB L3, 16 GB RAM),
installed on the incoming 2 TB SSD — not WSL2, not a VM, and not the existing Windows
drive repartitioned.** The 2 TB drive is partitioned 1 TB Windows / 1 TB Ubuntu 24.04 and
dual-booted on its own hardware; this removes the repartitioning risk that made a laptop
dual-boot unattractive, and leaves the current installation untouched.

WSL2 is excluded because it measures the Hyper-V scheduler's decisions; a virtualised TSC
and an un-controllable host make the numbers non-comparable to anything published.

**The laptop is the declared benchmark platform, deliberately.** The budget in §4 is
specified for a mobile part. A rented Xeon would make 25 µs trivially attainable and would
require re-deriving every number in §1.1 and §4 for a different platform — the budget would
no longer be the budget that was pre-registered. Keeping the declared platform fixed keeps
the document coherent.

**A second platform, if it happens, is an appendix, not the primary result.** If the
laptop numbers come out weak or unstable, a rented bare-metal host becomes a confirmation
run reported as such, with the platform difference stated. Two platforms would be a
strength; substituting one to rescue a headline would not be.

**One host, named and recorded, for the primary result.** No cloud instances: noisy
neighbours defeat pinning.

### 5.2 Isolation

- `intel_pstate` in `passive`; `no_turbo=1`; `energy_performance_preference=performance`.
- Pinned to a **physical core**, SMT sibling left idle (`taskset -c 2`, sibling verified
  idle via `/proc`). Single-threaded throughout.
- All non-essential services stopped; no network activity during runs; the run script
  records `uptime` and the process list at start.
- `perf_event_paranoid` set to permit `perf stat`; capabilities recorded.

### 5.3 Clocks

`CLOCK_MONOTONIC_RAW` via `clock_gettime` as the reported clock; TSC (`rdtsc`, invariant
on this part) read alongside as a cross-check. Resolution and the observed
clock-read-to-clock-read floor are reported as measured, not assumed.

### 5.4 Runs and warm-up

- Warm-up: 200,000 messages discarded (feature ring buffers and the model array must be
  resident in cache before the first counted message).
- **10 runs × 1,000,000 messages**, each run a fresh process.
- Every run is kept. No run is deleted, ever. A run excluded under the thermal rule
  (§5.6) is reported *as excluded, with its numbers*, not removed.

### 5.5 Instrumentation

`perf stat -e cycles,instructions,cache-references,cache-misses,LLC-loads,LLC-load-misses,branch-misses`
per run; report IPC and miss rates next to the latency figures, because they explain the
latency rather than merely accompanying it.

### 5.6 Thermal and frequency discipline

Package temperature recorded before and after each run, plus `cpufreq` policy state.
**Pre-declared rule:** if a run's start temperature differs from the median start
temperature by more than 5 °C, it is reported as thermally compromised and excluded from
the headline percentile, with its numbers still shown. This rule is fixed now precisely
because it is the one most tempting to invoke after the fact.

---

## 6. Arrival process and message mix (frozen)

### 6.1 Two measurement modes

- **Primary — unloaded service time.** Messages fed back-to-back with no inter-arrival
  gap. Measures the compute cost of the path and nothing else. This is what §4's budget
  is written against.
- **Secondary — loaded.** A Markov-modulated Poisson arrival process: baseline
  λ = 1,000 msg/s, burst state λ = 10,000 msg/s, mean burst duration 100 ms, mean
  non-burst duration 900 ms, fixed seed. Reports sojourn time (queueing included), which
  is a different quantity and is **not** compared against §4's budget.

Uniform arrivals are not used. Real feeds are bursty, and a uniform-arrival benchmark
flatters the implementation by never building a queue.

### 6.2 Message mix

`add` 1% · `modify` 84% · `delete` 1% · `trade` 14%.

Typical of an L2 equities feed. Fixed now, not tuned after seeing which mix looks best.
Order quantities lognormal; prices on a tick grid with a seeded random walk; the
generator is committed as code + seed, never as data (§12).

---

## 7. Statistics

- Reported per run: **p50, p99, p99.9**.
- Aggregated across the 10 runs: **median of the per-run percentiles**, with min and max.
- **No means. No mean of percentiles. No average latency, anywhere, in any table.**
  A mean of a heavy tail is a number that describes nobody's experience.
- Between-run variance is reported as the standard deviation of the per-run p99 **and**
  the full spread. If the sd of per-run p99 exceeds 1 µs, the result is reported as
  **unstable**, and the headline becomes the spread rather than a point.
- The primary decision statistic is the median-of-runs p99 of the total path against
  10 µs.

---

## 8. Pre-committed readings of each outcome

Written before measurement so that no result has to be interpreted under pressure.

| outcome | reading |
|---------|---------|
| Total p99 ≤ 10 µs, features dominate | Claim (a) and (b) both hold. §6-equivalent feature boundary becomes the load-bearing section; the budget is quoted as **features + inference as one path**. |
| Total p99 ≤ 10 µs, inference dominates | (a) holds, (b) is **reversed**. The write-up leads with the reversal: feature assembly was not the constraint, and the layout work went to the tree walk. No reframing as "as expected". |
| Total within budget at p99, but p99.9 breaches 25 µs | Reported as "meets p99, misses the tail", with the tail's cause identified. Budget is **not** revised. |
| Total p99 > 10 µs | Reported as a miss. Then, and only then, an optimisation round is opened as v2 with its own budget, and v1's numbers stay published. No silent re-run. |
| Native path less than **4×** faster than the naive C++ baseline | The layout work **bought nothing**. Reported plainly; the exercise becomes a negative result about the value of the flat-array rewrite, which is worth publishing precisely because it is unwelcome. |
| Measured naive baseline p99 < 30 µs | §1.1's premise is wrong and 10 µs proves little. Reported as "budget met without forcing any layout decision" — a null result for the exercise. Budget is not tightened after the fact. |
| Correctness gate (§9) fails | The end-to-end claim is **void**. Publish the divergence analysis. Latency numbers may be shown only as conditional on a failed gate, never as a claim. |
| Between-run sd of p99 > 1 µs | Reported as unstable; no headline figure. |

---

## 9. Correctness gate — the kill criterion

The native evaluator must reproduce the Python library's predictions, or the end-to-end
claim does not exist.

- **Sample:** ≥ 1,000,000 feature rows spanning the generator's full range, plus an
  adversarial set constructed to break naive evaluators:
  - values landing exactly on a split threshold (float equality, both sides);
  - `NaN` in each of the 100 columns, one at a time;
  - `±inf`, denormals, and `-0.0`;
  - rows with a single non-default feature;
  - duplicate feature values across columns (catches index confusion).
- **Criterion:** max absolute difference ≤ **1e-5** against the float64 evaluation of the
  same model, and **identical leaf assignments** on every row. Both are required: the
  tolerance catches accumulation error, the leaf-identity check catches a model that
  happens to land close on average while taking wrong paths.
- **Split semantics:** `x <= threshold` goes left, matching the library's default;
  missing values follow each node's `default_left` flag. These are documented per node in
  the export and asserted, not assumed.
- **Float width:** evaluated in float32, compared in float64 after the fact. A divergence
  bound is a tolerance, not zero, and the report will say which.

Failure of any item stops the latency claim. It does not stop the repository; it changes
what the repository is allowed to say.

---

## 10. Deliberately not built, and the condition that would change that

Each omission is a decision with a stated trigger. Building the whole technique menu
would be a compliance signal, not an engineering one.

| omitted | why | what would change it |
|---------|-----|----------------------|
| lock-free SPSC queue | Single-threaded replay: one producer, one consumer, same thread. Contention is **identically zero by construction**, so a queue would measure the cost of synchronisation that cannot occur. | A separate feed-handler thread, or a shared ring between decode and strategy. Then the queue is necessary, and its cost is real. |
| kernel bypass (DPDK / Solarflare) | No NIC in the measurement. The path begins at a decoded record. | Any wire-to-strategy measurement. At that point the claim is different and so is the repo. |
| SIMD feature kernels | Measure the scalar version against the budget first. Optimising before the profile is exactly the error this repo is meant to avoid. | Scalar feature assembly breaches its p99 budget. |
| custom allocator / memory pool | Nothing allocates on the hot path; fixed arrays, allocated at start-up. | A feature or model that requires per-message dynamic growth. |
| multi-instrument, cross-sectional features | The single-instrument message path is the unit of measurement. Cross-sectional features change the path's shape. | A portfolio-level model — which is where artefact 1 would begin to bite, and why this is flagged rather than hidden. |
| batching / vectorised inference | Batch latency and single-message latency are different quantities, and batching hides the per-message cost that a trading path actually pays. | A use case that scores on a timer rather than per message. |

---

## 11. Threats to validity, stated before results rather than after

1. **One host, laptop thermal envelope.** The 1165G7 in a laptop chassis throttles. The
   thermal rule (§5.6) mitigates but does not remove this. No claim of representativeness
   for server hardware.
2. **Synthetic messages.** Real feeds have gaps, out-of-order updates, snapshot recovery
   and exchange-specific quirks. The generator does not, so decode costs here are a
   lower bound on a real feed handler.
3. **Service time, not tick-to-trade.** See §0. This is the largest gap between what is
   measured and what a desk means by latency, and it is a scope decision, not a caveat
   discovered late.
4. **1e-5 is a chosen tolerance.** Float32 accumulation differs from float64. The number
   is defensible for a p&l-neutral comparison and is not a proof of equality.
5. **Self-imposed budget.** No external requirement fixes 10 µs. The budget's
   justification is comparative (§1.1), not economic.
6. **Compiler and flags are part of the result.** `-O3 -march=native` on this host is not
   portable; the flags are recorded in the results and the comparison baseline is built
   with the same flags where applicable.

---

## 12. Data and repository integrity

- **No vendor data is committed.** The message generator is committed as code with a
  fixed seed (`20261101`); every stream is regenerable from `git checkout` plus that seed.
  Hashes of any generated artefacts that are used in figures are committed under
  `results/`.
- Structure: `/prereg /src /notebooks /results /data /writeup`. `data/` is gitignored
  except the generator and any committed hashes.
- **First commit = this pre-registration + README together**, tagged `v0.1-prereg`,
  pushed public from that commit. A private repository's timestamp proves nothing.
- **Never rebase, amend or force-push past `v0.1-prereg`.**
- Pre-registration also lodged on **OSF** for an independent, third-party timestamp.
- `prereg/DEVIATIONS.md` is the deviation journal. Every departure from this document —
  including "changed the compiler flags", "added a feature", "ran three extra
  replications because the first ten looked odd" — gets an entry: date, what changed by,
  why, and what it does to the claim. The journal is published with the results.

---

## 13. Outputs, pre-declared

So that the analysis cannot be shaped after the fact, the finished artefact reports
exactly these:

- **T1** — latency by stage: p50 / p99 / p99.9 against §4 budgets, pass or miss per cell.
- **T2** — per-run percentile table, all 10 runs, including any thermally excluded run.
- **T3** — `perf` counters per run, with IPC and LLC miss rate.
- **T4** — correctness: sample size, max divergence, count of leaf mismatches, adversarial
  case results.
- **T5** — loaded-mode sojourn percentiles (secondary), clearly separated from T1.
- **T6** — both baselines: naive C++ against the 4× rule, and the Python bridge for context only.
- **F1** — latency distribution by stage (percentile plot, log-x), with the budget lines
  drawn on it.
- **Prose** — one paragraph per pre-committed outcome in §8 that actually occurred, and a
  deviations section reproducing the journal.

No other headline statistic will be introduced after the fact. If something worth
reporting turns up that is not in this list, it is reported as an exploratory finding,
labelled as such, and not folded into the claim.

---

## 14. Sign-off

This document is the commitment. It is deliberately specific enough to be wrong: the
budget numbers, the feature count, the tree depth, the arrival process, the thermal rule
and the correctness tolerance could all turn out to be badly chosen. That is the point of
writing them down first.
