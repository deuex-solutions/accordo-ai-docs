# Master Task Sequencer: 3-Track LangGraph Migration

This document defines the **sequential execution order** for all tasks across Vatsal (Track 1), Yug (Track 2), and Adarsh (Track 3). It identifies dependencies, critical path items, and optimal task pickup sequence.

---

## Execution Philosophy

1. **Contract-First**: State schema must be finalized before any agent implementation.
2. **Foundation-First**: Core agents (Parsing, Decision, State) must be ready before orchestration.
3. **Consumer-After-Producer**: Agents that consume state data wait for agents that produce it.
4. **Integration Sprints**: Weeks 5-6 are high-integration; Weeks 7-8 are polish and parity.

---

## Phase-by-Phase Sequential Breakdown

### PHASE 0: CONTRACT FINALIZATION (Week 1, Days 1-2) - **DONE**

| Order | Task                                 | Owner        | Output                                         | Status |
| ----- | ------------------------------------ | ------------ | ---------------------------------------------- | ------ |
| 1     | Research existing interfaces         | Vatsal       | Interface inventory                            | [DONE] |
| 2     | **Define `NegotiationState` schema** | Vatsal       | `state.ts`                                     | [DONE] |
| 3     | Review & approve schema              | Yug + Adarsh | Sign-off                                       | [DONE] |
| 4     | Design analysis-specific channels    | Yug          | `ToneAnalysis`, `BehavioralSignals` interfaces | [DONE] |
| 5     | Design strategy-specific channels    | Adarsh       | `VendorProfile`, `MESOOptions` interfaces      | [DONE] |

**Gate**: All three developers have approved `NegotiationState`. Proceeding to Foundation Agents.

---

### PHASE 1: FOUNDATION AGENTS (Weeks 1-2) - **PARALLEL EXECUTION**

Once the state contract is approved, these can run in parallel:

#### Week 1, Days 3-5 (Independent Tasks)

| Order | Task                                  | Owner  | Dependencies          | Deliverable                  |
| ----- | ------------------------------------- | ------ | --------------------- | ---------------------------- |
| 1.1   | `StateManagementAgent` implementation | Vatsal | State schema approved | Checkpointer + transitions   |
| 1.2   | Analysis interface finalization       | Yug    | State schema approved | Analysis type definitions    |
| 1.3   | `VendorProfilingAgent` foundation     | Adarsh | State schema approved | Profile schema + persistence |

**No cross-dependencies here** - all three can work simultaneously.

#### Week 2 (Independent Tasks)

| Order | Task                         | Owner  | Dependencies                | Deliverable                           |
| ----- | ---------------------------- | ------ | --------------------------- | ------------------------------------- |
| 2.1   | `OfferParsingAgent`          | Vatsal | None (uses state schema)    | Regex parsing, currency normalization |
| 2.2   | `ToneAnalysisAgent`          | Yug    | Analysis interfaces         | 11-signal tone detection              |
| 2.3   | `BehavioralAnalysisAgent`    | Yug    | ToneAnalysisAgent preferred | Concession velocity tracking          |
| 2.4   | `MESOGenerationAgent` Part 1 | Adarsh | VendorProfile helpful       | Utility calculations                  |

**Note**: Yug can work on Tone independently, but Behavioral benefits from understanding Tone output structure.

---

### PHASE 2: THE BRAIN & VOICE (Weeks 3-4) - **SEQUENTIAL DEPENDENCIES EMERGE**

#### Week 3: Decision Engine (Critical Path) + Supporting Agents

| Order | Task                                 | Owner  | Dependencies          | Deliverable              |
| ----- | ------------------------------------ | ------ | --------------------- | ------------------------ |
| 3.1   | **DecisionAgent Foundation**         | Vatsal | ParsingAgent complete | Accept/Escalate triggers |
| 3.2   | `ConcernExtractionAgent`             | Yug    | None (independent)    | Semantic issue detection |
| 3.3   | `ResponseGenerationAgent` foundation | Yug    | DecisionAgent helpful | PM persona rendering     |
| 3.4   | `MESOGenerationAgent` Part 2         | Adarsh | MESO Part 1 complete  | Pareto-optimal packages  |

**Sequence Note**:

- Vatsal's DecisionAgent (3.1) should complete before Yug fully commits to ResponseGeneration (3.3) because responses depend on decision outputs.
- However, Yug can start ResponseGen scaffolding while DecisionAgent is in progress.

#### Week 4: Advanced Logic & Validation

| Order | Task                              | Owner  | Dependencies          | Deliverable                         |
| ----- | --------------------------------- | ------ | --------------------- | ----------------------------------- |
| 4.1   | **DecisionAgent Advanced**        | Vatsal | Foundation complete   | Walk-away logic, utility thresholds |
| 4.2   | `ValidationAgent` - Two-tier bans | Yug    | ResponseGen helpful   | Output safety layer                 |
| 4.3   | `StallRecoveryAgent`              | Adarsh | DecisionAgent helpful | Deadlock detection                  |

**Sequence Note**:

- ValidationAgent (4.2) validates ResponseGeneration outputs, so ResponseGen should have v1 ready before Validation is complete.
- StallRecovery (4.3) monitors negotiation health which requires DecisionAgent outputs.

---

### PHASE 3: ORCHESTRATION & INTELLIGENCE (Week 5-6) - **HIGH INTEGRATION**

#### Week 5: The Skeleton Assembly

| Order | Task                          | Owner  | Dependencies                    | Deliverable            |
| ----- | ----------------------------- | ------ | ------------------------------- | ---------------------- |
| 5.1   | **NegotiationOrchestrator**   | Vatsal | DecisionAgent + ParsingAgent    | Main StateGraph wiring |
| 5.2   | `VectorSearchAgent`           | Yug    | Independent                     | Semantic retrieval     |
| 5.3   | `VendorSimulationAgent`       | Adarsh | StateManagementAgent            | Testing engine         |
| 5.4   | **Integrate Yug's agents**    | Vatsal | Tone + Behavior + Concern ready | Analysis nodes wired   |
| 5.5   | **Integrate Adarsh's agents** | Vatsal | MESO + Stall ready              | Strategy nodes wired   |

**Critical Sequence**:

1. Vatsal starts Orchestrator (5.1) with his own agents first.
2. As Yug and Adarsh complete their Week 4 agents, Vatsal integrates them (5.4, 5.5).
3. VectorSearch (5.2) and VendorSimulator (5.3) can be developed in parallel but need integration testing.

#### Week 6: RAG, Context, and Auxiliary Services

| Order | Task                      | Owner  | Dependencies           | Deliverable                        |
| ----- | ------------------------- | ------ | ---------------------- | ---------------------------------- |
| 6.1   | `WeightedUtilityNode`     | Vatsal | DecisionAgent complete | Multi-parameter scoring            |
| 6.2   | HITL Hooks                | Vatsal | Orchestrator helpful   | Human-in-the-loop interrupts       |
| 6.3   | `RAGContextAgent`         | Yug    | VectorSearchAgent      | Context assembly                   |
| 6.4   | Parallel execution setup  | Yug    | All analysis agents    | Tone + Behavior + Concern parallel |
| 6.5   | `EmailNotificationAgent`  | Adarsh | Orchestrator helpful   | Async email triggers               |
| 6.6   | `DocumentGenerationAgent` | Adarsh | EmailAgent pattern     | PDF generation                     |

**Sequence Notes**:

- RAGContext (6.3) consumes VectorSearch output, so 5.2 must finish before 6.3 completes.
- Email (6.5) and Document (6.6) are triggered by orchestrator decisions, so orchestrator wiring should exist.

---

### PHASE 4: ADVANCED SYSTEMS & POLISH (Week 7) - **INDEPENDENT TRACKS**

| Order | Task                             | Owner  | Dependencies            | Deliverable                     |
| ----- | -------------------------------- | ------ | ----------------------- | ------------------------------- |
| 7.1   | State persistence tuning         | Vatsal | Orchestrator + HITL     | Compression, query optimization |
| 7.2   | `PhrasingHistoryAgent`           | Yug    | ResponseGenerationAgent | Opener deduplication            |
| 7.3   | Safety audit                     | Yug    | ValidationAgent         | System-wide policy review       |
| 7.4   | `BidComparisonOrchestratorAgent` | Adarsh | VendorSimulator helpful | Multi-vendor workflows          |
| 7.5   | Document system polish           | Adarsh | DocumentGen v1          | Template personalization        |

**Note**: Week 7 tasks are largely independent - each track owner works on their specialty areas.

---

### PHASE 5: FINAL INTEGRATION & LAUNCH (Week 8) - **SEQUENTIAL LOCK-STEP**

| Order | Task                            | Owner  | Dependencies          | Deliverable             |
| ----- | ------------------------------- | ------ | --------------------- | ----------------------- |
| 8.1   | `ConversationOrchestratorAgent` | Adarsh | ALL other agents      | Entry-point routing     |
| 8.2   | API Documentation               | Yug    | All agents defined    | Swagger updates         |
| 8.3   | Infrastructure update           | Vatsal | All system components | `.env`, `Dockerfile`    |
| 8.4   | **Final E2E Testing - Track 2** | Yug    | Full system           | Analysis layer tests    |
| 8.5   | **Final E2E Testing - Track 3** | Adarsh | Full system           | Strategy layer tests    |
| 8.6   | **Final E2E System Check**      | Vatsal | All tracks            | End-to-end validation   |
| 8.7   | Parity validation               | Adarsh | E2E tests             | 95%+ parity with legacy |
| 8.8   | **DEPLOY**                      | Vatsal | All validations       | Production release      |

**Critical Sequence**:

- ConversationOrchestrator (8.1) MUST come after all other agents exist - it's the entry point.
- API docs (8.2) should document the complete system.
- E2E testing happens in sequence: Track 2, Track 3, then full system.
- Deploy is the final step after all validation.

---

## Visual Dependency Map

```
WEEK 1 (Foundation)
===================
Vatsal: NegotiationState ──────────────────────────────────────────┐
Yug:    Review + Analysis interfaces ──────────────────────────────┤ All three must complete
Adarsh: Review + Strategy interfaces ──────────────────────────────┘
            ↓                    ↓                    ↓
            └────────────────────┴────────────────────┘
                           ↓
                  [STATE CONTRACT LOCKED]
                           ↓
WEEK 2 (Independent Agents)
=============================
Vatsal: StateManagement ─────┐
       OfferParsing ──────────┤
Yug:    ToneAnalysis ─────────┤ Can work in parallel
       BehavioralAnalysis ────┤
Adarsh: VendorProfiling ──────┤
       MESO Part 1 ───────────┘
            ↓
WEEK 3 (Decision Dependencies)
==============================
Vatsal: DecisionAgent Foundation ───┬──→ Yug: ResponseGeneration (foundation)
Yug:    ConcernExtraction ──────────┘
Adarsh: MESO Part 2 ────────────────┐
                                   └──→ Uses Decision logic
            ↓
WEEK 4 (Advanced Logic)
=======================
Vatsal: DecisionAgent Advanced ───┬──→ Yug: ValidationAgent
Yug:    (continues) ──────────────┤    (validates responses)
Adarsh: StallRecovery ────────────┘    (uses Decision output)
            ↓
WEEK 5 (Orchestrator Integration)
=================================
Vatsal: NegotiationOrchestrator ───┬──→ Integrates Yug's agents
                                   ├──→ Integrates Adarsh's agents
Yug:    VectorSearch ──────────────┤    (parallel development)
Adarsh: VendorSimulator ───────────┘    (parallel development)
            ↓
WEEK 6 (RAG + Aux Services)
===========================
Vatsal: WeightedUtility + HITL ───┐
Yug:    RAGContextAgent ──────────┤──→ Uses VectorSearch (Week 5)
       Parallel execution ────────┤
Adarsh: EmailNotification ────────┤──→ Uses Orchestrator triggers
       DocumentGeneration ───────┘    (follows Email pattern)
            ↓
WEEK 7 (Polish)
===============
Vatsal: Persistence tuning ─────────┐
Yug:    PhrasingHistory ───────────┤ Independent tracks
       Safety audit ───────────────┤
Adarsh: BidComparison ─────────────┤
       Document polish ────────────┘
            ↓
WEEK 8 (Lock-Step Launch)
=========================
Adarsh: ConversationOrchestrator ──┐
Yug:    API Documentation ─────────┤ Sequential
Vatsal: Infrastructure update ─────┤
Yug:    E2E Testing Track 2 ───────┤
Adarsh: E2E Testing Track 3 ────────┤
Vatsal: Final E2E + Deploy ────────┘
```

---

## Critical Path Summary

These tasks form the **critical path** - any delay here delays the entire project:

1. **Week 1, Day 2**: `NegotiationState` schema finalized (Vatsal)
2. **Week 2, Day 5**: `OfferParsingAgent` complete (Vatsal)
3. **Week 3, Day 5**: `DecisionAgent` Foundation complete (Vatsal)
4. **Week 5, Day 3**: `NegotiationOrchestrator` v1 ready (Vatsal)
5. **Week 8, Day 3**: `ConversationOrchestrator` complete (Adarsh)
6. **Week 8, Day 5**: Final E2E + Deploy (Vatsal)

**Critical Path Owner**: Vatsal (4 of 6 critical tasks)

---

## Handoff Protocol

When a dependency is complete, the owner must:

1. **Update their execution plan**: Mark the task [DONE]
2. **Update MIGRATION_TASK_BOARD.md**: Change status column
3. **Notify dependent owner**: Post in shared channel with:
   - Task name completed
   - Branch name
   - Key interface/output details
   - Any caveats or usage notes

Example:

```
@yug @adarsh - DecisionAgent Foundation is DONE
Branch: feature/decision-agent-foundation
Output: Decision object with {action, confidence, reason}
Caveat: Walk-away logic is placeholder - will complete in Week 4
```

---

## Risk Adjustments

### If Vatsal Falls Behind (Critical Path Risk)

| Delay                      | Impact                                         | Adjustment                                     |
| -------------------------- | ---------------------------------------------- | ---------------------------------------------- |
| State schema delayed 1 day | Yug/Adarsh Week 1 blocked                      | Parallel interface design without final schema |
| DecisionAgent delayed      | ResponseGen, Validation, StallRecovery blocked | Use mock Decision objects for development      |
| Orchestrator delayed       | Full integration blocked                       | Develop agents in isolation, integrate later   |

### If Yug Falls Behind

| Delay                   | Impact                           | Adjustment                           |
| ----------------------- | -------------------------------- | ------------------------------------ |
| Analysis agents delayed | Orchestrator integration delayed | Orchestrator wires placeholder nodes |
| ResponseGen delayed     | Validation blocked               | Validation tests with mock responses |
| RAGContext delayed      | MESO context assembly affected   | MESO uses simpler context fallback   |

### If Adarsh Falls Behind

| Delay                            | Impact                           | Adjustment                     |
| -------------------------------- | -------------------------------- | ------------------------------ |
| MESO delayed                     | Advanced counter-offers affected | Use simple counter-offer logic |
| StallRecovery delayed            | Deadlock detection missing       | Manual monitoring fallback     |
| ConversationOrchestrator delayed | Launch blocked                   | Vatsal/Yug assist in Week 8    |

---

## Weekly Sync Agenda

**Every Monday, 30 minutes**:

1. **5 min**: Review critical path status (Vatsal reports first)
2. **10 min**: Blocker escalation - anyone blocked on a dependency?
3. **10 min**: Integration preview - what handoffs happen this week?
4. **5 min**: Risk review - any tasks at risk of delay?

**Outcome**: Updated task priorities for the week.

---

## Quick Reference: Who Works on What Week

| Week | Vatsal (Track 1)                          | Yug (Track 2)                        | Adarsh (Track 3)                                |
| ---- | ----------------------------------------- | ------------------------------------ | ----------------------------------------------- |
| 1    | State schema, StateManagement             | Interface design, schema approval    | Interface design, schema approval               |
| 2    | OfferParsing                              | ToneAnalysis, BehavioralAnalysis     | VendorProfiling, MESO Part 1                    |
| 3    | **DecisionAgent Foundation**              | ConcernExtraction, ResponseGen start | MESO Part 2                                     |
| 4    | DecisionAgent Advanced                    | ValidationAgent                      | StallRecovery                                   |
| 5    | **NegotiationOrchestrator** + Integration | VectorSearch                         | VendorSimulator                                 |
| 6    | WeightedUtility, HITL                     | RAGContext, Parallel exec            | Email, DocumentGen                              |
| 7    | Persistence tuning                        | PhrasingHistory, Safety audit        | BidComparison, Doc polish                       |
| 8    | Infrastructure, Final E2E, **Deploy**     | API Documentation                    | **ConversationOrchestrator**, Parity validation |

---

**Last Updated**: June 8, 2026
**Next Review**: Weekly (Mondays)
