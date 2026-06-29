# Backend Technical Implementation Specification: Total Deletion of INSIGHTS Mode

## Executive Summary
This technical specification outlines the precise code removals, validator cleanups, schema edits, and service method purges required in [Accordo-ai-backend](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend) to permanently eliminate `INSIGHTS` mode.

---

## 1. Database Model & Schema Purge Specifications

### 1.1 Model Definition Clean-up: `ChatbotDeal`
* **File Path**: [chatbot-deal.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/models/chatbot-deal.ts)
* **Target Lines**: L22, L125–L130
* **Required Modifications**:
  1. Update TypeScript type definitions:
     ```typescript
     // BEFORE
     export type DealMode = "INSIGHTS" | "CONVERSATION";

     // AFTER
     export type DealMode = "CONVERSATION";
     ```
  2. Update Sequelize column schema definition:
     ```typescript
     // BEFORE (Lines 125-130)
     mode: {
       type: DataTypes.ENUM("INSIGHTS", "CONVERSATION"),
       allowNull: false,
       defaultValue: "INSIGHTS",
     }

     // AFTER
     mode: {
       type: DataTypes.ENUM("CONVERSATION"),
       allowNull: false,
       defaultValue: "CONVERSATION",
     }
     ```

### 1.2 Seed Data Purge
* **File Paths**: 
  * [contracts.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/seeders/data/contracts.ts) (L32)
  * [index.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/seeders/index.ts) (L1928, L1978, L2001, L2046)
* **Required Modifications**: Replace all occurrences of `mode: "INSIGHTS"` with `mode: "CONVERSATION"`.

---

## 2. Request Validator Stripping Specifications

### 2.1 Schema Purge: `chatbot.validator.ts`
* **File Path**: [chatbot.validator.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/modules/chatbot/chatbot.validator.ts)
* **Target Lines**: L17, L37–L40, L247–L255, L264
* **Required Modifications**:
  1. In `createDealSchema` (Lines 37-40): Remove `'INSIGHTS'` from `.valid(...)`.
  2. In `modeQuerySchema` (Lines 247-255):
     ```typescript
     // BEFORE
     export const modeQuerySchema = Joi.object({
       mode: Joi.string()
         .valid('INSIGHTS', 'CONVERSATION')
         .default('INSIGHTS')
         .optional()
     });

     // AFTER
     export const modeQuerySchema = Joi.object({
       mode: Joi.string()
         .valid('CONVERSATION')
         .default('CONVERSATION')
         .optional()
     });
     ```

---

## 3. Service Core & Controller Code Purge Specifications

### 3.1 Controller Method Purge: `chatbot.service.ts`
* **File Path**: [chatbot.service.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/modules/chatbot/chatbot.service.ts)
* **Target Lines**: L1510, L1830–L1845, L2105, L4570–L4575
* **Required Modifications**:
  1. Remove conditional `dealMode === "INSIGHTS"` execution branches in message routing routines.
  2. Delete or refactor demo execution methods (`runDemo`) that throw errors or rely on legacy deterministic INSIGHTS simulation steps.

### 3.2 Route Javadoc & Swagger Cleanup
* **File Paths**:
  * [chatbot.routes.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/modules/chatbot/chatbot.routes.ts) (L303-L315)
  * [swagger.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-backend/src/config/swagger.ts) (L93)
* **Required Modifications**: Remove references to INSIGHTS mode from route comments and API schema definitions.

---
*Backend purge specification approved for implementation.*
