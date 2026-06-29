# Architectural Purge Roadmap: Complete Removal & Deletion of INSIGHTS Mode

## Executive Overview
This roadmap outlines the systematic, complete deletion and purging of **`INSIGHTS` mode** (legacy automated demo/simulation mode) from the Accordo AI platform. Rather than maintaining backward-compatible fallbacks or dual-mode option checks, `INSIGHTS` mode will be completely excised across all layers—database schemas, seeders, validation schemas, backend controllers, frontend wizards, and UI dashboards. The platform will operate exclusively through standard conversational negotiation.

---

## Purge Architecture & Transition Model

```
  +-------------------------------------------------------------------------+
  |                        LEGACY DUAL-MODE ARCHITECTURE                    |
  |  [ Database / Validators / UI ] ---> ENUM("INSIGHTS", "CONVERSATION")    |
  |                                        |                 |              |
  |                                        v                 v              |
  |                               Deterministic      LLM Conversation       |
  +-------------------------------------------------------------------------+
                                    |
                                    v (COMPLETE PURGE OF INSIGHTS BRANCHES)
  +-------------------------------------------------------------------------+
  |                       UNIFIED SINGLE-FRAMEWORK ARCHITECTURE             |
  |  [ Database / Validators / UI ] ---> Standard Conversational Negotiation |
  |                                 |                                       |
  |                                 v                                       |
  |                       Natural LLM Negotiation                           |
  |      (MAUT valuation engines execute as background validation only)      |
  +-------------------------------------------------------------------------+
```

---

## Phase 1: Complete Backend Schema & Validator Excision

### Step 1.1: Database Schema & Enum Deletion
* **Target File**: [chatbot-deal.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/models/chatbot-deal.ts)
* **Action**: 
  * Remove `"INSIGHTS"` from `DealMode` type definitions (`export type DealMode = "CONVERSATION";` or deprecate the redundant mode property).
  * Update Sequelize model column definitions in `ChatbotDeal` to strip `"INSIGHTS"` from `DataTypes.ENUM`.

### Step 1.2: Request Validator Strict Stripping
* **Target File**: [chatbot.validator.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/modules/chatbot/chatbot.validator.ts)
* **Action**:
  * Remove `'INSIGHTS'` from all `Joi.string().valid(...)` rules (Lines 17, 37, 249, 264).
  * Hardcode or default valid mode exclusively to `'CONVERSATION'` and reject any incoming payloads referencing `INSIGHTS`.

### Step 1.3: Seeder & Mock Data Purge
* **Target Files**: 
  * [contracts.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/seeders/data/contracts.ts)
  * [index.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/seeders/index.ts)
* **Action**:
  * Replace or delete all hardcoded `mode: "INSIGHTS"` properties in database seed generators with `"CONVERSATION"`.

---

## Phase 2: Backend Controller & Service Code Clean-up

### Step 2.1: Message Processing & Controller Purge
* **Target Files**:
  * [chatbot.service.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/modules/chatbot/chatbot.service.ts)
  * [chatbot.routes.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/modules/chatbot/chatbot.routes.ts)
* **Action**:
  * Excise conditional branches matching `if (dealMode === "INSIGHTS")` or `if (resetDeal.mode !== "INSIGHTS")`.
  * Delete demo simulation runner functions that rely on deterministic `INSIGHTS` processing.

### Step 2.2: API Documentation & Swagger Deletion
* **Target File**: [swagger.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/config/swagger.ts)
* **Action**:
  * Remove references to `INSIGHTS` mode from OpenAPI parameter specs and schemas.

---

## Phase 3: Frontend Component & Type Definition Deletion

### Step 3.1: Deal Wizard Selector & Conditional Removal
* **Target Files**:
  * [StepOne.tsx](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/components/chatbot/deal-wizard/StepOne.tsx)
  * [ReviewStep.tsx](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/components/chatbot/deal-wizard/ReviewStep.tsx)
* **Action**:
  * Permanently delete the mode `<select>` dropdown (Lines 381-396 in `StepOne.tsx`).
  * Remove conditional `mode === 'INSIGHTS'` checks in `ReviewStep.tsx`.

### Step 3.2: Negotiation Room UI Clean-up
* **Target Files**:
  * [NegotiationRoom.tsx](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/pages/chatbot/NegotiationRoom.tsx)
  * [NegotiationInsights.tsx](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/components/chatbot/NegotiationInsights.tsx)
* **Action**:
  * Remove dedicated `NegotiationInsights.tsx` rendering components or references from negotiation dashboards.
  * Strip simulation and demo trigger buttons.

### Step 3.3: API Client & Service Deletions
* **Target Files**:
  * [chatbot.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/types/chatbot.ts)
  * [chatbot.service.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/services/chatbot.service.ts)
  * [vendorChat.service.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/services/vendorChat.service.ts)
* **Action**:
  * Update `DealMode` union types from `'INSIGHTS' | 'CONVERSATION'` to exclusively `'CONVERSATION'`.

---

## Phase 4: Clean-up Verification Protocol

1. **Static Analysis & Type Check**: Run TypeScript compiler (`tsc`) on both frontend and backend to verify zero remaining references to `INSIGHTS`.
2. **Database Clean Execution**: Re-run database migrations and seeders to confirm clean table instantiation without legacy enum values.
3. **End-to-End Deal Flow**: Execute complete negotiation deal flows in the browser to ensure seamless natural conversation execution.

---
*Purge implementation roadmap approved for total removal phase.*
