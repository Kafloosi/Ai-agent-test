# 15 — Commission Review of the Design

A five-seat bench was convened to adjudicate this design against its own admissibility rules
(§7.4). Four seats reported; **C10 (Evidence & Process) failed to complete** — so arithmetic,
cross-reference integrity, and schema/doc drift are **unverified**. Under §7.5 an incomplete
record is itself a C10 dissent, which is the position this review is in.

**VERDICT: REJECTED — 4 of 4 reporting seats dissented.**

| Seat | Verdict | Admissible dissents |
|---|---|---:|
| C1 Requirements | DISSENT | 4 |
| C2 Correctness | DISSENT | 5 |
| C3 Completion | DISSENT | 10 + tail |
| C5 Architecture | DISSENT | 6 |
| C10 Evidence | **did not report** | — |

## 15.1 Convergent findings

Independently raised by more than one seat. Strongest signal in the set.

### F1 — The low-risk bench is four seats, not five · C1, C2

O2 (§14.1) demoted C10 to a mechanical Clerk check below high risk. The low-risk bench is
therefore C1, C2, C3, C8 — **four judging seats** — while §14.4 simultaneously rejects
"reduce the bench below five" as unsafe. The reduction was never stated.

*Clears when:* §7.12 states the seated judging count explicitly and either restores an LLM
C10 to the reduced bench or names a fifth seat, and §14.4 is reconciled with O2.

### F2 — The Clerk and Chair perform judgement while classified as code · C3, C5

§7.13's anti-gaming argument rests entirely on the Clerk being deterministic. But the Clerk
must verify that prose claims cite artifacts and must distil a rejected run into a
≤2,000-token pack with `falsified_hypotheses` and `next_step` — summarisation. The Chair must
rule that a dissent "restates a passing check" and that two natural-language
`remediation_condition` strings cannot both hold. Built as code these are brittle string
matchers; built as LLMs they void the structural protection.

*Clears when:* every Clerk and Chair responsibility is either mechanically decidable from
typed fields, or reclassified as a judged component with its own guardrails and error budget.

### F3 — The risk classifier is undefined, and everything rests on it · C3, C5

`risk_class` drives model tier, bench size, human approval, canary window, and evidence
depth. Only *low* is defined, by exclusion (§3.2). **Nothing distinguishes medium from high.**
§7.12 concedes the five-seat bench is "safe only insofar as the risk classifier is
conservative" and never specifies the classifier. The root-cause classifier (§3.3) has the
same problem at higher stakes: it gates instance replacement and the entire no-loop thesis,
yet has no spec, no owner, no schema, no precision metric, and no breaker.

*Clears when:* both classifiers are first-class components with stated rules, owners,
schemas, measured precision per class, and breakers.

### F4 — Tenure and succession cost more than the body they maintain · C2, C5

Every accepted work order credits a passing to all ten seats, so the bench turns over roughly
every three merges — several successions per merge, each costing a 12-case bench exam, all of
it in front of the merge queue that §5.5 names as the binding constraint. Neither §13 nor §14
models it. Worse, the stagger offsets only `passings`, not `adjudications`; at the
12-adjudication backstop **all ten seats can enter SUCCESSION in the same round**, and the
exam-retry loop has no ceiling. The bench can empty with no state transition for "no bench",
leaving work orders in ADJUDICATING indefinitely.

*Clears when:* tenure is defined against a rate that does not scale with throughput,
concurrent succession is capped, the exam-retry loop has a ceiling that escalates to a human,
and succession appears as a line item in the cost model.

## 15.2 Correctness defects

| # | Defect | Seat |
|---|---|---|
| **F5** | **No `ADJUDICATING → REPLAN` edge.** §3.3 and §10.4 state a C1 or C10 dissent must route upstream and never to the coder — but the state machine only offers `REVISION` (defined as "refiner assigned") or `ESCALATED`. A requirements dissent is *forced* into the action the design calls impossible, then burns rounds to an identical failure signature. | C2 |
| **F6** | **Gate C verdicts merge stale.** §3.2 says re-verification after rebase is mandatory and stale verdicts never merge. The Integrator re-runs Gate 0 only, and there is no `INTEGRATING → ADJUDICATING` edge. A semantic change in a merged dependency invalidates C6's security judgement, and the patch merges carrying it. | C2 |
| **F7** | **Rollback does not cascade.** `MERGED → ROLLED_BACK` leaves dependents in flight against reverted code, violating invariant 1 with no detector. §4.2④ cascades on contract *change*, not on revert. | C2 |
| **F8** | **Genesis probation is an unbounded outer loop.** §11.9 caps tournaments in flight and per-tournament cost, but nothing caps genesis *rounds per capability*. Fail probation → revise charter → new tournament → repeat, indefinitely. Every other loop in the design has a ceiling. | C2 |
| **F9** | **The amendment floor is unreachable.** §11.8 requires ≥6 absolute approvals; with the five-seat bench now default, no ordinary amendment can ever pass. A later change disabled an earlier requirement. | C1 |
| **F10** | **C4's mandate is unseated by default.** "Used wrong skills" is one of the four concerns the Commission was created to catch; C4 is dropped on ~half of all work orders, and its claimed backstop is a log, not a check that can fail anything. | C1 |

## 15.3 Specification gaps that block building

| # | Gap | Why it blocks |
|---|---|---|
| **F11** | Spec gate and design gate | State-machine transitions and three agents' exit criteria, specified nowhere. The `verdict` enum (`gate0/gate2/gateC`) cannot even represent them |
| **F12** | Gate 2 | Four words in a diagram. No owner, no check sequence, no timeout on `blocked_pending_human` |
| **F13** | `ArchitectureDecision` and `DesignDecision` schemas | Absent, while sixteen lesser artifacts have them. These are the frozen contracts all parallel implementation builds against |
| **F14** | `contract_conformance` | Exists only as an enum string. It is the backstop justifying C5's removal from the low-risk bench |
| **F15** | The golden task set | Sole promotion gate for every lesson, skill and amendment. No curator, no size, no definition of "measurably better" |
| **F16** | *q*, the false-dissent rate | Gates phase 7 and is called the Commission's vital sign. No estimator exists; the one defined proxy covers only voluntarily disputed dissents |
| **F17** | Size threshold, similarity metric, confidence threshold θ, complexity classifier cutoffs | Gate decomposition, the no-progress detector, and tier escalation. Presented as settled; unvalued |
| **F18** | Policy Engine | Holds production-approval authority with three mentions and no spec |

## 15.4 Architecture

| # | Finding |
|---|---|
| **F19** | **The control plane is never scheduled.** §6.2 phases enumerate agents; the orchestrator, task store, event log, policy engine, lease manager, budget ledger, sandbox, retrieval index and registries are all prerequisite to phase 1 and appear in no phase. Phase 1 also cannot produce a schema-valid `WorkOrder` — `acceptance_criteria` is required and authored in phase 3 |
| **F20** | **Replay is claimed but not achievable.** Only state transitions replay. Prompt, model, lesson, skill, repo-map and precedent-pack versions are not on the run record, and rejected contexts are discarded by design. "Why did WO-2418 fail" is answerable as a sequence of verdicts, not as a cause |
| **F21** | **No tenancy model.** §5.4's federation shares exactly the stores §9.8 forbids sharing, and the poisoning countermeasure depends on the forbidden case |
| **F22** | **Cold start / brownfield onboarding is absent.** The Decomposer needs historical sizing data, the Commission needs a precedent corpus and 12 graded cases per seat, the eval harness needs a golden set, implementers need frozen contracts. Nothing bootstraps any of it |

Also absent entirely: disaster recovery, model-deprecation handling for per-seat calibration,
tests for the system's own deterministic components, audit/compliance mode, data residency,
cost attribution, and any human interface for the six mandatory checkpoints and the DLQ.

## 15.5 Recommended cuts

C5's judgement, endorsed here: the system's own complexity is now a larger risk than the
problem it solves.

1. **Genesis tournaments and self-amendment (§11)** — two governance subsystems for a fleet
   that has not delivered a single work order. The Supervisor's eval-gated lesson path
   already covers the real cases.
2. **Tenure, succession and bench exams (§7.8–7.11)** — keep the ten seats and the Precedent
   Pack as an accumulating rejected-pattern catalogue; drop rotation. It costs more than the
   bench and its benefit is unmeasured (F4).
3. **Web skill acquisition (§12)** — highest attack surface, lowest marginal value while the
   skill registry is empty.

## 15.6 What the bench affirmed

Consistent across all four seats:

- The four-plane split with a deterministic control plane, and §1.6's "what is deliberately
  not an agent", called the strongest page in the set.
- Determinism-first gate ordering with signed, base-pinned verdicts.
- Test immutability enforced by CODEOWNERS plus a Gate 0 check — a mechanical guarantee, not
  a prompt.
- Blind parallel voting plus a deterministic admissibility filter, which is what makes
  unanimity survivable and per-seat precision measurable.
- Mandatory remediation conditions turning dissents into acceptance criteria — the actual
  reason Gate C terminates.
- The no-progress detector applying across instances with budget inherited, closing the
  respawn loophole.
- Sizing the fleet backwards from the merge queue (§5.5).
- §14.5's "move judgement to determinism or remove it" as the right evolutionary pressure.

## 15.7 Status

Nothing in this document has been remediated. Under §7.5 the rejection stands until each
clearance condition is met, and the review itself is incomplete until C10 reports.
