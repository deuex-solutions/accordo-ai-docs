# YUG'S EXECUTION PLAN: 8-Week LangGraph Migration

This document serves as your personalized, daily reference for the Accordo AI migration.

**Core Responsibilities**: The "Senses" (Analysis Layer), the "Voice" (Response Generation), RAG/Vector Systems, and Output Safety.

## SYNERGY & LOGIC MAPPING (CRITICAL)

Before implementing any agent, you must:

1. **Locate the Legacy Workflow**: Identify the specific files in `src/modules/chatbot/engine/` or `src/modules/vector/` that handle this logic today.
2. **Extract Rules**: List the "must-have" business rules from that legacy code.
3. **Trace in Comments**: Add a `@source` comment in your agent code pointing to the original file.

---

## WEEK 1: ANALYSIS INTERFACE REVIEW & FOUNDATION

**Focus**: Understanding the State Contract and preparing analysis interfaces.

### [x] PHASE 1.1: State Contract Review (Day 1-2)

- Review Vatsal's `NegotiationState` interface in `src/modules/chatbot/engine/graph/state.ts`.
- Identify analysis-specific channels needed: `toneAnalysis`, `behavioralSignals`, `concerns`.
- **Validation**: Confirm state channels with Vatsal and Adarsh.

### [x] PHASE 1.2: Analysis Interface Design (Day 3-5)

- Design `ToneAnalysis` interface (11 style signals: formality, urgency, sentiment, etc.).
- Design `BehavioralSignals` interface (concession velocity, momentum, rigidity).
- Design `VendorConcern` interface (semantic issue extraction).
- **Validation**: Get sign-off on analysis output schemas.

---

## WEEK 2: TONE & BEHAVIORAL ANALYSIS (The "Senses")

**Focus**: Port semantic analysis agents from legacy code.

### [x] PHASE 2.1: `ToneAnalysisAgent` (Day 1-3)

- Extract 11 style signals from `tone-detector.ts`.
- Implement formality detection logic (formal vs casual).
- Implement urgency detection (urgent vs relaxed).
- Port sentiment classification (positive, negative, neutral, mixed).
- **Validation**: Unit test against 30 sample vendor messages.

### [x] PHASE 2.2: `BehavioralAnalysisAgent` (Day 4-5)

- Port concession velocity tracking from `behavioral-analyzer.ts`.
- Implement momentum tracking (accelerating vs decelerating).
- Port rigidity detection patterns.
- **Validation**: Compare analysis against historical deal logs.

---

## WEEK 3: CONCERN EXTRACTION & RESPONSE GENERATION

**Focus**: Semantic understanding and voice generation.

### [x] PHASE 3.1: `ConcernExtractionAgent` (Day 1-2)

- Port LLM-based semantic issue identification from `concern-extractor.ts`.
- Implement supply chain concern detection.
- Implement timeline/budget concern extraction.
- **Validation**: Verify concern extraction accuracy on 20 edge cases.

### [x] PHASE 3.2: `ResponseGenerationAgent` (Day 3-5)

- Convert `persona-renderer.ts` to state-aware LLM rendering node.
- Implement PM persona rendering (professional, vendor-friendly tone).
- Port template-based fallback logic.
- Integrate with existing LLM services.
- **Validation**: Generate 20 sample responses, compare against legacy output.

---

## WEEK 4: VALIDATION & SAFETY

**Focus**: Output validation and safety guardrails.

### [x] PHASE 4.1: `ValidationAgent` - Two-Tier Bans (Day 1-3)

- Port strict ban list from `validate-llm-output.ts` (disallowed content).
- Port price normalization validation (format checking).
- Implement policy enforcement layer.
- **Validation**: Test validation against known problematic outputs.

### [x] PHASE 4.2: Safety Refinements (Day 4-5)

- Implement content filtering for off-topic requests.
- Add automated correction mechanisms.
- **Validation**: Full validation test suite passing.

---

## WEEK 5: VECTOR SEARCH & SEMANTIC RETRIEVAL

**Focus**: RAG foundation and historical deal search.

### [x] PHASE 5.0: Intelligence Node Integration (Completed on branch `yug/week5-intelligence-node`)
- [x] Integrate `ToneAnalysisAgent`, `BehavioralAnalysisAgent`, and `ConcernExtractionAgent` into a unified `analyzeSentimentNode`.
- [x] Wire node output to comply with the structured `IntelligenceAnalysis` NegotiationState channel schema.
- [x] **Validation**: Implement and pass intelligence node tests (`tests/ai-evals/intelligence-node.eval.test.ts`).

### [x] PHASE 5.1: `VectorSearchAgent` (Day 1-3)

- [x] Port semantic search logic from `vector.service.ts`.
- [x] Implement embedding generation for deal history.
- [x] Port multi-provider fallback logic.
- [x] Implement search result ranking.
- [x] **Validation**: Test retrieval accuracy on 50 historical queries.

### [x] PHASE 5.1: `VectorSearchAgent` (Day 4-5)

- [x] Coordinate with Vatsal on graph integration points.
- [x] Define vector search state channels.
- [x] **Validation**: Agent ready for orchestrator integration.

---

## WEEK 6: RAG CONTEXT & INTELLIGENCE LAYER

**Focus**: Context assembly and advanced RAG.

### [x] PHASE 6.1: `RAGContextAgent` (Day 1-3)

- [x] Port context assembly logic from `context.service.ts`.
- [x] Implement dynamic context window management.
- [x] Port relevance scoring algorithms.
- [x] Implement multi-source context fusion (deal history + vendor profile + market data).
- [x] **Validation**: Context quality assessment with Adarsh's MESO agent.

### [x] PHASE 6.2: Parallel Execution Setup (Day 4-5)

- [x] Configure parallel analysis execution (tone + behavior + concern).
- [x] Implement merge node for combined analysis results.
- [x] **Validation**: Parallel execution benchmark (target: <2s latency).

---

## WEEK 7: PHRASING HISTORY & SAFETY AUDIT

**Focus**: Historical tracking and system-wide safety.

### [x] PHASE 7.1: `PhrasingHistoryAgent` (Day 1-3)

- [x] Port fingerprinting logic from `phrasing-history.ts`.
- [x] Implement opener deduplication (prevent repetitive openers).
- [x] Port phrase variation tracking.
- [x] **Validation**: Test dedup logic on 100+ historical messages.

### [x] PHASE 7.2: Safety Audit (Day 4-5)

- [x] Conduct system-wide policy audit.
- [x] Review all prompt templates for safety.
- [x] Document safety procedures.
- [x] **Validation**: Safety audit checklist complete.

---

## WEEK 8: API DOCUMENTATION & FINAL E2E

**Focus**: Documentation and final integration testing.

### [x] PHASE 8.1: API Documentation (Day 1-3)

- [x] Update Swagger docs for graph endpoints.
- [x] Document analysis agent outputs.
- [x] Document RAG/vector search APIs.
- [x] **Validation**: API docs reviewed by team.

### [x] PHASE 8.2: Final E2E Integration Testing (Day 4-5)

- [x] Full integration test with Vatsal's orchestrator.
- [x] Integration test with Adarsh's strategy agents.
- [x] Performance benchmarking.
- [x] **Validation**: All Track 2 agents passing E2E tests.

---

## WEEK-BY-WEEK QUICK REFERENCE


| Week | Primary Agent(s)                 | Legacy Source File(s)                         | Key Deliverable            |
| ---- | -------------------------------- | --------------------------------------------- | -------------------------- |
| 1    | Interface Design                 | N/A                                           | Analysis schemas defined   |
| 2    | ToneAnalysis, BehavioralAnalysis | `tone-detector.ts`, `behavioral-analyzer.ts`  | Analysis layer agents      |
| 3    | ConcernExtraction, ResponseGen   | `concern-extractor.ts`, `persona-renderer.ts` | Understanding + Voice      |
| 4    | ValidationAgent                  | `validate-llm-output.ts`                      | Safety layer               |
| 5    | VectorSearchAgent                | `vector.service.ts`                           | Semantic retrieval         |
| 6    | RAGContextAgent                  | `context.service.ts`                          | Context assembly           |
| 7    | PhrasingHistory, Safety Audit    | `phrasing-history.ts`                         | History + Safety           |
| 8    | API Docs, E2E Testing            | N/A                                           | Documentation + Validation |


---

## CRITICAL DEPENDENCIES

### From Track 1 (Vatsal):

- **Week 1**: `NegotiationState` schema must be finalized.
- **Week 5**: Orchestrator must be ready for agent integration.
- **Week 6**: DecisionAgent output needed for ResponseGeneration input.

### From Track 3 (Adarsh):

- **Week 6**: MESOGenerationAgent output may influence context assembly.
- **Week 8**: Final E2E testing coordination.

---

## UPDATE PROTOCOL

1. **Mark [DONE]**: Update this file when a Phase is finished.
2. **Sync Board**: Update `MIGRATION_TASK_BOARD.md` status column immediately after.
3. **Git Workflow**:
  - Cut task branches from `**epic/multi-agent-workflow`**.
  - Target all Pull Requests to `**epic/multi-agent-workflow**` (NOT `main`).
4. **Notify Team**: Post the update in the GitHub issue comment thread.
5. **Escalate Blockers**: If a dependency from another track is blocking you, escalate within 24 hours.

