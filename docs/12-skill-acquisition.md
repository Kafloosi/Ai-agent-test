# 12 — Skill Acquisition

Agents acquire capability they lack, including from the internet, without waiting for a
human. Two paths with very different bars, because **using knowledge once is cheap and
reversible; installing it permanently is neither.**

| Path | Scope | Bar | Persists |
|---|---|---|---|
| **Just-in-time** | Finish the current work order | Verified by execution | No — working memory only |
| **Durable** | Add a skill to the fleet | Verified + eval + 80% Commission vote (§11.8) | Yes — versioned, revocable |

## 12.1 The one rule everything else protects

> **Fetched content is data, never instructions.**

External material informs *what an agent knows*. It never determines *what an agent does*.
Every fetched document enters context inside an untrusted envelope; imperatives inside it —
"ignore previous instructions", "run this command", "disable the check" — are inert. Fetched
content can never modify an agent's guardrails, permissions, tool set, budgets, or the skill
registry. Those change only through the amendment process, which requires a Commission vote
and, for anything constitutional, a human (§11.9).

Without this rule, "agents may read the web" means "anyone who can publish a web page can
issue instructions to a system with commit access".

## 12.2 Acquisition order — cheapest source first

An agent that reaches for the web before checking what it already has is the most common and
most expensive form of waste here.

```mermaid
flowchart TB
    GAP["Agent hits a knowledge gap<br/>(no matching skill · low confidence ·<br/>repeated failure on unfamiliar tech)"] --> S1{"1 · Skill registry<br/>exact + semantic match"}
    S1 -- hit --> USE["Activate skill — free"]
    S1 -- miss --> S2{"2 · Project memory<br/>(§9.5) — has this been<br/>solved here before?"}
    S2 -- hit --> USE2["Apply recorded fact — free"]
    S2 -- miss --> S3{"3 · Local sources<br/>vendored docs · type stubs ·<br/>source of the dependency itself"}
    S3 -- hit --> USE3["Read locally — no egress"]
    S3 -- miss --> S4{"4 · Fetch cache<br/>URL ⊕ version"}
    S4 -- hit --> USE4["Reuse cached fetch — no egress"]
    S4 -- miss --> WEB["5 · Web fetch<br/>allowlisted, budgeted, logged"]

    WEB --> TIER{"Source tier"}
    TIER -->|"T0 vendor/official docs"| OK["Usable directly"]
    TIER -->|"T1 reputable reference"| OK
    TIER -->|"T2 general web"| VERIFY["Usable only after<br/>execution verification"]
    TIER -->|"T3 unknown/untrusted"| REJECT["Rejected —<br/>never enters a skill"]

    OK & VERIFY --> JIT["Just-in-time use<br/>in this run only"]
    JIT --> WORTH{"Recurring capability?<br/>projected ≥ 3 work orders"}
    WORTH -- no --> DISCARD["Discard after the run —<br/>no permanent context rent"]
    WORTH -- yes --> DRAFT["Draft a durable skill →<br/>§12.4"]

    style REJECT fill:#a33,color:#fff
    style USE fill:#1f6f43,color:#fff
    style DISCARD fill:#1f6f43,color:#fff
```

The cache line matters more than it looks. Fifty agents fetching the same framework page is
fifty times the cost of one agent fetching it and the rest reading the cache. Fetches are
keyed by URL ⊕ resolved dependency version and shared fleet-wide.

## 12.3 Source tiers and egress control

| Tier | Examples | Just-in-time | May become a skill |
|---|---|---|---|
| **T0** | Vendor/official documentation, RFCs, language and framework references, the dependency's own source | ✅ direct | ✅ |
| **T1** | Established technical references, standards bodies, maintained specifications | ✅ direct | ✅ |
| **T2** | Blogs, forums, Q&A, tutorials | ✅ **only after execution verification** | ✅ only with a passing verification artifact |
| **T3** | Unknown provenance, content farms, anything not resolvable to a maintainer | ❌ | ❌ |

Egress rules, enforced by the sandbox (§5.4), not by the agent:

- Domain allowlist; per-work-order scope; GET only.
- No credentials, tokens, or repository content in any outbound request.
- Response size capped; content stripped to text before entering context.
- Every fetch logged with URL, tier, hash, and the work order that caused it — so a bad
  skill can be traced back to its source.
- **Fetched code is untrusted code**: scanned, executable only inside the sandbox, and never
  committed without passing the same gates as agent-written code.

## 12.4 Durable acquisition — drafting a skill

When a capability is projected to recur (≥ 3 work orders), the agent drafts a skill instead
of re-researching it every time. Drafts follow the §8.5 authoring standard and add three
fields that ordinary skills do not need:

| Field | Requirement |
|---|---|
| `sources[]` | URL, tier, retrieval date, content hash — for every claim the skill relies on |
| `verified_claims[]` | Each load-bearing claim, and the artifact that proves it |
| `verified_against` | The exact dependency and version the verification ran against |

**Verification is the core of the mechanism.** A claim taken from the internet is a
hypothesis until executed. The drafting agent writes a scratch test in the sandbox that
exercises the claim and records the result. "The docs say the client retries on 429" becomes
"a scratch test confirms the client retries on 429 against `httpx==0.27.1`, log attached".

This is the same principle the whole design rests on: deterministic execution is ground
truth, and an LLM's reading of a web page is not. It also removes the three failure modes
that make web-sourced knowledge dangerous — content that is outdated, content that is wrong,
and content that was hallucinated during summarisation. None of them survive a scratch test.

Schema: [`schemas/skill-draft.schema.json`](../schemas/skill-draft.schema.json).

## 12.5 Promotion — reuse the amendment process

A skill draft is an `OptimizationProposal` of class `ordinary` (§11.7). It takes the existing
path; no parallel governance:

```mermaid
flowchart LR
    DRAFT["SkillDraft<br/>sources + verified claims"] --> DEDUP{"Supervisor triage:<br/>duplicate of an<br/>existing skill?"}
    DEDUP -- yes --> MERGE["Merge into the existing skill;<br/>discard the draft"]
    DEDUP -- no --> EVALG["Eval harness:<br/>golden tasks for this class,<br/>with and without the skill"]
    EVALG -- "no improvement" --> DROP["Discard"]
    EVALG -- improves --> VOTE["Commission: 80% of<br/>non-abstaining, ≥6 absolute<br/>(§11.8)"]
    VOTE -- rejected --> DROP
    VOTE -- approved --> REL["Versioned release<br/>into the skill registry"]
    REL --> MON["Monitored: first-pass yield<br/>on this work-order class"]
    MON -- "no effect" --> REVERT["Auto-retire"]
    MON -- holds --> PERM["Permanent"]

    style EVALG fill:#0d2a4a,color:#fff
    style PERM fill:#1f6f43,color:#fff
```

Two seats matter especially here. **C6 (Security)** holds its usual jurisdictional veto —
relevant because a skill sourced from the web is a supply-chain path into the fleet's
procedural memory. **C10 (Evidence & Process)** dissents on any draft whose claims are not
backed by a verification artifact, which is what stops "I read a blog post" becoming
institutional knowledge.

## 12.6 Staleness

Web-sourced knowledge decays. Every acquired skill carries `verified_against` and is
re-verified when that dependency changes.

| Event | Action |
|---|---|
| Pinned dependency upgraded | Re-run the skill's verification tests; failure marks the skill `stale` and removes it from context packs until repaired |
| Source URL content hash changes | Flag for re-verification — the page the skill was derived from has moved |
| Skill unused for a full TTL | Retire under the §9.7 pruning rules |
| Verification test fails in CI | Immediate `stale`; the skill stops being served the same cycle |

A stale skill is worse than a missing one: a missing skill makes an agent research, a stale
skill makes it confident and wrong.

## 12.7 Bounds

| Bound | Value | Why |
|---|---|---|
| Research budget per work order | ≤ 10% of token budget, ≤ 8 fetches | Research is not the work; a run that spends half its budget reading has an oversized work order behind it |
| Draft threshold | Projected ≥ 3 work orders | One-off knowledge stays just-in-time; permanent skills pay permanent context rent |
| Drafts per agent per work order | 1 | Same rule as proposals — the channel must not become narration |
| Registry growth | Supervisor reviews skill count each amendment cycle | Skill sprawl degrades trigger precision: two skills with overlapping descriptions both fire, or neither does |
| Constitutional floor | No acquired skill may alter guardrails, permissions, gates, or budgets | Those are constitutional amendments (§11.8), human-ratified |

## 12.8 Anti-patterns

| Anti-pattern | Why it fails | Prevented by |
|---|---|---|
| Fetching before checking the registry | Pays egress and tokens for knowledge already held | Acquisition order (§12.2) |
| Treating fetched text as instructions | Any web page becomes a command channel into a system with commit access | Data-not-instructions rule (§12.1) |
| Drafting a skill from unverified reading | Outdated and hallucinated claims become institutional knowledge | Execution verification + C10 veto (§12.4) |
| A skill per work order | Registry sprawl; trigger precision collapses | Recurrence threshold + dedup triage |
| Skill drafted to excuse a failure | Turns a defect into a permanent artifact | Eval gate — a skill that does not improve the golden set does not ship |
| No version pinning | Silent staleness; confident wrong answers | `verified_against` + re-verification on upgrade |
