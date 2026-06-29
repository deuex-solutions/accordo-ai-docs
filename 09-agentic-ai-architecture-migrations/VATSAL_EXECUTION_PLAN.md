# VATSAL'S EXECUTION PLAN: 8-Week LangGraph Migration

This document serves as your personalized, daily reference for the Accordo AI migration. 

**Core Responsibilities**: The "Brain" (Decision Engine) and the "Skeleton" (Graph Orchestration).

## SYNERGY & LOGIC MAPPING (CRITICAL)
Before implementing any agent, you must:
1.  **Locate the Legacy Workflow**: Identify the specific files in `src/modules/chatbot/convo/` or `src/modules/chatbot/engine/` that handle this logic today.
2.  **Extract Rules**: List the "must-have" business rules from that legacy code.
3.  **Trace in Comments**: Add a `@source` comment in your agent code pointing to the original file.

---

## WEEK 1: GLOBAL CONTRACT & STATE ENGINE
**Focus**: Locking the schema and building the persistence layer.

### [x] PHASE 1.1: Define `NegotiationState` (Day 1-2)
- [x] Research all current `Negotiation` and `Deal` interfaces in `src/types/`.
- [x] Create `src/modules/chatbot/engine/graph/state.ts`.
- [x] Define the `NegotiationState` interface (Channels: `messages`, `parsedOffer`, `decision`, `dealStatus`, `history`).
- [x] **Validation**: Get sign-off from Yug and Adarsh.

### [x] PHASE 1.2: `StateManagementAgent` (Day 3-5)
- [x] Extract state transition logic from `negotiation-state-machine.ts`.
- [x] Implement the `StateManagementNode` as a LangGraph node.
- [x] Setup `PostgresSaver` from `@langchain/langgraph-checkpoint-postgres`.
- [x] **Validation**: Unit test state transitions (e.g., OPEN -> ACCEPTED).

---

## WEEK 2: INPUT PARSING (The "Eyes")
**Focus**: Structural extraction from raw vendor messages.

### [x] PHASE 2.1: `OfferParsingAgent` (Day 1-5)
- [x] Port regex patterns from `parse-offer.ts`.
- [x] Implement Indian currency normalization (3.5L -> 350,000).
- [x] Handle "Net 30", "Advance 50%" extraction for payment terms.
- [x] **Validation**: Run parser against 50 sample vendor messages.

---

## WEEK 3: THE BRAIN - PART 1 (Decision Engine)
**Focus**: Strategic logic extraction (ACCEPT/COUNTER).

### [x] PHASE 3.1: `DecisionAgent` Foundation (Day 1-5)
- [x] Extract target price vs. current offer comparison from `decide.ts`.
- [x] Implement the basic "Accept" trigger if `price <= target_price`.
- [x] Implement the "Escalate" trigger if `round_number > 5`.
- [x] **Validation**: Verify decision output for 5 static scenarios.

---

## WEEK 4: THE BRAIN - PART 2 (Logic Refinement)
**Focus**: Complex strategic moves.

### [x] PHASE 4.1: `DecisionAgent` Advanced (Day 1-5)
- [x] Port the weighted utility scoring from `weighted-utility.ts`.
- [x] Implement "Walk Away" logic based on utility thresholds.
- [x] Add post-decision safety guard (Strict Ceiling check).
- [x] **Validation**: Full parity test against 100 legacy `decide.ts` logs.

---

## WEEK 5: THE SKELETON (Graph Assembly)
**Focus**: Wiring all 19 nodes together.

### [x] PHASE 5.1: Main Orchestrator (Day 1-5)
- [x] Initialize `StateGraph` in `src/modules/chatbot/engine/index.ts`.
- [x] Add nodes from Track 2 (Yug) and Track 3 (Adarsh) as they arrive.
- [x] Define conditional edges (e.g., `if (parsedOffer) -> analyzeBehavior`).
- [x] **Validation**: Visualize the graph using `graph.drawMermaid()`.

---

## WEEK 6: UTILITY & HITL HOOKS
**Focus**: Human-in-the-loop and parameter scoring.

### [x] PHASE 6.1: `WeightedUtilityNode` (Day 1-3)
- [x] Finalize the multi-parameter scoring node (Price/Delivery/Terms).

### [x] PHASE 6.2: HITL Hooks (Day 4-5)
- [x] Implement `interrupt_before` for deals > ₹10L.
- [x] Create the "Approval Required" state transition.
- [x] **Validation**: Manual interruption test in the dev environment.

---

## WEEK 7: PERSISTENCE & TUNING
**Focus**: Optimization and state compression.

### [x] PHASE 7.1: Persistence Tuning (Day 1-5)
- [x] Implement state compression for long conversation threads.
- [x] Optimize Postgres checkpointer queries.
- [x] **Validation**: Benchmarking state load/save times.

---

## WEEK 8: INFRASTRUCTURE & LAUNCH
**Focus**: Production readiness.

### [x] PHASE 8.1: Infrastructure Update (Day 1-3)
- [x] Update `.env` with new LangGraph environment variables.
- [x] Update `Dockerfile` to include any new native dependencies.

### [x] PHASE 8.2: Launch & Final Audit (Day 4-5)
- [x] Final E2E system check.
- [x] **DEPLOY**.

---

## UPDATE PROTOCOL
1. **Mark [DONE]**: Update this file when a Phase is finished.
2. **Sync Board**: Update `MIGRATION_TASK_BOARD.md` status column immediately after.
3. **Git Workflow**: 
   - Cut task branches from **`epic/multi-agent-workflow`**.
   - Target all Pull Requests to **`epic/multi-agent-workflow`** (NOT `main`).
4. **Notify Team**: Post the update in the GitHub issue comment thread.
