# Sprint 4 Fix — Remaining `concerns: []` in ASK_CLARIFY Path

## Problem

The verification audit found one remaining `concerns: []` at line ~409 of `src/modules/chatbot/convo/conversation-service.ts`. This is in the **ASK_CLARIFY early-return path** — the branch that fires when a vendor sends a price without payment terms (lines 373–477). It builds a minimal `buildNegotiationIntent` call with hardcoded empty concerns instead of extracting real concerns from the vendor's message.

While this path only fires for a specific edge case (price-without-terms), the vendor's message might still contain real concerns (e.g., "Our budget is tight, best we can do is $50,000" — has a price but no terms, AND mentions budget). Passing empty concerns here means the LLM fabricates or ignores them in its "please share your terms" response.

## File to Change

**File: `src/modules/chatbot/convo/conversation-service.ts`**

The `extractVendorConcerns` function is already imported in this file (Sprint 2 wired it). It just needs to be called in this early-return path too.

Find this block (around lines 402–412):

```typescript
      // Generate LLM-rendered "ask for terms" message
      const termsAskIntent = buildNegotiationIntent({
        action: "ASK_CLARIFY",
        utilityScore: 0,
        counterPrice: null,
        counterPaymentTerms: null,
        counterDelivery: null,
        concerns: [],
        tone: "formal",
        currencyCode: (config.currency as string) || "USD",
      });
```

Replace `concerns: []` with:

```typescript
        concerns: extractVendorConcerns(vendorMessage),
```

So the full block becomes:

```typescript
      // Generate LLM-rendered "ask for terms" message
      const termsAskIntent = buildNegotiationIntent({
        action: "ASK_CLARIFY",
        utilityScore: 0,
        counterPrice: null,
        counterPaymentTerms: null,
        counterDelivery: null,
        concerns: extractVendorConcerns(vendorMessage),
        tone: "formal",
        currencyCode: (config.currency as string) || "USD",
      });
```

## Verification

After the fix, run:

```bash
grep -n "concerns: \[\]" src/modules/chatbot/convo/conversation-service.ts
```

Should return **zero matches**. Every `concerns:` field in the file should now use `extractVendorConcerns(vendorMessage)` or the `vendorConcerns` variable.

## Scope

- ONE line changed in ONE file
- `extractVendorConcerns` is already imported — no new imports needed
- No other files touched
- Tests should still pass unchanged (this path's tests don't assert on concern content)
