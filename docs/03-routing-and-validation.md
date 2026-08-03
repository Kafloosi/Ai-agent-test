# 03 — Routing, Validation, and Feedback Loops

## 3.1 Routing overview

Routing answers three questions in order, and only the third may involve an LLM:

1. **Is it ready?** — deterministic topological check against `depends_on`.
2. **Who can do it?** — deterministic capability match against the worker pool.
3. **Which of several eligible paths?** — rules first; LLM router only for the residue.

```mermaid
flowchart TB
    NEW["Work order enters QUEUED"] --> READY{"All depends_on<br/>MERGED?"}
    READY -- no --> WAIT["Hold in PLANNED<br/>(re-evaluated on each merge event)"]
    READY -- yes --> PRIO["Priority score =<br/>critical-path length × risk × age"]
    PRIO --> CAP{"Capability match<br/>in routing table"}
    CAP -- "exactly one pool" --> LEASE
    CAP -- "no match" --> ESC["Escalate:<br/>missing capability"]
    CAP -- "ambiguous / multi-pool" --> LLMR["LLM router (T1)<br/>classify + justify"]
    LLMR --> LEASE["Lease worker<br/>timeout = budget.wall_clock_s"]
    LEASE --> AFF{"File-scope affinity:<br/>module lease free?"}
    AFF -- "held by another WO" --> SERIAL["Serialise behind holder<br/>(prevents merge conflicts)"]
    AFF -- free --> TIER{"Model tier selection"}
    SERIAL --> TIER
    TIER --> T1["T1 — mechanical:<br/>templated, low risk, small diff"]
    TIER --> T2["T2 — default"]
    TIER --> T3["T3 — high risk, security,<br/>ambiguity, retry ≥ 2"]
    T1 & T2 & T3 --> RUN["IN_PROGRESS with context pack"]

    style ESC fill:#a33,color:#fff
```

### Routing table (deterministic first pass)

| Work-order type | Signals | Pool | Default tier |
|---|---|---|---|
| `implement` | paths `src/api/**`, `src/services/**` | Backend | T2 |
| `implement` | paths `src/ui/**`, `*.tsx`, styling | Frontend | T2 |
| `implement` | paths `migrations/**`, schema changes | Data | T3 |
| `implement` | paths `infra/**`, `*.tf`, CI config | Infra | T3 |
| `implement` | templated CRUD, codegen, mechanical refactor | Any matching | T1 |
| `test` | any | Test Engineer | T2 |
| `fix` | attempt 1 | Refiner | T2 |
| `fix` | attempt ≥ 2 | Refiner | T3 |
| `review` | risk `low`/`medium` | Reviewer | T2 |
| `review` | touches auth/payments/PII/crypto | Reviewer + Security | T3 + human |
| `adjudicate` | any reaching Gate C | Commission (seated bench, parallel) | T2/T3 by seat |
| `integrate` | any | Integrator | T2 |
| `document` | any | Docs | T1 |

**Why rules before LLM.** Routing is executed on every state change of every work order —
it is the hottest path in the system. A deterministic table is free, reproducible, and
auditable; an LLM call here multiplies cost by traffic and introduces non-determinism into
scheduling. The LLM router handles only genuinely ambiguous cases (typically < 5% of
routing decisions) and must return a justification that is logged.

### Scheduling policy

- **Priority** = `critical_path_length × risk_weight × age_factor`. Work that unblocks the
  most downstream nodes runs first.
- **WIP limits** per pool prevent context-window thrash, rate-limit exhaustion, and a
  merge queue that grows faster than it drains.
- **Leases** with timeouts: a crashed worker's work order returns to `QUEUED` automatically
  when its lease expires — this is why workers must be stateless and idempotent.
- **Module affinity**: work orders touching the same files are serialised, not
  parallelised. Parallelising them trades a scheduling delay for a merge conflict, which is
  strictly worse.
- **Starvation guard**: `age_factor` grows over time so low-priority work eventually runs.

## 3.2 Validation: three gates

Validation is layered cheapest-and-most-objective first. Each gate's output is a signed
verdict stored against the work order.

```mermaid
flowchart LR
    P["Patch + tests"] --> G0

    subgraph G0["GATE 0 — Deterministic (no LLM)"]
        direction TB
        A0["format · lint"] --> B0["typecheck"] --> C0["build"]
        C0 --> D0["affected tests → full unit"]
        D0 --> E0["integration tests"] --> F0["coverage delta ≥ 0"]
        F0 --> H0["SAST · SCA · secrets · licence"]
        H0 --> I0["protected-path check<br/>(no test/CI edits by implementers)"]
    end

    G0 -- fail --> RC(("Root-cause<br/>classifier"))
    G0 -- pass --> G1

    subgraph G1["GATE 1 — Semantic (LLM judgement)"]
        direction TB
        A1["Reviewer: correctness,<br/>contract fidelity, edge cases"]
        B1["Security: authz, trust<br/>boundaries, data handling"]
        C1["Performance: budgets,<br/>query plans, hot paths"]
    end

    G1 -- "blocker / major" --> RC
    G1 -- pass --> G2

    subgraph G2["GATE 2 — Acceptance"]
        direction TB
        A2["Acceptance tests<br/>vs original criteria"]
        B2["e2e on integrated build"]
        C2["NFR budgets on real env"]
        D2["Human sign-off<br/>(risk class dependent)"]
    end

    G2 -- fail --> RC
    G2 -- pass --> GC["GATE C — The Commission<br/>10 seats · blind parallel vote<br/>unanimity required (§7)"]
    GC -- "≥ 1 admissible dissent<br/>whole dossier void" --> RC
    GC -- "10 × PASS" --> OK["APPROVED → merge queue"]

    style G0 fill:#1f6f43,color:#fff
    style G1 fill:#8a6d1f,color:#fff
    style G2 fill:#7a3b8f,color:#fff
    style GC fill:#4a1d5c,color:#fff
    style OK fill:#1f6f43,color:#fff
    style RC fill:#a33,color:#fff
```

### Gate design rules

| Rule | Rationale |
|---|---|
| Gate 0 runs before any LLM gate | Never spend a review call on code that does not compile |
| Gate 0 is fail-fast and cheapest-first | Median failure is caught in seconds, not minutes |
| A gate may only block on its own domain | The Reviewer restating "add tests" after coverage passed is noise that stalls loops |
| Blocking findings need reproducible evidence | Prevents unfalsifiable objections from creating infinite loops |
| Verdicts are immutable and signed | Re-verification after rebase is mandatory; stale verdicts never merge |
| Gate strictness scales with `risk_class` | A copy change and a payments change should not cost the same |

### Gate applicability by risk class

| Check | Low | Medium | High |
|---|---|---|---|
| Gate 0 full suite | ✅ | ✅ | ✅ |
| Reviewer | T1 | T2 | T3 |
| Security agent | on security-path diffs | ✅ | ✅ + human |
| Performance agent | on hot-path diffs | ✅ | ✅ + benchmarks |
| Human approval | ❌ | on escalation | ✅ mandatory |
| Commission bench | reduced (C1,C2,C3,C8,C10) | full 10 | full 10 |
| Canary window | 15 min | 1 hour | 24 hours + staged flags |

Bench composition governs which seats are *seated* for a risk class. It never softens the
decision rule: a dissent from any seated commissioner voids the entire adjudication. Full
bench is the default until per-seat precision has been measured and is stable (§7.12).

## 3.3 The feedback loop: root-cause routing

The central idea: **a failure is routed to the stage that caused it, not always back to the
coder.** Most agent pipelines send every failure back to the implementer, which is why they
loop — an implementer cannot fix an ambiguous requirement or an oversized work order, so it
churns until the budget dies.

```mermaid
flowchart TB
    FAIL["Gate failure"] --> BUNDLE["Build Failure Bundle:<br/>verdict · evidence · diff ·<br/>prior attempts · falsified hypotheses"]
    BUNDLE --> CLASS{"Root-cause classifier<br/>(rules + T2 LLM)"}

    CLASS -->|"Code defect<br/>logic, edge case, contract violation"| C1["→ Refiner<br/>same work order, attempt+1"]
    CLASS -->|"Test defect<br/>wrong assertion, bad fixture"| C2["→ Test Engineer<br/>test-change-request, reviewed"]
    CLASS -->|"Decomposition defect<br/>WO too large, missing dependency"| C3["→ Decomposer<br/>split, re-plan subtree"]
    CLASS -->|"Design defect<br/>contract wrong, infeasible"| C4["→ Architect<br/>revise contract + ADR"]
    CLASS -->|"Spec defect<br/>criteria ambiguous/contradictory"| C5["→ Analyst<br/>+ likely human question"]
    CLASS -->|"Environment / flake"| C6["→ Quarantine, re-run,<br/>ticket. Non-blocking"]
    CLASS -->|"Integration conflict"| C7["→ Integrator or Refiner<br/>depending on ambiguity"]

    C1 --> GUARD
    C2 --> GUARD
    C7 --> GUARD
    GUARD{"Loop guard"}
    GUARD -->|"attempts < max<br/>AND budget left<br/>AND progress detected"| RETRY["Re-enter IN_PROGRESS"]
    GUARD -->|"any condition fails"| ESCALATE["Escalation ladder →"]

    C3 --> REPLAN["Subtree re-planned;<br/>downstream WOs invalidated"]
    C4 --> REPLAN
    C5 --> REPLAN
    REPLAN --> HUMANQ{"Human input<br/>required?"}
    HUMANQ -- yes --> HQ["BLOCKED: ask human"]
    HUMANQ -- no --> RESCHED["Re-enter PLANNED"]

    style CLASS fill:#a33,color:#fff
    style ESCALATE fill:#8a6d1f,color:#fff
```

### Root-cause classification signals

The classifier is rules-first, LLM-assisted. Strong deterministic signals:

| Signal | Implied root cause |
|---|---|
| Same test fails identically across ≥ 2 different agents | Spec or design defect, not code |
| Failure is a contract-schema violation | Design defect if the contract is wrong; code defect if the patch deviates |
| Diff touches > N files or exceeds size budget repeatedly | Decomposition defect — work order too large |
| Test passes locally, fails in CI, non-reproducible | Environment/flake |
| Acceptance criteria contradict each other | Spec defect |
| Two work orders' tests pass alone but fail together | Integration/semantic conflict |
| Reviewer blocker cites a criterion the WO never covered | Decomposition defect (coverage gap) |
| C1 or C10 dissent (requirements unmet, evidence missing) | Spec or decomposition defect — never routed to the coder |
| C3 dissent (workaround, not a fix) | Code defect, routed to the Refiner with the dissent as an added acceptance criterion |
| C4 dissent (wrong skills or method) | Method defect — re-run with corrected tooling; recurring instances are a lesson, not a patch |

### The no-progress detector

The loop guard trips on **any** of these, before the attempt ceiling is reached:

1. **Identical failure signature** — the same test fails with the same assertion message
   two attempts in a row.
2. **Diff oscillation** — attempt *n* diff is ≥ 90% similar to attempt *n−2* (the agent is
   cycling between two wrong answers).
3. **Monotonic diff growth** — the patch grows every attempt without reducing the failure
   count (the agent is guessing, not diagnosing).
4. **Failure-count plateau** — failing checks unchanged across two attempts.
5. **Budget burn rate** — > 60% of token budget spent with < 30% of failures resolved.

Tripping the detector escalates **immediately**. Detecting non-convergence early is worth
more than any additional retry: the third identical attempt has never, in practice, been
the one that works.

### Loop budgets (defaults)

| Scope | Limit | On exhaustion |
|---|---|---|
| Attempts per work order at one gate | 3 | Escalation ladder |
| Commission adjudication rounds | 3 | Human arbitration with full dissent ledger |
| Total attempts per work order | 6 | Human escalation |
| Re-plans per subtree | 2 | Human escalation |
| Spec revisions per epic | 3 | Human product review |
| Token budget per work order | as assigned | Hard stop, escalate |
| Wall-clock per work order | as assigned | Lease expiry, requeue once, then escalate |

## 3.4 Convergence properties

The loop terminates because every cycle strictly decreases at least one bounded quantity:
attempts remaining, tokens remaining, or wall-clock remaining — and the no-progress
detector shortcuts cycles that decrease nothing else. Re-plan cycles are separately bounded
so a spec ↔ design ↔ decomposition ping-pong cannot recur indefinitely. There is no path
through the state machine that returns to the same state with all budgets unchanged.
