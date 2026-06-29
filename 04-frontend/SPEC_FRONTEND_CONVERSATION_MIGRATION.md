# Frontend Technical Implementation Specification: Total Deletion of INSIGHTS Mode

## Executive Summary
This technical specification details the complete removal of `INSIGHTS` mode UI elements, component conditionals, type definitions, and client service parameters across [Accordo-ai-frontend](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend).

---

## 1. Deal Wizard Component Clean-up Specifications

### 1.1 Mode Selector Dropdown Deletion: `StepOne.tsx`
* **File Path**: [StepOne.tsx](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/components/chatbot/deal-wizard/StepOne.tsx)
* **Target Lines**: L380–L402
* **Required Modifications**:
  1. Delete the entire HTML block containing `<select id="mode">` and `<option value="INSIGHTS">`.
  2. Remove the ternary helper text checking `data.mode === 'INSIGHTS'`.
  3. Hardcode initial state data so `mode` is implicitly set to `'CONVERSATION'`.

### 1.2 Review Step Display Clean-up: `ReviewStep.tsx`
* **File Path**: [ReviewStep.tsx](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/components/chatbot/deal-wizard/ReviewStep.tsx)
* **Target Lines**: L198–L206
* **Required Modifications**:
  1. Remove `data.stepOne.mode === 'INSIGHTS'` conditional logic.
  2. Display static text: `"Conversational AI Negotiation"`.

---

## 2. Component & Workspace Clean-up Specifications

### 2.1 Component Deletion: `NegotiationInsights.tsx`
* **File Path**: [NegotiationInsights.tsx](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/components/chatbot/NegotiationInsights.tsx)
* **Action**: Delete or unmount this standalone demo insights panel component completely from the project workspace.

### 2.2 Dashboard Clean-up: `NegotiationRoom.tsx`
* **File Path**: [NegotiationRoom.tsx](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/pages/chatbot/NegotiationRoom.tsx)
* **Target Lines**: L1420–L1465
* **Required Modifications**: Remove references to demo mode insights buttons, simulation controls, and legacy insight badges.

---

## 3. API Services & Type Definition Clean-up Specifications

### 3.1 Type Definition Deletion: `chatbot.ts` & `vendorChat.service.ts`
* **File Paths**:
  * [chatbot.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/types/chatbot.ts) (L13)
  * [vendorChat.service.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/services/vendorChat.service.ts) (L85)
* **Required Modifications**:
  ```typescript
  // BEFORE
  export type DealMode = 'INSIGHTS' | 'CONVERSATION';

  // AFTER
  export type DealMode = 'CONVERSATION';
  ```

### 3.2 Service Query Param Clean-up: `chatbot.service.ts`
* **File Path**: [chatbot.service.ts](file:///Users/safayavatsal/Downloads/deuex/Accordo-AI/Accordo-ai-frontend/src/services/chatbot.service.ts)
* **Target Lines**: L457–L465
* **Required Modifications**: Update API client methods (`sendMessage`) to remove `mode=INSIGHTS` dual-branch parameter logic.

---
*Frontend purge specification approved for implementation.*
