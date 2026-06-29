# PM Response Enhancement - Combined Implementation Plan

## Overview

This plan combines TWO critical enhancements:

1. **Part A: Instant Vendor Message Display** - Vendor messages appear immediately, PM responds asynchronously
2. **Part B: Human-like PM Responses** - Context-aware responses with delivery terms and adaptive tone

**Priority**: CRITICAL (Core Features)
**Scope**: Backend API restructuring + Frontend UX + LLM response enhancement

---

## Part A: Instant Vendor Message Display

### Problem Statement
Currently, when vendor clicks a suggestion chip:
1. Frontend waits for entire API call to complete
2. Both vendor message AND PM response appear together
3. Creates perception of slow/unresponsive UI

### Solution
Split into two-phase flow:
1. **Phase 1 (Instant)**: Save vendor message → Display immediately
2. **Phase 2 (Async)**: Generate PM response → Display when ready with typing indicator

### Requirements Summary (Part A)

| Requirement | Decision |
|-------------|----------|
| Loading Indicator | Typing dots animation (...) |
| Input While Thinking | Allow but queue |
| Timeout Handling | Show fallback after 5 seconds |
| Scroll Behavior | Keep existing behavior |
| API Architecture | Two separate APIs |
| Message Status | Show as sent (optimistic) |
| Round Enforcement | Queue system (sequential processing) |
| Queue Feedback | Inline message grayed out with 'Queued' badge |

### UX Flow (Part A)

```
User clicks chip → Vendor message appears INSTANTLY
                 → Typing indicator shows in PM area (...)
                 → PM analyzes in background
                 → Typing indicator replaced with PM response
                 → If queued message exists, process next
```

### Visual Representation

```
┌─────────────────────────────────────────────┐
│ Chat Area                                   │
├─────────────────────────────────────────────┤
│                                             │
│    [PM] Previous message...                 │
│                                             │
│                    [VENDOR] $95 Net 30 ←─── Appears INSTANTLY
│                                             │
│    [PM] ● ● ●  ←─────────────────────────── Typing animation
│                                             │
│                    [VENDOR] $93? (Queued) ← Grayed out, queued
│                                             │
└─────────────────────────────────────────────┘
```

---

## Part B: Human-like PM Responses

### Problem Statement
Current PM responses are:
- Generic templates without delivery terms
- Expose internal utility scores
- Same response every time
- Don't adapt to vendor's tone or acknowledge their concerns

### Solution
Transform to human-like, context-aware responses with:
- Delivery terms always included
- Adaptive tone matching vendor
- Acknowledge vendor concerns when relevant
- High variation (10+ styles per action)
- Dynamic length based on complexity

### Requirements Summary (Part B)

| Requirement | Decision |
|-------------|----------|
| Target Mode | Both modes (Route INSIGHTS to LLM too) |
| Context Analysis | Full - Mirror tone, reference specifics, acknowledge concerns |
| Delivery Format | Flexible phrasing (AI chooses best format) |
| Action-Based Tone | Yes - ACCEPT=warm, COUNTER=firm but collaborative, WALK_AWAY=professional regret |
| Vendor Acknowledgment | Acknowledge when relevant (concerns, justifications) |
| Response Variation | High (10+ styles per action) |
| Response Length | Dynamic (short for simple, longer for complex) |
| Vendor Delivery Parsing | Yes - full parsing with utility calculation |
| Proactive Delivery | Always include delivery in PM responses |
| Tone Style | Adaptive to vendor (mirror their tone) |
| Reasoning Disclosure | High-level business reasoning (no internal calculations) |
| LLM Fallback | Hybrid (typing indicator, templates only if LLM fails) |
| Conversation Memory | Full history reference |

### Example Transformation (Part B)

**Vendor**: "I can offer $98 per unit with Net 45. We're dealing with supply chain delays."

**Before (Current)**:
> "Thank you for your offer. Based on our analysis, I'd like to counter with $94.00 and Net 60 payment terms. This would give us a utility score of 72%."

**After (Enhanced)**:
> "I understand the supply chain challenges you're facing. Given our project timeline, we'd like to propose $94 per unit with Net 60 payment, with delivery by March 10th. Extended payment terms should help offset some of your logistics costs. Can we make this work?"

---

## Combined Implementation Phases

### Phase 1: Backend API Restructuring (Part A)

**Goal**: Split message handling into two APIs

#### 1.1 New API: Save Vendor Message Only

**Endpoint**: `POST /api/chatbot/.../deals/:dealId/vendor-message`

```typescript
// Request
{ content: string }

// Response (instant, <100ms)
{
  success: true,
  data: {
    message: Message,        // Saved vendor message
    deal: Deal,              // Updated deal (round incremented)
    pmProcessing: true,      // Flag indicating PM response pending
  }
}
```

**Backend Logic**:
1. Save vendor message to database
2. Parse offer (price, terms, delivery)
3. Increment round
4. Return immediately
5. Do NOT generate PM response yet

#### 1.2 New API: Get PM Response (Async)

**Endpoint**: `POST /api/chatbot/.../deals/:dealId/pm-response`

```typescript
// Request
{ vendorMessageId: string }  // ID of vendor message to respond to

// Response (may take 1-5 seconds)
{
  success: true,
  data: {
    message: Message,        // PM response message
    decision: Decision,      // Decision details
    utility: UtilityResult,  // Utility breakdown
    deal: Deal,              // Updated deal status
  }
}
```

**Backend Logic**:
1. Retrieve vendor message and offer
2. Run decision engine
3. Generate human-like response (LLM)
4. Save PM message
5. Return with full context

#### 1.3 Message Queue System

```typescript
interface MessageQueue {
  dealId: string;
  pendingMessages: Array<{
    content: string;
    queuedAt: Date;
    status: 'queued' | 'processing';
  }>;
}
```

**Processing Logic**:
- When vendor sends while PM is thinking → Add to queue
- When PM response completes → Process next queued message
- Sequential: 1 vendor → 1 PM → repeat

---

### Phase 2: Frontend Instant Display (Part A)

**Goal**: Vendor messages appear immediately with typing indicator

#### 2.1 Update Message Sending Flow

**File**: `src/hooks/chatbot/useDealActions.ts` (or similar)

```typescript
const sendVendorMessage = async (content: string) => {
  // Step 1: Optimistically add vendor message to UI
  const tempMessage = createTempMessage(content, 'VENDOR');
  setMessages(prev => [...prev, tempMessage]);

  // Step 2: Show PM typing indicator
  setIsPMTyping(true);

  // Step 3: Save vendor message (fast API)
  const saveResponse = await chatbotService.saveVendorMessage(context, content);

  // Step 4: Replace temp message with server message
  setMessages(prev => prev.map(m =>
    m.id === tempMessage.id ? saveResponse.data.message : m
  ));

  // Step 5: Get PM response (slow API)
  try {
    const pmResponse = await chatbotService.getPMResponse(context, saveResponse.data.message.id);
    setMessages(prev => [...prev, pmResponse.data.message]);
  } catch (error) {
    // Timeout or error - show fallback
    const fallbackMessage = createFallbackPMMessage(saveResponse.data);
    setMessages(prev => [...prev, fallbackMessage]);
  } finally {
    setIsPMTyping(false);
  }

  // Step 6: Process queued messages if any
  processMessageQueue();
};
```

#### 2.2 Typing Indicator Component

**File**: `src/components/chatbot/chat/TypingIndicator.tsx`

```tsx
export function TypingIndicator() {
  return (
    <div className="flex items-center gap-1 px-4 py-2">
      <div className="flex items-center gap-1 bg-gray-100 dark:bg-gray-800 rounded-full px-3 py-2">
        <span className="w-2 h-2 bg-gray-500 rounded-full animate-bounce" style={{ animationDelay: '0ms' }} />
        <span className="w-2 h-2 bg-gray-500 rounded-full animate-bounce" style={{ animationDelay: '150ms' }} />
        <span className="w-2 h-2 bg-gray-500 rounded-full animate-bounce" style={{ animationDelay: '300ms' }} />
      </div>
      <span className="text-xs text-gray-500 ml-2">Accordo is typing...</span>
    </div>
  );
}
```

#### 2.3 Queued Message Display

```tsx
// In MessageBubble or ChatTranscript
{message.status === 'queued' && (
  <div className="opacity-50">
    <MessageBubble message={message} />
    <span className="text-xs text-gray-400 ml-2">Queued</span>
  </div>
)}
```

#### 2.4 Timeout Handling

```typescript
const PM_RESPONSE_TIMEOUT = 5000; // 5 seconds

const getPMResponseWithTimeout = async (context, messageId) => {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), PM_RESPONSE_TIMEOUT);

  try {
    const response = await chatbotService.getPMResponse(context, messageId, {
      signal: controller.signal,
    });
    clearTimeout(timeoutId);
    return response;
  } catch (error) {
    if (error.name === 'AbortError') {
      // Request fallback from backend
      return chatbotService.getPMFallbackResponse(context, messageId);
    }
    throw error;
  }
};
```

---

### Phase 3: Delivery Term Parsing (Part B)

**Goal**: Parse vendor delivery offers and calculate utility

#### 3.1 Extend Offer Interface

**File**: `src/modules/chatbot/engine/types.ts`

```typescript
interface Offer {
  unit_price: number | null;
  payment_terms: string | null;
  // NEW: Delivery fields
  delivery_date: string | null;      // ISO date string
  delivery_days: number | null;      // Days from today
  meta: {
    raw_terms_days?: number;
    non_standard_terms?: boolean;
    delivery_source?: 'explicit_date' | 'relative_days' | 'timeframe';
  };
}
```

#### 3.2 Enhance Offer Parser

**File**: `src/modules/chatbot/engine/parseOffer.ts`

Add delivery parsing patterns:
```typescript
// Explicit dates
const DATE_PATTERNS = [
  /(?:by|before|on|deliver(?:y|ed)?(?:\s+by)?)\s+(\w+\s+\d{1,2}(?:st|nd|rd|th)?(?:,?\s+\d{4})?)/i,
  /(\d{4}-\d{2}-\d{2})/,  // ISO format
  /(\d{1,2}\/\d{1,2}\/\d{2,4})/,  // MM/DD/YYYY
];

// Relative days
const RELATIVE_PATTERNS = [
  /(?:within|in)\s+(\d+)\s*(?:days?|business\s+days?)/i,
  /(\d+)\s*(?:days?|weeks?)\s*(?:delivery|shipping|lead\s+time)/i,
];

// Timeframes
const TIMEFRAME_PATTERNS = [
  /(?:early|mid|late|end\s+of)\s+(\w+)/i,  // "early March"
];
```

#### 3.3 Delivery Utility Calculation

**File**: `src/modules/chatbot/engine/deliveryUtility.ts` (NEW)

```typescript
export function calculateDeliveryUtility(
  offeredDate: Date | null,
  config: DeliveryConfig
): number {
  if (!offeredDate || !config.requiredDate) {
    return 0.5; // Neutral if not specified
  }

  const required = new Date(config.requiredDate);
  const preferred = config.preferredDate ? new Date(config.preferredDate) : required;

  // Calculate days difference
  const daysFromPreferred = Math.floor((offeredDate.getTime() - preferred.getTime()) / (1000 * 60 * 60 * 24));
  const daysFromRequired = Math.floor((offeredDate.getTime() - required.getTime()) / (1000 * 60 * 60 * 24));

  if (daysFromPreferred <= 0) {
    return 1.0; // On or before preferred = excellent
  } else if (daysFromRequired <= 0) {
    // Between preferred and required
    const range = Math.abs((required.getTime() - preferred.getTime()) / (1000 * 60 * 60 * 24));
    return 0.7 + (0.3 * (1 - daysFromPreferred / range));
  } else if (daysFromRequired <= 14) {
    // Up to 2 weeks late
    return 0.3 + (0.4 * (1 - daysFromRequired / 14));
  } else {
    // More than 2 weeks late
    return Math.max(0, 0.3 - (daysFromRequired - 14) * 0.02);
  }
}
```

#### 3.4 Update Decision Engine

**File**: `src/modules/chatbot/engine/decide.ts`

```typescript
// Add delivery to weighted utility
const totalUtility =
  (priceUtility * weights.price) +
  (termsUtility * weights.terms) +
  (deliveryUtility * weights.delivery);  // NEW

// Include delivery in counter-offer
const counterOffer = {
  unit_price: calculateCounterPrice(...),
  payment_terms: calculateCounterTerms(...),
  delivery_date: calculateCounterDeliveryDate(config, vendorOffer),  // NEW
  delivery_days: calculateDeliveryDays(counterDeliveryDate),
};
```

---

### Phase 4: Human-like Response Generation (Part B)

**Goal**: Context-aware LLM responses with delivery

#### 4.1 Create Unified Response Generator

**File**: `src/modules/chatbot/engine/responseGenerator.ts` (NEW)

```typescript
interface ResponseGeneratorInput {
  decision: Decision;
  config: NegotiationConfig;
  conversationHistory: Message[];
  vendorOffer: Offer;
  counterOffer: Offer | null;
  dealConfig: DealWizardConfig;
}

export async function generateHumanLikeResponse(
  input: ResponseGeneratorInput
): Promise<string> {
  // 1. Detect vendor tone
  const tone = detectVendorTone(input.conversationHistory);

  // 2. Extract vendor concerns
  const concerns = extractVendorConcerns(input.conversationHistory);

  // 3. Build context-aware prompt
  const prompt = buildResponsePrompt(input, tone, concerns);

  // 4. Generate via LLM
  try {
    const response = await generateLLMResponse(prompt, {
      timeout: 4000, // Leave 1 second buffer for 5 second total
    });

    // 5. Validate response quality
    if (validateResponseQuality(response, input.decision.action)) {
      return response;
    }
  } catch (error) {
    logger.warn('[ResponseGenerator] LLM failed, using fallback');
  }

  // 6. Fallback to enhanced template
  return generateEnhancedFallback(input, tone, concerns);
}
```

#### 4.2 Tone Detector

**File**: `src/modules/chatbot/engine/toneDetector.ts` (NEW)

```typescript
type VendorTone = 'formal' | 'casual' | 'urgent' | 'firm' | 'friendly';

export function detectVendorTone(messages: Message[]): VendorTone {
  const vendorMessages = messages.filter(m => m.role === 'VENDOR');
  const latestMessage = vendorMessages[vendorMessages.length - 1]?.content || '';

  // Formal indicators
  if (/dear|sir|madam|respectfully|please find|we would like to/i.test(latestMessage)) {
    return 'formal';
  }

  // Urgent indicators
  if (/asap|urgent|immediately|deadline|time-sensitive/i.test(latestMessage)) {
    return 'urgent';
  }

  // Firm indicators
  if (/final offer|best we can|non-negotiable|take it or leave/i.test(latestMessage)) {
    return 'firm';
  }

  // Casual indicators
  if (/hey|hi|sure|sounds good|cool|works for me/i.test(latestMessage)) {
    return 'casual';
  }

  return 'friendly'; // Default
}
```

#### 4.3 Concern Extractor

**File**: `src/modules/chatbot/engine/concernExtractor.ts` (NEW)

```typescript
interface VendorConcern {
  type: 'cost' | 'timeline' | 'quality' | 'volume' | 'logistics' | 'other';
  text: string;
}

export function extractVendorConcerns(messages: Message[]): VendorConcern[] {
  const concerns: VendorConcern[] = [];
  const vendorMessages = messages.filter(m => m.role === 'VENDOR');

  for (const msg of vendorMessages) {
    const content = msg.content.toLowerCase();

    // Cost concerns
    if (/material cost|supply chain|inflation|price increase|margin/i.test(content)) {
      concerns.push({ type: 'cost', text: extractConcernPhrase(content, 'cost') });
    }

    // Timeline concerns
    if (/lead time|production schedule|shipping delay|backlog/i.test(content)) {
      concerns.push({ type: 'timeline', text: extractConcernPhrase(content, 'timeline') });
    }

    // Quality concerns
    if (/premium quality|certification|testing|compliance/i.test(content)) {
      concerns.push({ type: 'quality', text: extractConcernPhrase(content, 'quality') });
    }

    // Logistics concerns
    if (/shipping|freight|logistics|warehouse|inventory/i.test(content)) {
      concerns.push({ type: 'logistics', text: extractConcernPhrase(content, 'logistics') });
    }
  }

  return concerns;
}
```

#### 4.4 Enhanced LLM Prompts by Action

```typescript
const ACTION_PROMPTS = {
  ACCEPT: (input, tone, concerns) => `
You are Accordo, a Procurement Manager. Generate a ${tone === 'formal' ? 'formal' : 'warm, friendly'} acceptance response.

VENDOR'S OFFER:
- Price: $${input.vendorOffer.unit_price}/unit
- Payment: ${input.vendorOffer.payment_terms}
- Delivery: ${formatDelivery(input.vendorOffer)}

${concerns.length > 0 ? `VENDOR'S CONCERNS (acknowledge positively): ${concerns.map(c => c.text).join(', ')}` : ''}

TONE: ${tone} (mirror the vendor's communication style)

REQUIREMENTS:
1. Confirm ALL THREE terms: price, payment, delivery
2. Express genuine appreciation
3. ${concerns.length > 0 ? 'Acknowledge their concerns/efforts positively' : 'Thank them for the negotiation'}
4. Keep to 2-3 sentences
5. DO NOT mention: utility, algorithm, score, threshold, calculation

Generate response:
`,

  COUNTER: (input, tone, concerns) => `
You are Accordo, a Procurement Manager. Generate a ${tone === 'formal' ? 'formal' : 'collaborative'} counter-offer response.

VENDOR'S OFFER:
- Price: $${input.vendorOffer.unit_price}/unit
- Payment: ${input.vendorOffer.payment_terms}
- Delivery: ${formatDelivery(input.vendorOffer)}

OUR COUNTER:
- Price: $${input.counterOffer.unit_price}/unit
- Payment: ${input.counterOffer.payment_terms}
- Delivery: ${formatDelivery(input.counterOffer)}

${concerns.length > 0 ? `VENDOR'S CONCERNS (acknowledge but pivot): ${concerns.map(c => c.text).join(', ')}` : ''}

BUSINESS REASONING (use one, don't reveal numbers):
- "This aligns better with our budget constraints"
- "We need faster delivery to meet project timelines"
- "Extended payment terms help our cash flow planning"

TONE: ${tone} (match formality level)

REQUIREMENTS:
1. ${concerns.length > 0 ? `First acknowledge: "I understand ${concerns[0].type} is a factor..."` : 'Acknowledge their offer respectfully'}
2. Present ALL THREE counter terms clearly
3. Provide brief business reasoning
4. Keep door open for negotiation
5. Length: 2-4 sentences
6. DO NOT mention: utility, algorithm, score, threshold, calculation

Generate response:
`,

  WALK_AWAY: (input, tone, concerns) => `
You are Accordo, a Procurement Manager. Generate a professional, regretful response declining to continue.

FINAL SITUATION:
- Vendor's offer: $${input.vendorOffer.unit_price}, ${input.vendorOffer.payment_terms}, ${formatDelivery(input.vendorOffer)}
- Round: ${input.deal.round} of ${input.config.max_rounds}
- Gap: Too far from our requirements

TONE: Professional regret (regardless of vendor's tone)

REQUIREMENTS:
1. Express genuine appreciation for their time and effort
2. Be clear but kind that we cannot proceed
3. Brief reason: "budget constraints" or "timeline requirements" (not specific numbers)
4. Leave door open for future opportunities
5. 2-3 sentences maximum
6. DO NOT blame or criticize their offer
7. DO NOT mention: utility, algorithm, score, threshold

Generate response:
`,

  ESCALATE: (input, tone) => `
You are Accordo, a Procurement Manager. The negotiation needs human review.

SITUATION: Complex situation requiring human decision-maker involvement

TONE: Reassuring and professional

REQUIREMENTS:
1. Reassure vendor their offer is being seriously considered
2. Explain a colleague will review (don't say "escalate")
3. Maintain positive tone about potential partnership
4. 2-3 sentences

Generate response:
`,
};
```

#### 4.5 Enhanced Fallback Templates

```typescript
const ENHANCED_FALLBACKS = {
  ACCEPT: (input) => {
    const templates = [
      `Great news! We accept your offer of $${input.vendorOffer.unit_price}/unit with ${input.vendorOffer.payment_terms} and delivery by ${formatDeliveryShort(input.vendorOffer)}. Thank you for working with us!`,
      `Excellent! We have a deal - $${input.vendorOffer.unit_price} per unit, ${input.vendorOffer.payment_terms}, delivered by ${formatDeliveryShort(input.vendorOffer)}. Looking forward to a great partnership.`,
      `I'm pleased to confirm we accept: $${input.vendorOffer.unit_price}/unit, ${input.vendorOffer.payment_terms}, delivery ${formatDeliveryShort(input.vendorOffer)}. Appreciate your flexibility!`,
    ];
    return templates[Math.floor(Math.random() * templates.length)];
  },

  COUNTER: (input) => {
    const templates = [
      `Thank you for your offer. We'd like to propose $${input.counterOffer.unit_price}/unit with ${input.counterOffer.payment_terms} and delivery by ${formatDeliveryShort(input.counterOffer)}. This better aligns with our project requirements.`,
      `I appreciate the offer of $${input.vendorOffer.unit_price}. Our counter: $${input.counterOffer.unit_price}, ${input.counterOffer.payment_terms}, delivery by ${formatDeliveryShort(input.counterOffer)}. Can we meet in the middle?`,
      `Your proposal is noted. Given our constraints, we can offer $${input.counterOffer.unit_price}/unit, ${input.counterOffer.payment_terms}, with delivery by ${formatDeliveryShort(input.counterOffer)}. Let me know your thoughts.`,
    ];
    return templates[Math.floor(Math.random() * templates.length)];
  },

  WALK_AWAY: (input) => {
    const templates = [
      `I appreciate your time and effort in this negotiation. Unfortunately, we're unable to proceed with the current terms. I hope we can work together on future opportunities.`,
      `Thank you for the discussions. Regrettably, the offer doesn't align with our current requirements. We'd welcome the chance to reconnect on future projects.`,
      `We've valued this negotiation, but can't move forward with these terms. Thank you for your understanding, and we hope to collaborate in the future.`,
    ];
    return templates[Math.floor(Math.random() * templates.length)];
  },

  ESCALATE: () => {
    const templates = [
      `This requires some additional review from our team. A colleague will follow up with you shortly to continue the discussion.`,
      `I'd like to bring in a colleague to review this further. Someone will be in touch soon to continue our conversation.`,
      `Let me involve a team member to give this the attention it deserves. You'll hear from us shortly.`,
    ];
    return templates[Math.floor(Math.random() * templates.length)];
  },
};
```

---

### Phase 5: Integration & Update processVendorTurn

**Goal**: Wire everything together

#### 5.1 Update Main Processing Function

**File**: `src/modules/chatbot/engine/processVendorTurn.ts`

```typescript
// BEFORE: Single function that does everything
export async function processVendorTurn(input) {
  // ... parse, decide, generate response, save both messages
  return { vendorMessage, accordoMessage, decision };
}

// AFTER: Split into two functions

/**
 * Phase 1: Save vendor message only (instant)
 */
export async function saveVendorMessage(input: {
  dealId: string;
  userId: number;
  content: string;
}): Promise<{
  message: Message;
  deal: Deal;
  extractedOffer: Offer;
}> {
  // 1. Load deal
  const deal = await ChatbotDeal.findByPk(input.dealId);

  // 2. Parse offer (now includes delivery)
  const extractedOffer = parseOfferWithDelivery(input.content);

  // 3. Save vendor message
  const message = await ChatbotMessage.create({
    dealId: input.dealId,
    role: 'VENDOR',
    content: input.content,
    extractedOffer,
  });

  // 4. Increment round
  await deal.update({ round: deal.round + 1 });

  return { message, deal, extractedOffer };
}

/**
 * Phase 2: Generate PM response (async, may be slow)
 */
export async function generatePMResponse(input: {
  dealId: string;
  vendorMessageId: string;
  userId: number;
}): Promise<{
  message: Message;
  decision: Decision;
  utility: WeightedUtilityResult;
  deal: Deal;
}> {
  // 1. Load context
  const deal = await ChatbotDeal.findByPk(input.dealId);
  const vendorMessage = await ChatbotMessage.findByPk(input.vendorMessageId);
  const messages = await ChatbotMessage.findAll({ where: { dealId: input.dealId } });
  const config = deal.configJson;

  // 2. Run decision engine (now with delivery utility)
  const decision = decideNextMove(config, vendorMessage.extractedOffer, deal.round);

  // 3. Compute explainability
  const explainability = computeExplainability(config, vendorMessage.extractedOffer, decision);

  // 4. Generate human-like response (LLM with fallback)
  const responseText = await generateHumanLikeResponse({
    decision,
    config,
    conversationHistory: messages,
    vendorOffer: vendorMessage.extractedOffer,
    counterOffer: decision.counterOffer,
    dealConfig: deal.configJson,
  });

  // 5. Save PM message
  const pmMessage = await ChatbotMessage.create({
    dealId: input.dealId,
    role: 'ACCORDO',
    content: responseText,
    engineDecision: decision,
    decisionAction: decision.action,
    utilityScore: decision.utilityScore,
    counterOffer: decision.counterOffer,
    explainabilityJson: explainability,
  });

  // 6. Update deal status if terminal
  if (['ACCEPT', 'WALK_AWAY'].includes(decision.action)) {
    await deal.update({ status: decision.action === 'ACCEPT' ? 'ACCEPTED' : 'WALKED_AWAY' });
  }

  return {
    message: pmMessage,
    decision,
    utility: explainability.utility,
    deal,
  };
}
```

#### 5.2 Update Controller

**File**: `src/modules/chatbot/chatbot.controller.ts`

```typescript
// NEW: Save vendor message only
export const saveVendorMessageOnly = async (req, res, next) => {
  const { dealId } = req.params;
  const { content } = req.body;
  const userId = req.context.userId;

  const result = await saveVendorMessage({ dealId, userId, content });

  res.json({
    message: 'Vendor message saved',
    data: {
      message: result.message,
      deal: result.deal,
      pmProcessing: true,
    },
  });
};

// NEW: Get PM response
export const getPMResponseAsync = async (req, res, next) => {
  const { dealId } = req.params;
  const { vendorMessageId } = req.body;
  const userId = req.context.userId;

  const result = await generatePMResponse({ dealId, vendorMessageId, userId });

  res.json({
    message: 'PM response generated',
    data: result,
  });
};

// NEW: Get PM fallback (for timeout)
export const getPMFallbackResponse = async (req, res, next) => {
  const { dealId } = req.params;
  const { vendorMessageId } = req.body;

  // Generate and save fallback response
  const result = await generateFallbackPMResponse({ dealId, vendorMessageId });

  res.json({
    message: 'PM fallback response generated',
    data: result,
  });
};
```

#### 5.3 Update Routes

**File**: `src/modules/chatbot/chatbot.routes.ts`

```typescript
// NEW endpoints
router.post(
  '/requisitions/:rfqId/vendors/:vendorId/deals/:dealId/vendor-message',
  authMiddleware,
  validateRequest(vendorMessageSchema),
  chatbotController.saveVendorMessageOnly
);

router.post(
  '/requisitions/:rfqId/vendors/:vendorId/deals/:dealId/pm-response',
  authMiddleware,
  validateRequest(pmResponseSchema),
  chatbotController.getPMResponseAsync
);

router.post(
  '/requisitions/:rfqId/vendors/:vendorId/deals/:dealId/pm-fallback',
  authMiddleware,
  chatbotController.getPMFallbackResponse
);
```

---

### Phase 6: Frontend Service Updates

**File**: `src/services/chatbot.service.ts`

```typescript
/**
 * Save vendor message only (instant API)
 * Returns immediately with saved message
 */
saveVendorMessage: async (
  ctx: DealContext,
  content: string
): Promise<{ data: { message: Message; deal: Deal; pmProcessing: boolean } }> => {
  const url = buildDealUrl(ctx.rfqId, ctx.vendorId, ctx.dealId, 'vendor-message');
  const res = await authApi.post(url, { content });
  return res.data;
},

/**
 * Get PM response (async API)
 * May take 1-5 seconds
 */
getPMResponse: async (
  ctx: DealContext,
  vendorMessageId: string,
  options?: { signal?: AbortSignal }
): Promise<{ data: { message: Message; decision: Decision; utility: any; deal: Deal } }> => {
  const url = buildDealUrl(ctx.rfqId, ctx.vendorId, ctx.dealId, 'pm-response');
  const res = await authApi.post(url, { vendorMessageId }, { signal: options?.signal });
  return res.data;
},

/**
 * Get PM fallback response (for timeout scenarios)
 */
getPMFallbackResponse: async (
  ctx: DealContext,
  vendorMessageId: string
): Promise<{ data: { message: Message; decision: Decision; deal: Deal } }> => {
  const url = buildDealUrl(ctx.rfqId, ctx.vendorId, ctx.dealId, 'pm-fallback');
  const res = await authApi.post(url, { vendorMessageId });
  return res.data;
},
```

---

### Phase 7: Testing & Validation

#### Test Scenarios

1. **Instant Display**: Vendor clicks chip → Message appears < 100ms
2. **Typing Indicator**: Dots animate while PM generates
3. **PM Response**: Human-like response with delivery terms
4. **Timeout Fallback**: After 5s, fallback template appears
5. **Message Queue**: Send while PM typing → Queued message shows grayed
6. **Tone Mirroring**: Formal vendor → Formal PM, Casual → Casual
7. **Concern Acknowledgment**: Vendor mentions costs → PM acknowledges
8. **Delivery Parsing**: "by March 15" correctly parsed
9. **All Actions**: ACCEPT, COUNTER, WALK_AWAY, ESCALATE all work
10. **Round Enforcement**: 1 vendor : 1 PM per round maintained

---

## File Changes Summary

| File | Change Type | Phase | Description |
|------|-------------|-------|-------------|
| `engine/types.ts` | Modify | 3 | Add delivery to Offer interface |
| `engine/parseOffer.ts` | Modify | 3 | Add delivery parsing |
| `engine/deliveryUtility.ts` | **New** | 3 | Delivery utility calculation |
| `engine/decide.ts` | Modify | 3 | Include delivery in decisions |
| `engine/responseGenerator.ts` | **New** | 4 | Unified LLM response generator |
| `engine/toneDetector.ts` | **New** | 4 | Vendor tone detection |
| `engine/concernExtractor.ts` | **New** | 4 | Vendor concern identification |
| `engine/processVendorTurn.ts` | Modify | 5 | Split into two functions |
| `chatbot.controller.ts` | Modify | 5 | Add new endpoints |
| `chatbot.routes.ts` | Modify | 5 | Add new routes |
| `chatbot.service.ts` (frontend) | Modify | 6 | Add new API methods |
| `useDealActions.ts` or hook | Modify | 6 | Instant display logic |
| `TypingIndicator.tsx` | **New** | 2 | Typing animation component |
| `ChatTranscript.tsx` | Modify | 2 | Support queued messages |

---

## Success Criteria

### Part A (Instant Display)
- [ ] Vendor message appears in < 100ms after click
- [ ] Typing indicator shows while PM generates
- [ ] PM response appears when ready (1-5 seconds)
- [ ] Fallback shown if PM takes > 5 seconds
- [ ] Queued messages show grayed with badge
- [ ] Sequential processing maintained (1:1 turns)

### Part B (Human-like Responses)
- [ ] Every PM response includes delivery terms
- [ ] PM acknowledges vendor concerns when relevant
- [ ] PM tone matches vendor tone
- [ ] No utility scores exposed to vendor
- [ ] High variation in responses
- [ ] Dynamic length based on complexity

---

## Dependencies

- Ollama LLM running on localhost:11434
- llama3.1 or llama3.2 model available
- Existing decision engine infrastructure
- React hooks for state management

---

## Rollback Plan

1. Feature flag `USE_SPLIT_API` can revert to single API
2. Feature flag `USE_ENHANCED_RESPONSES` can use old templates
3. Timeout fallback ensures responses always appear
