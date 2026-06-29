# Implementation Plan: Add Delivery Terms to Suggestion Chips

## Overview

This plan outlines the implementation to add delivery terms to the negotiation suggestion chips in the Accordo chatbot. The suggestions will now include price, payment terms, AND delivery information with human-like message variations.

---

## Requirements Summary (Based on User Input)

| Requirement | Decision |
|-------------|----------|
| **Display Format** | Both: Date + Days (e.g., "Jan 15 (14 days)") |
| **Scenario Logic** | Fixed from config (use configured delivery date) |
| **UI Placement** | Inline with Price/Terms as third chip |
| **Editability** | Click-to-send only (no editing before send) |
| **Date Source** | Preferred date if available, else required date |
| **Message Content** | New human-like format including all three terms |
| **Migration** | Full migration - no backward compatibility |
| **Days Reference** | From today (current date) |
| **Missing Date Fallback** | Default to 30 days from today |
| **Message Variety** | Emphasis variation (price/terms/delivery/value focus) |
| **Fallback Quality** | Match LLM quality with pre-written human-like templates |

---

## Phase 1: Data Structure Changes

### 1.1 Define New TypeScript Interfaces

**Backend** (`src/modules/chatbot/engine/types.ts`):
```typescript
export interface StructuredSuggestion {
  message: string;              // Human-like message text
  price: number;                // Unit price value
  paymentTerms: string;         // e.g., "Net 30", "Net 60"
  deliveryDate: string;         // ISO date string (YYYY-MM-DD)
  deliveryDays: number;         // Days from today
  emphasis: 'price' | 'terms' | 'delivery' | 'value';  // What this message emphasizes
}

export type ScenarioSuggestions = Record<'HARD' | 'MEDIUM' | 'SOFT' | 'WALK_AWAY', StructuredSuggestion[]>;
```

**Frontend** (`src/types/chatbot.ts`):
```typescript
export interface StructuredSuggestion {
  message: string;
  price: number;
  paymentTerms: string;
  deliveryDate: string;
  deliveryDays: number;
  emphasis: 'price' | 'terms' | 'delivery' | 'value';
}

export interface OfferChip {
  label: string;
  value: string;
  type: 'price' | 'terms' | 'delivery';
}
```

### 1.2 Update Suggestion Cache Structure

**File**: `src/modules/chatbot/suggestionCache.ts`

```typescript
interface CacheEntry {
  suggestions: ScenarioSuggestions;  // Changed from Record<string, string[]>
  timestamp: number;
  source: 'llm' | 'fallback';
  dealId: string;
}
```

---

## Phase 2: Backend Changes

### 2.1 Update LLM Prompt for Suggestion Generation

**File**: `src/modules/chatbot/chatbot.service.ts` (lines 1672-1706)

**Changes**:
1. Add delivery date extraction from deal config
2. Include delivery requirements in LLM prompt
3. Request structured JSON output with all three terms
4. Specify emphasis variation requirement

**New LLM Prompt Structure**:
```
You are a procurement negotiation expert. Generate 4 counter-offer suggestions for each scenario.

NEGOTIATION CONFIG:
- Target Unit Price: $${target}
- Maximum Acceptable: $${max}
- Payment Terms Options: ${paymentOptions.join(', ')}
- Required Delivery Date: ${deliveryDate} (${deliveryDays} days from today)

For each scenario (HARD, MEDIUM, SOFT, WALK_AWAY), generate 4 suggestions with DIFFERENT emphasis:
1. Price-focused: Emphasize the competitive pricing
2. Terms-focused: Emphasize favorable payment terms
3. Delivery-focused: Emphasize meeting delivery requirements
4. Value-focused: Emphasize overall value proposition

Each suggestion must be a complete, human-like negotiation message that naturally incorporates:
- Unit price
- Payment terms
- Delivery commitment

Return JSON format:
{
  "HARD": [
    {
      "message": "Looking at our costs, the best we can do is $92 per unit with Net 30 terms. We can guarantee delivery within 14 days to meet your timeline.",
      "price": 92,
      "paymentTerms": "Net 30",
      "deliveryDate": "2026-02-04",
      "deliveryDays": 14,
      "emphasis": "price"
    },
    ...
  ],
  ...
}
```

### 2.2 Update Fallback Suggestion Generator

**File**: `src/modules/chatbot/chatbot.service.ts` (lines 1715-1741)

Create high-quality pre-written templates that match LLM quality:

```typescript
const generateFallbackSuggestions = (
  priceConfig: PriceConfig,
  paymentTerms: string[],
  deliveryDate: string,
  deliveryDays: number
): ScenarioSuggestions => {
  const idealTerms = paymentTerms[0] || 'Net 30';
  const midTerms = paymentTerms[1] || 'Net 45';
  const flexTerms = paymentTerms[2] || 'Net 60';

  return {
    HARD: [
      {
        message: `After careful analysis, our best offer is $${priceConfig.anchor.toFixed(2)} per unit with ${idealTerms} payment terms. We commit to delivery within ${deliveryDays} days.`,
        price: priceConfig.anchor,
        paymentTerms: idealTerms,
        deliveryDate,
        deliveryDays,
        emphasis: 'price'
      },
      {
        message: `We can work with ${idealTerms} payment terms at $${priceConfig.anchor.toFixed(2)} per unit, ensuring delivery by ${formatDate(deliveryDate)} as requested.`,
        price: priceConfig.anchor,
        paymentTerms: idealTerms,
        deliveryDate,
        deliveryDays,
        emphasis: 'terms'
      },
      {
        message: `To meet your ${deliveryDays}-day delivery requirement, we're offering $${priceConfig.anchor.toFixed(2)} per unit with ${idealTerms} terms.`,
        price: priceConfig.anchor,
        paymentTerms: idealTerms,
        deliveryDate,
        deliveryDays,
        emphasis: 'delivery'
      },
      {
        message: `Considering the complete package - competitive pricing at $${priceConfig.anchor.toFixed(2)}, ${idealTerms} terms, and guaranteed ${deliveryDays}-day delivery - this represents excellent value.`,
        price: priceConfig.anchor,
        paymentTerms: idealTerms,
        deliveryDate,
        deliveryDays,
        emphasis: 'value'
      }
    ],
    MEDIUM: [...], // Similar structure with target prices
    SOFT: [...],   // Similar structure with max acceptable prices
    WALK_AWAY: [...] // Polite decline messages
  };
};
```

### 2.3 Update API Response Format

**File**: `src/modules/chatbot/chatbot.controller.ts`

Update the `/suggestions` endpoint response:

```typescript
// Response format
{
  message: "Suggestions generated successfully",
  data: {
    suggestions: ScenarioSuggestions,
    source: 'llm' | 'fallback',
    deliveryConfig: {
      date: string,
      daysFromToday: number
    }
  }
}
```

### 2.4 Add Delivery Date Helper Functions

**File**: `src/modules/chatbot/chatbot.service.ts` (new section)

```typescript
function getDeliveryDate(config: ExtendedNegotiationConfig): { date: string; days: number } {
  const delivery = config.delivery;
  const targetDate = delivery?.preferredDate || delivery?.requiredDate;

  if (!targetDate) {
    // Fallback: 30 days from today
    const fallbackDate = new Date();
    fallbackDate.setDate(fallbackDate.getDate() + 30);
    return {
      date: fallbackDate.toISOString().split('T')[0],
      days: 30
    };
  }

  const deliveryDate = new Date(targetDate);
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  deliveryDate.setHours(0, 0, 0, 0);

  const diffTime = deliveryDate.getTime() - today.getTime();
  const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));

  return {
    date: targetDate,
    days: Math.max(0, diffDays)
  };
}

function formatDeliveryForDisplay(date: string, days: number): string {
  const dateObj = new Date(date);
  const formatted = dateObj.toLocaleDateString('en-US', {
    month: 'short',
    day: 'numeric'
  });
  return `${formatted} (${days} days)`;
}
```

---

## Phase 3: Frontend Changes

### 3.1 Update Scenario Generator

**File**: `src/utils/scenarioGenerator.ts`

Update to generate `StructuredSuggestion` objects:

```typescript
export function generateScenarioMessages(
  config: NegotiationConfig,
  delivery: { date: string; days: number }
): ScenarioSuggestions {
  // Generate structured suggestions with all three terms
  // Use emphasis variation for the 4 messages per scenario
}
```

### 3.2 Update Composer Component

**File**: `src/components/chatbot/chat/Composer.tsx`

**Changes**:

1. **Parse structured suggestions**:
```typescript
const [currentSuggestion, setCurrentSuggestion] = useState<StructuredSuggestion | null>(null);
```

2. **Generate offer chips from suggestion**:
```typescript
const generateChips = (suggestion: StructuredSuggestion): OfferChip[] => [
  {
    type: 'price',
    label: 'Price',
    value: `$${suggestion.price.toFixed(2)}`
  },
  {
    type: 'terms',
    label: 'Terms',
    value: suggestion.paymentTerms
  },
  {
    type: 'delivery',
    label: 'Delivery',
    value: formatDeliveryDisplay(suggestion.deliveryDate, suggestion.deliveryDays)
  }
];
```

3. **Update chip display section** (lines 192-203):
```tsx
{/* Offer Details Chips - Now shown for all modes */}
<div className="flex flex-wrap gap-2 mb-2">
  {currentChips.map((chip, idx) => (
    <span
      key={idx}
      className={`px-2 py-1 rounded-full text-xs font-medium ${
        chip.type === 'price' ? 'bg-green-100 text-green-800' :
        chip.type === 'terms' ? 'bg-blue-100 text-blue-800' :
        'bg-purple-100 text-purple-800'  // delivery
      }`}
    >
      {chip.label}: {chip.value}
    </span>
  ))}
</div>
```

4. **Update suggestion click handler**:
```typescript
const handleSuggestionClick = (suggestion: StructuredSuggestion) => {
  onSend(suggestion.message);  // Send the human-like message
};
```

### 3.3 Add Delivery Formatting Utility

**File**: `src/utils/formatters.ts` (new or existing)

```typescript
export function formatDeliveryDisplay(date: string, days: number): string {
  const dateObj = new Date(date);
  const formatted = dateObj.toLocaleDateString('en-US', {
    month: 'short',
    day: 'numeric'
  });
  return `${formatted} (${days} days)`;
}
```

### 3.4 Update Chatbot Service

**File**: `src/services/chatbot.service.ts`

Update `getSuggestedCounters` to handle new response format:

```typescript
async getSuggestedCounters(ctx: DealContext): Promise<ScenarioSuggestions> {
  const response = await authApi.post<ApiResponse<{
    suggestions: ScenarioSuggestions;
    source: string;
    deliveryConfig: { date: string; daysFromToday: number };
  }>>(
    buildDealUrl(ctx.rfqId, ctx.vendorId, ctx.dealId, '/suggestions')
  );
  return response.data.data.suggestions;
}
```

---

## Phase 4: Testing & Validation

### 4.1 Backend Tests

1. **LLM Response Parsing**: Verify structured JSON is correctly parsed
2. **Fallback Generation**: Verify all 4 emphasis types are generated
3. **Delivery Date Calculation**: Test various date scenarios
4. **Cache Behavior**: Verify new structure is cached correctly

### 4.2 Frontend Tests

1. **Chip Display**: Verify 3 chips appear (price, terms, delivery)
2. **Date Formatting**: Verify "Jan 15 (14 days)" format
3. **Click-to-Send**: Verify message is sent on chip click
4. **Scenario Switching**: Verify chips update when scenario changes

### 4.3 Integration Tests

1. **End-to-End Flow**: Generate suggestions → Display chips → Send message
2. **LLM Timeout**: Verify fallback suggestions appear correctly
3. **Missing Config**: Verify 30-day default works

---

## Phase 5: Files to Modify

### Backend Files

| File | Changes |
|------|---------|
| `src/modules/chatbot/engine/types.ts` | Add `StructuredSuggestion` interface |
| `src/modules/chatbot/suggestionCache.ts` | Update cache entry type |
| `src/modules/chatbot/chatbot.service.ts` | Update LLM prompt, fallback generator, add delivery helpers |
| `src/modules/chatbot/chatbot.controller.ts` | Update response format |

### Frontend Files

| File | Changes |
|------|---------|
| `src/types/chatbot.ts` | Add `StructuredSuggestion` and `OfferChip` types |
| `src/utils/scenarioGenerator.ts` | Update to generate structured suggestions |
| `src/utils/formatters.ts` | Add delivery date formatting utility |
| `src/components/chatbot/chat/Composer.tsx` | Update chip display and click handlers |
| `src/services/chatbot.service.ts` | Update API response handling |

---

## Implementation Order

1. **Phase 1**: Data structure changes (types and interfaces)
2. **Phase 2.4**: Add delivery date helper functions
3. **Phase 2.2**: Update fallback generator first (allows testing without LLM)
4. **Phase 3**: Frontend changes (can test with fallback suggestions)
5. **Phase 2.1**: Update LLM prompt
6. **Phase 2.3**: Update API response format
7. **Phase 4**: Testing and validation

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| LLM response format inconsistency | Robust JSON parsing with fallback |
| Breaking existing conversations | Full migration - clear cache on deploy |
| Date calculation edge cases | Comprehensive helper functions with defaults |
| UI layout issues with 3 chips | Responsive flex layout with wrapping |

---

## Estimated Changes

- **Backend**: ~300-400 lines of code changes
- **Frontend**: ~150-200 lines of code changes
- **New Files**: 0 (all changes to existing files)
- **Types/Interfaces**: ~50 lines

---

## Approval Checklist

Please confirm the following before implementation:

- [ ] Data structure changes are acceptable
- [ ] Delivery chip color (purple) is approved
- [ ] Message format examples are approved
- [ ] Fallback template quality is sufficient
- [ ] Implementation order is acceptable

---

## Questions for Final Approval

1. Is the purple color (`bg-purple-100 text-purple-800`) acceptable for delivery chips?
2. Should we add any additional emphasis types beyond (price/terms/delivery/value)?
3. Are the example fallback messages acceptable in tone and style?

