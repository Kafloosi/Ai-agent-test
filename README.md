# Multi-Agent AI System for App Development

A reference design for a production-grade multi-agent system that takes a product
request from natural language to released software, with deterministic validation
gates, bounded feedback loops, explicit failure handling, and horizontal scalability.

## Design thesis

Four principles drive every decision in this design:

1. **Ground truth is deterministic, not conversational.** Compilers, type checkers,
   test runners and scanners decide whether work is correct. LLM agents propose;
   deterministic gates dispose. No agent may declare its own work done.
2. **Separation of duties.** The agent that writes the code never writes its own
   acceptance tests and never approves its own merge. This is the single most
   important defence against agents optimising for "looks finished".
3. **Loops must terminate.** Every refinement cycle carries an attempt counter, a
   token/cost budget, and a no-progress detector. When a loop cannot converge it
   escalates up a fixed ladder and ultimately to a human — it never spins.
4. **State lives outside the agents.** Agents are stateless workers over a durable
   task graph. That is what makes the system restartable, parallelisable, and
   horizontally scalable.

## Documentation map

| Document | Contents |
|---|---|
| [`docs/01-architecture.md`](docs/01-architecture.md) | Layered topology, master flowchart, task state machine, control plane |
| [`docs/02-agent-specs.md`](docs/02-agent-specs.md) | Every agent: role, inputs, outputs, design logic, guardrails, exit criteria |
| [`docs/03-routing-and-validation.md`](docs/03-routing-and-validation.md) | Routing rules, the three validation gates, root-cause driven feedback loops |
| [`docs/04-failure-handling.md`](docs/04-failure-handling.md) | Failure taxonomy, retry/escalation ladder, circuit breakers, rollback |
| [`docs/05-optimization-and-scaling.md`](docs/05-optimization-and-scaling.md) | Cost/latency/quality optimisation, scaling model, capacity and governance |
| [`docs/06-implementation-guide.md`](docs/06-implementation-guide.md) | Stack choices, phased rollout, metrics, anti-patterns |
| [`schemas/`](schemas/) | JSON Schemas for the message contracts between agents |

## Master flowchart

```mermaid
flowchart TB
    subgraph INTAKE["① Intake &amp; Definition"]
        REQ["Human request<br/>(idea, ticket, bug)"] --> RA["Requirements Analyst"]
        RA --> SPEC{{"Product Spec<br/>+ acceptance criteria"}}
        SPEC --> G_SPEC{"Spec gate:<br/>testable? unambiguous?"}
        G_SPEC -- "open questions" --> HUMAN1["Human clarification"]
        HUMAN1 --> RA
    end

    subgraph PLAN["② Design &amp; Decomposition"]
        G_SPEC -- pass --> ARCH["Architect"]
        ARCH --> ADR{{"Architecture:<br/>ADRs, interfaces, risks"}}
        ADR --> G_ARCH{"Design review<br/>+ risk class"}
        G_ARCH -- "high risk" --> HUMAN2["Human design approval"]
        HUMAN2 --> DEC
        G_ARCH -- "low risk" --> DEC["Task Decomposer"]
        DEC --> DAG[("Work-order DAG<br/>in task store")]
    end

    subgraph EXEC["③ Parallel Execution"]
        DAG --> ORCH["Orchestrator / Router"]
        ORCH --> POOL{"Capability match<br/>+ dependency ready?"}
        POOL --> BE["Backend Implementer"]
        POOL --> FE["Frontend Implementer"]
        POOL --> DATA["Data / Migration Implementer"]
        POOL --> INFRA["Infra Implementer"]
        DEC -.->|"acceptance criteria<br/>(independent path)"| TE["Test Engineer"]
        TE --> TESTS{{"Executable tests<br/>(locked from implementers)"}}
        BE & FE & DATA & INFRA --> PATCH{{"Patch + self-tests + notes"}}
    end

    subgraph VAL["④ Validation"]
        PATCH --> GATE0["Gate 0 — Deterministic<br/>build · lint · types · tests · SAST"]
        TESTS --> GATE0
        GATE0 -- fail --> RC
        GATE0 -- pass --> GATE1["Gate 1 — Semantic<br/>Reviewer · Security · Performance"]
        GATE1 -- "blocking findings" --> RC
        GATE1 -- pass --> GATE2["Gate 2 — Acceptance<br/>e2e · budgets · human sign-off"]
        GATE2 -- fail --> RC
    end

    subgraph LOOP["⑤ Refinement Loop"]
        RC{"Root-cause<br/>classifier"}
        RC -- "code defect" --> REF["Refiner / Repair Agent"]
        REF --> BUDGET{"attempts &lt; N<br/>and budget left<br/>and progress made?"}
        BUDGET -- yes --> PATCH
        BUDGET -- no --> ESC["Escalation ladder"]
        RC -- "test defect" --> TE
        RC -- "decomposition defect" --> DEC
        RC -- "design defect" --> ARCH
        RC -- "spec defect" --> RA
        RC -- "flake / infra" --> QUAR["Quarantine + retry"]
        QUAR --> GATE0
        ESC --> HUMAN3["Human intervention"]
        HUMAN3 --> ORCH
    end

    subgraph SHIP["⑥ Integration &amp; Release"]
        GATE2 -- pass --> INTEG["Integrator<br/>merge queue · semantic conflicts"]
        INTEG -- "conflict" --> RC
        INTEG -- merged --> DOC["Documentation Agent"]
        DOC --> REL["Release Agent<br/>flags · canary · rollback"]
        REL --> DONE(["Released"])
    end

    subgraph LEARN["⑦ Learning &amp; Supervision"]
        SUP["Supervisor / Meta-agent"]
        DONE --> SUP
        ESC --> SUP
        SUP --> LESSON[("Lesson store:<br/>rules, checklists, evals")]
        LESSON -.->|"injected into prompts"| POOL
        LESSON -.-> ARCH
        LESSON -.-> TE
        MEM[("Memory service:<br/>repo map · ADRs · retrieval")]
        MEM -.-> RA & ARCH & POOL & GATE1
    end

    style GATE0 fill:#1f6f43,color:#fff
    style GATE1 fill:#8a6d1f,color:#fff
    style GATE2 fill:#7a3b8f,color:#fff
    style RC fill:#a33,color:#fff
    style DONE fill:#1f6f43,color:#fff
```

## The one-paragraph version

A **Requirements Analyst** turns a request into a testable spec; an **Architect** turns
the spec into interface contracts and an **ADR** trail; a **Decomposer** turns that into a
DAG of small work orders, each carrying its own acceptance criteria. An **Orchestrator**
schedules ready work orders onto pools of specialised **Implementers**, while a separate
**Test Engineer** writes the acceptance tests those implementers are not allowed to edit.
Every patch runs the **Gate 0** deterministic suite, then LLM **Gate 1** review
(correctness, security, performance), then **Gate 2** acceptance and human sign-off for
risky classes. Any failure is classified by root cause and routed back to the *specific*
stage that caused it — code, tests, decomposition, design, or spec — under a hard attempt
and budget ceiling with no-progress detection. Exhausted loops climb an escalation ladder
(retry → stronger model → repair specialist → re-plan → human). Merges pass through a
serialised **merge queue** with semantic-conflict detection, ship behind feature flags with
automated rollback, and every escalation deposits a **lesson** that is injected into future
agent prompts and added to the evaluation suite.

## Quick reference: agents at a glance

| Agent | Consumes | Produces | Blocking authority |
|---|---|---|---|
| Orchestrator | Task graph, worker health | Assignments, budgets | Can halt any branch |
| Requirements Analyst | Human request, product context | Product Spec, acceptance criteria | Blocks on ambiguity |
| Architect | Product Spec | ADRs, interface contracts, risk register | Blocks on infeasibility |
| Task Decomposer | Architecture + spec | Work-order DAG | Blocks on unsizable work |
| Implementers (×N) | Work order, contracts, repo context | Patch, self-tests, notes | None |
| Test Engineer | Acceptance criteria, contracts | Executable test suites | Owns test files |
| Verifier (deterministic) | Patch + tests | Machine verdict | **Hard block** |
| Code Reviewer | Diff, contracts, lessons | Findings with severity | Blocks on ≥ major |
| Security Agent | Diff, threat model, deps | Vulnerability findings | **Hard block** on high |
| Performance Agent | Diff, benchmarks, budgets | Regression report | Blocks on budget breach |
| Refiner | Failure bundle | Minimal corrective patch | None |
| Integrator | Approved patches | Merge, conflict resolution | Blocks on conflict |
| Documentation Agent | Merged diff, ADRs | Docs, changelog | None |
| Release Agent | Merged main | Deploy, canary, rollback | **Hard block** on canary regression |
| Memory Curator | All artifacts | Repo map, retrieval, summaries | None |
| Supervisor | Metrics, escalations | Lessons, routing tuning, alerts | Can trip circuit breakers |

See [`docs/02-agent-specs.md`](docs/02-agent-specs.md) for the full specification of each.
