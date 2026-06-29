# Accordo AI: 3-Track Migration Task Board (Tabular - 8-Week)

This board provides a surgical, tabular view of the LangGraph migration over an 8-week timeline.

---

## 0. GLOBAL CONTRACT (Pre-requisite)


| Week | Task                 | File                                        | Objective                         | Status |
| ---- | -------------------- | ------------------------------------------- | --------------------------------- | ------ |
| 1    | **NegotiationState** | `src/modules/chatbot/engine/graph/state.ts` | Define the global schema contract | [DONE] |


---

## TRACK 1: Vatsal (The Core Foundations)

*Ownership: Decision Brain, Input Processing, and Graph Skeleton.*


| Week | Task                  | Legacy Source File                    | Key Objectives                                 | Status |
| ---- | --------------------- | ------------------------------------- | ---------------------------------------------- | ------ |
| 1    | **StateManagement**   | `negotiation-state-machine.ts`        | Port state transitions & Postgres checkpointer | [DONE] |
| 2    | **OfferParsing**      | `parse-offer.ts`                      | Extract regex & Lakh/Crore logic into node     | [DONE] |
| 3    | **DecisionAgent (1)** | `decide.ts` (~1,700 lines)            | Port core strategic logic (Part 1)             | [DONE] |
| 4    | **DecisionAgent (2)** | `decide.ts`                           | Refine logic & thresholding (Part 2)           | [DONE] |
| 5    | **Orchestrator**      | `src/modules/chatbot/engine/index.ts` | Define StateGraph & initial edge wiring        | [DONE] |
| 6    | **WeightedUtility**   | `weighted-utility.ts`                 | Port parameter scoring logic                   | [DONE] |
| 6    | **HITL Hooks**        | N/A                                   | Add `interrupt_before` for high-value deals    | [DONE] |
| 7    | **Persistence**       | N/A                                   | Optimize checkpointer & state compression      | [DONE] |
| 8    | **Infrastructure**    | `.env`, `Dockerfile`                  | Update backend configs for agentic mode        | [DONE] |


---

## TRACK 2: Yug (Intelligence & Interaction)

*Ownership: Analysis, LLM Rendering, Safety, and Retrieval.*


| Week | Task                   | Legacy Source File       | Key Objectives                             | Status |
| ---- | ---------------------- | ------------------------ | ------------------------------------------ | ------ |
| 1    | **ToneAnalysis**       | `tone-detector.ts`       | 11 style signals & formality detection     | [DONE] |
| 2    | **BehavioralAnalysis** | `behavioral-analyzer.ts` | Concession velocity & momentum tracking    | [DONE] |
| 3    | **ConcernExtraction**  | `concern-extractor.ts`   | Semantic identification of vendor pain points | [DONE] |
| 4    | **ResponseGeneration** | `persona-renderer.ts`    | PM persona LLM rendering with fallbacks    | [DONE] |
| 4    | **Validation**         | `validate-llm-output.ts` | Two-tier bans & price normalization        | [DONE] |
| 5    | **VectorSearch**       | `vector.service.ts`      | Semantic retrieval across historical deals | [DONE] |
| 6    | **RAGContext**         | `context.service.ts`     | Context window assembly optimization       | [DONE] |
| 7    | **PhrasingHistory**    | `phrasing-history.ts`    | Fingerprinting & opener dedup logic        | [DONE] |
| 7    | **Safety Audit**       | N/A                      | System-wide policy & prompt audit          | [DONE] |
| 8    | **API Docs**           | N/A                      | Update Swagger for graph endpoints         | [DONE] |


---

## TRACK 3: Adarsh (Strategy & Systems)

*Ownership: Advanced Algorithms, Simulation, and Auxiliary Services.*


| Week | Task                  | Legacy Source File             | Key Objectives                            | Status |
| ---- | --------------------- | ------------------------------ | ----------------------------------------- | ------ |
| 1    | **VendorProfiling**   | `meso.ts` (Profile section)    | Long-term preference learning persistence | [DONE] |
| 2    | **MESOGen (1)**       | `meso.ts` (~2,000 lines)       | Port Pareto-optimal algorithm (Part 1)    | [DONE] |
| 3    | **MESOGen (2)**       | `meso.ts`                      | Port Pareto-optimal algorithm (Part 2)    | [DONE] |
| 4    | **StallRecovery**     | `stall-detector.ts`            | Deadlock detection & recovery probes      | [DONE] |
| 5    | **VendorSimulator**   | `vendor-simulator.service.ts`  | Update simulator for graph interaction    | [DONE] |
| 6    | **EmailNotification** | `email.service.ts`             | Async node for deal update emails         | [DONE] |
| 7    | **DocumentGen**       | `pdf-generator.ts`             | Background node for deal summary PDFs     | [DONE] |
| 7    | **BidComparison**     | `bid-comparison.service.ts`    | Port multi-vendor bid collection logic    | [DONE] |
| 8    | **ConvoOrchestrator** | `process-conversation-turn.ts` | Entry-point routing & intent management   | [DONE] |
| 8    | **Parity Validation** | N/A                            | Final E2E comparison vs legacy code       | [DONE] |


