# 13 — Token Model

A cost model, not a measurement. Every number here is derived from the budget table in §10.5
and stated assumptions below — the system has no telemetry yet. Treat these as sizing
estimates to be replaced with measured values once phase 1 is running.

## 13.1 Assumptions

| Assumption | Value | Basis |
|---|---|---|
| Epic size | 12 work orders | Definition-stage agents run per epic, not per work order |
| Risk mix | 50% low / 50% medium-or-high | Low risk = touches no security path, contract, migration or infra (§3.2) |
| Bench | 5 seats on low risk, 10 on medium/high | §7.12 |
| First-pass yield — optimized | 70% | Target band once gates are trusted (§6.3) |
| First-pass yield — unoptimized | 50% | No gate discipline, no failure bundles |
| Docs changed | 60% of work orders | Not every change is public-facing |
| Gate 0 | 0 LLM tokens | Deterministic (§2.7) |
| Verify cycle (rebase + full suite) | 480 s | Sets merge-queue throughput (§5.5) |

**Tokens *processed*, not tokens *billed*.** Cached input is processed but billed at roughly
a tenth; §13.5 converts.

## 13.2 Per work order — this design

| Stage | Calculation | Tokens |
|---|---|---:|
| Definition (amortized) | (43k + 86k + 54k + 54k) ÷ 12 | 19.8k |
| Construction | Implementer 48k + Test Engineer 48k | 96.0k |
| Gate 0 | deterministic | 0 |
| Gate 1 | Reviewer 31.5k + Security 31.5k + Perf 20.8k | 83.8k |
| **Gate C — Commission** | 10 seats × (25k in + 50 out) | **250.5k** |
| Integration | 32k | 32.0k |
| Documentation | 28k × 0.6 | 16.8k |
| Memory Curator | pack assembly + summarisation | 5.0k |
| Release (amortized) | 27k ÷ 12 | 2.3k |
| **First-pass total** | | **506k** |

**Rework.** One replacement cycle = Refiner 29k + Gate 1 83.8k + Gate C 250.5k = **363k**.
At 70% first-pass yield the expected number of extra cycles is 0.3 + 0.09 + 0.027 ≈ 0.42.

> **≈ 660k tokens per merged work order — medium/high risk, full 10-seat bench.**

The Commission is **49% of the happy path** and 69% of every rework cycle. It is by a wide
margin the dominant line item, which is what makes the §10.7 controls load-bearing rather
than housekeeping.

### 13.2a Low-risk work order — five-seat bench

Gate C becomes 5 × 25.05k = **125.3k** instead of 250.5k.

| | Full bench (10) | Reduced bench (5) |
|---|---:|---:|
| First-pass total | 506k | **381k** |
| Rework cycle | 363k | **238k** |
| Expected extra cycles @ 70% yield | 0.42 | 0.42 |
| **Per merged work order** | **660k** | **481k** |

A low-risk work order costs **27% less**. At a 50/50 risk mix the blended figure is:

> **≈ 570k tokens per merged work order, blended** — down from 660k, a **14% reduction**
> across the whole fleet.

The five-seat bench also raises first-round acceptance on clean low-risk work from
`(1 − q)^10` to `(1 − q)^5` — 95% instead of 90% at q = 0.01 — so the yield term improves
slightly too. Not modelled above; the figures are conservative.

## 13.3 Per work order — an unoptimized baseline

Same delivered work, without the design's controls: no context budgets (whole-repo context,
~200k per call), narration in every output (~6–10k per call), rationale on every Commission
vote, no failure bundles, lower first-pass yield.

| Stage | Tokens |
|---|---:|
| Definition (amortized) | 68.7k |
| Construction | 420.0k |
| Gate 1 | 609.0k |
| Gate C — Commission | 2,004.0k |
| Integration | 203.0k |
| Documentation | 122.4k |
| Curator | 20.0k |
| Release (amortized) | 17.2k |
| **First-pass total** | **3,464k** |

Rework cycle = 2,823k; at 50% first-pass yield the expected extra cycles ≈ 0.94 → +2,653k.

> **≈ 6.1M tokens per merged work order.**

## 13.4 Comparison

| | Optimized | Unoptimized | Ratio |
|---|---:|---:|---:|
| First-pass (medium/high) | 506k | 3,464k | 6.8× |
| Per merged work order (medium/high) | 660k | 6,120k | 9.3× |
| **Per merged work order (blended, 50% low risk)** | **570k** | **6,120k** | **10.7×** |

Where the 9.3× comes from, in order:

| Lever | Share of the reduction |
|---|---|
| Context/retrieval budgets (200k → 25–40k per call) | ~70% |
| Higher first-pass yield (0.42 vs 0.94 extra cycles) | ~20% |
| Narration removed from every output | ~9% |
| Commission PASS votes carrying no rationale | ~0.5% |

### Correction to §10.2

I claimed there that PASS-without-rationale is "the largest single saving in the design."
This model contradicts that: it is worth ~3.5k per round against a ~660k total — roughly
**0.5%**, not the largest. Output tokens are dwarfed by input tokens throughout. The largest
savings are **input-side**: context budgets, then first-pass yield, then bench composition.
The PASS rule is still worth keeping (it is free, and it removes prose nobody reads), but it
should not be presented as the headline lever. §10.2 has been corrected.

## 13.5 Billed-equivalent, after caching

Cached input is billed at ~0.1× and cache writes at ~1.25×. Two large cacheable blocks:

| Block | Processed | After caching |
|---|---:|---:|
| Commission dossier (1 write + 9 reads × 25k) | 250k | ~47k |
| Stable prefix on other calls (~60% of 350k input) | 210k | ~21k |
| Everything else | ~200k | ~200k |
| **Total** | **660k** | **≈ 268k** |

> **≈ 660k processed / ≈ 270k billed-equivalent per merged work order** — caching is worth
roughly another 2.4×, on top of the 9.3×.

## 13.6 Tokens per minute

Rate is throughput × cost, and throughput is set by the merge queue, not by agent count
(§5.5). At a 480 s verify cycle a single merge queue sustains **7.5 merges/hour**.

### Steady state (merge-bound)

Using the blended 570k figure:

| Fleet | Merge queues | Merges/hr | Optimized | Unoptimized |
|---|---:|---:|---:|---:|
| Small — one repo, WIP 4 | 1 | 7.5 | **71k tok/min** | 765k tok/min |
| Mid — 3 module-sharded queues | 3 | 22.5 | **214k tok/min** | 2.3M tok/min |
| Large — 10 queues, federated | 10 | 75 | **713k tok/min** | 7.7M tok/min |

Optimized small fleet: 7.5 × 570k = 4.28M/hr = **71k tokens/minute** (≈ 29k/min
billed-equivalent). All-medium-risk work would run at 82k/min.

### Burst (agents saturated, queue backing up)

An agent run averages ~48k tokens over ~4 minutes ≈ **12k tokens/minute per active agent**.

| Concurrent agents | Burst rate |
|---:|---:|
| 4 | 48k tok/min |
| 12 | 144k tok/min |
| 30 | 360k tok/min |
| 100 | 1.2M tok/min |

**Burst above the merge-bound rate is waste, not throughput.** If agents sustain 360k/min
while the merge queue drains 82k/min worth of work, the difference is spent on patches that
go stale before they land and get re-verified or discarded. This is the quantitative form of
§5.5's rule: size the fleet backwards from the merge queue.

### Rate-limit implication

A large fleet at 825k tokens/minute needs provider capacity around **50M tokens/hour**
sustained, with headroom for burst. That is a provisioning conversation before it is an
engineering one — and the reason the token-bucket governor and priority classes in §5.4 are
not optional at that scale.

## 13.7 What moves these numbers most

| Change | Effect on per-work-order cost |
|---|---|
| ✅ Five-seat bench on low risk (**now default**) | −90k blended (−14%); −179k on a low-risk work order |
| Raising the low-risk share 50% → 70% | −36k (−6%) |
| First-pass yield 70% → 85% | −90k (−14%) |
| Larger epics (12 → 25 work orders) | −10k (−1.5%) — definition is already well amortized |
| Context budgets +50% | +150k (+23%) |
| First-pass yield 70% → 55% | +180k (+27%) |

The asymmetry is the point: **yield is worth more than frugality.** A 15-point drop in
first-pass yield costs more than a 50% increase in every context budget saves. Any
optimization that trades quality for context size is very likely a net loss — which is what
§10.8's guardrails exist to detect.

## 13.8 Confidence

Low-to-moderate, and unevenly distributed:

| Component | Confidence | Why |
|---|---|---|
| Relative ratio (~9×) | Moderate | Driven by the context-budget ratio, which is structural |
| Absolute per-work-order (660k) | Low | Depends entirely on real context-pack sizes and diff sizes |
| Commission share (~49%) | Moderate–high | Falls directly out of seat count × pack size |
| Yield assumptions (70% / 50%) | Low | Unvalidated; the single largest source of error in the model |
| Merge-bound framing | High | Structural, not empirical |

Re-derive this table from the first 50 merged work orders. The instrumentation in §6.3
already emits every field needed.
