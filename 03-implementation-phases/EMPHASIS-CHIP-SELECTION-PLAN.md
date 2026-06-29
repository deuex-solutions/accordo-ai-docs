# Implementation Plan: Interactive Emphasis Chip Selection

## Overview
Enable users to click on emphasis chips (Price, Terms, Delivery) to filter/prioritize suggestions based on selected emphases. Supports multi-selection with weighted blending.

---

## Requirements Summary

| Aspect | Decision |
|--------|----------|
| **Chip Action** | Hybrid approach - show reordered cache immediately, fetch fresh in background |
| **Click Behavior** | Update options only - user clicks chip to filter, then clicks suggestion to send |
| **Visual Feedback** | Scale + glow effect on selected chips |
| **Multi-select** | Full multi-select - any combination of Price, Terms, Delivery |
| **Multi-emphasis Logic** | Weighted blend - balance all selected emphases equally |
| **Suggestion Count** | Same as now (3-4 per scenario) |
| **Persistence** | Keep selection across scenario changes (HARD→MEDIUM→SOFT) |
| **Default State** | Show all emphases when no chips selected |
| **Reset Option** | Click to toggle (clicking selected chip deselects it) |
| **Chip Position** | Keep current position in UI |
| **Loading State** | No indicator (hybrid shows cached, silently replaces) |

---

## Implementation Phases

### Phase 1: Backend API Updates

#### 1.1 Update Suggestions Endpoint
**File:** `Accordo-ai-backend/src/modules/chatbot/chatbot.service.ts`

- Add optional `emphasis` query parameter to suggestions endpoint
- Accept array of emphases: `emphasis=price,terms` or `emphasis=delivery`
- When emphasis provided:
  - Generate more suggestions internally (e.g., 8-10 per scenario)
  - Filter to those matching ANY selected emphasis
  - Score/rank by how well they match selected emphases
  - Return top 3-4 per scenario

#### 1.2 Update LLM Prompt for Emphasis Filtering
**File:** `Accordo-ai-backend/src/modules/chatbot/chatbot.service.ts`

- Modify prompt to request emphasis-specific suggestions when filter provided
- For weighted blend: "Generate suggestions that equally emphasize: {selected emphases}"
- Ensure each returned suggestion has `emphasis` field matching one of the selected

#### 1.3 Update Fallback Generator
**File:** `Accordo-ai-backend/src/modules/chatbot/chatbot.service.ts`

- Update template-based fallback to filter by emphasis
- Create templates for each emphasis combination

#### 1.4 Cache Strategy
**File:** `Accordo-ai-backend/src/modules/chatbot/chatbot.service.ts`

- Cache key includes emphasis selection: `suggestions:${dealId}:${emphases.sort().join(',')}`
- Allow different cached results for different emphasis combinations

---

### Phase 2: Frontend State Management

#### 2.1 Add Emphasis Selection State
**File:** `Accordo-ai-frontend/src/components/chatbot/chat/Composer.tsx`

```typescript
// New state for selected emphases
const [selectedEmphases, setSelectedEmphases] = useState<Set<SuggestionEmphasis>>(new Set());

// Toggle function for chips
const toggleEmphasis = (emphasis: SuggestionEmphasis) => {
  setSelectedEmphases(prev => {
    const next = new Set(prev);
    if (next.has(emphasis)) {
      next.delete(emphasis);
    } else {
      next.add(emphasis);
    }
    return next;
  });
};
```

#### 2.2 Update Service Call
**File:** `Accordo-ai-frontend/src/services/chatbot.service.ts`

- Add `emphasis` parameter to `getSuggestedCounters`
- Pass selected emphases as comma-separated query param

---

### Phase 3: Frontend UI Updates

#### 3.1 Make Chips Clickable
**File:** `Accordo-ai-frontend/src/components/chatbot/chat/Composer.tsx`

- Add `onClick` handler to each chip (Price, Terms, Delivery)
- Chips become interactive filter toggles
- Clicking toggles selection state

#### 3.2 Visual Feedback (Scale + Glow)
**File:** `Accordo-ai-frontend/src/components/chatbot/chat/Composer.tsx`

```css
/* Selected chip styles */
.chip-selected {
  transform: scale(1.05);
  box-shadow: 0 0 8px rgba(147, 51, 234, 0.5);
  border-color: #9333ea;
}
```

#### 3.3 Hybrid Filtering Logic
**File:** `Accordo-ai-frontend/src/components/chatbot/chat/Composer.tsx`

```typescript
// On emphasis change:
// 1. Immediately filter/reorder cached suggestions
const filteredSuggestions = filterByEmphasis(cachedSuggestions, selectedEmphases);
setDisplayedSuggestions(filteredSuggestions);

// 2. Fetch fresh suggestions in background
fetchSuggestionsWithEmphasis(selectedEmphases).then(fresh => {
  setDisplayedSuggestions(fresh);
  updateCache(fresh);
});
```

#### 3.4 Persistence Across Scenarios
**File:** `Accordo-ai-frontend/src/components/chatbot/chat/Composer.tsx`

- `selectedEmphases` state persists when user changes scenario
- When scenario changes, re-apply emphasis filter to new scenario's suggestions

---

### Phase 4: Integration & Testing

#### 4.1 End-to-End Flow
1. User sees suggestions with Price/Terms/Delivery chips (default: all emphases shown)
2. User clicks "Price" chip → chip scales up with glow
3. Cached suggestions immediately reorder (price-focused first)
4. Background API call fetches price-focused suggestions
5. When ready, suggestions silently update
6. User clicks "Delivery" chip → both Price + Delivery selected
7. Suggestions show weighted blend of price AND delivery focus
8. User clicks "Price" again → deselects, only Delivery remains
9. User changes scenario (HARD→MEDIUM) → emphasis selection preserved

#### 4.2 Test Cases
- [ ] Single emphasis selection works
- [ ] Multi-emphasis selection works
- [ ] Toggle off works
- [ ] Scenario change preserves selection
- [ ] Hybrid loading works (cache then fresh)
- [ ] Scale + glow visual feedback
- [ ] Default state shows all emphases

---

## Files to Modify

### Backend
1. `src/modules/chatbot/chatbot.service.ts` - Add emphasis filtering logic
2. `src/modules/chatbot/chatbot.controller.ts` - Accept emphasis query param
3. `src/modules/chatbot/chatbot.types.ts` - Update types if needed

### Frontend
1. `src/components/chatbot/chat/Composer.tsx` - Main UI changes
2. `src/services/chatbot.service.ts` - Add emphasis param to API call
3. `src/types/chatbot.ts` - Ensure types support emphasis array

---

## API Contract

### Request
```
POST /api/chatbot/requisitions/:rfqId/vendors/:vendorId/deals/:dealId/suggestions
Query: ?emphasis=price,delivery (optional, comma-separated)
```

### Response (unchanged structure)
```json
{
  "data": {
    "HARD": [
      {
        "message": "...",
        "price": 374.00,
        "paymentTerms": "Net 30",
        "deliveryDate": "2026-01-28T00:00:00.000Z",
        "deliveryDays": 7,
        "emphasis": "price"
      }
    ],
    "MEDIUM": [...],
    "SOFT": [...],
    "WALK_AWAY": [...]
  }
}
```

---

## Estimated Complexity

| Phase | Complexity | Key Challenge |
|-------|------------|---------------|
| Phase 1 | Medium | LLM prompt engineering for weighted blend |
| Phase 2 | Low | Simple state management |
| Phase 3 | Medium | Hybrid loading + visual effects |
| Phase 4 | Low | Standard testing |

---

## Approval Checklist

- [ ] Backend API accepts emphasis filter parameter
- [ ] LLM generates weighted blend suggestions
- [ ] Fallback templates support emphasis filtering
- [ ] Chips are clickable with toggle behavior
- [ ] Scale + glow visual feedback on selected chips
- [ ] Multi-select works with weighted results
- [ ] Selection persists across scenario changes
- [ ] Hybrid loading (cache first, then fresh)
- [ ] No loading indicator (silent background fetch)
