# 16 — Filled Specifications

Components the review (§15) found named but undefined. Each was load-bearing: something else
in the design depends on it and could not be built without it. Collected here rather than
scattered because they share one property — they were all treated as obvious and none is.

## 16.1 Risk classifier

`risk_class` drives model tier, bench composition, human approval, canary window, and evidence
depth. Only *low* was defined, by exclusion. Medium versus high was undefined (§15 F3).

Deterministic, evaluated by the Policy Engine on the Decomposer's output, **before** any
implementation begins. Highest matching rule wins.

| Class | Assigned when **any** holds |
|---|---|
| **High** | Touches auth, authz, payments, PII, crypto, or secrets · destructive or non-reversible migration · public API or contract change · infrastructure, CI, or deployment configuration · new runtime dependency · data deletion or retention change · a component with an SLO the change could breach · previously caused a production incident |
| **Medium** | Touches a measured hot path or benchmark budget · crosses a module boundary · modifies shared or framework code · adds a new external call · diff exceeds the size threshold (§16.7) · work order has ≥ 3 dependents |
| **Low** | None of the above |

Two properties make this safe to automate: it is **monotone** (adding files can only raise
the class, never lower it) and **conservative by construction** (every rule is an inclusion
into a higher class, so an unmatched novel case falls to Low only if it matched nothing —
which is why the Low definition is by exclusion and the promotion triggers in §3.2 are
evaluated independently as a second net).

**Owner:** Policy Engine. **Override:** a human may raise a class, never lower it.
**Metric:** escape rate by assigned class; any escape from Low promotes its work-order class
to full bench for 30 days (§7.12).

## 16.2 Root-cause classifier

Decides which of seven stages every failure routes to, gates instance replacement, and is
what the entire no-loop thesis rests on. It had no spec, owner, schema, or breaker (§15 F3).

**It is a first-class component**, not a footnote in §3.3: it sits in the control plane, is
rules-first with an LLM fallback, and emits a schema-validated artifact.

```
1. Deterministic signals (§3.3 table) — if any fires, that is the class. No LLM call.
2. If none fires: T2 LLM classification against the seven-value enum, with a required
   justification citing at least one artifact.
3. If the LLM returns `unknown` or cites nothing: route to `ESCALATED`, not to the coder.
```

| Property | Value |
|---|---|
| Owner | Control plane (Orchestrator subsystem) |
| Output | `root_cause_class` + `evidence[]` + `confidence`, per `failure-bundle.schema.json` |
| `unknown` handling | Escalate to human. Previously the enum allowed it with no routing |
| Precision metric | Per class: reclassifications ÷ classifications, measured when a downstream stage rejects the routing |
| Breaker | If `code_defect` share exceeds 80% over 50 classifications, trip and alert — that is the signature of the "send every failure back to the coder" anti-pattern (§6.4) |

The breaker matters more than the classifier's accuracy. A miscalibrated classifier degrades
silently into exactly the anti-pattern the design was built to avoid, and nothing else in the
system would notice.

## 16.3 Spec gate, design gate, Gate 2

Three gates referenced as state transitions and agent exit criteria, specified nowhere — and
unrepresentable in the verdict enum (§15 F11, F12).

| Gate | Owner | Checks | Emits |
|---|---|---|---|
| **Spec gate** | Policy Engine (mechanical) | Every criterion parses as Given/When/Then · `falsifiable: true` on all · `open_questions` empty · non-goals present · every criterion has ≥ 1 testable predicate | `Verdict{gate: "spec"}` |
| **Design gate** | Policy Engine + human on high risk | Contracts machine-validate (OpenAPI/SDL/protobuf parse) · every high-risk area has a stated mitigation · every acceptance criterion maps to ≥ 1 component · Design Agent state inventory covers every criterion | `Verdict{gate: "design"}` |
| **Gate 2** | Release Agent | Acceptance tests green against a provisioned environment · e2e suite green · NFR budgets measured and within bounds · human sign-off where the risk class requires it | `Verdict{gate: "gate2"}` |

The verdict enum becomes `spec | design | gate0 | gate2 | gateC`.

**Human sign-off has a bound.** `blocked_pending_human` carries a `deadline`. On expiry the
work order escalates to the approver's escalation contact; it never merges by default and
never waits forever.

## 16.4 `contract_conformance`

The Gate 0 check that justifies dropping C5 from the reduced bench existed only as an enum
string (§15 F14).

Runs in Gate 0 after typecheck, before tests:

1. For every contract in `work_order.contracts`, resolve the frozen artifact at the pinned
   version.
2. Extract the implemented surface from the diff — route signatures, exported types, event
   payloads, schema definitions.
3. Compare structurally: added and removed operations, parameter and field names, types,
   required/optional, status codes, enum members.
4. **Fail on any difference not declared in `patch.contract_deviations`.** A declared
   deviation fails differently — it routes to the Architect rather than the implementer.

Nothing here is judgement, which is precisely why it can back a dropped seat.

## 16.5 The golden task set

The sole promotion gate for every lesson, skill, and ordinary amendment — undefined (§15 F15).

| Property | Value |
|---|---|
| Composition | ≥ 30 completed work orders, stratified across type, risk class, and the four defect classes that actually recur |
| Curation | Supervisor proposes additions from merged history; a human ratifies. A task enters only after being merged and surviving 30 days without an escape |
| Refresh | 10% replaced quarterly; retired tasks are archived, not deleted |
| Run protocol | 3 runs per task per variant, pinned model version, temperature-equivalent settings fixed |
| **Improvement criterion** | Mean first-pass yield improves by **≥ 3 percentage points** with **no individual task regressing by more than 1 point**, across all 3 runs |
| Guard | A change that improves the mean by regressing one task badly is rejected — that is overfitting to the aggregate |

## 16.6 Estimating *q*

The per-seat false-dissent rate gates phase 7 and is called the Commission's vital sign. No
estimator existed; the one proxy covered only voluntarily disputed dissents (§15 F16).

*q* = P(seat dissents | work is genuinely acceptable). Three estimators, used together:

| Estimator | Method | Bias |
|---|---|---|
| **Clean-canary** | Inject known-good dossiers (previously merged, 90 days incident-free, lightly re-worded) into the adjudication stream at ~2% rate. Any dissent on one is a false dissent | Cleanest signal; limited volume |
| **Overturn** | Upheld ÷ (upheld + overturned) from disputes | Undercounts — most false dissents are silently complied with |
| **Cleared-without-change** | Dissents whose clearance came from a diff that did not touch the cited `file:line` | Catches the silent-compliance cases the overturn rate misses |

Report *q* as the clean-canary rate with the other two as bounds. Phase 7's ship criterion
(*q* < 0.01) is measured on the clean-canary estimator specifically.

## 16.7 Unvalued thresholds

Presented as settled; all were guesses (§15 F17). Now stated with basis and owner, and marked
as placeholders where they are.

| Threshold | Value | Basis | Owner |
|---|---|---|---|
| Work-order size threshold | 400 changed lines or 8 files | Placeholder — recalibrate from the p75 of merged first-pass work orders | Decomposer |
| Diff-oscillation similarity | ≥ 90% | Computed on the **normalised AST diff**, not text — whitespace and rename churn otherwise mask a repeat | Loop guard |
| Budget-burn trip | > 60% budget with < 30% of failing checks resolved | Placeholder | Loop guard |
| Implementer confidence θ | 0.7 | **Uncalibrated and self-reported.** Until calibration exists, θ escalates tier but never *avoids* a gate — a self-report may not shorten the verification path | Orchestrator |
| Complexity classifier | Inputs: files touched, cyclomatic delta, contracts touched, novelty (no similar merged work order in 90 days). T1 < 2 inputs, T2 2–3, T3 ≥ 4 or any high-risk marker | Placeholder | Orchestrator |

Every value above is a starting point to be replaced from the first 50 merged work orders —
the same caveat §5.5 and §10.5 already carry, now applied consistently.

## 16.8 Policy Engine

Held production-approval authority with three mentions and no spec (§15 F18).

| Field | Specification |
|---|---|
| **Role** | Evaluate deterministic policy over work orders and transitions. The only component permitted to assign `risk_class` and to grant or withhold an approval requirement |
| **Inputs** | Work order, diff manifest, risk rules (§16.1), approval matrix, budget ledger |
| **Outputs** | `risk_class`, required-gate set, required-approval set, budget authorisation, spec- and design-gate verdicts |
| **Implementation** | Plain code — rules as versioned configuration, evaluated deterministically. **Not an LLM** |
| **Guardrails** | Cannot be modified by an amendment below constitutional class; every evaluation is logged with the rule version that produced it |
| **Failure mode** | Fail closed. An unevaluable policy blocks the transition rather than allowing it |

## 16.9 What remains open

These are recorded, not fixed. They are architecture-level and change the shape of the
system rather than filling a hole in it.

| # | Open finding | Why deferred |
|---|---|---|
| F2 | Clerk and Chair perform judgement while classified as code | Needs a decision: constrain their duties to typed fields, or reclassify them as judged components and accept the weakening of §7.13. Both are real designs; the choice is yours |
| F19 | Control plane absent from the rollout plan | The phase plan needs rewriting around platform components, not agents |
| F20 | Replay claimed but not achievable | Requires a versioned run record spanning prompt, model, lesson, skill, repo-map and pack versions |
| F21 | No tenancy model; federation contradicts §9.8 | Requires a tenancy boundary per store |
| F22 | No cold-start or brownfield path | The Decomposer, Commission, and eval harness all require history that a new installation does not have |
| — | C5's recommended cuts (genesis, tenure, web acquisition) | Each implements an explicit user request; removing them is a product decision, not a defect fix |
| — | C10's jurisdiction | The Evidence seat never reported. Arithmetic, cross-references, and schema/doc drift remain unverified |
