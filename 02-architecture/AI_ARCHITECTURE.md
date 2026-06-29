# Accordo AI Architecture

> **Enterprise AI System Design for Procurement Negotiation**
>
> Version: 1.0 | Date: May 2026 | Classification: External Consultant Briefing

---

## Executive Summary

Accordo is an **AI-powered procurement negotiation platform** that automates deal negotiations between procurement managers and vendors. The system combines a **deterministic utility-based decision engine** with **LLM-driven conversational AI** to deliver consistent, explainable, and human-like negotiation outcomes.

### Key Differentiators

| Capability           | Traditional Systems  | Accordo AI                            |
| -------------------- | -------------------- | ------------------------------------- |
| Decision Making      | Rule-based (brittle) | Utility-optimized with explainability |
| Vendor Interaction   | Email/phone (slow)   | Real-time AI chat (24/7)              |
| Offer Evaluation     | Manual comparison    | Automated utility scoring             |
| Strategy Adaptation  | Static playbooks     | Dynamic tone-aware responses          |
| Negotiation Outcomes | Unpredictable        | Consistent, data-driven results       |

---

## High-Level AI Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                    │
│  ┌────────────────┐  ┌─────────────────┐  ┌────────────────────┐            │
│  │  Web App       │  │  Vendor Portal  │  │  Mobile (Future)   │            │
│  │  (React/Vite)  │  │  (Token-based)  │  │                    │            │
│  └────────┬───────┘  └────────┬────────┘  └──────────┬─────────┘            │
└───────────┼──────────────────┼────────────────────────┼──────────────────────┘
            │                  │                        │
            └──────────────────┴────────────────────────┘
                               │
                    HTTP/HTTPS (REST API)
                               │
┌──────────────────────────────┼──────────────────────────────────────────────┐
│                         API GATEWAY (Express)                                │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │  - JWT Authentication                                             │     │
│  │  - Rate Limiting                                                  │     │
│  │  - Request Routing                                                │     │
│  │  - CORS & Security Headers                                        │     │
│  └──────────────────────────────────────────────────────────────────┘     │
└───────────────────────────────┼──────────────────────────────────────────────┘
                                │
┌───────────────────────────────┼──────────────────────────────────────────────┐
│                      AI ORCHESTRATION LAYER                                │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                    NEGOTIATION ENGINE                               │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │    │
│  │  │   INSIGHTS   │  │ CONVERSATION │  │    MESO      │            │    │
│  │  │    MODE      │  │    MODE      │  │   SYSTEM     │            │    │
│  │  │              │  │              │  │              │            │    │
│  │  │ Deterministic│  │ Deterministic│  │ Multi-Issue  │            │    │
│  │  │ + Templates  │  │ + LLM Render │  │ Options Gen  │            │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                    VENDOR SIMULATION ENGINE                           │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │    │
│  │  │   Policy     │  │   Scenario   │  │   Auto-Reply │            │    │
│  │  │   Engine     │  │   Detection  │  │   Generator  │            │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                    INTELLIGENCE LAYER                               │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │    │
│  │  │   Intent     │  │    Tone      │  │   Offer      │            │    │
│  │  │  Classification│ │  Detection  │  │   Parsing    │            │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │    │
│  │  │   Utility    │  │  Explain-    │  │   RAG        │            │    │
│  │  │ Calculation  │  │  ability     │  │   Context    │            │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │    │
│  └────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                │
┌───────────────────────────────┼──────────────────────────────────────────────┐
│                         LLM GATEWAY LAYER                                    │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │    │
│  │  │   Ollama     │  │   OpenAI     │  │   AWS        │             │    │
│  │  │   (Local)    │  │  (Fallback)  │  │   Bedrock    │             │    │
│  │  │   qwen3      │  │   GPT-4o     │  │  (Embeddings)│             │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘             │    │
│  │                                                                   │    │
│  │  ┌──────────────────────────────────────────────────────────┐    │    │
│  │  │              OUTPUT VALIDATION & SAFETY                   │    │    │
│  │  │  • Banned word filtering    • Price constraint checking    │    │    │
│  │  │  • Length validation        • Anti-fabrication rules       │    │    │
│  │  │  • Template fallback on failure                            │    │    │
│  │  └──────────────────────────────────────────────────────────┘    │    │
│  └────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                │
┌───────────────────────────────┼──────────────────────────────────────────────┐
│                         DATA & INFRASTRUCTURE                              │
│  ┌────────────────┐  ┌─────────────────┐  ┌────────────────────┐            │
│  │  PostgreSQL    │  │     Redis       │  │   Vector Store     │            │
│  │  (Primary DB)  │  │   (Pub/Sub)     │  │  (Embeddings)      │            │
│  └────────────────┘  └─────────────────┘  └────────────────────┘            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Core AI Components

### 1. Deterministic Negotiation Engine

The **heart of the system** — makes all strategic decisions without relying on LLM reasoning.

**Key Capabilities:**

- **Offer Parsing**: Regex-based extraction of price, terms, and delivery from vendor messages
- **Utility Calculation**: Weighted scoring of price + payment terms + delivery
- **Decision Algorithm**: Threshold-based action selection (ACCEPT / COUNTER / WALK_AWAY)
- **Monotonic Constraints**: Ensures counter-offers always improve for Accordo

**Decision Flow:**

```
Vendor Message
    ↓
┌─────────────────┐
│  parseOffer()   │ ← Regex extracts: unit_price, payment_terms, delivery_days
└────────┬────────┘
         ↓
┌─────────────────┐
│ calculateUtility()│ ← Weighted: priceUtility*0.6 + termsUtility*0.3 + deliveryUtility*0.1
└────────┬────────┘
         ↓
┌─────────────────┐
│  decideNextMove() │ ← Thresholds: utility >= 75 → ACCEPT, < 30 → WALK_AWAY, else COUNTER
└────────┬────────┘
         ↓
┌─────────────────┐
│  buildCounter() │ ← Humanized pricing within [targetPrice, maxAcceptablePrice]
└─────────────────┘
```

### 2. LLM Persona Renderer (CONVERSATION Mode)

**Hard Boundary Design:** The LLM is explicitly treated as **untrusted** and only used for message rendering.

**What the LLM Sees:**

- `NegotiationIntent` (filtered decision data)
- Vendor's message content
- Deal title, vendor name, product category
- `allowedPrice` only for COUNTER actions

**What the LLM NEVER Sees:**

- Utility scores
- Weights and thresholds
- Target price, max acceptable price
- Internal configuration

**Safety Controls:**

```
┌─────────────────────────────────────────────────────────┐
│                    LLM OUTPUT VALIDATOR                  │
├─────────────────────────────────────────────────────────┤
│  1. Banned word check (internal terms, competitor refs) │
│  2. Price constraint validation (matches allowed range)   │
│  3. Length check (≤160 words)                             │
│  4. Anti-fabrication (no invented vendor concerns)       │
│  5. Date format check (Month Day, not YYYY-MM-DD)       │
└─────────────────────────────────────────────────────────┘
         ↓
    ┌────────┴────────┐
    ↓                 ↓
Validation OK    Validation Failed
    ↓                 ↓
Return message   Use fallback template
```

### 3. MESO (Multi-Issue Offer) System

Generates **Pareto-optimal option packages** for complex negotiations.

**Concept:** Instead of single-issue haggling, present vendors with 3-4 complete packages trading off:

- Price vs. Payment Terms
- Price vs. Delivery Timeline
- Price vs. Warranty/Support

**Generation Algorithm:**

```typescript
// Each MESO option varies one dimension while keeping utility constant
const mesoOptions = [
  { price: targetPrice, paymentTerms: 30, deliveryDays: 14 }, // Balanced
  { price: targetPrice * 0.98, paymentTerms: 15, deliveryDays: 21 }, // Better price, stricter terms
  { price: targetPrice * 1.02, paymentTerms: 60, deliveryDays: 7 }, // Worse price, better terms
];
```

### 4. Vendor Simulation Engine

**Purpose:** Generate realistic vendor personas for testing and training.

**Components:**

- **Policy Engine**: Defines vendor constraints (min price, max concession step)
- **Scenario Detector**: Classifies vendor behavior (HARD / SOFT / WALK_AWAY)
- **Auto-Reply Generator**: LLM-based vendor responses using the same safety controls

### 5. Vector & RAG System

**Embeddings Pipeline:**

1. Generate embeddings for all negotiation messages
2. Store in `message_embeddings` table
3. Enable semantic similarity search

**RAG Use Cases:**

- Retrieve similar past negotiations for strategy hints
- Context-aware responses based on historical patterns
- Deal outcome prediction from similar scenarios

**Providers:**

- Local: HuggingFace Transformers (ONNX)
- Cloud: OpenAI Embeddings / AWS Bedrock

---

## Data Flow: Complete Negotiation Lifecycle

### Scenario: Vendor Makes Counter-Offer

```
┌──────────┐                                              ┌────────────┐
│  Vendor  │ ──"We can do $95,000 with Net 45 terms"──→ │  Accordo   │
│  Portal  │                                              │  Backend   │
└──────────┘                                              └─────┬──────┘
                                                                │
                                                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 1: OFFER PARSING                                                   │
│  Input: "We can do $95,000 with Net 45 terms"                           │
│  Output: { total_price: 95000, payment_terms: 45, delivery_days: null } │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 2: UTILITY CALCULATION                                            │
│  Target: $90,000 | Max Acceptable: $100,000 | Vendor Offer: $95,000      │
│  Price Utility: 50% (midpoint of range)                                 │
│  Terms Utility: 60% (Net 45 is reasonable)                              │
│  Overall Utility: 54% → DECISION: COUNTER                               │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 3: DECISION & INTENT BUILDING                                     │
│  Action: COUNTER                                                        │
│  Counter Price: $92,000 (humanized via humanRoundPrice)                 │
│  Reasoning: "what we can work with" (vague, self-referential)            │
│  Tone: Professional but not overly formal                               │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 4: LLM RENDERING (if CONVERSATION mode)                           │
│  System Prompt: "You are a procurement negotiation assistant..."         │
│  Temperature: 0.5 (consistent, not creative)                            │
│  Input: NegotiationIntent + vendor message                              │
│  Output: "Thanks for the update. Based on the quantities discussed,     │
│           we could work with $92,000 if the terms align. What do you   │
│           think of moving forward at that figure?"                     │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 5: VALIDATION & DELIVERY                                          │
│  ✓ Price within allowed range ($92,000 vs target $90k-$100k)            │
│  ✓ No banned internal terms                                              │
│  ✓ Length: 28 words (under 160 limit)                                   │
│  ✓ No fabricated concerns                                                │
│  ✓ Date format check passed                                              │
│  Delay: 6-12s (simulated typing, based on complexity)                   │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────┐                                              ┌────────────┐
│  Vendor  │ ←──────────"we could work with $92,000..."─────│  Accordo   │
│  Portal  │                                              │  Backend   │
└──────────┘                                              └────────────┘
```

---

## AI Safety & Guardrails

### 1. Deterministic Safety

- **Monotonic Floor**: Counter-offers never worsen from Accordo's perspective
- **Threshold Guards**: Auto-accept above utility 75, walk-away below 30
- **Price Floors**: MESO prices floored at targetPrice, delivery floored at 1 day

### 2. LLM Safety

- **Hard Boundary**: Intent object filters all sensitive data
- **Output Validator**: Multi-layer validation before delivery
- **Fallback Templates**: 40+ humanized variations ready if LLM fails
- **No Fabrication Rule**: LLM cannot invent vendor motivations

### 3. Behavioral Safety

- **Tone Detection**: Casual short messages get short direct replies
- **Refusal Handling**: Tracks vendor refusals, escalates after 3 consecutive
- **Stall Detection**: Monitors inactive deals, sends gentle reminders

---

## Integration Architecture

### Bid Analysis Integration

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   Requisition   │────→│   Bid Analysis  │────→│   Negotiation   │
│   Published     │      │   Collects      │      │   Auto-Creates  │
│                 │      │   Vendor Bids   │      │   Deals         │
└─────────────────┘      └─────────────────┘      └─────────────────┘
```

### Email Notification Integration

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   Deal Accepted │────→│   Redis Pub/Sub │────→│   Email Service   │
│   (Negotiation) │      │   Event:        │      │   Sends Summary  │
│                 │      │   deal.accepted │      │   PDF to Vendor  │
└─────────────────┘      └─────────────────┘      └─────────────────┘
```

### Vector Search Integration

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   New Message   │────→│   Embedding     │────→│   Similar Past  │
│   Received      │      │   Generated     │      │   Deals Found   │
└─────────────────┘      └─────────────────┘      └─────────────────┘
                                │
                                ↓
                    ┌─────────────────┐
                    │   Context Added │
                    │   to RAG Prompt │
                    └─────────────────┘
```

---

## Performance Characteristics

| Metric                | Target      | Current     |
| --------------------- | ----------- | ----------- |
| Offer Parsing Latency | <50ms       | ~20ms       |
| Utility Calculation   | <10ms       | ~5ms        |
| LLM Response (local)  | 3-8s        | 4-6s        |
| LLM Response (OpenAI) | 1-3s        | 1-2s        |
| End-to-End Message    | <15s        | 8-12s       |
| Concurrent Deals      | 100+        | 50+ tested  |
| Token Usage (avg)     | <200 tokens | ~150 tokens |

---

## Scalability Considerations

### Horizontal Scaling

- **Stateless Backend**: Any instance can handle any request
- **Database**: Connection pooling (max 20), query optimization
- **LLM Gateway**: Can run multiple Ollama instances behind load balancer

### Caching Strategy

- **Deal Configs**: Short-term cache (30s) for active negotiations
- **Embeddings**: Permanent storage, semantic index for retrieval
- **Templates**: In-memory cache for fallback templates

### Future Enhancements

- **Model Fine-tuning**: Train on negotiation outcomes for better performance
- **Multi-Agent**: Parallel vendor negotiations with shared learning
- **Predictive Analytics**: Deal outcome probability before first message
- **Voice Integration**: Speech-to-text for vendor phone interactions

---

## Technology Stack Summary

| Layer       | Technology            | Purpose                    |
| ----------- | --------------------- | -------------------------- |
| Runtime     | Node.js 18+           | JavaScript execution       |
| Framework   | Express.js            | HTTP API framework         |
| Language    | TypeScript 5.9        | Type safety                |
| Database    | PostgreSQL 14+        | Primary data store         |
| ORM         | Sequelize 6.37        | Database abstraction       |
| Local LLM   | Ollama (qwen3)        | Primary LLM inference      |
| Cloud LLM   | OpenAI GPT-4o         | Fallback for complex cases |
| Embeddings  | HuggingFace / Bedrock | Vector generation          |
| Message Bus | Redis Pub/Sub         | Async event handling       |
| Validation  | Joi + Zod             | Schema validation          |
| Logging     | Pino / Winston        | Structured logging         |

---

## Key Design Principles

1. **Deterministic First**: AI augments decision-making but doesn't replace it
2. **Safety by Design**: Hard boundaries prevent information leakage
3. **Graceful Degradation**: System works even when LLM fails
4. **Explainability**: Every decision can be traced and explained
5. **Vendor-Centric**: Natural conversation flow, no robotic responses
6. **Auditability**: Complete negotiation history with decision logs

---

## Document Control

| Version | Date       | Author              | Changes                                       |
| ------- | ---------- | ------------------- | --------------------------------------------- |
| 1.0     | 2026-05-19 | Accordo Engineering | Initial external consultant briefing document |

---

_End of Document_
