# ADARSH'S EXECUTION PLAN: 8-Week LangGraph Migration

This document serves as your personalized, daily reference for the Accordo AI migration.

**Core Responsibilities**: The "Tactics" (MESO Strategy, Stall Recovery), "Vendor Memory" (Profiling), "Simulation" (Testing Engine), and Supporting Systems (Email, PDF, Bid Comparison).

## SYNERGY & LOGIC MAPPING (CRITICAL)

Before implementing any agent, you must:

1.  **Locate the Legacy Workflow**: Identify the specific files in `src/modules/chatbot/engine/`, `src/modules/chatbot/vendor/`, or `src/modules/bid-comparison/` that handle this logic today.
2.  **Extract Rules**: List the "must-have" business rules from that legacy code.
3.  **Trace in Comments**: Add a `@source` comment in your agent code pointing to the original file.

---

## WEEK 1: STRATEGY INTERFACE REVIEW & VENDOR PROFILING FOUNDATION

**Focus**: Understanding the State Contract and designing vendor memory systems.

### [x] PHASE 1.1: State Contract Review (Day 1-2)

- [x] Review Vatsal's `NegotiationState` interface in `src/modules/chatbot/engine/graph/state.ts`.
- [x] Identify strategy-specific channels needed: `vendorProfile`, `mesoOptions`, `stallStatus`.
- [x] **Validation**: Confirm state channels with Vatsal and Yug.

### [x] PHASE 1.2: `VendorProfilingAgent` - Foundation (Day 3-5)

- [x] Review `meso.ts` profile builder section (~200 lines).
- [x] Review `src/models/vendor-negotiation-profile.ts` for data structures.
- [x] Design `VendorProfile` interface for state channel.
- [x] Implement preference learning persistence layer.
- [x] **Validation**: Profile schema approved by team.

---

## WEEK 2-3: MESO GENERATION (The "Tactics Engine")

**Focus**: Port the complex Pareto-optimal offer generation algorithm.

### [x] PHASE 2.1: `MESOGenerationAgent` - Part 1 (Week 2, Day 1-5)

- [x] Extract core MESO logic from `meso.ts` (~2,000 lines total).
- [x] Port utility function calculations (price, delivery, terms).
- [x] Implement Pareto frontier calculation.
- [x] Port trade-off logic between parameters.
- [x] **Validation**: Generate 10 MESO packages, verify Pareto-optimality.

### [x] PHASE 2.2: `MESOGenerationAgent` - Part 2 (Week 3, Day 1-5)

- [x] Port preference-learned option tuning.
- [x] Implement multi-parameter optimization.
- [x] Port equivalent utility calculation (2-3 options with same utility).
- [x] Integrate with VendorProfilingAgent for personalized options.
- [x] **Validation**: Full parity test against 50 legacy MESO generations.

---

## WEEK 4: STALL RECOVERY & DEADLOCK DETECTION

**Focus**: Negotiation health monitoring and recovery.

### [x] PHASE 4.1: `StallRecoveryAgent` (Day 1-3)

- [x] Port deadlock detection from `stall-detector.ts`.
- [x] Implement "no progress" detection (3+ rounds without movement).
- [x] Port momentum analysis integration.
- [x] **Validation**: Detect stalls in 10 historical deadlocked negotiations.

### [x] PHASE 4.2: Recovery Probe Generation (Day 4-5)

- [x] Implement recovery probe strategies (value-add offers, deadline extensions).
- [x] Port escalation timing optimization.
- [x] Implement automated nudge generation.
- [x] **Validation**: Recovery probe effectiveness test.

---

## WEEK 5: VENDOR SIMULATION ENGINE

**Focus**: Testing and training environment.

### [x] PHASE 5.0: Strategy/Offers Node Integration (Completed on branch `adarsh/week5-strategy-node`, PR #58)
- [x] Integrate `VendorProfilingAgent`, `MESOGenerationAgent`, and `StallRecoveryAgent` into a unified `generateOffersNode`.
- [x] Wire node output to comply with the structured `mesoOptions` and `counterOffer` NegotiationState channels.
- [x] **Validation**: Implement and pass strategy node tests (`tests/ai-evals/generate-offers.eval.test.ts`).

### [x] PHASE 5.1: `VendorSimulationAgent` (Day 1-3)

- [x] Port `vendor-simulator.service.ts` to graph-compatible format.
- [x] Port `vendor-agent.ts` persona logic.
- [x] Update simulator for graph interaction (state-based responses).
- [x] Implement realistic vendor persona generation.
- [x] **Validation**: Simulator produces plausible vendor responses.

### [x] PHASE 5.2: Multi-Vendor Testing (Day 4-5)

- [x] Implement parallel scenario simulation.
- [x] Port multi-vendor strategy testing.
- [x] **Validation**: A/B test simulator against real vendor behavior patterns.

---

## WEEK 6: AUXILIARY SERVICES - EMAIL & DOCUMENTS

**Focus**: Background service nodes for notifications and reporting.

### [x] PHASE 6.1: `EmailNotificationAgent` (Day 1-3)

- [x] Port `email.service.ts` to async LangGraph node.
- [x] Port `bid-comparison.email.ts` templates.
- [x] Implement deal update email triggers.
- [x] Add async email queue processing.
- [x] **Validation**: Email delivery test in dev environment.

### [x] PHASE 6.2: `DocumentGenerationAgent` (Day 4-5)

- [x] Port `pdf-generator.ts` to background node.
- [x] Port `deal-summary-pdf-generator.ts` logic.
- [x] Implement deal summary PDF generation.
- [x] Add batch processing capability.
- [x] **Validation**: Generate 5 sample PDFs, verify formatting.

---

## WEEK 7: BID COMPARISON & ADVANCED SYSTEMS

**Focus**: Multi-vendor orchestration and document generation.

### [x] PHASE 7.1: `BidComparisonOrchestratorAgent` (Day 1-3)

- [x] Port `bid-comparison.service.ts` multi-vendor logic.
- [x] Implement vendor bid collection workflows.
- [x] Port automated comparison triggers.
- [x] Implement deadline management.
- [x] Port multi-criteria ranking algorithms.
- [x] **Validation**: Test with 3+ simulated vendor bids.

### [x] PHASE 7.2: Document & System Polish (Day 4-5)

- [x] Polish DocumentGenerationAgent templates.
- [x] Implement template personalization.
- [x] **Validation**: Full bid comparison workflow E2E test.

---

## WEEK 8: CONVERSATION ORCHESTRATION & FINAL VALIDATION

**Focus**: Entry-point routing and system-wide parity.

### [x] PHASE 8.1: `ConversationOrchestratorAgent` (Day 1-3)

- [x] Port `process-conversation-turn.ts` entry-point logic.
- [x] Port `enhanced-convo-router.ts` intent classification.
- [x] Implement state transition management.
- [x] Implement conversation flow recovery.
- [x] **Validation**: Route 20 sample conversations correctly.

### [x] PHASE 8.2: Final E2E Parity Validation (Day 4-5)

- [x] Execute E2E comparison test: new system vs legacy code.
- [x] Document any behavioral differences.
- [x] Final bug fixes and tuning.
- [x] **Validation**: 95%+ parity with legacy system.

---

## WEEK-BY-WEEK QUICK REFERENCE

| Week | Primary Agent(s)               | Legacy Source File(s)                                | Key Deliverable        |
| ---- | ------------------------------ | ---------------------------------------------------- | ---------------------- |
| 1    | VendorProfilingAgent           | `meso.ts` (profile), `vendor-negotiation-profile.ts` | Vendor memory system   |
| 2-3  | MESOGenerationAgent            | `meso.ts` (~2,000 lines)                             | Pareto-optimal offers  |
| 4    | StallRecoveryAgent             | `stall-detector.ts`                                  | Deadlock recovery      |
| 5    | VendorSimulator                | `vendor-simulator.service.ts`, `vendor-agent.ts`     | Testing engine         |
| 6    | EmailNotification, DocumentGen | `email.service.ts`, `pdf-generator.ts`               | Async services         |
| 7    | BidComparisonOrchestrator      | `bid-comparison.service.ts`                          | Multi-vendor system    |
| 8    | ConversationOrchestrator       | `process-conversation-turn.ts`                       | Entry routing + Parity |

---

## CRITICAL DEPENDENCIES

### From Track 1 (Vatsal):

- **Week 1**: `NegotiationState` schema must be finalized.
- **Week 5**: NegotiationOrchestrator must be ready for integration.
- **Week 6**: DecisionAgent output needed for MESO trigger conditions.
- **Week 8**: Final E2E coordination.

### From Track 2 (Yug):

- **Week 6**: Tone/Behavior analysis may influence vendor profiling.
- **Week 6**: RAGContextAgent needed for MESO context assembly.
- **Week 8**: Final E2E testing coordination.

---

## COMPLEXITY NOTES

### MESOGenerationAgent (~2,000 lines from meso.ts)

This is the **most complex agent** in Track 3:

- Pareto-optimality algorithm requires careful porting.
- Multi-parameter optimization (price x delivery x terms).
- Utility equivalence calculations.
- Integration with vendor profiles for personalization.

**Recommendation**: Break into sub-modules:

1. Utility calculator
2. Pareto frontier generator
3. Option packager
4. Preference tuner

### VendorSimulationAgent

Critical for testing but complex:

- Must simulate realistic vendor behavior.
- Integrates with graph state for context-aware responses.
- Supports training and A/B testing.

---

## UPDATE PROTOCOL

1. **Mark [DONE]**: Update this file when a Phase is finished.
2. **Sync Board**: Update `MIGRATION_TASK_BOARD.md` status column immediately after.
3. **Git Workflow**:
   - Cut task branches from **`epic/multi-agent-workflow`**.
   - Target all Pull Requests to **`epic/multi-agent-workflow`** (NOT `main`).
4. **Notify Team**: Post the update in the GitHub issue comment thread.
5. **Escalate Blockers**: If a dependency from another track is blocking you, escalate within 24 hours.

---

## RISK MITIGATION

| Risk                                   | Impact | Mitigation                                             |
| -------------------------------------- | ------ | ------------------------------------------------------ |
| MESO algorithm too complex for 2 weeks | High   | Break into 4 sub-modules; prioritize core utility calc |
| Vendor simulator lacks realism         | Medium | Use real historical data for training patterns         |
| Bid comparison delays                  | Medium | Start with 2-vendor case, expand to N-vendor           |
| Week 8 parity test failures            | High   | Begin parity testing in Week 6 with partial system     |
