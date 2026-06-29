# Quick Reference Guide - Phase 2 & Phase 3

Fast reference for all new components and modules.

---

## Frontend Components

### useConversation Hook

```typescript
import { useConversation } from '@/hooks/chatbot';

const {
  deal,              // Current deal object
  messages,          // Message array
  convoState,        // Conversation state
  loading,           // Initial load state
  sending,           // Message send state
  error,             // Error message
  sendMessage,       // Send vendor message
  startConversation, // Start new conversation
  reload,            // Reload deal data
  currentPhase,      // 'WAITING_FOR_OFFER' | 'NEGOTIATING' | 'TERMINAL'
  refusalCount,      // Number of refusals
  turnCount,         // Number of turns
  canSend,           // Can send message?
  warningLevel,      // 'none' | 'low' | 'medium' | 'high'
} = useConversation(dealId);
```

### ConversationAnalytics Component

```typescript
import { ConversationAnalytics } from '@/components/chatbot/analytics';

// Single deal
<ConversationAnalytics dealId="deal-123" />

// Multiple deals
<ConversationAnalytics
  deals={dealsList}
  timeRange="week" // 'day' | 'week' | 'month' | 'all'
/>
```

### ConversationEnhanced Component

```typescript
import { ConversationEnhanced } from '@/components/chatbot/chat';

<ConversationEnhanced
  dealId={dealId}
  showSidebar={true}
  showSuggestedResponses={true}
  className="h-full"
/>
```

### useHistoryTracking Hook

```typescript
import { useHistoryTracking, getRelativeTime } from '@/hooks/chatbot';

const {
  recentDeals,       // Array of recent deals
  addToHistory,      // Add deal to history
  removeFromHistory, // Remove deal from history
  clearHistory,      // Clear all history
  getLastViewed,     // Get last viewed deal
  isInHistory,       // Check if deal is in history
} = useHistoryTracking();

// Session-only tracking
const { ... } = useHistoryTracking(true);
```

### HistoryPanel Component

```typescript
import { HistoryPanel } from '@/components/chatbot/navigation';

<HistoryPanel
  onNavigate={(dealId) => navigate(`/deals/${dealId}`)}
  onClose={() => setOpen(false)}
  showClearButton={true}
  maxItems={10}
  className="w-64"
/>
```

---

## Backend Modules

### processVendorTurn Module

```typescript
import { processVendorTurn } from '@/modules/chatbot/engine/processVendorTurn';

const result = await processVendorTurn({
  dealId: 'deal-123',
  vendorMessage: 'I can offer $95 with Net 60 terms',
  userId: 1,
});

// Returns:
// {
//   extractedOffer: Offer | null,
//   decision: Decision,
//   accordoMessage: Message,
//   vendorMessageRecord: Message,
//   explainability: Explainability | null,
//   updatedDeal: Deal
// }
```

### Error Recovery Module

```typescript
import {
  retryWithBackoff,
  recoverConvoState,
  fallbackClassifyIntent,
  addToDeadLetterQueue,
} from '@/modules/chatbot/utils/errorRecovery';

// Retry with backoff
const result = await retryWithBackoff(
  () => llmService.classify(message),
  3,    // max retries
  1000  // base delay (ms)
);

// Recover corrupted state
const fixed = await recoverConvoState(dealId, corruptedState);

// Fallback classification
const intent = await fallbackClassifyIntent(message);

// Dead letter queue
const id = addToDeadLetterQueue('operation', input, error);
```

---

## Common Patterns

### Complete Conversation Page

```typescript
function ConversationPage({ dealId }) {
  const conversation = useConversation(dealId);
  const history = useHistoryTracking();

  useEffect(() => {
    if (conversation.deal) {
      history.addToHistory(conversation.deal);
    }
  }, [conversation.deal]);

  return (
    <div className="h-screen flex">
      <aside className="w-64">
        <HistoryPanel onNavigate={(id) => navigate(`/deals/${id}`)} />
      </aside>
      <main className="flex-1">
        <ConversationEnhanced dealId={dealId} />
      </main>
    </div>
  );
}
```

### Backend Message Handler

```typescript
import { processVendorTurn } from '@/modules/chatbot/engine/processVendorTurn';
import { retryWithBackoff } from '@/modules/chatbot/utils/errorRecovery';

export async function handleVendorMessage(req, res) {
  try {
    const result = await retryWithBackoff(
      () => processVendorTurn({
        dealId: req.params.dealId,
        vendorMessage: req.body.content,
        userId: req.context.userId,
      }),
      3
    );

    res.json({ success: true, data: result });
  } catch (error) {
    logger.error('Failed to process vendor message:', error);
    res.status(500).json({ success: false, error: error.message });
  }
}
```

### Analytics Dashboard

```typescript
function AnalyticsDashboard() {
  const { data: deals, loading } = useDeals();

  if (loading) return <Spinner />;

  return (
    <div className="p-6 space-y-6">
      <h1>Conversation Analytics</h1>
      <ConversationAnalytics
        deals={deals}
        timeRange="month"
      />
    </div>
  );
}
```

---

## Type Definitions

### Frontend Types

```typescript
// Phase
type ConversationPhase = 'WAITING_FOR_OFFER' | 'NEGOTIATING' | 'TERMINAL';

// Intent
type ConversationIntent =
  | 'GREET'
  | 'ASK_FOR_OFFER'
  | 'SMALL_TALK'
  | 'ASK_CLARIFY'
  | 'PROVIDE_OFFER'
  | 'REFUSAL'
  | 'UNKNOWN';

// Warning Level
type WarningLevel = 'none' | 'low' | 'medium' | 'high';

// History Entry
interface HistoryEntry {
  dealId: string;
  title: string;
  counterparty: string | null;
  status: DealStatus;
  timestamp: Date;
}
```

### Backend Types

```typescript
// Offer
interface Offer {
  unit_price: number | null;
  payment_terms: 'Net 30' | 'Net 60' | 'Net 90' | null;
  meta?: {
    raw_terms_days?: number;
    non_standard_terms?: boolean;
  };
}

// Decision
interface Decision {
  action: 'ACCEPT' | 'COUNTER' | 'WALK_AWAY' | 'ESCALATE' | 'ASK_CLARIFY';
  reasons: string[];
  counterOffer?: Offer | null;
  utilityScore?: number;
}

// Explainability
interface Explainability {
  vendorOffer: Offer;
  utilities: {
    priceUtility: number | null;
    termsUtility: number | null;
    weightedPrice: number | null;
    weightedTerms: number | null;
    total: number | null;
  };
  decision: Decision;
  configSnapshot: object;
}
```

---

## Color Codes

### Status Colors
- **NEGOTIATING**: Blue (`bg-blue-100 text-blue-700`)
- **ACCEPTED**: Green (`bg-green-100 text-green-700`)
- **WALKED_AWAY**: Gray (`bg-gray-100 text-gray-700`)
- **ESCALATED**: Orange (`bg-orange-100 text-orange-700`)

### Warning Levels
- **none**: No color
- **low**: Yellow (`bg-yellow-100 text-yellow-700`)
- **medium**: Orange (`bg-orange-100 text-orange-700`)
- **high**: Red (`bg-red-100 text-red-700`)

### Intent Colors
- **GREET**: Green
- **ASK_FOR_OFFER**: Blue
- **SMALL_TALK**: Gray
- **ASK_CLARIFY**: Yellow
- **PROVIDE_OFFER**: Purple
- **REFUSAL**: Red
- **UNKNOWN**: Gray

---

## File Paths

### Frontend
```
src/
├── hooks/chatbot/
│   ├── useConversation.ts
│   ├── useHistoryTracking.ts
│   └── index.ts
├── components/chatbot/
│   ├── analytics/
│   │   ├── ConversationAnalytics.tsx
│   │   └── index.ts
│   ├── chat/
│   │   ├── ConversationEnhanced.tsx
│   │   └── index.ts
│   └── navigation/
│       ├── HistoryPanel.tsx
│       └── index.ts
```

### Backend
```
src/modules/chatbot/
├── engine/
│   └── processVendorTurn.ts
└── utils/
    └── errorRecovery.ts
```

---

## Common Issues & Solutions

### Issue: "Deal not loading in useConversation"
**Solution**: Check that `dealId` is valid and backend API is running

### Issue: "History not persisting"
**Solution**: Check localStorage/sessionStorage is enabled and not full (5MB limit)

### Issue: "Warning level not updating"
**Solution**: Ensure `convoState.refusalCount` is being updated in backend

### Issue: "Transaction rollback not working"
**Solution**: Verify all database operations are within the transaction scope

### Issue: "Retry exhausted without success"
**Solution**: Check LLM service availability, increase max retries, or use fallback

---

## Keyboard Shortcuts

### ConversationEnhanced
- **Enter**: Send message
- **Shift+Enter**: New line in composer
- **Esc**: Close sidebar (if open)

---

## Performance Tips

1. **Memoize expensive calculations** in analytics
2. **Debounce scroll events** for auto-scroll
3. **Use optimistic updates** for messages
4. **Batch storage writes** in history tracking
5. **Use transaction pooling** in backend
6. **Cache negotiation configs** to reduce DB queries

---

## Testing Commands

### Frontend
```bash
npm test -- useConversation.test.ts
npm test -- ConversationAnalytics.test.tsx
npm test -- ConversationEnhanced.test.tsx
npm test -- useHistoryTracking.test.ts
npm test -- HistoryPanel.test.tsx
```

### Backend
```bash
npm test -- processVendorTurn.test.ts
npm test -- errorRecovery.test.ts
```

---

## Quick Debugging

### Frontend Console
```javascript
// Check history
localStorage.getItem('accordo_deal_history')

// Check session history
sessionStorage.getItem('accordo_deal_history_session')

// Clear history
localStorage.removeItem('accordo_deal_history')
```

### Backend Logs
```bash
# Search for processVendorTurn logs
grep "processVendorTurn" logs/app.log

# Search for retry attempts
grep "retry" logs/app.log
```

---

**Quick Ref Version**: 1.0.0
**Last Updated**: January 4, 2026
