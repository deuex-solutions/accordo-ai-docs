# Plan: Remove Suggestions Feature Completely

## Overview

Remove the HARD/MEDIUM/SOFT suggestion system from both frontend and backend, leaving only a clean chat input with send button.

---

## Files to Modify/Delete

### FRONTEND (Accordo-ai-frontend)

#### DELETE Files
| File | Reason |
|------|--------|
| `src/utils/scenarioGenerator.ts` | Contains all scenario generation logic |

#### MODIFY Files

| File | Changes |
|------|---------|
| `src/components/chatbot/chat/Composer.tsx` | Remove scenario buttons, emphasis chips, quick message chips. Keep only text input + send button |
| `src/services/chatbot.service.ts` | Remove `getSuggestedCounters()` function (lines 495-505) |
| `src/services/vendorChat.service.ts` | Remove `getSuggestions()` function and vendor scenario methods |
| `src/types/chatbot.ts` | Remove `StructuredSuggestion`, `ScenarioSuggestions`, `SuggestionEmphasis`, `ScenarioType` types |

---

### BACKEND (Accordo-ai-backend)

#### DELETE Files
| File | Reason |
|------|--------|
| `src/modules/chatbot/suggestionCache.ts` | Suggestion caching system |

#### MODIFY Files

| File | Changes |
|------|---------|
| `src/modules/chatbot/chatbot.service.ts` | Remove `generateScenarioSuggestionsService()` function (lines 2855-3300+) |
| `src/modules/chatbot/chatbot.service.ts` | Remove `precomputeSuggestionsBackground()` calls |
| `src/modules/chatbot/chatbot.controller.ts` | Remove suggestions endpoint handler |
| `src/modules/chatbot/chatbot.routes.ts` | Remove `/suggestions` route |
| `src/modules/vendor-chat/vendor-chat.service.ts` | Remove vendor suggestion functions |
| `src/modules/vendor-chat/vendor-chat.controller.ts` | Remove vendor suggestions endpoint |
| `src/modules/vendor-chat/vendor-chat.routes.ts` | Remove vendor suggestions route |
| `src/modules/chatbot/engine/types.ts` | Remove suggestion-related types if not used elsewhere |

---

## Implementation Steps

### Phase 1: Frontend Cleanup

1. **Simplify Composer.tsx**
   - Remove all state related to suggestions (scenarios, selectedScenario, structuredSuggestions, etc.)
   - Remove useEffect hooks for fetching/generating suggestions
   - Remove scenario buttons JSX (lines 362-378)
   - Remove emphasis chips JSX (lines 381-409)
   - Remove quick message chips JSX (lines 425-463)
   - Keep only: text input, send button, basic styling

2. **Remove scenarioGenerator.ts**
   - Delete the entire file

3. **Clean chatbot.service.ts**
   - Remove `getSuggestedCounters()` function

4. **Clean vendorChat.service.ts**
   - Remove `getSuggestions()` and related vendor scenario functions

5. **Clean chatbot.ts types**
   - Remove unused suggestion types

### Phase 2: Backend Cleanup

1. **Remove suggestionCache.ts**
   - Delete the entire file

2. **Clean chatbot.service.ts**
   - Remove `generateScenarioSuggestionsService()` function
   - Remove all `precomputeSuggestionsBackground()` calls
   - Remove imports from suggestionCache.ts

3. **Clean chatbot.controller.ts**
   - Remove suggestions endpoint handler

4. **Clean chatbot.routes.ts**
   - Remove `/suggestions` route definition

5. **Clean vendor-chat module**
   - Remove vendor suggestion endpoint and service functions

6. **Clean engine/types.ts**
   - Remove `ScenarioSuggestions`, `StructuredSuggestion` types if only used for suggestions

---

## Final UI State

After removal, the Composer component will be:

```
┌─────────────────────────────────────────────────────────────────┐
│  [                    Type a message...                ] [Send] │
└─────────────────────────────────────────────────────────────────┘
```

- Clean, minimal text input
- Send button
- No suggestion chips
- No scenario buttons
- No emphasis filters

---

## Verification Checklist

- [ ] Composer renders with only text input and send button
- [ ] User can type and send messages manually
- [ ] No console errors related to missing suggestion functions
- [ ] Backend `/suggestions` endpoint returns 404
- [ ] Vendor portal chat works without suggestions
- [ ] TypeScript compilation passes (no unused type errors)
- [ ] No dead code or imports remaining

---

## Rollback Plan

Since we're deleting code completely, rollback would require:
- Git revert of the commits
- No feature flag or toggle available

---

## Estimated Changes

| Area | Files Modified | Files Deleted | Lines Removed |
|------|---------------|---------------|---------------|
| Frontend | 4 | 1 | ~800 |
| Backend | 6 | 1 | ~600 |
| **Total** | **10** | **2** | **~1400** |
