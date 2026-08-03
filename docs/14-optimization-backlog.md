# 14 — Optimization Backlog

What is left after the §10 controls and the five-seat bench, ranked by modelled impact
against the 570k blended baseline in §13.

Blended cost decomposition, which is where the candidates come from:

| Component | Blended | Share |
|---|---:|---:|
| Gate C — Commission | 187.9k | 33% |
| Rework cycles | 126.3k | 22% |
| Construction | 96.0k | 17% |
| Gate 1 — semantic review | 83.8k | 15% |
| Integration | 32.0k | 6% |
| Definition (amortized) | 19.8k | 3% |
| Documentation | 16.8k | 3% |
| Curator + Release | 7.3k | 1% |

## 14.1 Implemented

### O1 — Collapse Gate 1 into Gate C · −119k (−21%)

**Gate 1 and Gate C were judging the same things with different agents.** The Code Reviewer
and seat C2 both assess correctness. The Security Agent and C6 both assess security. The
Performance Agent and C7 both assess budgets. Three jurisdictions, adjudicated twice, at full
price each time.

This is duplication I introduced by designing the two gates independently; the cost model made
it visible. The fix is to make the Gate 1 agents *be* the seats:

| Was | Now |
|---|---|
| Code Reviewer (Gate 1) + C2 Correctness | **C2 Correctness** — one agent, one pass |
| Security Agent (Gate 1) + C6 Security | **C6 Security & Privacy** |
| Performance Agent (Gate 1) + C7 Performance | **C7 Performance & Cost** |

The gate chain becomes **Gate 0 → Gate 2 → Gate C**. The "cheap filter before the expensive
gate" argument that justified Gate 1 applies to Gate 0 — which is deterministic and free — not
to a second tier of LLM judgement costing 84k a pass.

Two consequences handled:

- Low-risk work no longer seats C6 or C7, so the promotion triggers in §3.2 now include
  **any diff on a measured hot path or with a benchmark budget attached** (previously the
  Performance Agent ran conditionally at Gate 1). Security-relevant paths already forced
  promotion.
- The seats keep their §7.4 admissibility rules, which are stricter than the old Gate 1 —
  jurisdiction, evidence, concrete failure, and a clearance condition. Review quality goes up,
  not down.

### O2 — C10 becomes a deterministic validation · −25k (−4%)

C10 (Evidence & Process) asks whether every claim in the dossier is backed by an artifact.
Almost all of that is **mechanically checkable and does not need a model**:

| Check | Mechanical? |
|---|---|
| Every applicable verdict present | ✅ |
| Verdict hashes match the patch and base commit | ✅ |
| Base commit is current (no stale verdict) | ✅ |
| Attempt history contiguous, no gaps | ✅ |
| Every acceptance criterion mapped to a test id | ✅ |
| Every claim in the patch's `notes` cites an artifact | ✅ |
| Whether the evidence is *sufficient* for this risk class | ❌ judgement |

The Clerk now runs the mechanical checks **before** sealing the dossier — a dossier that fails
them never reaches the bench at all, which also removes a whole round-trip on incomplete
records. An LLM C10 seat is retained **only for high-risk work**, judging sufficiency.

This is the best class of optimization available anywhere in the design: moving a judgement
into a deterministic check makes it free, faster, and more reliable at once.

### O6 — Test Engineer context pack 40k → 20k · −20k (−4%)

The Test Engineer works from acceptance criteria and contracts and is explicitly forbidden
from reading the implementation (§2.6). Its 40k pack was sized like an implementer's. Half of
it was repo context it may not use.

### Combined effect

| | Before | After |
|---|---:|---:|
| First-pass (blended) | 443.6k | **314.8k** |
| Rework cycle (blended) | 300.7k | **191.9k** |
| **Per merged work order** | **570k** | **≈ 395k** |
| Ratio vs unoptimized baseline | 10.7× | **15.5×** |
| Single merge queue | 71k tok/min | **49k tok/min** |

## 14.2 Proposed — needs measurement first

These are real savings with real risk. None should ship before the metrics in §10.8 exist,
because each trades context or coverage for cost and the model in §13.7 says that trade is
usually a loss.

| # | Optimization | Modelled saving | Risk |
|---|---|---:|---|
| **O3** | Diff-scoped packs for judgement seats — 25k → 15k. Seats need the diff, its neighbourhood, contracts, criteria and precedent, not the full repo map | −65k (−16%) | Starves reviewers of context; §10.8 lists this as a top over-optimization signal. Ship only if escape rate holds |
| **O7** | Two-stage adjudication — seat C3 and C8 first (the two cheapest-to-detect rejection classes); if either dissents, never run the rest | −22k on rejected work | Adds one round of latency to every adjudication, including the happy path |
| **O8** | Tier C3 down to T1 — completion checking is largely pattern recognition once the marker scan (below) is deterministic | −8k | Small; measure C3's dissent precision at T1 first |
| **O9** | Batch trivial work orders into one dossier | −100k+ per batch | Breaks unanimity granularity — one dissent voids several unrelated changes |

**O3 is the largest single remaining item and the one most likely to backfire.** Judgement
agents are the last line before merge; the §13.7 asymmetry (yield is worth more than frugality)
applies to them most strongly.

## 14.3 Infrastructure — savings not visible in the per-work-order model

| # | Optimization | Effect |
|---|---|---|
| **O4** | **Verdict reuse on semantically identical rebase.** If a merge-queue rebase produces a normalised diff with an unchanged hash and Gate 0 passes on the new base, reuse the Gate C verdict instead of re-adjudicating | Removes most re-adjudication caused by queue contention — the busier the fleet, the larger this gets |
| **O5** | **Dossier stable-prefix caching across rounds.** Re-seal as a stable prefix (work order, criteria, contracts, repo slice) plus a volatile suffix (current diff, dissent ledger). Only the suffix changes between rounds | ~60% off the billed cost of every re-adjudication |
| **O10** | **Marker scan into Gate 0.** `TODO`, `FIXME`, `skip`/`xfail`, `@ignore`, empty catch blocks, blanket suppressions, stub returns — all detectable deterministically. Feeds C3 rather than replacing it | Free detection; raises C3's first-pass signal and enables O8 |
| **O11** | **Fetch-cache and result-cache hit-rate targets.** Both already specified (§12.2, §5.2) but neither has a target. Set one and alert on regression | Cache misses are silent cost |

## 14.4 Rejected — attractive and wrong

| Candidate | Why not |
|---|---|
| Cut reasoning depth across the board | §13.7: a 15-point yield drop costs more than a 50% context increase saves. Depth is not a cost lever, it is a yield lever |
| Drop evidence fields (`root_cause`, `failure_scenario`, `remediation_condition`) | ~0.5% saving, and each one prevents a full replacement cycle. Net loss by an order of magnitude (§10.1) |
| Reduce the bench below five | The five remaining seats are exactly the jurisdictions with no deterministic backstop (§7.12). Below five, defects have no path to detection at all |
| Skip Gate 0 on small diffs | Gate 0 is free and is the ground truth the entire cascade depends on |
| Single-seat "chief judge" instead of a bench | Correlated failure: one judge's blind spot becomes the system's blind spot, with no independent signal |
| Compress the work journal | It is already ≤2k and is what makes replacement cheap. Compressing the thing that prevents re-derivation is a false economy |

## 14.5 The pattern worth generalising

Every large win so far has the same shape: **move work from judgement to determinism, or
remove it entirely.**

| Win | Shape |
|---|---|
| Gate 0 before every LLM gate | Determinism first |
| C10 → Clerk validation (O2) | Judgement → determinism |
| Marker scan → Gate 0 (O10) | Judgement → determinism |
| Collapse Gate 1 into Gate C (O1) | Remove duplication |
| Result and fetch caching | Remove the call |
| Five-seat bench | Remove work with a deterministic backstop |

Compression — shorter prompts, terser outputs, smaller packs — is the *smallest* category of
saving and the one most likely to cost more than it saves. Look for duplicated judgement and
for judgement that has a mechanical answer before looking for anything to compress.
