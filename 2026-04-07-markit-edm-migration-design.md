# Markit EDM Replacement Migration Design

**Date:** 2026-04-07
**Status:** Draft

## Context

Markit EDM currently acts as the orchestrator for trade lifecycle management, supporting multiple asset types (OTC, SEC, FX, etc.) with built-in rules/enrichments and downstream system integration. A new Java application is being built to replace it, with a Java-based rule engine (backed by YAML rules converted from Markit via a Python converter tool). Migration will be gradual, asset type by asset type.

## Migration Strategy: Asset-by-Asset Phased with Simple+Complex Pilot

Start with one simple and one complex asset type simultaneously. The simple type provides a quick win and validates the end-to-end process. The complex type reveals converter gaps early when there's maximum flexibility to fix them. All remaining asset types migrate using the proven process.

## Architecture

### Component Overview

```
┌─────────────────────────────────────────────────────────┐
│  Trade Capture                    Markit EDM (Legacy)   │
│  - Primary trade source         - All asset types      │
│    for New App                      active initially   │
│  - Unified JSON CMF format      - Rules built-in       │
│  - Will eventually be           - Receives most trades │
│    the sole trade source          from clients         │
└────────────────────┬─────────────────┬─────────────────┘
                     │                 │
                     │                 ▼
                     │    ┌─────────────────────┐
                     │    │ CMF Bridge (Temp)     │
                     │    │ Markit → CMF          │
                     │    │ Shadow mode only      │
                     └───►│ Discarded after migration │
                          └────────┬──────────────┘
                                   │
                                   ▼
                     ┌─────────────────────────┐
                     │   Java App (New)         │
                     │  ┌───────────────────┐   │
                     │  │ Orchestrator       │   │
                     │  │ Trade lifecycle     │   │
                     │  │ (per asset type):   │   │
                     │  │   A → B → C chain   │   │
                     │  └───────────────────┘   │
                     │  ┌───────────────────┐   │
                     │  │ Rule Engine (YAML) │   │
                     │  │ - Converted from   │   │
                     │  │   Markit rules     │   │
                     │  └───────────────────┘   │
                     │  ┌───────────────────┐   │
                     │  │ Downstream         │   │
                     │  │ Integration        │   │
                     │  └───────────────────┘   │
                     └────────────┬──────────────┘
                                  │
                    ┌─────────────┼──────────────┐
                    ▼             ▼              ▼
                Downstream A  Downstream B  Downstream C

┌──────────────────────────────────┐
│  Python Converter                │
│  Markit rules → YAML             │
│  - One-time + incremental runs   │
│  - Validation mode               │
└──────────────────────────────────┘

┌──────────────────────────────────┐
│  Reconciliation Framework         │
│  - Input comparison (CMF bridge) │
│  - Output comparison (downstream)│
│  - Exception reporting            │
│  - Reconciliation gate criteria   │
│  (Discarded after migration)      │
└──────────────────────────────────┘
```

### CMF Bridge (Temporary)

A thin adapter that reads Markit's internal trade representation and converts it to unified JSON CMF format. Used only during shadow mode for reconciliation. Built as a separate utility, not part of the main Java app or Markit. Discarded after all asset types are migrated.

### Reconciliation Framework (Temporary)

Two-tier comparison during shadow mode:

1. **Input-level reconciliation**: CMF bridge feeds same trade data (in CMF) to new app. Compare field-by-field to verify Markit trade data round-trips correctly.
2. **Downstream output comparison**: Compare what each system produces for downstream systems - same enrichment data, same state transitions, same routing decisions. This is the ultimate validation.

Reconciliation gate: N consecutive passes with zero critical discrepancies before cutover is allowed.

## Migration Phases

### Phase 0: Infrastructure Foundation

- Build the CMF Bridge (Markit → CMF conversion)
- Build the Reconciliation Framework (input + output comparison)
- Python converter improvements (validation mode, gap detection)
- Pick 1 simple + 1 complex asset type for Phase 1
- Asset type selection criteria: availability, business priority, rule complexity, downstream dependencies

### Phase 1: Pilot Migration (Simple + Complex Asset Types)

1. Convert all Markit rules for both asset types via Python converter → YAML
2. Shadow mode: Trade flows through Markit AND new app (via CMF bridge)
3. Compare downstream outputs → fix discrepancies (converter bugs, missing rules, edge cases)
4. **Reconciliation gate**: N consecutive passes, zero critical discrepancies
5. Cutover: New app owns downstream communication for migrated types; Markit suppresses downstream for those types
6. Markit continues processing un-migrated types only

### Phase 2-N: Remaining Asset Types

- Same process per asset type (or batch of related types)
- Asset types ordered by: dependency order, business priority, complexity, client impact
- Each phase gets faster as process is proven

### Phase Final: Decommission

- All asset types migrated to new app
- Remove CMF Bridge
- Remove Reconciliation Framework
- Markit EDM retired

## Key Risk Mitigations

| Risk | Mitigation |
|------|-----------|
| Rule conversion accuracy | Validation mode in converter; reconciliation gate; manual review of edge cases |
| Downstream message mismatch | Output-level comparison ensures downstream systems receive correct data |
| Migration timeline | Parallel simple/complex pilot; process reuse across phases |
| Data loss during cutover | Trade Capture as single source of truth; no-trade-during-cutover windows |
| Downstream system disruption | New app owns downstream only AFTER reconciliation passed; cutover during low-volume windows |

## Trade Capture Transition

Eventually Trade Capture will capture ALL trades (not just a subset). During migration:
- For migrated asset types: Trade Capture → New App (CMF format) → downstream
- For un-migrated asset types: Trade Capture → Markit → downstream
- Trade Capture routing logic updated per asset type as migration progresses

## Options Analysis by Topic

This section lists all considered options with pros/cons. Decisions will be made later once all topics are cataloged.

### 1. Migration Strategy

| Option | Pros | Cons |
|---|---|---|
| **A: Asset-by-asset with simple+complex pilot** *(provisional)* | Quick win from simple type; early gap detection from complex type; process is reusable | Requires parallel work streams; complex type may reveal issues that delay simple type |
| **B: All-asset-types-same-process** | Uniform approach; simpler planning | No early win; gap detection is late |
| **C: Complexity-sorted order (all simple first, then all complex)** | Easy to predict timeline | Complex types at end = highest risk when schedule pressure is highest |
| **D: Big bang** | Shortest total timeline | Highest risk; no validation per asset type; single point of failure |

### 2. Reconciliation Approach

| Option | Pros | Cons |
|---|---|---|
| **A: Only downstream output comparison** | No bridge needed; tests the real outcome | Cannot isolate input vs. processing discrepancies |
| **B: Markit → CMF Bridge (bridge feeds new app)** | Exact input parity; isolates conversion vs. processing issues | Build-and-discard infrastructure; bridge accuracy must itself be verified |
| **C: Bridge + Downstream Compare (provisional)** | Most confidence: bridge validates input, downstream validates output; discrepancies are precisely locatable | Most infrastructure to build; two comparison systems to maintain |
| **D: Trade Capture routing (skip bridge, compare downstream only)** | No bridge needed | Cannot verify input parity; format differences between Markit and CMF make diffs ambiguous |

### 3. Data Migration (History Depth)

| Option | Pros | Cons |
|---|---|---|
| **A: Open trades only** | Minimal migration effort; lowest risk | No historical context for rule engine; reports on recent history are incomplete |
| **B: Open trades + 30-90 days history (provisional)** | Enough for rule engine context; manageable migration scope | Still requires bulk migration tool; boundary (how far back) is arbitrary |
| **C: Full 7-year migration** | Complete data set; reports on new app are self-contained | Massive migration effort; PostgreSQL performance with 7 years of trade detail; unnecessary data |
| **D: Markit SQL Server as read-only archive** | Zero migration cost for old data; always accessible | Requires legacy API or direct DB access; reporting spans two systems |

### 4. Reporting Strategy

| Option | Pros | Cons |
|---|---|---|
| **A: Hybrid: JSON + extracted key columns (provisional)** | Fast aggregate queries; flexible JSON stored; no separate infrastructure | Deciding which fields to extract requires report audit; some data duplication |
| **B: Pure JSON + PostgreSQL GIN indexes** | No duplication; single source of truth | Slower aggregate queries; limited to PostgreSQL JSON operators; index size grows |
| **C: Pure JSON + materialized views** | Reports hit optimized structure; main store stays pure JSON | Refresh strategy complexity; views lag real data; maintenance overhead |
| **D: Separate analytics layer (ELT to column store)** | Dedicated reporting infrastructure; can scale independently | Significant additional infrastructure; data pipeline to build and maintain |

### 5. Downstream Cutover Mechanics

| Option | Pros | Cons |
|---|---|---|
| **A: Same queue, Markit stops / new app starts** | No downstream changes; same consumption point | Timing gap risk: duplicates or missed messages during switch |
| **B: Separate queues, dual subscription** *(fact: downstream uses separate queues per source)* | Clean separation; no collision; Markit and new app can coexist indefinitely | Downstream must subscribe to both queues; more infrastructure |
| **C: Fan-out to unified downstream queue** | Downstream reads one queue; no downstream changes | Fan-out infrastructure needed; message ordering complexity |
| **D: Downstream subscription updated at cutover (provisional)** | Cleanest: each system owns its queue; no message mixing | Cutover runbook step; downstream must be coordinated per asset type |

### 6. Reference Data / Market Data Dependencies

| Option | Pros | Cons |
|---|---|---|
| **A: Direct MRD API calls** | No data duplication; always fresh; simple | API latency adds to trade processing time; MRD becomes hard dependency |
| **B: Local cache with batch sync** | Fast local lookups; MRD downtime doesn't block trade processing | Data staleness between syncs; sync mechanism to build |
| **C: Local cache with event-driven sync (Kafka/JMS)** | Near-real-time freshness; fast local lookups | Requires MRD event support; complex event handling and ordering |
| **D: Hybrid: local cache + MRD fallback (provisional)** | Fast + always available; flexible per ref data type | Most complex: two lookup paths; cache staleness vs. freshness tuning needed |
| **E: One-time Markit ref data migration + ongoing MRD sync** | Captures Markit-specific enrichment; nothing missed | Markit ref data schema is proprietary and complex; may contain stale data |

### 7. Rollback Strategy

| Option | Pros | Cons |
|---|---|---|
| **A: Feature flag rollback (provisional)** | Instant; no data movement; Trade Capture still captures everything | Requires feature flags built into both systems; Markit must be able to re-activate quickly |
| **B: Full reversion to Markit processing** | Cleanest rollback state | Trade Capture routing must change back; messages during rollback window may be lost |
| **C: Dual-write during cutover (both systems active)** | No rollback needed - both systems already active | Message duplication risk; downstream must handle duplicates; expensive |

### 8. Regulatory / Compliance

| Option | Pros | Cons |
|---|---|---|
| **A: Map Markit compliance rules 1:1 to new app** | Audit trail shows exact equivalence; easiest to justify to regulators | May carry over obsolete rules; limits new app design flexibility |
| **B: Rebuild compliance logic fresh, validate against regulatory requirements directly** | Clean implementation; can incorporate modern best practices; not tied to Markit's interpretation of rules | Requires legal/compliance re-validation; higher effort; risk of interpretation divergence |
| **C: Dual compliance: Markit validates, new app mirrors, compare outputs** | Markit's authority backs new app during migration; discrepancies surface immediately | Regulatory bodies may require formal cutover notification |

### 9. Audit Trail

| Option | Pros | Cons |
|---|---|---|
| **A: Append-only event log in PostgreSQL** | Co-located with trade data; queryable; simple | Storage grows; may not meet regulatory immutability requirements |
| **B: Separate audit storage (append-only files or WORM)** | Meets strict regulatory requirements; tamper-evident | Separate infrastructure; harder to query |
| **C: Stream audit events to external audit system** | Decoupled; professional audit system can handle retention/legal holds | External system dependency; stream reliability concerns |

### 10. Operational Procedures & Training

| Option | Pros | Cons |
|---|---|---|
| **A: Rewrite Markit runbooks for new app before cutover** | Clean handover; ops teams ready from day 1 | Writing runbooks for unproven system; may need rewrites post-cutover |
| **B: Co-develop runbooks during shadow mode** | Based on real operational experience during shadow; more accurate | Ops teams learn on the fly; potential delays at cutover |
| **C: Dual runbooks during parallel run, merge after decommission** | Markit runbooks stay authoritative during transition; new app runbooks mature organically | Two sets of procedures to maintain; confusion risk |

### 11. Alerting & Monitoring

| Option | Pros | Cons |
|---|---|---|
| **A: New app monitoring mirrors Markit dashboards** | Ops teams see familiar metrics; easier transition | May replicate Markit's blind spots or irrelevant metrics |
| **B: Purpose-built monitoring for new app** | Can modernize observability; better signal-to-noise; cloud-native tooling | Ops teams need to learn new dashboards; transition period confusion |
| **C: Unified dashboard: both Markit and new app in one place** | Single pane of glass during migration; easy comparison | More complex dashboard setup; may be over-engineered for migration-only |

### 12. Batch Processing / End-of-Day

| Option | Pros | Cons |
|---|---|---|
| **A: Replicate Markit batch jobs 1:1 in new app** | Exact behavior match; easier reconciliation | Replicates potentially inefficient timing; ties new app to Markit's paradigm |
| **B: Redesign with streaming/real-time where possible** | Reduces batch dependency; faster processing; less overnight work | More complex architecture; harder to reconcile against Markit's batch output |
| **C: Hybrid: streaming for real-time, batch only for end-of-day aggregations** | Best of both: low latency + predictable EOD | Need to decide what goes where; reconciliation between streaming and batch sources |

### 13. Error Handling & Exception Management

| Option | Pros | Cons |
|---|---|---|
| **A: Mirror Markit's error handling workflow** | Ops teams already know the process; predictable | May replicate outdated or convoluted error workflows |
| **B: Build new exception framework (DLQ, retry, escalation)** | Modern patterns; configurable retry/escalation; better observability | New workflow for ops teams to learn; risk of gaps in the early design |
| **C: Markit handles errors for un-migrated types, new app handles for migrated** | Clean ownership per asset type | Two different error handling paradigms to operate during migration |

### 14. Performance / SLA

| Option | Pros | Cons |
|---|---|---|
| **A: Benchmark against Markit's current SLA, match or exceed** | Clear, measurable target; no regression for business | May inherit Markit's under-built SLA; competitive disadvantage |
| **B: Set new SLA based on business requirements, not Markit's numbers** | Potentially better performance; modernizes expectations | Requires business to re-define requirements; harder to negotiate |
| **C: Shadow SLA: observe new app performance during shadow, set post-migration SLA based on data** | Data-driven; realistic; avoids over-promising | No SLA commitment until after migration; business may not accept uncertainty |

### 15. User Access & Authorization

| Option | Pros | Cons |
|---|---|---|
| **A: Mirror Markit's existing RBAC roles** | Familiar to users; minimal change | Markit's roles may be outdated or overly permissive |
| **B: Redesign RBAC from scratch** | Clean access model; follows least-privilege; audit-friendly | Requires stakeholder alignment; user re-onboarding |
| **C: Map Markit roles to new app roles (translation layer)** | Low user disruption; can improve incrementally | Role mapping adds complexity; may perpetuate over-permissioned roles |

### 16. Knowledge Transfer

| Option | Pros | Cons |
|---|---|---|
| **A: Systematic Markit rule audit: every rule documented with rationale** | Complete knowledge capture; discovers dead rules that can be dropped | Time-intensive; requires Markit SME availability |
| **B: Shadow mode as knowledge capture: discrepancies reveal undocumented behavior** | Automated discovery; surfaces rules that behave differently than documented | Only covers active rules; edge cases may never trigger during shadow |
| **C: Interview Markit SMEs + document tribal knowledge** | Captures "this rule exists because of 2019 incident" context | Time-limited: SMEs won't be available forever; hard to validate completeness |

### 17. Vendor Dependencies

| Option | Pros | Cons |
|---|---|---|
| **A: Transfer existing vendor contracts from Markit to new app** | No new vendor negotiations; continuity of service | May not be transferable; Markit contracts may have exit fees |
| **B: Re-negotiate vendor contracts for new app** | Opportunity to get better terms; can switch vendors | Negotiation lead time; risk of service gaps during transition |
| **C: Build in-house alternatives where feasible** | No vendor dependency; full control; cost savings over time | Development effort; in-house tools may lack vendor features |

### 18. Client Communication

| Option | Pros | Cons |
|---|---|---|
| **A: No client impact => no notification needed** | Minimal administrative overhead | Risk: clients discover change indirectly and lose confidence |
| **B: Proactive client notification for each asset type cutover** | Transparent; manages expectations; clients can prepare their testing | More communication overhead; may raise concerns where none exist |
| **C: Single notification at migration start, status updates at each phase** | One upfront message; clients know the full roadmap | Early notification about distant phases may cause premature concern |

### 19. Testing Environment Parity

| Option | Pros | Cons |
|---|---|---|
| **A: Mirror Markit UAT structure for new app** | Testers use familiar environment; comparable test results | Replicates Markit's environment limitations; may not reflect prod accurately |
| **B: Cloud-native test environment with synthetic + sanitized prod data** | Scalable; repeatable; can test edge cases Markit's UAT can't | Cost of cloud infrastructure; data sanitization effort |
| **C: Shared test data pipeline: Markit test data feeds both systems** | One test data source, two consumers; ensures apples-to-apples testing | Data pipeline to build; Markit test data may not be in CMF format |

### 20. Cost Comparison & Budget

| Option | Pros | Cons |
|---|---|---|
| **A: Track Markit license savings vs. new app infra/team cost** | Clear ROI narrative; justifies migration investment | Markit costs are known, new app costs are estimates |
| **B: Include migration tooling + parallel run as project cost** | Realistic total cost; avoids under-budgeting | Higher upfront number; harder to get budget approval |
| **C: Phase-gated budget: each migration phase gets its own budget approval** | Controlled spending; can stop/adjust at each phase | Budget uncertainty; procurement overhead per phase |

| Option | Pros | Cons |
|---|---|---|
| **A: Feature flag rollback (provisional)** | Instant; no data movement; Trade Capture still captures everything | Requires feature flags built into both systems; Markit must be able to re-activate quickly |
| **B: Full reversion to Markit processing** | Cleanest rollback state | Trade Capture routing must change back; messages during rollback window may be lost |
| **C: Dual-write during cutover (both systems active)** | No rollback needed - both systems already active | Message duplication risk; downstream must handle duplicates; expensive |

## Open Questions

- What determines "N" in the reconciliation gate? (e.g., N=500 trades, N=5 business days?)
- Will there be a rollback plan if something goes wrong post-cutover?
- How are Markit rules that depend on external data sources (market data, reference data) handled in the new app?
- What is the interface between Trade Capture and the new app (API, message queue)?
- How many existing Markit reports need to be preserved, and what fields do they use?

## Discarded Approaches

**Approach: Big bang migration**
- Rejected due to high risk and inability to validate per asset type.

**Approach: Markit as router to new app**
- Rejected due to format incompatibility (Markit proprietary vs. CMF). Building a bridge in Markit would be more complex than a standalone CMF bridge utility.
