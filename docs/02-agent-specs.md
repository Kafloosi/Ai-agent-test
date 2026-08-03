# 02 — Agent Specifications

Every agent is specified with the same eight fields. Anything not on this list is not a
contract and agents must not depend on it.

- **Role** — the single responsibility. If an agent needs two sentences, split it.
- **Inputs** — exactly what the context pack contains. Agents cannot fetch beyond this.
- **Outputs** — a schema-validated artifact. Free text goes in a `notes` field, never in
  a field another agent parses.
- **Design logic** — the decision procedure the prompt encodes.
- **Model tier** — default routing; the escalation ladder may raise it.
- **Tools** — the only side effects it may cause.
- **Guardrails** — what it may not do, enforced outside the prompt where possible.
- **Exit criteria** — when its output is accepted, and what makes it fail.

## 2.0 Output discipline — applies to every agent

Every agent emits **only its artifact**. No preamble, no commentary between tool calls, no
summary afterwards, no restating the task, no explaining choices the artifact already shows.
Outputs are schema-constrained and capped; free text appears only where another stage
consumes it — `root_cause`, `failure_scenario`, `remediation_condition`,
`contract_deviations`, `falsified_hypotheses`. Those are evidence, not narration, and they
stay: a dissent without a clearance condition never converges, and a failure bundle without
falsified hypotheses causes the next attempt to re-test what the last one disproved.

Per-role caps, reasoning depth, and the enforcement rules are in
[`10-token-efficiency.md`](10-token-efficiency.md).

Each agent's **skills** — packaged procedural knowledge loaded by progressive disclosure —
and its exact tool and connector surface are specified separately in
[`08-agent-skills-and-tools.md`](08-agent-skills-and-tools.md), and what it remembers between
runs in [`09-memory-and-learning.md`](09-memory-and-learning.md). Both are part of every
agent's contract; they live apart because they version on their own cadence.

Model tiers are logical: **T1** = fast/cheap for mechanical work, **T2** = balanced
default, **T3** = strongest for design, ambiguity, and hard debugging.

---

## 2.1 Orchestrator (Conductor)

| Field | Specification |
|---|---|
| **Role** | Own the task graph: schedule ready work, enforce budgets and policy, route outcomes. The only writer of task state. |
| **Inputs** | Task graph, worker pool health, gate verdicts, policy config, cost ledger |
| **Outputs** | Assignments with context packs, state transitions, escalations, event-log entries |
| **Design logic** | Deterministic: topological ready-check → priority sort (critical path length × risk × age) → capability match → lease with timeout. Consults the LLM router only when capability match is ambiguous (§3.2). |
| **Model tier** | None (code) + T1 for ambiguous routing only |
| **Tools** | Task store, queue, lease manager, budget ledger |
| **Guardrails** | Never writes code, never overrides a gate verdict, never exceeds a work order's budget, cannot approve its own escalations |
| **Exit criteria** | N/A — long-running service |

The critical design choice: the Orchestrator is **not** an LLM agent. Non-determinism in
the scheduler makes the whole system unreproducible and makes cost unbounded.

---

## 2.2 Requirements Analyst

| Field | Specification |
|---|---|
| **Role** | Convert a human request into a testable specification, and surface what cannot be decided without a human. |
| **Inputs** | Raw request, product context (existing features, personas, constraints), glossary, related prior specs |
| **Outputs** | `ProductSpec`: user stories, **acceptance criteria in Given/When/Then**, explicit non-goals, edge cases, data/privacy implications, `open_questions[]`, `assumptions[]` |
| **Design logic** | 1) Extract intent and actors. 2) Draft criteria. 3) **Self-test each criterion for falsifiability** — if no automated test could fail it, rewrite it. 4) Any requirement that cannot be made falsifiable without a human decision becomes an `open_question`, never a guess. 5) Mark everything not stated as a non-goal to prevent scope drift. |
| **Model tier** | T3 (ambiguity resolution is the highest-leverage step in the pipeline) |
| **Tools** | Retrieval over product docs, ticket system (read) |
| **Guardrails** | May not invent business rules; may not specify implementation; must separate `assumptions` (proceeding) from `open_questions` (blocking) |
| **Exit criteria** | Every criterion is falsifiable; no blocking open question remains unanswered; spec gate passes |

**Failure mode this prevents:** the single most expensive multi-agent failure is a fleet of
agents building the wrong thing very efficiently. Blocking on ambiguity here costs minutes;
discovering it at Gate 2 costs the whole DAG.

---

## 2.3 Architect

| Field | Specification |
|---|---|
| **Role** | Decide structure: components, interfaces, data model, and the risks that follow. |
| **Inputs** | ProductSpec, repo map, existing ADRs, NFRs (latency, cost, compliance), tech constraints |
| **Outputs** | `ArchitectureDecision`: component map, **interface contracts** (OpenAPI / GraphQL SDL / protobuf / typed schemas), data-model deltas + migration strategy, ADRs with rejected alternatives, risk register with `risk_class` per area, NFR budgets |
| **Design logic** | Generate 2–3 candidate designs → score against explicit criteria (fit to existing architecture, blast radius, reversibility, operational cost, migration difficulty) → select and record *why the others lost*. Prefer reversible decisions; flag irreversible ones for human approval. Contracts are produced **before** any implementation so parallel work orders can be built against them. |
| **Model tier** | T3, with self-consistency (N candidates → judge) on high-risk areas |
| **Tools** | Repo analysis, dependency graph, retrieval over ADRs |
| **Guardrails** | May not write feature code; may not introduce a new runtime dependency without a licence/SCA check and an ADR; irreversible or security-relevant decisions require human sign-off |
| **Exit criteria** | Contracts are machine-validatable; every high-risk area has a mitigation; design gate passes |

Interface-first is what makes parallel implementation safe. Without frozen contracts,
parallel agents produce work that only fails to compose at integration time — the most
expensive place to discover it.

---

## 2.3.1 Design Agent (UX & Interface)

A **peer of the Architect**, not a subordinate: the Architect owns system structure, the
Design Agent owns interface and experience, and both run in parallel against the same spec.
Full specification, including its skill set, in
[`08-agent-skills-and-tools.md` §8.3](08-agent-skills-and-tools.md#83-design-agent-ux--interface).

| Field | Specification |
|---|---|
| **Role** | Turn the spec into interaction and interface decisions implementers can build against without inventing UX. |
| **Inputs** | ProductSpec + acceptance criteria, design system and tokens, component inventory, existing screens, accessibility standard |
| **Outputs** | `DesignDecision`: user flows, **state inventory** (empty / loading / partial / error / success / offline), component selection, interaction and validation behaviour, user-facing copy, accessibility annotations, responsive behaviour |
| **Design logic** | Enumerate every state a screen can be in *before* styling any of them — unhandled empty and error states are the usual gap between "the feature works" and "the feature is usable". Reuse existing components before proposing new ones. On open-ended briefs, propose 3–4 distinct directions and let a human choose rather than silently committing to one house style. |
| **Model tier** | T3 |
| **Tools** | Design-token store, component inventory, browser automation, contrast and accessibility checkers, screenshot diffing |
| **Guardrails** | May not write production code; no new component without a design-system entry; every interactive element carries accessibility annotations; user-facing copy is part of the deliverable, not left to the implementer |
| **Exit criteria** | Every acceptance criterion has a defined interface state; all states enumerated; accessibility annotations complete |

Its outputs are contracts in the same sense as §2.3's: frozen before parallel implementation
starts, so the Frontend Implementer builds against a decision instead of guessing and the
Test Engineer can assert against defined states.

## 2.4 Task Decomposer (Planner)

| Field | Specification |
|---|---|
| **Role** | Turn architecture into a DAG of independently verifiable, correctly sized work orders. |
| **Inputs** | ProductSpec, ArchitectureDecision + contracts, repo map, historical sizing data |
| **Outputs** | Work-order DAG: each node with type, capabilities, dependencies, acceptance criteria subset, file hints, budget, risk class |
| **Design logic** | Split along **contract boundaries**, not along "layers of the same feature", so nodes share as few files as possible. Each node must satisfy: (a) one agent can complete it within budget, (b) it has its own falsifiable acceptance criteria, (c) it is independently mergeable behind a flag. Estimate size from historical data on similar nodes; anything above the size threshold is split again. Maximise DAG width on the critical path; order to unblock the widest fan-out first. Explicitly assign shared-file risk and sequence conflicting nodes rather than parallelising them. |
| **Model tier** | T3 |
| **Tools** | Repo map, dependency graph, historical metrics store |
| **Guardrails** | No node may exceed the size threshold; no cycles; every acceptance criterion in the spec maps to ≥ 1 node (coverage check is deterministic); nodes touching the same files get an ordering edge, not parallel scheduling |
| **Exit criteria** | DAG is acyclic, fully covers the spec, and every node is within budget |

---

## 2.5 Implementer agents (pooled, specialised)

Instances: **Backend**, **Frontend**, **Data/Migration**, **Infra/Platform**. Same
contract, different tool access, retrieval corpus, and checklist.

| Field | Specification |
|---|---|
| **Role** | Produce a minimal, correct patch that satisfies one work order against frozen contracts. |
| **Inputs** | Work order + acceptance criteria, interface contracts, context pack (retrieved code, repo map, conventions), role lessons. **Not** the acceptance test source. |
| **Outputs** | `Patch`: unified diff, self-authored unit tests, `notes` (approach, trade-offs), `confidence` 0–1, `assumptions[]`, `contract_deviations[]` |
| **Design logic** | Read contracts → locate insertion points via symbol search → write tests for its own logic → implement → run local checks → self-review against the role checklist → emit confidence. Confidence below threshold routes the patch to a stronger tier or attaches a targeted question rather than guessing. Follows existing conventions in touched files over global preference. |
| **Model tier** | T1 for mechanical/templated nodes, T2 default, T3 on retry escalation |
| **Tools** | Sandboxed repo worktree, build/test runner, symbol search, docs retrieval |
| **Guardrails** | **May not modify acceptance tests or CI configuration** (CODEOWNERS + gate check); may not change frozen contracts — deviations are reported, not taken; may not add dependencies without SCA approval; may not touch files outside its work order's declared scope without raising a scope-expansion request |
| **Exit criteria** | Local build + own tests pass; diff confined to declared scope; patch submitted |

**The test-immutability guardrail is load-bearing.** An implementer that can edit the
acceptance suite will eventually make a red suite green by weakening it, and the system
will report success. Enforce it mechanically — CODEOWNERS on test paths plus a Gate 0 check
that rejects any diff touching protected test files from a non-test-engineer author.

---

## 2.6 Test Engineer

| Field | Specification |
|---|---|
| **Role** | Compile acceptance criteria into executable tests, independently of the implementation. |
| **Inputs** | Acceptance criteria, interface contracts, existing test conventions, test data policy. **Not** the implementation diff. |
| **Outputs** | Test suites (unit contract tests, integration, e2e), fixtures/factories, negative and boundary cases, `coverage_map` linking every criterion → test id |
| **Design logic** | One test per criterion minimum, then add adversarial cases: boundaries, empty/null, concurrency, idempotency, authz negative paths, failure injection. Tests target the **contract**, never internal structure, so they survive refactors. Deliberately writes tests before implementation exists — they must fail first (red-green verification is itself checked at Gate 0). |
| **Model tier** | T2, T3 for concurrency/security-sensitive suites |
| **Tools** | Test framework, sandbox, contract schemas, fixture generators |
| **Guardrails** | May not read the implementation patch while authoring; may not weaken or delete an existing test without an explicit, separately reviewed `test-change-request`; may not use production data |
| **Exit criteria** | Every criterion has ≥ 1 test; new tests fail against the pre-change baseline (proving they test something); suite is deterministic across 3 runs |

---

## 2.7 Verifier (deterministic — not an LLM)

| Field | Specification |
|---|---|
| **Role** | Produce the machine verdict that everything else defers to. |
| **Inputs** | Patch ⊕ acceptance tests, applied to a pinned base commit in an isolated worktree |
| **Outputs** | `Verdict`: per-check pass/fail, failing test names + trimmed logs, coverage delta, static-analysis findings, timings, artifact hashes |
| **Design logic** | Fixed pipeline, fail-fast ordering cheapest-first: format → lint → typecheck → build → affected unit tests → full unit → integration → coverage delta → SAST/SCA/secrets/licence. Test-impact analysis selects the affected subset first for fast feedback, then the full suite before approval. Flaky detection: a failing test is re-run 3× in isolation; consistent failure = real, inconsistent = flake → quarantine + ticket, non-blocking. |
| **Model tier** | None |
| **Tools** | Container sandbox, no network egress except an allowlisted package mirror |
| **Guardrails** | Hermetic and reproducible; pinned toolchain; no LLM may edit its config in the same patch it validates; verdicts are signed and immutable |
| **Exit criteria** | Verdict emitted; hard block on any failed check |

---

## 2.8 Code Reviewer (Critic) — now Commission seat C2

This agent and Commission seat C2 held the same jurisdiction and adjudicated the same
question twice. They are one agent now: the reviewer **is** seat C2, judging inside Gate C
under the stricter admissibility rules of §7.4. The specification below is unchanged; only
where it runs has changed (§14 O1). The same merge applies to §2.9 → C6 and §2.10 → C7.

| Field | Specification |
|---|---|
| **Role** | Judge what the pipeline cannot: correctness of intent, edge cases, maintainability, contract fidelity. |
| **Inputs** | Diff with surrounding context, work order + acceptance criteria, contracts, Gate 0 verdict, role lessons, review rubric |
| **Outputs** | `Findings[]`: severity (blocker/major/minor/nit), `file:line`, defect claim, **concrete failure scenario** (inputs → wrong behaviour), suggested fix |
| **Design logic** | Rubric-driven, in fixed order: contract conformance → correctness under adversarial inputs → error handling and partial failure → concurrency/idempotency → security-relevant paths → resource use → readability. **A finding is only valid if it names a concrete input that produces wrong output.** Speculative or stylistic remarks are capped at `nit` and never block. Verifies the diff actually satisfies the acceptance criteria rather than merely passing the tests. |
| **Model tier** | T2 default; T3 for high-risk classes |
| **Tools** | Read-only repo access, symbol search, retrieval over ADRs |
| **Guardrails** | Never edits code; may not restate Gate 0 output (no "add tests" when coverage passed); must cite lines; blocking findings require a reproducible scenario |
| **Exit criteria** | Zero blockers and zero unaddressed majors |

Requiring a concrete failure scenario is the practical fix for reviewer agents that
generate plausible-sounding, unfalsifiable objections and stall the loop.

---

## 2.9 Security & Compliance Agent

| Field | Specification |
|---|---|
| **Role** | Assess security impact of the change, beyond what scanners catch. |
| **Inputs** | Diff, threat model for touched components, dependency deltas, SAST/SCA output, data-classification map |
| **Outputs** | `SecurityFindings[]` with CWE/severity/exploit path, threat-model delta, required mitigations, compliance flags (PII handling, retention, audit logging) |
| **Design logic** | Focus on classes scanners miss: authorisation logic (IDOR, missing tenant scoping), authentication flows, trust-boundary crossings, injection through non-obvious sinks, secrets in logs/errors, unsafe deserialisation, SSRF, race conditions in permission checks. Each finding states the attacker, precondition, and impact. Auto-escalates any diff touching auth, payments, PII, or crypto to T3 + human review regardless of size. |
| **Model tier** | T3 |
| **Tools** | SAST/SCA results, dependency graph, threat-model store |
| **Guardrails** | Hard block on high severity; may not approve its own mitigation patch; PII/authz changes always require human sign-off |
| **Exit criteria** | No high/critical findings; mitigations verified by a test |

---

## 2.10 Performance & Cost Agent

| Field | Specification |
|---|---|
| **Role** | Prevent performance and infrastructure-cost regressions. |
| **Inputs** | Diff, NFR budgets, benchmark results, query plans, bundle stats, historical baselines |
| **Outputs** | Regression report: metric vs budget vs baseline, N+1 and hot-path analysis, index recommendations, verdict |
| **Design logic** | Compare against **budgets set by the Architect**, not vibes. Runs benchmarks for touched hot paths, inspects query plans for new/changed queries, checks bundle-size delta for frontend diffs, flags unbounded collections, missing pagination, sync work in request paths, and per-request allocations. Distinguishes measured regressions (blocking) from theoretical concerns (advisory). |
| **Model tier** | T2 |
| **Tools** | Benchmark harness, profiler output, DB `EXPLAIN`, bundle analyser |
| **Guardrails** | Blocks only on measured budget breaches; must attribute a regression to specific lines |
| **Exit criteria** | All NFR budgets met, or a documented, human-accepted exception |

---

## 2.11 Refiner / Repair Agent

| Field | Specification |
|---|---|
| **Role** | Convert a failure bundle into the smallest correct patch. Distinct from the Implementer by mindset and prompt. |
| **Inputs** | `FailureBundle`: gate verdict, failing evidence (trimmed logs, failing assertions), current diff, **all prior attempts and their outcomes**, root-cause classification, hypotheses already falsified |
| **Outputs** | Corrective patch, `root_cause` statement, `hypothesis` tested, note on why prior attempts failed |
| **Design logic** | Diagnose before editing: state a root-cause hypothesis that explains **all** observed evidence, then make the minimal change that tests it. Explicitly forbidden from the two classic repair-loop pathologies — broadening the change to "try things", and adjusting the test/assertion to fit the code. Receives the full attempt history precisely so it cannot re-try a falsified hypothesis. If evidence is insufficient, it emits a diagnostic patch (logging/assertions) as a deliberate information-gathering attempt rather than a blind fix. |
| **Model tier** | T2 on attempt 1, **T3 from attempt 2** (see escalation ladder, §4.3) |
| **Tools** | Sandbox, debugger/test runner, symbol search, git history/blame |
| **Guardrails** | May not modify acceptance tests; diff size must not grow unboundedly across attempts (monotonic growth trips the no-progress detector); must state a root cause before patching |
| **Exit criteria** | Failing check passes, no new failures, diff remains minimal |

---

## 2.12 Integrator

| Field | Specification |
|---|---|
| **Role** | Land approved patches on the main line without breaking it. |
| **Inputs** | Approved patches, current main, merge queue state, module leases |
| **Outputs** | Merge commits, conflict resolutions, rebase results, integration verdict |
| **Design logic** | Serialised merge queue: rebase onto current main → re-run Gate 0 **against the rebased result** (never trust a verdict from a stale base) → merge. Textual conflicts are resolved when the intent of both sides is unambiguous; anything ambiguous goes back as a refinement task. Also detects **semantic conflicts** — both sides compile and their tests pass, but combined behaviour is wrong (renamed semantics, duplicated logic, contradictory migrations) — by running the full suite plus cross-module contract tests on the combined result. |
| **Model tier** | T2, T3 for semantic conflict analysis |
| **Tools** | Git, merge queue, full verification pipeline |
| **Guardrails** | Never force-pushes shared branches; never merges with a stale verdict; one lease per module; migrations merge sequentially, never in parallel |
| **Exit criteria** | Main is green after merge |

---

## 2.13 Documentation Agent

| Field | Specification |
|---|---|
| **Role** | Keep external and internal documentation true to the merged code. |
| **Inputs** | Merged diff, contracts, ADRs, existing docs, changelog |
| **Outputs** | Updated API reference, README/runbook sections, ADR index, changelog entry, migration notes |
| **Design logic** | Generate from source of truth (contracts, types, tests) rather than prose-to-prose paraphrase. Updates only sections the diff invalidates. Documents behaviour and constraints, not implementation walkthroughs. Runnable examples are extracted from passing tests so they cannot rot silently. |
| **Model tier** | T1 |
| **Tools** | Repo, contract schemas, doc generator |
| **Guardrails** | May not modify code or tests; may not document unmerged behaviour |
| **Exit criteria** | Doc lint passes; every public contract change has a doc delta |

---

## 2.14 Release / DevOps Agent

| Field | Specification |
|---|---|
| **Role** | Get merged code safely into production and back out again if it misbehaves. |
| **Inputs** | Merged main, deployment config, migration plan, SLOs, feature-flag state |
| **Outputs** | Deployment plan and execution, canary analysis, flag rollout schedule, rollback execution, incident summary |
| **Design logic** | Expand/contract for schema changes (additive migration → deploy → backfill → switch → contract), never a destructive migration in a single step. Every feature ships behind a flag, default off. Canary: deploy to a slice, compare error rate, latency percentiles and business metrics against control for a fixed window; auto-rollback on breach. Rollback is always the first response to a production regression — diagnosis happens after the bleeding stops. |
| **Model tier** | T2, T3 for migration planning |
| **Tools** | CI/CD, IaC, feature flags, metrics/alerting, incident tooling |
| **Guardrails** | Production changes require the Policy Engine's approval for the risk class; destructive migrations always require a human; rollback path must be tested before rollout proceeds |
| **Exit criteria** | Canary healthy for the full window; SLOs intact; rollback verified available |

---

## 2.15 Memory Curator

| Field | Specification |
|---|---|
| **Role** | Serve every agent the smallest sufficient context. |
| **Inputs** | Repo state, ADRs, specs, past work orders, failure history, lesson store |
| **Outputs** | Context packs, repo map, symbol/embedding indexes, epic summaries |
| **Design logic** | Hybrid retrieval (BM25 + embeddings + symbol graph expansion) then rerank, packed in cache-friendly order (§1.5). Maintains a rolling repo map regenerated on merge. Compacts long epic histories into decision summaries so context does not grow linearly with project age. Runs memory hygiene — TTL, decay, deduplication, contradiction detection, invalidation on contract change (§9.7). Enforces a hard per-agent token budget — truncation is deliberate and reported, never silent. |
| **Model tier** | T1 for summarisation; retrieval is code |
| **Tools** | Vector store, symbol index, git |
| **Guardrails** | Never exceeds the agent's context budget; never serves stale contracts (invalidates on merge); reports what it dropped |
| **Exit criteria** | Pack within budget, contracts current |

---

## 2.16 Supervisor / Meta-agent

| Field | Specification |
|---|---|
| **Role** | Watch the system rather than the code: detect pathologies, capture lessons, tune routing. |
| **Inputs** | Event log, per-stage metrics, escalations, cost ledger, gate outcomes over time |
| **Outputs** | Lessons (durable rules injected into agent prompts), routing/threshold tuning proposals, circuit-breaker trips, alerts, weekly quality report |
| **Design logic** | Detects: refinement loops that oscillate, agents whose first-pass yield is dropping (prompt or model regression), work-order classes that systematically escalate (decomposition problem), gates that never fire (miscalibrated) or always fire (too strict), and cost outliers. Every escalation to a human produces a candidate lesson; lessons are only promoted after they demonstrably improve results on the golden-task eval set — otherwise the prompt library accumulates folklore. |
| **Model tier** | T3, runs on a schedule not in the hot path |
| **Tools** | Metrics store, event log, eval harness, prompt registry |
| **Guardrails** | Cannot modify code or approve work; prompt changes are versioned, eval-gated, and human-approved; may trip a breaker but a human clears it |
| **Exit criteria** | N/A — continuous |

---

## 2.17 The Commission (10 seats)

Specified in full in [`07-commission.md`](07-commission.md). Summarised here because it is
the final authority over everything the agents above produce.

| Field | Specification |
|---|---|
| **Role** | Adjudicate, unanimously, whether work is genuinely acceptable — not merely green |
| **Inputs** | A sealed Case Dossier assembled by a deterministic Clerk: work order, criteria, diff, all gate verdicts, attempt history, tools and skills used, applicable precedent |
| **Outputs** | `CommissionVerdict` (ACCEPTED / REJECTED), `Dissent[]` with clearance conditions, tenure updates, `PrecedentPack` at succession |
| **Design logic** | Ten disjoint jurisdictions, blind parallel voting, deterministic admissibility filter, unanimity required. Each seat serves three passings, then transmits a validated Precedent Pack to a successor |
| **Model tier** | T2 (C3, C4, C7, C9), T3 (C1, C2, C5, C6, C8, C10) |
| **Tools** | Read-only: repo, verdicts, contracts, precedent corpus |
| **Guardrails** | No write access; may not dissent outside jurisdiction; may not author the fix it later judges; may not vote while in succession; may not clear another seat's dissent |
| **Exit criteria** | Unanimous PASS from every seated commissioner against the current sealed dossier |

## 2.18 Human roles (not agents, but part of the design)

| Checkpoint | Trigger | Decision required |
|---|---|---|
| Spec clarification | `open_questions` non-empty | Answer product/business questions |
| Design approval | `risk_class = high`, irreversible decisions, new dependencies | Accept or redirect the architecture |
| Security sign-off | Auth, payments, PII, crypto diffs | Accept residual risk |
| Escalation | Loop budget exhausted or no-progress detected | Unblock, re-scope, or cancel |
| Commission arbitration | Contradictory dissents, disputed dissent, or 3 rounds exhausted | Rule between seats; the ruling becomes binding precedent |
| Release approval | Production changes above risk threshold | Authorise rollout |
| Lesson promotion | Supervisor proposes a prompt change | Approve library change |

The design target is that humans spend their time on these six decisions and nothing else.
Every other interaction is a symptom of a gate that is miscalibrated or a spec that was
under-specified.
