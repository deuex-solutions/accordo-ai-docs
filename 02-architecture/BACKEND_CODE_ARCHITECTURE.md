# Accordo Backend Code Architecture

> **Module-to-Code Mapping for AI & Core Systems**
>
> Version: 1.0 | Date: May 2026 | Classification: Development Reference

---

## Quick Reference: Where Code Lives

### AI/Negotiation System

| Component                | Directory Path                | Key Files                                                                |
| ------------------------ | ----------------------------- | ------------------------------------------------------------------------ |
| **Decision Engine**      | `src/modules/chatbot/engine/` | `decide.ts`, `utility.ts`, `parseOffer.ts`                               |
| **Conversation Service** | `src/modules/chatbot/convo/`  | `conversation-service.ts`, `processConversationTurn.ts`                  |
| **LLM Renderer**         | `src/llm/`                    | `persona-renderer.ts`, `validate-llm-output.ts`, `fallback-templates.ts` |
| **Intent Builder**       | `src/negotiation/intent/`     | `build-negotiation-intent.ts`                                            |
| **Vendor Simulator**     | `src/modules/chatbot/vendor/` | `vendor-policy.ts`, `scenario-detector.ts`, `vendor-agent.ts`            |
| **Typing Delay**         | `src/delivery/`               | `simulateTypingDelay.ts`                                                 |
| **Negotiation Logging**  | `src/metrics/`                | `logNegotiationStep.ts`                                                  |

### Supporting AI Systems

| Component               | Directory Path                | Key Files                                                    |
| ----------------------- | ----------------------------- | ------------------------------------------------------------ |
| **LLM Gateway Service** | `src/services/`               | `llm.service.ts`, `openai.service.ts`                        |
| **Vector/RAG**          | `src/modules/vector/`         | `vector.service.ts`, `vector.controller.ts`                  |
| **Email Notifications** | `src/services/`               | `email.service.ts`                                           |
| **Bid Analysis**        | `src/modules/bid-analysis/`   | `bid-analysis.service.ts`, `bid-analysis.controller.ts`      |
| **Bid Comparison**      | `src/modules/bid-comparison/` | `bid-comparison.service.ts`, `scheduler/deadline-checker.ts` |

---

## Directory Tree: AI & Core Modules

```
Accordo-ai-backend/src/
│
├── modules/
│   ├── chatbot/                          ← PRIMARY NEGOTIATION MODULE
│   │   ├── engine/                       ← Deterministic Decision Engine
│   │   │   ├── decide.ts                  # Core decision algorithm (ACCEPT/COUNTER/WALK_AWAY)
│   │   │   ├── utility.ts                 # Weighted utility calculation
│   │   │   ├── parseOffer.ts              # Regex-based offer extraction
│   │   │   ├── weightedUtility.ts         # Price/terms/delivery scoring
│   │   │   ├── meso.ts                    # Multi-issue offer generation
│   │   │   ├── humanizePrice.ts           # Price formatting (e.g., 95234 → 95000)
│   │   │   ├── parseOfferRegex.ts         # Advanced regex patterns for extraction
│   │   │   ├── offerAccumulator.ts        # Offer accumulation and validation
│   │   │   ├── detectVendorTone.ts        # Tone classification (formal/casual)
│   │   │   ├── batnaCalculator.ts         # Best Alternative calculation
│   │   │   └── index.ts                   # Engine exports
│   │   │
│   │   ├── convo/                          ← CONVERSATION MODE (LLM)
│   │   │   ├── conversation-service.ts    # Main conversation orchestration
│   │   │   ├── processConversationTurn.ts # Single turn processing
│   │   │   ├── conversation-state.ts      # State machine (WAITING_FOR_OFFER → NEGOTIATING → TERMINAL)
│   │   │   ├── conversation-templates.ts  # 56 response templates
│   │   │   ├── llamaReplyGenerator.ts    # LLM integration
│   │   │   └── types.ts                   # Conversation type definitions
│   │   │
│   │   ├── vendor/                         ← VENDOR SIMULATION
│   │   │   ├── vendor-policy.ts            # Vendor constraint policies
│   │   │   ├── scenario-detector.ts        # HARD/SOFT/WALK_AWAY detection
│   │   │   ├── vendor-agent.ts             # Simulated vendor responder
│   │   │   ├── simulator.ts                # Test scenario generator
│   │   │   └── types.ts                    # Vendor simulation types
│   │   │
│   │   ├── pdf/                            ← PDF GENERATION
│   │   │   └── generate-deal-summary.ts    # Deal summary PDF export
│   │   │
│   │   ├── prompts/                        ← LLM PROMPTS (if any custom)
│   │   │
│   │   ├── chatbot.controller.ts           # 17 HTTP endpoint handlers
│   │   ├── chatbot.service.ts              # 15 business logic functions
│   │   ├── chatbot.repo.ts                 # 12 database query functions
│   │   ├── chatbot.routes.ts               # 23 REST API routes
│   │   ├── chatbot.validator.ts            # 8 Joi validation schemas
│   │   └── index.ts                        # Module exports
│   │
│   ├── bid-analysis/                       ← BID ANALYSIS MODULE
│   │   ├── bid-analysis.controller.ts
│   │   ├── bid-analysis.service.ts
│   │   ├── bid-analysis.repo.ts
│   │   ├── bid-analysis.routes.ts
│   │   └── types.ts
│   │
│   ├── bid-comparison/                     ← BID COMPARISON & SCHEDULER
│   │   ├── bid-comparison.controller.ts
│   │   ├── bid-comparison.service.ts
│   │   ├── bid-comparison.repo.ts
│   │   ├── bid-comparison.routes.ts
│   │   └── scheduler/
│   │       └── deadline-checker.ts         # Cron-based deadline monitor
│   │
│   ├── vendor-chat/                        ← PUBLIC VENDOR PORTAL
│   │   ├── vendor-chat.controller.ts       # Token-based auth endpoints
│   │   ├── vendor-chat.service.ts        # Public MESO negotiation
│   │   ├── vendor-chat.routes.ts
│   │   └── types.ts
│   │
│   ├── vector/                             ← VECTOR/RAG MODULE
│   │   ├── vector.controller.ts
│   │   ├── vector.service.ts               # Embedding generation & search
│   │   ├── vector.repo.ts
│   │   ├── vector.routes.ts
│   │   ├── embedding-providers/            # Provider abstraction
│   │   │   ├── local-embedding.ts          # HuggingFace ONNX
│   │   │   ├── openai-embedding.ts         # OpenAI embeddings
│   │   │   └── bedrock-embedding.ts        # AWS Bedrock
│   │   ├── vectorization-queue.ts         # In-memory processing queue
│   │   └── types.ts
│   │
│   ├── chat/                               ← LEGACY CHAT MODULE
│   │   ├── chat.controller.ts
│   │   ├── chat.service.ts
│   │   └── chat.routes.ts
│   │
│   ├── negotiation/                        ← NEGOTIATION TRACKING
│   │   ├── negotiation.controller.ts
│   │   ├── negotiation.service.ts
│   │   └── negotiation.routes.ts
│   │
│   ├── auth/                               ← AUTHENTICATION (extracting)
│   ├── user/                               ← USER MANAGEMENT
│   ├── company/                            ← COMPANY MANAGEMENT
│   ├── role/                               ← RBAC ROLES
│   ├── product/                            ← PRODUCT CATALOG
│   ├── project/                            ← PROJECT MANAGEMENT
│   ├── requisition/                        ← REQUISITION/RFQ
│   ├── contract/                           ← CONTRACT MANAGEMENT
│   ├── po/                                 ← PURCHASE ORDERS
│   ├── vendor/                             ← VENDOR MANAGEMENT
│   ├── customer/                           ← CUSTOMER MANAGEMENT
│   ├── document/                           ← DOCUMENT UPLOAD/OCR
│   ├── dashboard/                          ← ANALYTICS/DASHBOARD
│   └── permission/                         ← PERMISSION MODULE
│
├── llm/                                    ← LLM BOUNDARY LAYER
│   ├── persona-renderer.ts                 # ONLY LLM entry point (temp 0.5)
│   ├── validate-llm-output.ts              # Output validation & safety
│   ├── fallback-templates.ts               # 40+ humanized fallback templates
│   └── index.ts
│
├── negotiation/intent/                     ← NEGOTIATION INTENT (Hard Boundary)
│   └── build-negotiation-intent.ts         # Decision → Intent transformation
│
├── delivery/                               ← UX DELIVERY LAYER
│   └── simulateTypingDelay.ts              # Server-side typing simulation (6-15s)
│
├── metrics/                                ← AUDIT & METRICS
│   └── logNegotiationStep.ts               # Winston logging (action/firmness/round/tone)
│
├── services/                               ← SHARED SERVICES
│   ├── llm.service.ts                      # Ollama HTTP client
│   ├── openai.service.ts                   # OpenAI API client
│   ├── email.service.ts                    # Nodemailer + AWS SES
│   ├── context.service.ts                  # Request context builder
│   └── currency.service.ts                 # Currency conversion
│
├── models/                                 ← SEQUELIZE MODELS (40+)
│   ├── chatbot-deal.ts                     # Core negotiation deal
│   ├── chatbot-message.ts                  # Individual messages
│   ├── chatbot-template.ts                 # Deal templates
│   ├── chatbot-template-parameter.ts       # Template configuration
│   ├── meso-round.ts                     # MESO offer tracking
│   ├── message-embedding.ts                # Vector embeddings for messages
│   ├── deal-embedding.ts                   # Vector embeddings for deals
│   ├── negotiation-pattern.ts              # Pattern recognition data
│   ├── negotiation-training-data.ts        # Training dataset
│   ├── vector-migration-status.ts          # Embedding migration tracking
│   ├── vendor-bid.ts                       # Vendor submitted bids
│   ├── bid-comparison.ts                   # Bid analysis results
│   ├── vendor-selection.ts                 # Winner selection tracking
│   └── ... (30+ more models)
│
├── routes/                                 ← ROUTE AGGREGATION
│   └── index.ts                            # Mounts all module routes
│
├── middlewares/                            ← EXPRESS MIDDLEWARE
│   ├── auth.middleware.ts                  # JWT validation
│   ├── jwt.service.ts                      # Token operations
│   ├── error-handler.ts                    # Global error handling
│   ├── request-logger.ts                   # Request logging
│   ├── request-id.ts                       # Correlation IDs
│   └── upload.middleware.ts                # File upload handling
│
├── config/                                 ← CONFIGURATION
│   ├── database.ts                         # Sequelize setup
│   ├── env.ts                              # Environment variables
│   ├── logger.ts                           # Winston/Pino configuration
│   └── swagger.ts                          # API documentation
│
├── loaders/                                │
│   └── express.ts                          # Express app factory
│
├── types/                                  ← TYPESCRIPT TYPES
│   └── index.ts                            # Global type definitions
│
├── utils/                                  ← UTILITIES
│   └── ...                                 # Helper functions
│
├── seeders/                                ← DATABASE SEEDERS
│
└── index.ts                                ← APPLICATION ENTRY POINT
```

---

## File-by-File: AI System Components

### 1. Decision Engine (Deterministic Core)

**File:** `src/modules/chatbot/engine/decide.ts`

```typescript
// Core decision algorithm
function decideNextMove(utility: number, config: DealConfig): DecisionAction {
  if (utility >= config.acceptThreshold) return "ACCEPT";
  if (utility < config.walkAwayThreshold) return "WALK_AWAY";
  return "COUNTER";
}
```

**Purpose:** Makes all strategic decisions without LLM involvement

---

**File:** `src/modules/chatbot/engine/utility.ts`

```typescript
// Weighted utility calculation
function calculateUtility(
  offer: ParsedOffer,
  config: DealConfig,
): UtilityBreakdown {
  const priceUtility = calculatePriceUtility(offer.price, config);
  const termsUtility = calculateTermsUtility(offer.paymentTerms, config);
  const deliveryUtility = calculateDeliveryUtility(offer.deliveryDays, config);

  return {
    total: priceUtility * 0.6 + termsUtility * 0.3 + deliveryUtility * 0.1,
    breakdown: {
      price: priceUtility,
      terms: termsUtility,
      delivery: deliveryUtility,
    },
  };
}
```

**Purpose:** Scores vendor offers across multiple dimensions

---

**File:** `src/modules/chatbot/engine/parseOffer.ts`

```typescript
// Regex-based offer extraction
function parseOffer(message: string): ParsedOffer {
  const priceMatch = message.match(
    /(?:\$|USD|Rs\.?|INR)?\s*([\d,]+(?:\.\d{2})?)/i,
  );
  const termsMatch = message.match(/(?:net|payment terms?)\s*(\d+)/i);
  const deliveryMatch = message.match(/(\d+)\s*(?:days?|business days?)/i);

  return {
    price: priceMatch ? parseCurrency(priceMatch[1]) : null,
    paymentTerms: termsMatch ? parseInt(termsMatch[1]) : null,
    deliveryDays: deliveryMatch ? parseInt(deliveryMatch[1]) : null,
  };
}
```

**Purpose:** Extracts structured data from unstructured vendor messages

---

**File:** `src/modules/chatbot/engine/meso.ts`

```typescript
// Multi-issue option generation
function generateMESOOptions(
  targetPrice: number,
  config: DealConfig,
): MESOOption[] {
  return [
    {
      // Balanced
      totalPrice: humanRoundPrice(targetPrice),
      paymentTerms: 30,
      deliveryDays: 14,
      label: "Standard Package",
    },
    {
      // Better price, stricter terms
      totalPrice: humanRoundPrice(targetPrice * 0.98),
      paymentTerms: 15,
      deliveryDays: 21,
      label: "Fast Payment Package",
    },
    {
      // Worse price, better terms
      totalPrice: humanRoundPrice(targetPrice * 1.02),
      paymentTerms: 60,
      deliveryDays: 7,
      label: "Premium Delivery Package",
    },
  ];
}
```

**Purpose:** Creates Pareto-optimal negotiation packages

---

### 2. LLM Boundary Layer

**File:** `src/llm/persona-renderer.ts`

```typescript
// ONLY LLM entry point in system
async function renderNegotiationMessage(
  intent: NegotiationIntent,
  context: RenderContext,
): Promise<string> {
  const systemPrompt = buildSystemPrompt(); // Static, cached
  const userPrompt = buildUserPrompt(intent, context);

  const response = await callLLM({
    model: "qwen3",
    temperature: 0.5, // Consistent, not creative
    maxTokens: 200,
    system: systemPrompt,
    messages: [{ role: "user", content: userPrompt }],
  });

  return response.content;
}
```

**Purpose:** Renders human-like negotiation messages from structured intent

---

**File:** `src/llm/validate-llm-output.ts`

```typescript
// Output validation & safety
function validateLlmOutput(
  output: string,
  intent: NegotiationIntent,
): ValidationResult {
  // 1. Banned word check
  const bannedWords = [
    "utility",
    "threshold",
    "target price",
    "max acceptable",
  ];
  if (bannedWords.some((w) => output.toLowerCase().includes(w))) {
    return { valid: false, reason: "Contains internal terms" };
  }

  // 2. Price constraint check
  if (intent.action === "COUNTER") {
    const price = extractPrice(output);
    if (
      price &&
      (price < intent.allowedPrice.min || price > intent.allowedPrice.max)
    ) {
      return { valid: false, reason: "Price outside allowed range" };
    }
  }

  // 3. Length check
  const wordCount = output.split(/\s+/).length;
  if (wordCount > 160) {
    return { valid: false, reason: "Response too long" };
  }

  // 4. Date format check
  if (output.match(/\d{4}-\d{2}-\d{2}/)) {
    return { valid: false, reason: "Contains YYYY-MM-DD date format" };
  }

  return { valid: true };
}
```

**Purpose:** Ensures LLM output meets safety and business constraints

---

**File:** `src/llm/fallback-templates.ts`

```typescript
// 40+ humanized fallback templates
const fallbackTemplates: Record<ActionType, string[]> = {
  ACCEPT: [
    "That works for us. Let's move forward with {price}.",
    "Good, we're aligned at {price}. Preparing the paperwork now.",
    "Agreed. {price} works — I'll send the contract shortly.",
    // ... 5+ variations per action
  ],
  COUNTER: [
    "Thanks for the update. Could we work with {price}?",
    "Appreciate the offer. Where would you be on {price}?",
    // ... 5+ variations
  ],
  WALK_AWAY: [
    "Thanks for the discussion. We'll need to revisit this another time.",
    "Appreciate your time, but we're not aligned. Let's reconnect later.",
    // ... 5+ variations
  ],
  MESO: [
    "Here are a few options that could work. Which direction feels best?",
    // ... variations
  ],
};
```

**Purpose:** Provides humanized responses when LLM validation fails

---

### 3. Intent Building (Hard Boundary)

**File:** `src/negotiation/intent/build-negotiation-intent.ts`

```typescript
// Transforms engine decisions into LLM-safe intent
function buildNegotiationIntent(
  decision: EngineDecision,
  context: NegotiationContext,
): NegotiationIntent {
  // STRIP sensitive data
  const {
    utilityScore, // ← REMOVED
    targetPrice, // ← REMOVED
    maxAcceptablePrice, // ← REMOVED
    weights, // ← REMOVED
    ...safeData
  } = decision;

  return {
    action: decision.action, // ACCEPT/COUNTER/WALK_AWAY/MESO
    tone: detectTone(context), // formal/neutral/casual
    round: context.currentRound,
    allowedPrice:
      decision.action === "COUNTER"
        ? { min: decision.floorPrice, max: decision.ceilingPrice }
        : undefined,
    reasoningHint: getVagueReasoning(decision.action), // "what we can work with"
    vendorMessage: context.lastVendorMessage,
    dealTitle: context.dealTitle,
    vendorName: context.vendorName,
    productCategory: context.productCategory,
    // NOTE: NO utility scores, NO thresholds, NO internal config
  };
}
```

**Purpose:** Creates a hard data boundary between deterministic engine and LLM

---

### 4. Conversation Orchestration

**File:** `src/modules/chatbot/convo/conversation-service.ts`

```typescript
// Main conversation pipeline (15 steps)
async function processConversationMessage(
  dealId: string,
  vendorMessage: string,
  mode: "INSIGHTS" | "CONVERSATION",
): Promise<ConversationResult> {
  // 1. Load deal state
  const deal = await getDeal(dealId);

  // 2. Parse vendor offer
  const parsedOffer = parseOfferRegex(vendorMessage);

  // 3. Make deterministic decision
  const decision = decideNextMove(parsedOffer, deal.config);

  // 4. Detect vendor tone
  const tone = detectVendorTone(vendorMessage);

  // 5. Build intent (hard boundary)
  const intent = buildNegotiationIntent(decision, { deal, tone, parsedOffer });

  // 6. Generate response
  let accordoResponse: string;
  if (mode === "INSIGHTS") {
    accordoResponse = getTemplate(intent);
  } else {
    // CONVERSATION mode with LLM
    const rawLlmOutput = await renderNegotiationMessage(
      intent,
      buildContext(deal),
    );
    const validated = validateLlmOutput(rawLlmOutput, intent);

    accordoResponse = validated.valid
      ? rawLlmOutput
      : getFallbackTemplate(intent);
  }

  // 7. Simulate typing delay
  await simulateTypingDelay(intent.action, tone);

  // 8. Log negotiation step
  logNegotiationStep({
    dealId,
    action: intent.action,
    round: deal.round,
    tone,
  });

  // 9. Save messages to database
  await saveMessages(dealId, vendorMessage, accordoResponse, decision);

  // 10. Update deal state
  await updateDealState(dealId, decision, parsedOffer);

  return { vendorMessage, accordoMessage: accordoResponse, decision };
}
```

**Purpose:** Orchestrates the complete negotiation turn

---

### 5. Delivery Layer

**File:** `src/delivery/simulateTypingDelay.ts`

```typescript
// Server-side typing simulation
async function simulateTypingDelay(
  action: DecisionAction,
  tone: ToneType,
): Promise<void> {
  const baseDelays: Record<DecisionAction, [number, number]> = {
    GREET: [2, 4],
    OFFER_REQUEST: [3, 6],
    INFORMATION_REQUEST: [3, 5],
    COUNTER: [6, 12],
    MESO: [8, 15], // Longer for multi-option
    ACCEPT: [4, 8],
    WALK_AWAY: [5, 10],
    ESCALATE: [7, 12],
  };

  const [min, max] = baseDelays[action];
  const adjustedMin = tone === "casual" ? min * 0.7 : min;
  const adjustedMax = tone === "formal" ? max * 1.2 : max;

  const delayMs =
    Math.floor(Math.random() * (adjustedMax - adjustedMin) + adjustedMin) *
    1000;

  await sleep(delayMs);
}
```

**Purpose:** Creates realistic human-like response timing

---

### 6. Audit & Metrics

**File:** `src/metrics/logNegotiationStep.ts`

```typescript
// Winston-based negotiation logging
function logNegotiationStep(metrics: NegotiationMetrics): void {
  logger.info({
    event: "negotiation_step",
    dealId: metrics.dealId,
    action: metrics.action, // ACCEPT/COUNTER/etc.
    firmness: metrics.firmness, // 1-5 scale
    round: metrics.round,
    tone: metrics.tone,
    responseTimeMs: metrics.responseTimeMs,
    // NOTE: NO prices, NO utility scores (privacy)
  });
}
```

**Purpose:** Minimal audit logging for analytics without exposing sensitive data

---

## API Routes: Where Endpoints Live

### Chatbot Routes (`/api/chatbot/*`)

| Route                                                                | File                | Purpose                     |
| -------------------------------------------------------------------- | ------------------- | --------------------------- |
| `POST /requisitions/:rfqId/deals`                                    | `chatbot.routes.ts` | Create new deal             |
| `GET /requisitions/:rfqId/deals`                                     | `chatbot.routes.ts` | List deals for requisition  |
| `POST /requisitions/:rfqId/vendors/:vendorId/deals/:dealId/messages` | `chatbot.routes.ts` | Send message (INSIGHTS)     |
| `POST /conversation/deals/:dealId/messages`                          | `chatbot.routes.ts` | Send message (CONVERSATION) |
| `GET /requisitions/:rfqId/deals/:dealId/messages`                    | `chatbot.routes.ts` | Get message history         |
| `POST /requisitions/:rfqId/deals/:dealId/reset`                      | `chatbot.routes.ts` | Reset deal                  |
| `POST /requisitions/:rfqId/deals/:dealId/archive`                    | `chatbot.routes.ts` | Archive deal                |
| `POST /requisitions/:rfqId/deals/:dealId/summary`                    | `chatbot.routes.ts` | Get deal summary            |
| `GET /templates`                                                     | `chatbot.routes.ts` | List templates              |

### Bid Analysis Routes (`/api/bid-analysis/*`)

| Route                                  | File                     | Purpose                     |
| -------------------------------------- | ------------------------ | --------------------------- |
| `GET /requisitions`                    | `bid-analysis.routes.ts` | List requisitions with bids |
| `GET /requisitions/:id`                | `bid-analysis.routes.ts` | Get requisition bid details |
| `POST /requisitions/:id/select-winner` | `bid-analysis.routes.ts` | Select winning bid          |
| `POST /requisitions/:id/reject-bids`   | `bid-analysis.routes.ts` | Reject all bids             |
| `GET /requisitions/:id/export`         | `bid-analysis.routes.ts` | Export bid comparison PDF   |

### Vector Routes (`/api/vector/*`)

| Route                        | File               | Purpose                  |
| ---------------------------- | ------------------ | ------------------------ |
| `POST /embeddings/messages`  | `vector.routes.ts` | Create message embedding |
| `POST /embeddings/deals`     | `vector.routes.ts` | Create deal embedding    |
| `POST /search/similar`       | `vector.routes.ts` | Semantic search          |
| `GET /deals/:dealId/similar` | `vector.routes.ts` | Find similar past deals  |

### Vendor Chat Routes (`/api/vendor-chat/*`) — PUBLIC

| Route                         | File                    | Purpose                    |
| ----------------------------- | ----------------------- | -------------------------- |
| `GET /deals/:token`           | `vendor-chat.routes.ts` | Get deal by unique token   |
| `POST /deals/:token/messages` | `vendor-chat.routes.ts` | Vendor sends message       |
| `POST /deals/:token/meso`     | `vendor-chat.routes.ts` | Vendor selects MESO option |
| `POST /deals/:token/accept`   | `vendor-chat.routes.ts` | Vendor accepts deal        |

---

## Database Models: Where Data Lives

### Core Negotiation Models

| Model                      | File                                   | Key Fields                                                                                                                                |
| -------------------------- | -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `ChatbotDeal`              | `models/chatbot-deal.ts`               | id, status (NEGOTIATING/ACCEPTED/etc.), mode (INSIGHTS/CONVERSATION), round, latest_offer_json, convo_state_json, archived_at, deleted_at |
| `ChatbotMessage`           | `models/chatbot-message.ts`            | deal_id, role (VENDOR/ACCORDO/SYSTEM), content, extracted_offer, engine_decision, utility_score, counter_offer, explainability_json       |
| `ChatbotTemplate`          | `models/chatbot-template.ts`           | name, description, is_default                                                                                                             |
| `ChatbotTemplateParameter` | `models/chatbot-template-parameter.ts` | template_id, config_json (price/terms/delivery parameters)                                                                                |
| `MesoRound`                | `models/meso-round.ts`                 | deal_id, round_number, options_json (MESO packages), selected_option                                                                      |

### Vector/RAG Models

| Model                     | File                                  | Key Fields                                                         |
| ------------------------- | ------------------------------------- | ------------------------------------------------------------------ |
| `MessageEmbedding`        | `models/message-embedding.ts`         | message_id, deal_id, user_id, embedding_vector, model_name         |
| `DealEmbedding`           | `models/deal-embedding.ts`            | deal_id, user_id, embedding_vector, outcome (ACCEPTED/REJECTED)    |
| `VectorMigrationStatus`   | `models/vector-migration-status.ts`   | entity_type, last_processed_id, status                             |
| `NegotiationPattern`      | `models/negotiation-pattern.ts`       | pattern_type, pattern_data, occurrence_count                       |
| `NegotiationTrainingData` | `models/negotiation-training-data.ts` | input_features, output_action, confidence_score, used_for_training |

### Bid Analysis Models

| Model                | File                            | Key Fields                                                                                |
| -------------------- | ------------------------------- | ----------------------------------------------------------------------------------------- |
| `VendorBid`          | `models/vendor-bid.ts`          | requisition_id, vendor_id, bid_amount, bid_currency, delivery_days, validity_date, status |
| `BidComparison`      | `models/bid-comparison.ts`      | requisition_id, comparisons_json, generated_at                                            |
| `BidActionHistory`   | `models/bid-action-history.ts`  | requisition_id, action_type, performed_by, details_json                                   |
| `VendorSelection`    | `models/vendor-selection.ts`    | requisition_id, selected_vendor_id, selection_reason, negotiated_amount                   |
| `VendorNotification` | `models/vendor-notification.ts` | vendor_id, requisition_id, notification_type, sent_at, read_at                            |

---

## Service Dependencies Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SERVICE DEPENDENCY GRAPH                           │
└─────────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────┐
                    │   API Gateway   │
                    │    (Port 5002)  │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│   Chatbot     │  │ Bid Analysis  │  │  Bid Compare  │
│   Service     │  │   Service     │  │   Service     │
│               │  │               │  │               │
│ • engine/     │  │ • bid-analysis│  │ • bid-compari-│
│ • convo/      │  │   .service.ts │  │   son.service │
│ • vendor/     │  │ • bid-analysis│  │ • scheduler/  │
│ • chatbot.    │  │   .controller │  │   deadline-   │
│   service.ts  │  │ • bid-analysis│  │   checker.ts  │
│               │  │   .repo.ts    │  │               │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        │                    │                    │
        │    ┌───────────────┴────────────────────┘
        │    │
        ▼    ▼
┌─────────────────────────────────────────────────────────┐
│                     SHARED LAYER                         │
├─────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ LLM Services │  │ Email Service│  │ Vector/RAG   │  │
│  │ • llm.svc.ts │  │ • email.svc  │  │ • vector.svc │  │
│  │ • openai.svc │  │              │  │ • embedding  │  │
│  │              │  │              │  │   providers│  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│    Ollama     │  │   AWS SES     │  │  PostgreSQL   │
│   (Local AI)  │  │  (Email)      │  │   + Redis     │
│   Port:11434  │  │               │  │               │
└───────────────┘  └───────────────┘  └───────────────┘

LEGEND:
  ───►  Calls/Uses
  ░░░  External Service
```

---

## Key Constants & Configuration

### Decision Thresholds

```typescript
// Located in: src/modules/chatbot/engine/config.ts (or similar)
const DECISION_THRESHOLDS = {
  ACCEPT: 75, // Utility >= 75 → Auto-accept
  WALK_AWAY: 30, // Utility < 30 → Walk away
  COUNTER: "between", // 30-75 → Counter-offer
};

const UTILITY_WEIGHTS = {
  PRICE: 0.6, // 60% weight on price
  TERMS: 0.3, // 30% weight on payment terms
  DELIVERY: 0.1, // 10% weight on delivery speed
};
```

### LLM Configuration

```typescript
// Located in: src/config/env.ts or llm service
const LLM_CONFIG = {
  LOCAL_MODEL: "qwen3",
  FALLBACK_MODEL: "gpt-4o",
  TEMPERATURE: 0.5, // Low for consistency
  MAX_TOKENS: 200,
  TIMEOUT_MS: 60000,
  EMBEDDING_PROVIDER: "local", // or 'openai' or 'bedrock'
};
```

### Typing Delays (seconds)

```typescript
// Located in: src/delivery/simulateTypingDelay.ts
const TYPING_DELAYS = {
  GREET: [2, 4],
  COUNTER: [6, 12],
  MESO: [8, 15],
  ACCEPT: [4, 8],
  WALK_AWAY: [5, 10],
};
```

---

## Testing: Where Tests Live

```
Accordo-ai-backend/
├── tests/
│   ├── unit/                           # Pure logic tests (no DB)
│   │   ├── engine/
│   │   │   ├── decide.test.ts          # Decision algorithm tests
│   │   │   ├── utility.test.ts         # Utility calculation tests
│   │   │   └── parseOffer.test.ts      # Regex parsing tests
│   │   ├── llm/
│   │   │   ├── validate-output.test.ts # Validation logic tests
│   │   │   └── fallback-templates.test.ts
│   │   └── vendor/
│   │       └── scenario-detector.test.ts
│   │
│   ├── integration/                    # Service + DB tests
│   │   ├── chatbot/
│   │   │   ├── deal-lifecycle.test.ts  # Full deal flow
│   │   │   └── message-handling.test.ts
│   │   ├── bid-analysis/
│   │   │   └── winner-selection.test.ts
│   │   └── vector/
│   │       └── embedding-flow.test.ts
│   │
│   └── fixtures/                       # Test data
│       ├── deals.ts                    # Sample deal configs
│       ├── offers.ts                   # Sample vendor messages
│       └── templates.ts                # Sample templates
│
├── vitest.unit.config.ts               # Unit test config (no DB)
└── vitest.config.ts                    # Integration test config (with DB)
```

---

## Document Control

| Version | Date       | Author              | Changes                                                      |
| ------- | ---------- | ------------------- | ------------------------------------------------------------ |
| 1.0     | 2026-05-19 | Accordo Engineering | Initial code architecture reference for external consultants |

---

_End of Document_
