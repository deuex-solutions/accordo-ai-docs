# Final Implementation Plan - Complete Feature Integration

**Date**: January 4, 2026
**Scope**: Full feature parity with accordo-chatbot + enhanced features
**Timeline**: Aggressive (as fast as possible)
**Testing**: Full coverage (frontend + backend)

---

## Executive Summary

### What We're Building
Complete integration of all accordo-chatbot features into Accordo-ai platform while maintaining Accordo-ai's superior architecture (Sequelize, JWT, Template CRUD).

### Success Criteria
- ✅ All 3 demo scenarios work (HARD, SOFT, WALK_AWAY)
- ✅ Autopilot vendor simulation functional
- ✅ Natural conversation templates (5-7 variations per intent)
- ✅ Advanced conversation state machine
- ✅ Refusal handling, small talk, human escalation
- ✅ Full explainability panels
- ✅ Demo analytics and transcript replay
- ✅ Conversation analytics
- ✅ Full test coverage (>80%)

---

## Phase 1: Critical Demo Functionality
**Duration**: Aggressive timeline - Parallel development
**Deliverable**: Working demo scenarios with autopilot vendor

### Backend Tasks (Priority: CRITICAL)

#### 1.1 Vendor Simulation System
**Files to create**:
- `src/modules/chatbot/vendor/vendorSimulator.service.ts` (NEW)
- `src/modules/chatbot/vendor/vendorSimulator.controller.ts` (NEW)

**Endpoint**: `POST /api/chatbot/vendor/deals/:dealId/vendor/next`

**Request**:
```typescript
{
  "scenario": "HARD" | "SOFT" | "WALK_AWAY"
}
```

**Response**:
```typescript
{
  "message": Message,  // Generated vendor message
  "deal": Deal         // Updated deal state
}
```

**Source Reference**: `accordo-chatbot/backend/src/routes/vendorSim.ts`

**Logic**:
1. Load deal and current state
2. Determine vendor policy based on scenario
3. Use vendorAgent to generate response
4. Create vendor message in database
5. Return message

**Estimated Effort**: 4-6 hours

---

#### 1.2 Run Demo Endpoint
**Files to modify**:
- `src/modules/chatbot/chatbot.controller.ts` (ADD runDemo function)
- `src/modules/chatbot/chatbot.service.ts` (ADD runDemoService function)
- `src/modules/chatbot/chatbot.routes.ts` (ADD route)

**Endpoint**: `POST /api/chatbot/deals/:dealId/run-demo`

**Request**:
```typescript
{
  "scenario": "HARD" | "SOFT" | "WALK_AWAY",
  "maxRounds"?: number  // Default: 10
}
```

**Response**:
```typescript
{
  "deal": Deal,
  "messages": Message[],
  "steps": Array<{
    "vendorMessage": Message,
    "accordoMessage": Message,
    "round": number
  }>,
  "finalStatus": DealStatus,
  "totalRounds": number,
  "finalUtility": number
}
```

**Source Reference**: `accordo-chatbot/backend/src/routes/deals.ts` (lines 163-244)

**Logic**:
1. Reset deal to initial state
2. Set vendor scenario policy
3. Loop until terminal state (ACCEPTED, WALKED_AWAY, ESCALATED):
   - Generate vendor message using vendorAgent
   - Process vendor message (extract offer)
   - Run decision engine
   - Generate Accordo response
   - Save both messages
   - Check if deal is in terminal state
4. Return complete transcript with steps array

**Estimated Effort**: 6-8 hours

---

#### 1.3 Resume Escalated Deal Endpoint
**Files to modify**:
- `src/modules/chatbot/chatbot.controller.ts` (ADD resumeDeal function)
- `src/modules/chatbot/chatbot.service.ts` (ADD resumeDealService function)
- `src/modules/chatbot/chatbot.routes.ts` (ADD route)

**Endpoint**: `POST /api/chatbot/deals/:dealId/resume`

**Logic**:
1. Verify deal is ESCALATED
2. Change status to NEGOTIATING
3. Add system message: "Deal resumed by human"
4. Return updated deal

**Estimated Effort**: 2-3 hours

---

#### 1.4 Demo Analytics System
**Files to create**:
- `src/modules/chatbot/analytics/demoAnalytics.service.ts` (NEW)
- `src/modules/chatbot/analytics/demoAnalytics.repo.ts` (NEW)

**Database Changes**:
- Add `demo_runs` table (optional - can use deal metadata)

**Endpoints**:
- `GET /api/chatbot/analytics/demos` - List demo runs
- `GET /api/chatbot/analytics/demos/:dealId` - Get demo statistics
- `GET /api/chatbot/deals/:dealId/replay` - Get transcript for replay

**What to track**:
- Scenario type
- Number of rounds
- Final status
- Final utility
- Duration (timestamp tracking)
- Success rate per scenario

**Estimated Effort**: 4-5 hours

---

### Frontend Tasks (Priority: CRITICAL)

#### 1.5 VendorControls Component
**File to create**: `src/components/chatbot/chat/VendorControls.tsx`

**Source**: `accordo-chatbot/frontend/src/components/chat/VendorControls.tsx`

**Features**:
- Scenario selector dropdown (HARD, SOFT, WALK_AWAY)
- "Auto Next" button for single vendor turn
- "Run Full Demo" button for complete negotiation
- Progress indicator during autopilot
- Round counter
- Stop button to halt autopilot

**Props**:
```typescript
interface VendorControlsProps {
  dealId: string;
  dealStatus: DealStatus;
  currentRound: number;
  maxRounds: number;
  onVendorMessage: (message: Message) => void;
  disabled?: boolean;
}
```

**State Management**:
- Selected scenario
- Autopilot running state
- Current step in full demo

**API Integration**:
- `POST /api/chatbot/vendor/deals/:dealId/vendor/next` for single turn
- `POST /api/chatbot/deals/:dealId/run-demo` for full demo

**Estimated Effort**: 4-5 hours

---

#### 1.6 OutcomeBanner Component
**File to create**: `src/components/chatbot/chat/OutcomeBanner.tsx`

**Source**: `accordo-chatbot/frontend/src/components/chat/OutcomeBanner.tsx`

**Features**:
- Color-coded banner based on status:
  - ACCEPTED: Green with checkmark
  - WALKED_AWAY: Red with X
  - ESCALATED: Yellow with alert
- Final offer display
- Final utility score with progress bar
- Outcome message
- Action buttons (Reset, View Summary, Save Transcript)

**Props**:
```typescript
interface OutcomeBannerProps {
  status: DealStatus;
  finalOffer?: Offer;
  finalUtility?: number;
  onReset?: () => void;
  onViewSummary?: () => void;
  onSaveTranscript?: () => void;
}
```

**Estimated Effort**: 2-3 hours

---

#### 1.7 Complete DemoScenarios Page
**File to modify**: `src/pages/chatbot/DemoScenarios.tsx`

**Current State**: Skeleton with placeholder
**Target State**: Full integration with run-demo endpoint

**Features to add**:
1. **Scenario Cards** (existing, enhance):
   - HARD: Vendor starts at $95, reluctant to move
   - SOFT: Vendor starts at $90, willing to negotiate
   - WALK_AWAY: Vendor starts at $95, walks away at $92
   - Each card shows expected rounds, success rate

2. **Start Demo Flow**:
   - Click scenario → Create new deal
   - Auto-run demo with selected scenario
   - Real-time message display (streaming simulation)
   - Progress bar showing rounds

3. **Demo Results**:
   - OutcomeBanner with final results
   - Full transcript display
   - Save transcript button
   - "Try Again" button

4. **Demo History**:
   - List of recent demo runs
   - Quick replay functionality
   - Statistics (avg rounds, success rate per scenario)

**State Management**:
```typescript
const [running, setRunning] = useState(false);
const [currentDeal, setCurrentDeal] = useState<Deal | null>(null);
const [messages, setMessages] = useState<Message[]>([]);
const [demoSteps, setDemoSteps] = useState<DemoStep[]>([]);
const [selectedScenario, setSelectedScenario] = useState<ScenarioType | null>(null);
```

**API Integration**:
- `POST /api/chatbot/deals` - Create demo deal
- `POST /api/chatbot/deals/:dealId/run-demo` - Run demo
- `GET /api/chatbot/analytics/demos` - Load demo history

**Estimated Effort**: 6-8 hours

---

#### 1.8 ExplainabilityPanel Component (Enhanced)
**File to create**: `src/components/chatbot/ExplainabilityPanel.tsx`

**Source**: `accordo-chatbot/frontend/src/components/deal/ExplainabilityPanel.tsx`

**Current**: ExplainDrawer exists but simpler
**Enhancement**: Add detailed breakdowns

**Features**:
- **Offer Comparison Table**:
  - Vendor offer vs Target vs Anchor
  - Visual progress bars

- **Utility Breakdown**:
  - Price utility calculation step-by-step
  - Terms utility with explanations
  - Weighted scores
  - Total utility

- **Decision Reasoning**:
  - List of decision factors
  - Rule explanations (threshold checks)
  - Counter-offer justification

- **Config Snapshot**:
  - Show config used for this decision
  - Weights, thresholds, parameters

**Props**:
```typescript
interface ExplainabilityPanelProps {
  explainability: Explainability;
  isOpen: boolean;
  onClose: () => void;
  mode?: 'drawer' | 'modal' | 'inline';
}
```

**Estimated Effort**: 4-5 hours

---

### Testing (Phase 1)

#### Backend Tests
**Files to create**:
- `src/modules/chatbot/vendor/vendorSimulator.service.test.ts`
- `src/modules/chatbot/chatbot.service.test.ts` (add runDemo tests)
- `src/modules/chatbot/analytics/demoAnalytics.service.test.ts`

**Test Coverage**:
- Vendor simulation with all 3 scenarios
- Full demo run for each scenario
- Edge cases (max rounds, early termination)
- Resume escalated deals
- Demo analytics calculations

**Estimated Effort**: 6-8 hours

#### Frontend Tests
**Files to create**:
- `src/components/chatbot/chat/VendorControls.test.tsx`
- `src/components/chatbot/chat/OutcomeBanner.test.tsx`
- `src/components/chatbot/ExplainabilityPanel.test.tsx`
- `src/pages/chatbot/DemoScenarios.test.tsx`

**Test Coverage**:
- Component rendering
- User interactions (button clicks, scenario selection)
- API integration mocks
- Error handling
- Loading states

**Estimated Effort**: 6-8 hours

---

## Phase 2: Conversation Enhancement
**Duration**: Aggressive timeline - Parallel development
**Deliverable**: Production-ready conversation mode with natural templates

### Backend Tasks (Priority: HIGH)

#### 2.1 Conversation Template System
**File to port/enhance**: `src/modules/chatbot/convo/conversationManager.ts`

**Source**: `accordo-chatbot/backend/src/engine/conversationManager.ts` (255 lines)

**What to port**:
1. **Template Library** - 5-7 variations per intent:
   ```typescript
   const GREETING_TEMPLATES = [
     "Hello! I'm excited to discuss {projectName} with you.",
     "Hi there! Thanks for connecting about {projectName}.",
     "Good day! Let's talk about {projectName}.",
     // ... 4 more variations
   ];

   const ASK_OFFER_TEMPLATES = [
     "Could you share your pricing for {projectName}?",
     "What's your best offer for {projectName}?",
     "I'd love to hear your proposal for {projectName}.",
     // ... 4 more variations
   ];

   // Similar for: ASK_CLARIFY, COUNTER, ACCEPT, ESCALATE, WALK_AWAY
   ```

2. **Template Selection Logic**:
   ```typescript
   function selectTemplate(
     dealId: string,
     round: number,
     intent: ConversationIntent
   ): string {
     // Deterministic selection based on dealId + round + intent
     // Ensures same deal always gets same templates
     const templates = TEMPLATE_MAP[intent];
     const hash = hashDealRound(dealId, round);
     const index = hash % templates.length;
     return templates[index];
   }
   ```

3. **Variable Substitution**:
   ```typescript
   function substituteVariables(
     template: string,
     context: {
       projectName?: string;
       counterparty?: string;
       targetPrice?: number;
       currentOffer?: number;
       // ... other variables
     }
   ): string {
     return template.replace(/\{(\w+)\}/g, (match, key) => {
       return context[key]?.toString() || match;
     });
   }
   ```

4. **Phase Tracking**:
   - GREET → ASK_OFFER → NEGOTIATING → CLOSED
   - Ensure proper phase transitions

**Integration**:
- Update `conversationService.ts` to use template system
- Replace simple LLM calls with template selection + LLM enhancement (optional)

**Estimated Effort**: 8-10 hours

---

#### 2.2 Enhanced Conversation Router
**File to enhance**: `src/modules/chatbot/chatbot.routes.ts` and `chatbot.controller.ts`

**Source**: `accordo-chatbot/backend/src/convo/convoRouter.ts` (408 lines)

**Features to add**:

1. **Refusal Classification**:
   ```typescript
   type RefusalType = 'NO' | 'LATER' | 'ALREADY_SHARED' | 'CONFUSED';

   function classifyRefusal(message: string): RefusalType | null {
     // Detect refusal patterns
     if (/no|not interested|don't want/i.test(message)) return 'NO';
     if (/later|another time|maybe later/i.test(message)) return 'LATER';
     if (/already (told|shared|said)/i.test(message)) return 'ALREADY_SHARED';
     if (/don't understand|confused|what/i.test(message)) return 'CONFUSED';
     return null;
   }
   ```

2. **Refusal Handling Logic**:
   ```typescript
   async function handleRefusal(
     deal: Deal,
     refusalType: RefusalType,
     refusalCount: number
   ): Promise<ConversationMessageResponse> {
     if (refusalCount >= 3) {
       // Escalate after 3 refusals
       return escalateConversation(deal, "Multiple refusals detected");
     }

     // Select appropriate response based on refusal type
     const response = selectRefusalResponse(refusalType);
     return createConversationMessage(deal, response, 'ASK_CLARIFY');
   }
   ```

3. **Intent-based Branching**:
   - GREET → greeting response → ASK_OFFER
   - ASK_OFFER → wait for offer
   - PROVIDE_OFFER → extract offer → decide → respond
   - REFUSAL → handle refusal → retry or escalate
   - SMALL_TALK → acknowledge → redirect to offer
   - ASK_CLARIFY → provide clarification

4. **Multi-turn Context**:
   ```typescript
   interface ConversationContext {
     lastIntent: ConversationIntent;
     lastVendorMessage: string;
     turnsSinceOffer: number;
     refusalCount: number;
     offersReceived: Offer[];
     clarificationAsked: boolean;
   }
   ```

5. **Small Talk Detection**:
   ```typescript
   function detectSmallTalk(message: string): boolean {
     const smallTalkPatterns = [
       /how are you|how's it going/i,
       /weather|weekend|holiday/i,
       /nice to meet|pleasure to/i,
       // ... more patterns
     ];
     return smallTalkPatterns.some(pattern => pattern.test(message));
   }
   ```

**Controller Methods to Add**:
- `handleConversationTurn` - Main conversation processor
- `classifyIntent` - Intent classification
- `handleRefusal` - Refusal logic
- `handleSmallTalk` - Small talk responses
- `escalateConversation` - Human escalation

**Estimated Effort**: 12-15 hours

---

#### 2.3 Process Conversation Turn Module
**File to create**: `src/modules/chatbot/engine/processConversationTurn.ts`

**Source**: `accordo-chatbot/backend/src/engine/processConversationTurn.ts`

**Purpose**: Dedicated turn processor for conversation mode

**Functions**:
```typescript
export async function processConversationTurn(
  deal: Deal,
  vendorMessage: string,
  conversationState: ConversationState
): Promise<{
  accordoMessage: Message;
  updatedState: ConversationState;
  revealAvailable: boolean;
}> {
  // 1. Classify intent
  const intent = await classifyIntent(vendorMessage, conversationState);

  // 2. Handle based on intent
  switch (intent) {
    case 'PROVIDE_OFFER':
      return handleOfferProvided(deal, vendorMessage, conversationState);
    case 'REFUSAL':
      return handleRefusal(deal, vendorMessage, conversationState);
    case 'SMALL_TALK':
      return handleSmallTalk(deal, vendorMessage, conversationState);
    case 'ASK_CLARIFY':
      return handleClarification(deal, vendorMessage, conversationState);
    default:
      return handleDefault(deal, vendorMessage, conversationState);
  }
}
```

**Estimated Effort**: 4-5 hours

---

#### 2.4 Conversation Analytics System
**Files to create**:
- `src/modules/chatbot/analytics/conversationAnalytics.service.ts` (NEW)
- `src/modules/chatbot/analytics/conversationAnalytics.repo.ts` (NEW)

**Database Changes**:
- Add analytics fields to `ConversationState` or create `conversation_analytics` table

**Endpoints**:
- `GET /api/chatbot/analytics/conversations` - List conversations with stats
- `GET /api/chatbot/analytics/conversations/:dealId` - Get detailed analytics
- `GET /api/chatbot/analytics/conversations/aggregate` - Aggregate statistics

**What to track**:
- Total turns
- Refusal count by type
- Small talk occurrences
- Time to first offer
- Clarification requests
- Success rate (offer received vs escalated)
- Intent distribution
- Average conversation length
- Escalation reasons

**Analytics Models**:
```typescript
interface ConversationAnalytics {
  dealId: string;
  totalTurns: number;
  refusalsByType: Record<RefusalType, number>;
  smallTalkCount: number;
  turnsToFirstOffer: number;
  clarificationRequests: number;
  intentDistribution: Record<ConversationIntent, number>;
  outcome: 'OFFER_RECEIVED' | 'ESCALATED' | 'IN_PROGRESS';
  escalationReason?: string;
  duration: number; // milliseconds
  createdAt: Date;
  completedAt?: Date;
}
```

**Estimated Effort**: 5-6 hours

---

### Frontend Tasks (Priority: HIGH)

#### 2.5 useConversation Hook
**File to create**: `src/hooks/chatbot/useConversation.ts`

**Source**: `accordo-chatbot/frontend/src/hooks/useConversation.ts`

**Purpose**: Centralized conversation state management

**API**:
```typescript
export function useConversation(dealId: string) {
  const [state, setState] = useState<ConversationState | null>(null);
  const [messages, setMessages] = useState<Message[]>([]);
  const [loading, setLoading] = useState(false);
  const [sending, setSending] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [revealAvailable, setRevealAvailable] = useState(false);

  const startConversation = async (): Promise<void> => {
    // POST /api/chatbot/conversation/deals/:dealId/start
  };

  const sendMessage = async (content: string): Promise<void> => {
    // POST /api/chatbot/conversation/deals/:dealId/messages
  };

  const loadExplainability = async (): Promise<Explainability | null> => {
    // GET /api/chatbot/conversation/deals/:dealId/explainability
  };

  const reload = async (): Promise<void> => {
    // Reload deal and messages
  };

  return {
    state,
    messages,
    loading,
    sending,
    error,
    revealAvailable,
    startConversation,
    sendMessage,
    loadExplainability,
    reload,
  };
}
```

**Features**:
- Auto-load conversation state
- Message sending with optimistic updates
- Error handling with retry
- Conversation phase tracking
- Refusal count display
- Turn count display

**Estimated Effort**: 3-4 hours

---

#### 2.6 Conversation Analytics Dashboard
**File to create**: `src/pages/chatbot/ConversationAnalytics.tsx` (NEW)

**Features**:
1. **Overview Cards**:
   - Total conversations
   - Success rate (offers received)
   - Average turns to offer
   - Escalation rate

2. **Charts**:
   - Intent distribution (pie chart)
   - Refusal types (bar chart)
   - Turns per conversation (histogram)
   - Time to offer (line chart)

3. **Conversation List**:
   - Table with: Deal, Status, Turns, Refusals, Outcome, Duration
   - Click to view detailed conversation

4. **Detailed Conversation View**:
   - Full transcript
   - Intent annotations
   - Refusal highlights
   - Timeline visualization
   - Escalation details (if escalated)

**Dependencies**:
- React Chart.js (already in dependencies)
- Date formatting (date-fns already in dependencies)

**Estimated Effort**: 8-10 hours

---

#### 2.7 Enhanced Conversation UI Components
**Files to enhance**:
- `src/pages/chatbot/ConversationRoom.tsx` - Add phase indicator, refusal counter
- `src/components/chatbot/conversation/ConversationMessageBubble.tsx` - Add intent badges

**New Features**:
1. **Phase Indicator** (top of chat):
   ```
   [Greeting] → [Asking for Offer] → [Negotiating] → [Closed]
   ```
   - Visual progress bar
   - Current phase highlighted

2. **Refusal Counter** (warning banner):
   - Shows after first refusal
   - "Vendor has refused 2 times. One more refusal may escalate."

3. **Intent Badges** on messages:
   - Small colored badge showing intent (GREET, ASK_OFFER, REFUSAL, etc.)
   - Hover tooltip with explanation

4. **Small Talk Detection** (subtle indicator):
   - Icon showing small talk was detected
   - Tooltip: "Small talk detected - redirecting to business"

**Estimated Effort**: 4-5 hours

---

### Testing (Phase 2)

#### Backend Tests
**Files to create**:
- `src/modules/chatbot/convo/conversationManager.test.ts`
- `src/modules/chatbot/engine/processConversationTurn.test.ts`
- `src/modules/chatbot/analytics/conversationAnalytics.test.ts`

**Test Coverage**:
- Template selection determinism
- Variable substitution
- Intent classification
- Refusal handling (all 4 types)
- Small talk detection
- Multi-turn context preservation
- Phase transitions
- Escalation triggers
- Analytics calculations

**Estimated Effort**: 10-12 hours

#### Frontend Tests
**Files to create**:
- `src/hooks/chatbot/useConversation.test.ts`
- `src/pages/chatbot/ConversationAnalytics.test.tsx`
- Enhanced tests for ConversationRoom.tsx

**Test Coverage**:
- Hook state management
- Message sending flow
- Error handling
- Analytics dashboard rendering
- Chart data visualization
- Conversation detail view

**Estimated Effort**: 8-10 hours

---

## Phase 3: Polish & Advanced Features
**Duration**: Aggressive timeline
**Deliverable**: Fully featured, production-ready chatbot

### Backend Tasks

#### 3.1 Process Vendor Turn Module
**File to create**: `src/modules/chatbot/engine/processVendorTurn.ts`

**Source**: `accordo-chatbot/backend/src/engine/processVendorTurn.ts`

**Purpose**: Dedicated processor for vendor messages in INSIGHTS mode

**Functions**:
```typescript
export async function processVendorTurn(
  deal: Deal,
  vendorMessage: string
): Promise<{
  vendorMsg: Message;
  accordoMsg: Message;
  decision: Decision;
  updatedDeal: Deal;
}> {
  // 1. Create vendor message
  // 2. Extract offer from message
  // 3. Run decision engine
  // 4. Generate Accordo response
  // 5. Update deal state
  // 6. Return complete turn result
}
```

**Estimated Effort**: 3-4 hours

---

#### 3.2 Enhanced Error Handling
**Files to enhance**:
- `src/modules/chatbot/chatbot.service.ts` - Add detailed error messages
- `src/utils/custom-error.ts` - Add chatbot-specific error classes

**Error Classes to Add**:
```typescript
class DealNotFoundError extends CustomError {}
class InvalidDealStateError extends CustomError {}
class ConversationPhaseError extends CustomError {}
class OfferExtractionError extends CustomError {}
class VendorSimulationError extends CustomError {}
```

**Error Handling Improvements**:
- Detailed error messages for users
- Proper HTTP status codes
- Error logging with context
- Recovery suggestions

**Estimated Effort**: 3-4 hours

---

#### 3.3 Performance Optimizations
**Files to optimize**:
- `src/modules/chatbot/chatbot.repo.ts` - Add query optimization
- `src/modules/chatbot/chatbot.service.ts` - Add caching

**Optimizations**:
1. **Database Query Optimization**:
   - Add indexes on frequently queried fields
   - Use select fields to reduce data transfer
   - Implement pagination properly

2. **Caching Layer** (optional):
   - Cache negotiation configs
   - Cache templates
   - Cache deal states for active deals

3. **LLM Request Batching** (if using LLM heavily):
   - Batch multiple requests
   - Implement request queuing

**Estimated Effort**: 4-5 hours

---

### Frontend Tasks

#### 3.4 useHistoryTracking Hook
**File to create**: `src/hooks/chatbot/useHistoryTracking.ts`

**Source**: `accordo-chatbot/frontend/src/hooks/useHistoryTracking.ts`

**Purpose**: Browser history and navigation state persistence

**API**:
```typescript
export function useHistoryTracking() {
  const [recentDeals, setRecentDeals] = useState<Deal[]>([]);
  const [currentDealId, setCurrentDealId] = useState<string | null>(null);

  const trackDealView = (dealId: string): void => {
    // Add to recent deals
    // Update last viewed timestamp
    // Store in localStorage
  };

  const clearHistory = (): void => {
    // Clear recent deals
  };

  return {
    recentDeals,
    currentDealId,
    trackDealView,
    clearHistory,
  };
}
```

**Integration**:
- Use in DealsPage, NegotiationRoom, ConversationRoom
- Show "Recently Viewed" section in sidebar
- Persist across browser sessions

**Estimated Effort**: 2-3 hours

---

#### 3.5 History/Storage Services
**Files to create**:
- `src/services/storage/historyService.ts` (NEW)
- `src/services/storage/localStorageAdapter.ts` (NEW)
- `src/services/storage/types.ts` (NEW)

**Source**: `accordo-chatbot/frontend/src/services/storage/`

**Purpose**: Local caching and offline-first capabilities

**Features**:
1. **Deal State Caching**:
   - Cache deal data in localStorage
   - Auto-sync with server
   - Optimistic updates

2. **Offline Support**:
   - Queue messages when offline
   - Sync when back online
   - Show offline indicator

3. **History Management**:
   - Recently viewed deals
   - Search history
   - Favorite deals

**Storage Schema**:
```typescript
interface StorageSchema {
  deals: Record<string, Deal>;
  messages: Record<string, Message[]>;
  recentDeals: string[];
  favoriteDeals: string[];
  lastSync: number;
}
```

**Estimated Effort**: 3-4 hours

---

#### 3.6 Enhanced UI/UX Polish
**Files to enhance**:
- All chatbot pages - Add loading skeletons
- All components - Add animations
- Error states - Better error messages

**Improvements**:
1. **Loading States**:
   - Skeleton loaders for deal cards
   - Message typing indicators
   - Progress bars for long operations

2. **Animations**:
   - Smooth transitions between pages
   - Message slide-in animations
   - Status change animations

3. **Accessibility**:
   - ARIA labels
   - Keyboard navigation
   - Screen reader support

4. **Responsive Design**:
   - Mobile-optimized layouts
   - Tablet support
   - Touch-friendly controls

**Estimated Effort**: 6-8 hours

---

### Testing (Phase 3)

#### Integration Tests
**Files to create**:
- `src/modules/chatbot/integration/fullNegotiation.test.ts`
- `src/modules/chatbot/integration/fullConversation.test.ts`
- `src/modules/chatbot/integration/demoScenarios.test.ts`

**Test Scenarios**:
1. **Full HARD Demo**:
   - Start → Vendor refuses → Counter → Accept/Walk/Escalate

2. **Full SOFT Demo**:
   - Start → Vendor accepts → Deal closes

3. **Full WALK_AWAY Demo**:
   - Start → Vendor walks away → Deal ends

4. **Full Conversation**:
   - Start → Greet → Ask offer → Refusal → Retry → Offer → Negotiate → Close

5. **Edge Cases**:
   - Max rounds exceeded
   - Invalid offers
   - LLM failures
   - Network errors

**Estimated Effort**: 10-12 hours

#### E2E Tests (Optional but Recommended)
**Tool**: Playwright or Cypress

**Test Flows**:
1. Create deal → Run HARD demo → View results
2. Create deal → Start conversation → Send messages → Close
3. View analytics dashboard
4. Save and replay transcript

**Estimated Effort**: 8-10 hours (if implementing E2E)

---

## Total Effort Estimate

### Backend
| Task | Hours |
|------|-------|
| Phase 1: Demo System | 16-22 |
| Phase 2: Conversation | 29-36 |
| Phase 3: Polish | 10-13 |
| Backend Tests | 26-32 |
| **Total Backend** | **81-103 hours** |

### Frontend
| Task | Hours |
|------|-------|
| Phase 1: Demo UI | 20-26 |
| Phase 2: Conversation UI | 23-29 |
| Phase 3: Polish & Features | 11-15 |
| Frontend Tests | 24-30 |
| **Total Frontend** | **78-100 hours** |

### **Grand Total: 159-203 hours (20-25 full days)**

---

## Parallel Development Strategy

To achieve fastest timeline, we'll work in parallel:

### Week 1-2: Foundation (Parallel Tracks)
**Backend Track**:
- Day 1-2: Vendor simulation endpoint
- Day 3-4: Run demo endpoint
- Day 5-6: Demo analytics
- Day 7-8: Backend tests for Phase 1

**Frontend Track**:
- Day 1-2: VendorControls component
- Day 3-4: OutcomeBanner component
- Day 5-6: Complete DemoScenarios page
- Day 7-8: ExplainabilityPanel
- Day 9-10: Frontend tests for Phase 1

### Week 3-4: Conversation (Parallel Tracks)
**Backend Track**:
- Day 1-3: Conversation template system
- Day 4-6: Enhanced convoRouter
- Day 7-8: Conversation analytics
- Day 9-10: Backend tests for Phase 2

**Frontend Track**:
- Day 1-2: useConversation hook
- Day 3-5: Conversation analytics dashboard
- Day 6-7: Enhanced conversation UI
- Day 8-10: Frontend tests for Phase 2

### Week 5: Polish & Integration
**Backend Track**:
- Day 1-2: Process vendor/conversation turn modules
- Day 3: Error handling
- Day 4-5: Performance optimizations

**Frontend Track**:
- Day 1-2: History tracking
- Day 3: Storage services
- Day 4-5: UI/UX polish

### Week 6: Testing & Launch
- Day 1-3: Integration tests
- Day 4-5: E2E tests (optional)
- Day 6: Bug fixes
- Day 7: Final verification & deployment

---

## Questions Before Final Approval

I have a few remaining clarifying questions before we begin implementation:

### Q1: Database Migrations
**Question**: Should I create database migrations for new tables/fields, or will you handle database schema changes?

**New tables/fields needed**:
- `demo_runs` table (optional - for demo analytics)
- `conversation_analytics` table (optional - for conversation analytics)
- New fields on `chatbot_deals` table (if any missing)

**Options**:
- A) Create Sequelize migrations for all changes
- B) You'll handle migrations manually
- C) Use JSONB fields in existing tables (no schema changes)

---

### Q2: LLM Integration Strategy
**Question**: For conversation templates, should we:

**Options**:
- A) Pure template system (deterministic, no LLM for reply generation)
- B) Template + LLM enhancement (template selection + LLM polishing)
- C) Full LLM (templates guide prompts, LLM generates replies)

**Recommendation**: Option A (pure templates) for speed and reliability, Option B for best of both worlds.

---

### Q3: Demo Transcript Storage
**Question**: Where should we store demo transcripts for replay?

**Options**:
- A) Keep in deals table (use existing messages)
- B) Separate `demo_transcripts` table
- C) Export to JSON files (download only, no storage)
- D) Local storage (browser-side)

**Recommendation**: Option A (simplest) or Option C (no database overhead).

---

### Q4: Conversation Analytics Storage
**Question**: How should we store conversation analytics?

**Options**:
- A) New `conversation_analytics` table (normalized)
- B) JSONB field in `chatbot_deals` table (simpler)
- C) Calculate on-the-fly from messages (no storage)

**Recommendation**: Option B (JSONB) for balance of simplicity and queryability.

---

### Q5: Error Recovery
**Question**: If LLM fails or offers can't be extracted, how should we handle it?

**Options**:
- A) Auto-escalate to human
- B) Retry with fallback logic
- C) Ask vendor to rephrase
- D) All of the above (smart fallback chain)

**Recommendation**: Option D (comprehensive error recovery).

---

### Q6: Performance vs Features
**Question**: If we need to optimize for speed, which features can we defer?

**Priority ranking** (please confirm or adjust):
1. **Must Have (Phase 1)**: Demo scenarios, vendor simulation, run-demo
2. **Should Have (Phase 2)**: Conversation templates, refusal handling
3. **Nice to Have (Phase 3)**: Analytics dashboards, history tracking, offline support

---

### Q7: Testing Approach
**Question**: Should we test as we go, or in a dedicated testing phase?

**Options**:
- A) Test-Driven Development (write tests first)
- B) Test after each feature (continuous testing)
- C) Testing phase at end (faster development)
- D) Hybrid (critical paths tested immediately, others at end)

**Recommendation**: Option B or D for balance.

---

### Q8: Code Style & Patterns
**Question**: Should I follow exact accordo-chatbot patterns or adapt to Accordo-ai style?

**Specific examples**:
- Variable naming: camelCase vs snake_case
- Error handling: throw vs return error objects
- Async patterns: async/await vs promises
- Component structure: hooks vs class components (hooks already decided)

**Recommendation**: Adapt to Accordo-ai existing patterns for consistency.

---

### Q9: Documentation
**Question**: What level of documentation do you want?

**Options**:
- A) Code comments only
- B) JSDoc for all functions
- C) Separate documentation files (MD)
- D) All of the above

---

### Q10: Deployment Support
**Question**: Do you need deployment support, or just code delivery?

**Options**:
- A) Code only (you'll deploy)
- B) Code + deployment guide
- C) Code + help with deployment
- D) End-to-end deployment

---

## Next Steps

Once you answer the 10 questions above, I will:

1. ✅ Create final task breakdown (file-by-file)
2. ✅ Set up development environment (if needed)
3. ✅ Begin Phase 1 implementation immediately
4. ✅ Provide daily progress updates
5. ✅ Deliver working features incrementally

---

**Status**: ⏳ AWAITING FINAL APPROVAL
**Ready to Start**: Yes, pending answers to 10 questions above
**Estimated Total Timeline**: 5-6 weeks (can be compressed with parallel work)
