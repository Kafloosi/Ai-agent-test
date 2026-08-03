# 01 — Architecture

## 1.1 Topology

The system is a **hierarchical orchestrator over a shared task graph** (a blackboard),
not a free-form agent chat. Agents never talk to each other directly; they read and write
typed artifacts in a durable store and are dispatched by a control plane. This is what
makes the system debuggable, replayable, and horizontally scalable.

```mermaid
flowchart TB
    subgraph CP["Control Plane — durable, deterministic"]
        ORCH["Orchestrator<br/>scheduler · router · budget enforcer"]
        STATE[("Task Graph Store<br/>Postgres: work orders, states, attempts")]
        EVT[("Event Log<br/>append-only, replayable")]
        POL["Policy Engine<br/>budgets · risk classes · approvals"]
        ORCH <--> STATE
        ORCH --> EVT
        ORCH <--> POL
    end

    subgraph AP["Agent Plane — stateless LLM workers"]
        direction LR
        L0["L0 Definition<br/>Analyst · Architect · Decomposer"]
        L1["L1 Construction<br/>Implementers · Test Engineer"]
        L2["L2 Judgement<br/>Reviewer · Security · Performance"]
        L3["L3 Delivery<br/>Refiner · Integrator · Docs · Release"]
    end

    subgraph TP["Tool Plane — deterministic ground truth"]
        SBX["Sandbox executor<br/>container + git worktree per task"]
        CI["Build · lint · typecheck · test · coverage"]
        SEC["SAST · SCA · secret &amp; licence scan"]
        BENCH["Benchmarks · bundle &amp; query budgets"]
    end

    subgraph DP["Data Plane"]
        REPO[("Git repositories")]
        ART[("Artifact store<br/>patches · reports · logs")]
        MEM[("Memory service<br/>repo map · ADRs · embeddings")]
        LES[("Lesson store<br/>rules · checklists · evals")]
    end

    ORCH -->|"work order + context pack"| AP
    AP -->|"artifact + confidence"| ORCH
    AP --> TP
    TP -->|"machine verdict"| ORCH
    AP <--> DP
    TP <--> REPO
    SUP["Supervisor / Meta-agent"] -.->|"observe"| EVT
    SUP -.->|"tune routing, trip breakers"| ORCH
    SUP -.->|"write lessons"| LES

    style CP fill:#0d2a4a,color:#fff
    style TP fill:#123d28,color:#fff
```

### Why these four planes

| Plane | Property it guarantees | Consequence if merged with another |
|---|---|---|
| Control | Determinism, durability, replay | Agents that schedule themselves cannot be rate-limited, budgeted, or resumed after a crash |
| Agent | Statelessness, interchangeability | Stateful agents cannot be scaled out, retried, or swapped for a cheaper model |
| Tool | Objective truth | An LLM that judges its own build output will hallucinate a green build |
| Data | Shared, versioned context | Context living in conversation history is unbounded, lossy, and non-reproducible |

## 1.2 The work order — unit of scheduling

Everything the system does is a **work order**: a self-contained, independently
verifiable unit with its own acceptance criteria. Work orders are sized so a single agent
run can complete one within its context and token budget — the Decomposer enforces this.

```json
{
  "id": "WO-2418",
  "parent": "EPIC-31",
  "type": "implement",
  "title": "Add idempotency key to POST /payments",
  "risk_class": "high",
  "depends_on": ["WO-2411", "WO-2412"],
  "capabilities_required": ["backend", "postgres", "payments-domain"],
  "contracts": ["openapi://payments/v2#/paths/~1payments/post"],
  "acceptance_criteria": [
    "GIVEN a repeated request with the same Idempotency-Key WHEN POST /payments is called THEN the original response is returned and no second charge is created"
  ],
  "files_hint": ["src/payments/**", "migrations/**"],
  "budget": { "tokens": 400000, "usd": 6.0, "wall_clock_s": 900, "max_attempts": 3 },
  "state": "QUEUED",
  "attempts": 0
}
```

Full schema: [`schemas/work-order.schema.json`](../schemas/work-order.schema.json).

Three fields carry most of the system's intelligence:

- **`acceptance_criteria`** — written by the Analyst, compiled into executable tests by
  the Test Engineer, and the *only* definition of "done". They travel with the work order
  so no agent has to guess intent.
- **`risk_class`** — set by the Architect, consumed by the Policy Engine. It decides model
  tier, whether human approval is required, and how aggressive rollout may be.
- **`budget`** — hard ceilings. When exhausted, the work order escalates rather than
  continuing to spend.

## 1.3 Task state machine

Every work order moves through one state machine. The Orchestrator is the only writer;
transitions are recorded in the event log, which makes the entire run replayable.

```mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> SPECIFIED: analyst output passes spec gate
    DRAFT --> BLOCKED: open questions for human
    SPECIFIED --> PLANNED: architecture + decomposition done
    PLANNED --> QUEUED: dependencies satisfied
    QUEUED --> ASSIGNED: worker leased
    ASSIGNED --> IN_PROGRESS: agent started
    IN_PROGRESS --> SUBMITTED: patch produced
    IN_PROGRESS --> FAILED_RUN: crash / timeout / lease expiry
    FAILED_RUN --> QUEUED: transient, attempts left
    FAILED_RUN --> ESCALATED: attempts exhausted
    SUBMITTED --> VERIFYING: gates running
    VERIFYING --> REVISION: gate failure, root cause = this task
    VERIFYING --> REPLAN: root cause = decomposition/design/spec
    VERIFYING --> APPROVED: all gates pass
    REVISION --> IN_PROGRESS: refiner assigned, budget remains
    REVISION --> ESCALATED: budget or no-progress detector trips
    REPLAN --> PLANNED: upstream stage re-runs
    APPROVED --> INTEGRATING: merge queue
    INTEGRATING --> MERGED: clean integration
    INTEGRATING --> REVISION: semantic or textual conflict
    MERGED --> RELEASED: canary healthy
    MERGED --> ROLLED_BACK: canary regression
    ROLLED_BACK --> REVISION
    ESCALATED --> QUEUED: human unblocks
    ESCALATED --> ABANDONED: human cancels
    BLOCKED --> DRAFT: human answers
    RELEASED --> [*]
    ABANDONED --> [*]
```

**Invariants enforced by the Orchestrator:**

1. A work order in `QUEUED` has all `depends_on` in `MERGED` or `RELEASED`.
2. `attempts` increments on every entry to `IN_PROGRESS`; entering with
   `attempts >= max_attempts` is impossible — the transition goes to `ESCALATED`.
3. `APPROVED` requires a signed verdict from every gate applicable to the risk class.
4. Only one work order per module-lease may be in `INTEGRATING` at a time.
5. Every transition writes an event; the task graph is a projection of the event log and
   can be rebuilt from it.

## 1.4 End-to-end sequence for a single work order

```mermaid
sequenceDiagram
    autonumber
    participant O as Orchestrator
    participant M as Memory Curator
    participant I as Implementer
    participant T as Test Engineer
    participant V as Verifier — deterministic
    participant R as Reviewer / Security / Perf
    participant F as Refiner
    participant N as Integrator

    O->>M: request context pack (WO-2418)
    M-->>O: repo map slice, contracts, ADRs, lessons, ≤ budget tokens
    par Independent construction
        O->>I: work order + context pack
        I-->>O: patch + self-tests + confidence + notes
    and
        O->>T: acceptance criteria + contracts (no implementation shown)
        T-->>O: executable acceptance tests (CODEOWNER: test-engineer)
    end
    O->>V: patch ⊕ tests in isolated worktree
    V-->>O: machine verdict (build, types, tests, coverage Δ, SAST)
    alt Gate 0 fails
        O->>F: failure bundle (evidence, prior attempts, hypotheses)
        F-->>O: minimal corrective patch
        O->>V: re-verify (attempt n+1)
    else Gate 0 passes
        O->>R: diff + contracts + threat model + budgets
        R-->>O: findings with severity and file:line citations
        alt blocking findings
            O->>F: failure bundle
        else clean
            O->>N: approved patch
            N-->>O: merged (or conflict → refinement)
        end
    end
```

Note step 5–6: the Test Engineer works **from the acceptance criteria only** and is not
shown the implementation. Tests written against an implementation validate the
implementation; tests written against the spec validate the requirement. This is the
structural reason the system can trust its own green builds.

## 1.5 Context packing

Agents do not receive "the repository". The Memory Curator assembles a **context pack**
sized to a per-agent token budget, deterministically ordered so that prompt caching hits:

| Segment | Source | Cache behaviour |
|---|---|---|
| Agent system prompt + rubric | Static, versioned | Cached prefix, changes only on prompt release |
| Repo map (module tree, public signatures) | Regenerated on merge | Cached until next merge |
| Interface contracts for touched modules | Architect artifacts | Cached per epic |
| Relevant lessons for this agent role | Lesson store, top-k | Cached per lesson-store version |
| Retrieved code slices (top-k by hybrid search) | Embeddings + symbol index | Per-task |
| Work order + acceptance criteria | Task store | Per-task |
| Failure bundle (refinement runs only) | Prior attempts | Per-attempt |

Ordering matters: everything stable goes first so the cacheable prefix is as long as
possible. In steady state this is the difference between paying full price for a 200k-token
context on every attempt and paying full price once.

## 1.6 What is deliberately *not* an agent

A recurring failure mode in multi-agent designs is modelling deterministic work as an
agent. In this design the following are plain code, and must stay that way:

- **Scheduling and routing** — a rules table plus a topological ready-check. An LLM is
  consulted only for the residual ambiguous cases (§3.2).
- **Gate 0 verification** — the build/test/scan pipeline. It returns a signed verdict.
- **Merge mechanics** — rebase, conflict detection, merge queue serialisation.
- **Budget accounting, retries, backoff, leases** — durable workflow engine primitives.
- **Metrics and alerting** — standard observability, not a "monitoring agent".

The Supervisor meta-agent observes these systems and proposes changes; it does not perform
their work.
