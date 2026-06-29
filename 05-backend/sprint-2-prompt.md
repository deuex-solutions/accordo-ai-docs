# Sprint 2 Implementation Prompt — Architectural Enrichments (Days 2–3)

## Context

You are working on the **Accordo-AI** backend — an AI-powered procurement negotiation platform. Sprint 1 (quick wins) is complete. Sprint 2 enriches the CONVERSATION mode pipeline with contextual signals that currently don't exist: vendor concern extraction, term-question detection, varied counter reasoning, and improved phrasing deduplication.

**Key architectural rule**: The hard boundary between the deterministic engine and the LLM lives in `buildNegotiationIntent()`. The LLM NEVER sees utility scores, weights, thresholds, target prices, or max prices. All changes in this sprint add signals to the `NegotiationIntent` or improve how the persona-renderer uses existing signals — they don't leak commercial data.

All changes are in:
```
Accordo-ai-backend/src/
```

**Sprint 1 changes already applied** (don't revert these):
- `build-negotiation-intent.ts`: has `normalizeDelivery()`, `humanizeDeliveryDate()`, `humanRoundPrice()`
- `validate-llm-output.ts`: has multi-currency `extractPrices()`, expanded `AI_TELL_PHRASES`
- `fallback-templates.ts`: cleaned AI-tell phrases at lines 114 and 276

---

## Fix 1: A5 — Improve Phrasing Fingerprint (from 3 → 5 words)

### Problem
The fingerprint in `phrasing-history.ts` uses only the first 3 words of a rendered message. This is too coarse — "Thank you for" matches completely different templates ("Thank you for your proposal", "Thank you for the time", "Thank you for coming back"). The dedup system triggers false collisions and doesn't actually prevent repetition of distinct-but-similarly-starting messages.

### Root Cause
Line 67 in `phrasing-history.ts`: `.slice(0, 3)` — only takes 3 words.

### File to Change

**File: `src/llm/phrasing-history.ts`**

Change the `buildFingerprint()` function (lines 61–69):

```typescript
/**
 * Build a fingerprint from a rendered message + action type.
 * First 5 meaningful words, lowercased, punctuation-stripped.
 * 5 words provides enough discriminative power to distinguish
 * "Thank you for your proposal" from "Thank you for coming back"
 * while still being coarse enough that minor rephrasing still de-dups.
 */
export function buildFingerprint(action: string, message: string): string {
  const words = (message || "")
    .toLowerCase()
    .replace(/[^\p{L}\p{N}\s']/gu, " ")
    .split(/\s+/)
    .filter(Boolean)
    .slice(0, 5);
  return `${action}|${words.join(":")}`;
}
```

Also increase `MAX_PHRASINGS_PER_DEAL` from 12 to 20 (line 23), since 5-word fingerprints are more specific and we need more storage to cover a full negotiation without premature eviction:

```typescript
const MAX_PHRASINGS_PER_DEAL = 20;
```

### Acceptance Criteria
- `buildFingerprint("COUNTER", "Thank you for your proposal")` → `"COUNTER|thank:you:for:your:proposal"`
- `buildFingerprint("COUNTER", "Thank you for coming back")` → `"COUNTER|thank:you:for:coming:back"` (different from above!)
- Old 3-word version would have produced the SAME fingerprint for both
- `MAX_PHRASINGS_PER_DEAL` is now 20
- All existing imports of `buildFingerprint` still work (signature unchanged)

---

## Fix 2: A4 — Wire Up `detectTermsRequest()` in Conversation Service

### Problem
`detectTermsRequest()` exists in `parse-offer.ts` (lines 539–586) and correctly detects vendor questions like "can you do Net 30?" or "what's your best price for Net 60?". But it is **never imported or called** in `conversation-service.ts`. Vendor questions about specific payment terms are silently ignored — the engine just counters without addressing the question.

### What It Should Do
When a vendor asks about a specific term (e.g., "Can you do Net 30?"), the system should:
1. Detect the requested term days
2. Pass the question to the `openQuestions` field in `NegotiationIntent`
3. The LLM then addresses the question naturally in its response

### Files to Change

**File: `src/modules/chatbot/convo/conversation-service.ts`**

**Step 1**: Add the import (near line 23 where `parseOfferRegex` is imported):

```typescript
import { parseOfferRegex, detectTermsRequest } from "../engine/parse-offer.js";
```

**Step 2**: After the tone detection block (around line 974, after `detectVendorStyle` call), add term-request detection:

```typescript
    // 10c. Detect vendor term requests (e.g. "can you do Net 30?")
    //      If vendor is asking about specific terms, add to openQuestions so
    //      the LLM addresses it naturally. Do NOT override the engine decision.
    const termsRequest = detectTermsRequest(vendorMessage);
    if (termsRequest) {
      logger.info("[ConversationService] Vendor term request detected", {
        dealId,
        requestedDays: termsRequest.requestedDays,
        matchedText: termsRequest.matchedText,
      });
    }
```

**Step 3**: In the `buildNegotiationIntent` call (around line 1056), modify the `openQuestions` field to include the detected terms request:

```typescript
    // Merge stored open questions with any fresh term request from this round
    const freshQuestions: Array<{ question: string; askedAtRound: number }> = [];
    if (termsRequest) {
      freshQuestions.push({
        question: `Vendor asked about ${termsRequest.matchedText} terms`,
        askedAtRound: deal.round + 1,
      });
    }
    const mergedOpenQuestions = [...openQuestions, ...freshQuestions];

    const negotiationIntent = buildNegotiationIntent({
      // ... all existing fields unchanged ...
      openQuestions: mergedOpenQuestions,  // ← was: openQuestions
    });
```

The full `buildNegotiationIntent` call remains the same except `openQuestions` now uses `mergedOpenQuestions`.

### Acceptance Criteria
- Vendor sends "Can you do Net 30?" → `detectTermsRequest` returns `{ requestedDays: 30, matchedText: "Net 30" }`
- The question appears in `negotiationIntent.openQuestions` as `{ question: "Vendor asked about Net 30 terms", askedAtRound: N }`
- The persona-renderer's `openQuestionsHint` logic (already exists) picks it up and tells the LLM to address it
- Regular offers like "$50,000 with Net 60" do NOT trigger this (detectTermsRequest filters them out)
- The engine decision (ACCEPT/COUNTER/etc.) is NOT overridden — we only ADD context for the LLM

---

## Fix 3: H1 — Expand COUNTER Template Pool (3 → 7 per tone)

### Problem
Each tone variant for COUNTER in `fallback-templates.ts` has only 3 templates. After 3 rounds, all fingerprints are exhausted and `pickVariantIndex()` falls through to random — producing visible repetition. COUNTER is the most frequent action (60%+ of all responses), so this pool needs to be deepest.

### File to Change

**File: `src/llm/fallback-templates.ts`**

Expand each COUNTER tone pool from 3 to 7 templates. The new templates must follow the same pattern — inject `${i.currencySymbol}${i.allowedPrice?.toLocaleString()}` plus optional terms/delivery. They must NOT contain any AI-tell phrases, em-dashes, or exclamation marks (the sanitizer catches these but templates should be clean at source).

**Replace the entire `COUNTER_TEMPLATES` object** (lines 79–132) with:

```typescript
// COUNTER templates — 7 per tone for sufficient variety across multi-round negotiations
const COUNTER_TEMPLATES: Record<VendorTone | "default", TemplatePool> = {
  formal: [
    (i) =>
      `Thank you for your proposal. After careful review, we would like to counter with a total price of ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, with ${i.allowedPaymentTerms} payment terms` : ""}${i.allowedDelivery ? `, and delivery ${i.allowedDelivery}` : ""}. We believe these terms represent a fair path forward.`,
    (i) =>
      `We appreciate the offer and wish to propose the following: ${i.currencySymbol}${i.allowedPrice?.toLocaleString()} total${i.allowedPaymentTerms ? ` on ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. We trust this aligns with both parties' objectives.`,
    (i) =>
      `Following our review, we propose ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? ` and delivery ${i.allowedDelivery}` : ""} as our counter. We remain open to discussion.`,
    (i) =>
      `Having considered your proposal, our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. We are confident this reflects market conditions fairly.`,
    (i) =>
      `We have given your offer careful consideration. Our position is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` on ${i.allowedPaymentTerms} terms` : ""}${i.allowedDelivery ? `, with delivery ${i.allowedDelivery}` : ""}. We look forward to your response.`,
    (i) =>
      `Your proposal is noted. Based on our requirements, we are countering at ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. We value this relationship and hope we can align.`,
    (i) =>
      `We acknowledge your offer with appreciation. Our counter stands at ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? ` and delivery ${i.allowedDelivery}` : ""}. We believe this represents workable terms for both sides.`,
  ],
  casual: [
    (i) =>
      `Thanks for coming back to us. Here's what we can work with: ${i.currencySymbol}${i.allowedPrice?.toLocaleString()} total${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Think we can make that work?`,
    (i) =>
      `Appreciate the offer, our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` on ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, and delivery ${i.allowedDelivery}` : ""}. Let us know what you think.`,
    (i) =>
      `Good to hear from you. We're looking at ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Does that work on your end?`,
    (i) =>
      `Thanks for that. On our side, we're at ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Hoping we can land somewhere close to this.`,
    (i) =>
      `Noted, thanks. We can do ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` on ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, ${i.allowedDelivery} delivery` : ""}. Let me know if that's in the right ballpark.`,
    (i) =>
      `Alright, here's where we are: ${i.currencySymbol}${i.allowedPrice?.toLocaleString()} total${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms} terms` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Think there's room to meet here?`,
    (i) =>
      `Got it. From our end, ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""} is where we need to be. Can we work with that?`,
  ],
  urgent: [
    (i) =>
      `Given our timeline, we need to move quickly. Our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Can we confirm this today?`,
    (i) =>
      `To keep things on track, here's our counter: ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` / ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? ` / delivery ${i.allowedDelivery}` : ""}. We'd appreciate a fast turnaround.`,
    (i) =>
      `We're working against a deadline. Our offer is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Please let us know as soon as possible.`,
    (i) =>
      `Time-sensitive on our end. We're at ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Need a response by end of day if possible.`,
    (i) =>
      `Hoping to wrap this up quickly: ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` on ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Our team needs an answer to proceed with scheduling.`,
    (i) =>
      `We're pressed for time, so being direct: ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, ${i.allowedDelivery} delivery` : ""}. Can you confirm?`,
    (i) =>
      `Need to move this along. Our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Hoping for a quick resolution.`,
  ],
  firm: [
    (i) =>
      `We have reviewed the offer and our counter stands at ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. This reflects our firm position given current constraints.`,
    (i) =>
      `Our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, ${i.allowedDelivery} delivery` : ""}. We have limited flexibility beyond this.`,
    (i) =>
      `After internal review, our position is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` / ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? ` / delivery ${i.allowedDelivery}` : ""}. We hope we can reach agreement on these terms.`,
    (i) =>
      `To be direct: ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""} is where we are. This is based on firm budget constraints.`,
    (i) =>
      `We're holding at ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` on ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. We've reviewed internally and this is our best position.`,
    (i) =>
      `Our counter remains ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. The budget on this one is tight and we don't have much room.`,
    (i) =>
      `${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. That's our position. We're constrained on this and can't go higher.`,
  ],
  friendly: [
    (i) =>
      `Really appreciate your proposal. We think there's a fair middle ground here, our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. What do you think?`,
    (i) =>
      `Thanks so much for the offer, we're making progress. Our counter: ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, ${i.allowedDelivery} delivery` : ""}. Hoping we can find the right fit.`,
    (i) =>
      `We value this partnership and want to find terms that work for everyone. Our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` on ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Happy to discuss further.`,
    (i) =>
      `This is going well. From our end, ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""} feels fair for both sides. Thoughts?`,
    (i) =>
      `Appreciate you working through this with us. We're at ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Think we're getting close.`,
    (i) =>
      `Thanks for the flexibility so far. Our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` on ${i.allowedPaymentTerms} terms` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. We're optimistic we can land this.`,
    (i) =>
      `Good progress. We can work with ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Looking forward to wrapping this up together.`,
  ],
  default: [
    (i) =>
      `Thank you for your offer. Our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()} total${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. We believe this is a fair step forward.`,
    (i) =>
      `We appreciate your proposal. After consideration, we're countering with ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` / ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? ` / ${i.allowedDelivery}` : ""}. Can we find common ground here?`,
    (i) =>
      `I appreciate your offer and want to keep this moving. Our counter: ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. Open to your thoughts.`,
    (i) =>
      `Thanks for the proposal, here's our counter: ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms} terms` : ""}${i.allowedDelivery ? `, ${i.allowedDelivery} delivery` : ""}. Let's see if we can land on this.`,
    (i) =>
      `We've reviewed your offer carefully. Our counter position is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` on ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? ` with delivery ${i.allowedDelivery}` : ""}. Looking forward to your response.`,
    (i) =>
      `Noted, thank you. We're at ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? `, ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. We think this gives both sides room to move.`,
    (i) =>
      `Thanks for sharing that. Our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. We believe these terms work for both parties.`,
  ],
};
```

### Template Writing Rules (enforce for all new templates)
- NO em-dashes (—) — use commas or "and"
- NO exclamation marks
- NO "we'd love to", "feel free to", "don't hesitate", "let us know your thoughts"
- NO emojis
- Price MUST use `${i.currencySymbol}${i.allowedPrice?.toLocaleString()}`
- Each template must start with a DIFFERENT first 5 words (to work with improved fingerprint from Fix 1)
- Templates should sound like procurement professionals in an actual chat — not cover letters

### Acceptance Criteria
- Each tone pool (formal, casual, urgent, firm, friendly, default) has exactly 7 templates
- No two templates in the same pool start with the same first 5 words
- `sanitizeText()` applied to all templates produces no changes (templates are already clean)
- `pickVariantIndex()` logic still works — no interface change needed

---

## Fix 4: A3 — Extract Real Vendor Concerns (Replace Empty Array)

### Problem
Line 1071 in `conversation-service.ts` passes `concerns: []` to `buildNegotiationIntent`. The concerns field is always empty. The LLM then fabricates concerns on its own — inventing things like "cash flow considerations" or "supply chain worries" that the vendor never mentioned. This is a hallucination vector.

### Solution
Build a lightweight, deterministic concern-extractor that reads the vendor message and extracts REAL concerns. Only pass concerns the vendor actually expressed. If none are detected, pass an empty array (the LLM should not invent them).

### Files to Change

**File: `src/modules/chatbot/engine/tone-detector.ts`** (add new exported function)

Add at the end of the file (before the final export block):

```typescript
// ─────────────────────────────────────────────
// Vendor Concern Extraction (deterministic, no LLM)
// ─────────────────────────────────────────────

/**
 * Extract concrete vendor concerns from their message.
 * ONLY returns concerns the vendor explicitly mentioned — never fabricates.
 * Returns an array of short, normalized concern labels safe to pass to the LLM.
 */
const CONCERN_PATTERNS: Array<{ pattern: RegExp; label: string }> = [
  // Timeline / urgency
  { pattern: /\b(timeline|deadline|time.?sensitive|urgent|rush|asap|time.?frame)\b/i, label: "timeline pressure" },
  { pattern: /\b(delay|delayed|lead.?time|shipping.?time|production.?time)\b/i, label: "delivery timeline" },
  // Budget / cash flow
  { pattern: /\b(budget|cash.?flow|liquidity|payment.?cycle|fiscal)\b/i, label: "budget constraints" },
  { pattern: /\b(margin|margins|thin.?margins?|tight.?margins?)\b/i, label: "margin pressure" },
  // Supply chain
  { pattern: /\b(supply.?chain|raw.?material|shortage|availability|back.?order)\b/i, label: "supply chain" },
  // Volume / commitment
  { pattern: /\b(volume|bulk|large.?order|quantity.?discount|long.?term)\b/i, label: "volume commitment" },
  // Quality / compliance
  { pattern: /\b(quality|compliance|certification|standard|spec|specification)\b/i, label: "quality requirements" },
  // Relationship / trust
  { pattern: /\b(relationship|partnership|long.?standing|repeat.?business|loyal)\b/i, label: "relationship value" },
  // Competition / alternatives
  { pattern: /\b(competitor|alternative|other.?vendor|other.?supplier|market.?rate|going.?rate)\b/i, label: "competitive alternatives" },
  // Risk
  { pattern: /\b(risk|warranty|guarantee|liability|insurance)\b/i, label: "risk concerns" },
];

export function extractVendorConcerns(vendorMessage: string): string[] {
  if (!vendorMessage || vendorMessage.trim().length < 10) return [];
  
  const concerns: string[] = [];
  for (const { pattern, label } of CONCERN_PATTERNS) {
    if (pattern.test(vendorMessage) && !concerns.includes(label)) {
      concerns.push(label);
    }
  }
  // Cap at 3 most relevant — too many concerns dilutes the response
  return concerns.slice(0, 3);
}
```

Add `extractVendorConcerns` to the exports at the bottom of `tone-detector.ts`.

**File: `src/modules/chatbot/convo/conversation-service.ts`**

**Step 1**: Update the import from tone-detector (around line 60-64):

```typescript
import {
  detectVendorTone,
  detectStrictFirmness,
  detectVendorStyle,
  extractVendorConcerns,
  type VendorStyle,
} from "../engine/tone-detector.js";
```

**Step 2**: After the vendor style detection (around line 981), add:

```typescript
    // 10d. Extract real vendor concerns (deterministic — never fabricated)
    const vendorConcerns = extractVendorConcerns(vendorMessage);
```

**Step 3**: In the `buildNegotiationIntent` call (around line 1071), change:

```typescript
// BEFORE:
concerns: [],

// AFTER:
concerns: vendorConcerns,
```

### Acceptance Criteria
- Vendor says "We need this delivered within the timeline, our supply chain is strained" → `concerns: ["timeline pressure", "supply chain"]`
- Vendor says "50000" (just a number) → `concerns: []` (empty — no fabrication)
- Vendor says "Can you consider our long-term relationship?" → `concerns: ["relationship value"]`
- Maximum 3 concerns returned
- The LLM prompt already says `Acknowledge these vendor concerns naturally: X, Y` — this now receives REAL concerns instead of nothing
- NO fabrication: if the vendor didn't say it, it doesn't appear

---

## Fix 5: A2 — Vary Counter Reasoning in LLM Instruction

### Problem
The persona-renderer's COUNTER instruction (line 204 of `persona-renderer.ts`) always says `"Provide a brief, natural business reason (budget constraints, project requirements, or similar)"`. This means the LLM repeats "budget constraints" in nearly every counter. Real procurement managers vary their reasoning: market rates, volume expectations, competitive offers, project scope, internal approvals, etc.

### Solution
Rotate through a pool of reasoning hints deterministically (based on round number) so the LLM uses different justification language each round.

### File to Change

**File: `src/llm/persona-renderer.ts`**

**Step 1**: Add a reasoning hint pool (before `buildInstruction`, around line 110):

```typescript
// ─────────────────────────────────────────────
// Counter reasoning variety (rotates by round)
// ─────────────────────────────────────────────

const COUNTER_REASONING_HINTS: string[] = [
  "budget constraints",
  "market benchmarks we're seeing",
  "the scope of this project",
  "our volume expectations",
  "comparable pricing from other vendors",
  "internal procurement guidelines",
  "the timeline and delivery requirements",
  "our quarterly spend targets",
];

function getCounterReasoningHint(roundNumber: number): string {
  const idx = (roundNumber - 1) % COUNTER_REASONING_HINTS.length;
  return COUNTER_REASONING_HINTS[idx];
}
```

**Step 2**: In `buildInstruction()`, modify the COUNTER case (around line 197-208). Replace the static reasoning instruction:

```typescript
// BEFORE (line ~204):
actionInstruction = `Counter the vendor's offer. The EXACT counter is: total price ${intent.currencySymbol}${intent.allowedPrice.toLocaleString()}${termsText}${deliveryText}. You MUST include this exact price with the ${intent.currencySymbol} symbol. Provide a brief, natural business reason (budget constraints, project requirements, or similar). Keep it conversational.`;

// AFTER:
const reasonHint = getCounterReasoningHint(intent.roundNumber ?? 1);
actionInstruction = `Counter the vendor's offer. The EXACT counter is: total price ${intent.currencySymbol}${intent.allowedPrice.toLocaleString()}${termsText}${deliveryText}. You MUST include this exact price with the ${intent.currencySymbol} symbol. Provide a brief, natural business reason (${reasonHint}). Keep it conversational.`;
```

### Acceptance Criteria
- Round 1 → reasoning hint is "budget constraints"
- Round 2 → "market benchmarks we're seeing"
- Round 3 → "the scope of this project"
- Round 8 → cycles back to "budget constraints" (wraps around)
- The COUNTER instruction is still structurally identical — only the parenthetical reasoning hint changes
- No change to the hard price inclusion rule ("You MUST include this exact price")
- Other actions (ACCEPT, WALK_AWAY, etc.) are completely unaffected

---

## Implementation Order

Execute in this exact order:

1. **Fix 1 (A5)** — Phrasing fingerprint improvement in `phrasing-history.ts`
2. **Fix 3 (H1)** — Expand COUNTER templates in `fallback-templates.ts` (benefits from improved fingerprint)
3. **Fix 4 (A3)** — Vendor concern extraction in `tone-detector.ts` + wire into `conversation-service.ts`
4. **Fix 2 (A4)** — Wire `detectTermsRequest()` in `conversation-service.ts` (same file as Fix 4)
5. **Fix 5 (A2)** — Counter reasoning variety in `persona-renderer.ts`

---

## Scope Boundaries — DO NOT TOUCH

- Do NOT modify `decide.ts` — engine logic stays unchanged
- Do NOT modify `build-negotiation-intent.ts` interface (NegotiationIntent) — it already has all needed fields (concerns, openQuestions, etc.)
- Do NOT modify `validate-llm-output.ts` — Sprint 1 already handled that
- Do NOT add external dependencies — all logic is pure TypeScript regex/string processing
- Do NOT modify the LLM system prompt (first 73 lines of persona-renderer.ts) — only the instruction builder
- Do NOT create a new LLM call for concern extraction — it MUST be deterministic (regex-based)
- The `buildNegotiationIntent()` function body in `build-negotiation-intent.ts` should NOT change — we're only changing what gets PASSED IN from `conversation-service.ts`

---

## Testing Guidance

```typescript
// Fix 1:
buildFingerprint("COUNTER", "Thank you for your proposal") 
  // → "COUNTER|thank:you:for:your:proposal"
buildFingerprint("COUNTER", "Thank you for coming back")
  // → "COUNTER|thank:you:for:coming:back"

// Fix 2:
detectTermsRequest("Can you do Net 30?")
  // → { requestedDays: 30, matchedText: "Net 30" }
detectTermsRequest("$50,000 with Net 60")
  // → null (offer statement, not a question)

// Fix 4:
extractVendorConcerns("Our timeline is tight and margins are thin")
  // → ["timeline pressure", "margin pressure"]
extractVendorConcerns("50000")
  // → []

// Fix 5:
getCounterReasoningHint(1)  // → "budget constraints"
getCounterReasoningHint(2)  // → "market benchmarks we're seeing"
getCounterReasoningHint(9)  // → "budget constraints" (wraps)
```
