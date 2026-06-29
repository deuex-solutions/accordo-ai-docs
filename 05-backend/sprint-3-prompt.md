# Sprint 3 Implementation Prompt — Behavioral Changes (Days 4–5)

## Context

You are working on the **Accordo-AI** backend — an AI-powered procurement negotiation platform. Sprints 1 (quick wins) and 2 (architectural enrichments) are complete. Sprint 3 adds behavioral intelligence: the system now recognizes vendor movement, auto-accepts near-agreement deals, diversifies the commercial position phrasing, and adds firmness granularity. These are "awareness" changes — the system starts acting on signals it already has but currently ignores.

All changes are in:
```
Accordo-ai-backend/src/
```

**Previously completed (don't revert):**
- Sprint 1: `normalizeDelivery`, `humanizeDeliveryDate`, `humanRoundPrice`, multi-currency `extractPrices`, expanded `AI_TELL_PHRASES`
- Sprint 2: 5-word fingerprints, 7-per-tone COUNTER templates, `extractVendorConcerns`, `detectTermsRequest` wiring, counter reasoning rotation

---

## Fix 1: M2 — Proximity-Accept (Auto-Accept Within 2% After Round 5)

### Problem
The engine only accepts when utility reaches the `accept_threshold` (default 0.7). But when the vendor is at $51,000 and our target is $50,000 (a 2% gap), after 5+ rounds of negotiation it's commercially unreasonable to keep countering for $1,000. A human PM would close the deal. The current system keeps countering indefinitely over trivial gaps, frustrating vendors and damaging the relationship.

### User Requirements (locked in)
- **Gap threshold**: 2% between vendor price and PM's max acceptable price
- **Minimum rounds**: After round 5 (so we don't accept too early)
- **Terms check**: Payment terms must also be acceptable (within vendor's stated terms)
- This is a STRATEGY decision — it lives in `conversation-service.ts`, NOT in the engine

### File to Change

**File: `src/modules/chatbot/convo/conversation-service.ts`**

Add the proximity-accept check AFTER the engine decision but BEFORE the existing firmness/escape-hatch overrides (around line 510, after the affirmative override block ends). This ensures it fires after the engine has its say but before the existing escape hatches:

```typescript
    // 7-pre-B. Proximity-accept: if the vendor's price is within 2% of our
    // max_acceptable after round 5+, close the deal automatically. A human PM
    // wouldn't keep countering over a trivial gap after 5 rounds of back-and-forth.
    // Strategy lives here (conversation-service), never in the engine or LLM.
    if (
      decision.action === "COUNTER" &&
      !isVendorAffirmative &&
      deal.round >= 5 &&
      vendorOffer.total_price != null &&
      maxAcceptablePrice != null &&
      maxAcceptablePrice > 0
    ) {
      const gap = Math.abs(vendorOffer.total_price - maxAcceptablePrice);
      const gapPercent = gap / maxAcceptablePrice;

      if (
        vendorOffer.total_price <= maxAcceptablePrice &&
        gapPercent <= 0.02
      ) {
        // Price is within 2% AND within our ceiling — accept
        // Also verify payment terms are acceptable (if vendor specified them)
        const termsAcceptable =
          vendorOffer.payment_terms_days == null || // no terms specified → accept
          vendorOffer.payment_terms_days >= 30;     // at least Net 30

        if (termsAcceptable) {
          logger.info(
            "[ConversationService] Proximity-accept: vendor within 2% of max_acceptable after round 5+",
            {
              dealId,
              vendorPrice: vendorOffer.total_price,
              maxAcceptablePrice,
              gapPercent: (gapPercent * 100).toFixed(2) + "%",
              round: deal.round + 1,
            },
          );
          decision = {
            action: "ACCEPT",
            utilityScore: decision.utilityScore,
            counterOffer: null,
            reasons: [
              ...decision.reasons,
              `Proximity-accept: vendor at ${vendorOffer.total_price} is within 2% of max_acceptable ${maxAcceptablePrice} after round ${deal.round + 1}.`,
            ],
          };
        }
      }
    }
```

**IMPORTANT placement**: This block should go AFTER the `isVendorAffirmative` check (around line 510) and BEFORE the firmness check (`detectStrictFirmness`, around line 516). The `maxAcceptablePrice` variable is resolved later (around line 989), so you need to resolve it EARLIER or use the same source. Check where `maxAcceptablePrice` is first computed — it comes from `config.parameters.total_price.max_acceptable` (line 989-992).

Since `maxAcceptablePrice` is used later (line 989), you need to extract it BEFORE this proximity check. Move the price boundary resolution earlier:

```typescript
    // Resolve price boundaries early — needed for proximity-accept and later for intent builder
    const storedConfigEarly = deal.negotiationConfigJson as any;
    const targetPriceEarly: number | undefined =
      storedConfigEarly?.parameters?.total_price?.target ??
      storedConfigEarly?.wizardConfig?.priceQuantity?.targetUnitPrice ??
      undefined;
    const maxAcceptablePriceEarly: number | undefined =
      storedConfigEarly?.parameters?.total_price?.max_acceptable ??
      storedConfigEarly?.wizardConfig?.priceQuantity?.maxAcceptablePrice ??
      undefined;
```

Place this extraction BEFORE the proximity-accept block, and use `maxAcceptablePriceEarly` in the check. Then later (line 984-992), the existing `targetPrice` / `maxAcceptablePrice` resolution can reuse these or remain as-is (they'll compute the same values).

**ALTERNATIVELY** (simpler): Just move lines 984–992 (the `targetPrice` / `maxAcceptablePrice` resolution) to just before this new block, and use the same variable names. Then the later lines 984-992 become redundant and can be removed.

### Acceptance Criteria
- Vendor at $49,500, max_acceptable = $50,000 (1% gap), round 6 → ACCEPT
- Vendor at $48,000, max_acceptable = $50,000 (4% gap), round 6 → COUNTER (gap > 2%)
- Vendor at $49,500, max_acceptable = $50,000, round 3 → COUNTER (too early, < round 5)
- Vendor at $51,000, max_acceptable = $50,000 → COUNTER (above ceiling, never proximity-accept)
- Vendor terms = Net 15 → COUNTER (terms not acceptable, < Net 30)
- Does NOT fire when `isVendorAffirmative` is already true
- The engine decision is overridden — `decision.action` becomes `"ACCEPT"`
- Logged at INFO level for audit trail

---

## Fix 2: H3 — Expand Commercial Position Pool

### Problem
`selectCommercialPosition()` in `build-negotiation-intent.ts` has only 4 phrases for COUNTER (high/medium/low firmness + urgent). These are injected into `NegotiationIntent.commercialPosition` and passed to the LLM as `"Position context: ..."`. The LLM sees the same phrase round after round, contributing to repetitive output. The same applies to ACCEPT (3 phrases) and WALK_AWAY (2 phrases).

### File to Change

**File: `src/negotiation/intent/build-negotiation-intent.ts`**

Replace the `COMMERCIAL_POSITIONS` object and the `selectCommercialPosition` function.

**New `COMMERCIAL_POSITIONS`** (replace lines 242–267 approximately):

```typescript
const COMMERCIAL_POSITIONS: Record<
  NegotiationAction,
  Record<string, string[]>
> = {
  ACCEPT: {
    default: [
      "We are pleased to accept this offer as it meets our procurement requirements.",
      "This offer aligns with what we need and we are ready to move forward.",
      "The terms work for us and we are confirming our acceptance.",
    ],
    friendly: [
      "This offer works well for us and we are happy to proceed.",
      "We're glad we could reach an agreement that works for both sides.",
    ],
    formal: [
      "We confirm formal acceptance of the terms as presented.",
      "We hereby accept the terms under the conditions discussed.",
    ],
  },
  COUNTER: {
    high_firmness: [
      "Our budget constraints require us to maintain firm terms for this procurement.",
      "We have limited room to move on this and need to hold our position.",
      "Our internal guidelines require us to stay within these parameters.",
    ],
    medium_firmness: [
      "We are working toward mutually beneficial terms that align with our requirements.",
      "We think there is a fair middle ground and are proposing terms we can both work with.",
      "We are looking for an arrangement that reflects the value of this engagement for both parties.",
    ],
    low_firmness: [
      "We remain flexible and are committed to finding an arrangement that works for both parties.",
      "We see room for collaboration here and are proposing terms in that spirit.",
      "We value this relationship and are approaching this with flexibility.",
    ],
    urgent: [
      "Given our project timeline, we need to reach an agreement on these terms promptly.",
      "We are on a tight schedule and need to finalize terms soon.",
    ],
  },
  WALK_AWAY: {
    default: [
      "After careful consideration, the current terms do not align with our procurement requirements.",
      "We have evaluated the offer thoroughly and are unable to proceed under these conditions.",
    ],
    firm: [
      "We have reached the limit of what is workable within our constraints.",
      "The gap between our positions is too wide for us to bridge at this time.",
    ],
  },
  ESCALATE: {
    default: [
      "This negotiation requires senior review before we can proceed further.",
      "We would like to bring in our procurement leadership to continue this discussion.",
    ],
  },
  MESO: {
    default: [
      "We have prepared several options that may work for both parties.",
      "We are presenting a few alternatives to find the best fit.",
    ],
  },
  ASK_CLARIFY: {
    default: [
      "We need a few more details to move this negotiation forward.",
      "Some information is missing before we can respond with a complete offer.",
    ],
  },
};
```

**Note**: The type changes from `Record<string, string>` to `Record<string, string[]>` — each key maps to an ARRAY of phrases.

**New `selectCommercialPosition`** (replaces the function around lines 270–295):

```typescript
/**
 * Select a commercial position phrase deterministically.
 * Rotates through available phrases based on round number to avoid repetition.
 * No LLM involved — purely a lookup.
 */
function selectCommercialPosition(
  action: NegotiationAction,
  firmness: number,
  tone: VendorTone,
  roundNumber?: number,
): string {
  const pool = COMMERCIAL_POSITIONS[action];
  const round = roundNumber ?? 1;
  
  let phrases: string[];

  if (action === "COUNTER") {
    if (firmness >= 0.75) phrases = pool["high_firmness"];
    else if (firmness >= 0.55) phrases = pool["medium_firmness"];
    else if (tone === "urgent") phrases = pool["urgent"];
    else phrases = pool["low_firmness"];
  } else if (action === "ACCEPT") {
    if (tone === "friendly") phrases = pool["friendly"];
    else if (tone === "formal") phrases = pool["formal"];
    else phrases = pool["default"];
  } else if (action === "WALK_AWAY") {
    if (tone === "firm") phrases = pool["firm"];
    else phrases = pool["default"];
  } else {
    phrases = pool["default"] ?? ["We are working toward a mutually beneficial agreement."];
  }

  // Rotate by round number for variety
  const idx = (round - 1) % phrases.length;
  return phrases[idx];
}
```

**Update the call site** in `buildNegotiationIntent()` (around line 367-370). The existing call:

```typescript
const commercialPosition = selectCommercialPosition(
  finalAction,
  firmness,
  tone,
);
```

becomes:

```typescript
const commercialPosition = selectCommercialPosition(
  finalAction,
  firmness,
  tone,
  input.roundNumber,
);
```

### Acceptance Criteria
- COUNTER at high firmness, round 1 → phrase[0], round 2 → phrase[1], round 3 → phrase[2], round 4 → wraps to phrase[0]
- ACCEPT friendly → rotates through 2 phrases
- WALK_AWAY default → rotates through 2 phrases
- The return type is still `string` — no interface change to `NegotiationIntent`
- No fallback-safe issues: every key has at least 2 entries

---

## Fix 3: M3 — Vendor Movement Signal

### Problem
When a vendor drops their price from $60,000 → $55,000, the PM's response should acknowledge that movement ("Appreciate you coming down" or "Good to see some movement"). Currently, the LLM has no idea the vendor moved — it only sees the current vendor message, not the price history. So it never acknowledges concessions, making the PM sound unaware and ungrateful.

### Solution
Compare the current vendor price against `conversationState.lastVendorOffer.total_price`. If the vendor moved toward our target, pass a `vendorMovement` signal in the `NegotiationIntent`. The persona-renderer uses this to tell the LLM to acknowledge the movement.

### Files to Change

**File: `src/negotiation/intent/build-negotiation-intent.ts`**

**Step 1**: Add `vendorMovement` to the `NegotiationIntent` interface (around line 216, before the closing `}`):

```typescript
  /**
   * Indicates the vendor moved their price toward our position since last round.
   * 'significant' (>5% drop), 'moderate' (2-5%), 'minor' (<2%), or undefined (no movement / first round).
   * Used by the persona-renderer to acknowledge concessions naturally.
   */
  vendorMovement?: "significant" | "moderate" | "minor";
```

**Step 2**: Add `vendorMovement` to the `BuildIntentInput` interface (around line 328):

```typescript
  /** Vendor movement signal: 'significant', 'moderate', 'minor', or undefined. */
  vendorMovement?: "significant" | "moderate" | "minor";
```

**Step 3**: In the main `buildNegotiationIntent()` function body (around line 385, where other humanization signals are forwarded):

```typescript
  if (input.vendorMovement) intent.vendorMovement = input.vendorMovement;
```

**File: `src/modules/chatbot/convo/conversation-service.ts`**

**Step 1**: After vendor offer parsing and merging (around line 484, after `mergeWithLastOffer`), compute vendor movement:

```typescript
    // 6b. Compute vendor price movement (for LLM acknowledgment signal)
    let vendorMovement: "significant" | "moderate" | "minor" | undefined;
    const previousVendorPrice = conversationState.lastVendorOffer?.total_price;
    if (
      previousVendorPrice != null &&
      previousVendorPrice > 0 &&
      vendorOffer.total_price != null &&
      vendorOffer.total_price < previousVendorPrice
    ) {
      const dropPercent =
        (previousVendorPrice - vendorOffer.total_price) / previousVendorPrice;
      if (dropPercent >= 0.05) vendorMovement = "significant";
      else if (dropPercent >= 0.02) vendorMovement = "moderate";
      else if (dropPercent > 0) vendorMovement = "minor";
    }
```

**Step 2**: Pass `vendorMovement` into the `buildNegotiationIntent` call (around line 1056, in the object literal):

```typescript
    const negotiationIntent = buildNegotiationIntent({
      // ... all existing fields ...
      vendorMovement,   // ← ADD THIS
    });
```

**File: `src/llm/persona-renderer.ts`**

In `buildInstruction()`, add a movement acknowledgment hint (around line 195, near the other contextual hints):

```typescript
  // Vendor movement acknowledgment.
  const movementHint = intent.vendorMovement
    ? intent.vendorMovement === "significant"
      ? "The vendor has made a significant price concession since last round. Briefly acknowledge this positively before stating your counter."
      : intent.vendorMovement === "moderate"
        ? "The vendor moved their price down moderately. A brief positive acknowledgment is appropriate."
        : "The vendor made a small price adjustment. A subtle nod to their flexibility is fine but not required."
    : "";
```

Then include `movementHint` in the return array (around line 264, alongside the other hints):

```typescript
  return [
    `Vendor's message: "${vendorMessage}"`,
    "",
    `Your action: ${actionInstruction}`,
    "",
    formalityHint,
    languageHint,
    greetingHint,
    hostilityHint,
    movementHint,       // ← ADD THIS
    orderingHint,
    smalltalkHint,
    openQuestionsHint,
    phrasingHint,
    firmnessInstruction,
    concernsText,
    `Position context: ${intent.commercialPosition}`,
    "",
    `${lengthHint} Single message, no bullet points (except for MESO options).`,
  ]
    .filter(Boolean)
    .join("\n");
```

### Acceptance Criteria
- Vendor drops from $60,000 → $55,000 (8.3%) → `vendorMovement = "significant"`
- Vendor drops from $55,000 → $54,000 (1.8%) → `vendorMovement = "minor"`
- Vendor drops from $55,000 → $53,500 (2.7%) → `vendorMovement = "moderate"`
- Vendor stays at $55,000 → `vendorMovement = undefined`
- Vendor increases price → `vendorMovement = undefined` (only downward movement counts)
- First round (no previous offer) → `vendorMovement = undefined`
- The LLM receives "The vendor has made a significant price concession..." in the instruction
- The `NegotiationIntent` interface has a new optional `vendorMovement` field

---

## Fix 4: H5 — Firmness Granularity (5 Levels Instead of 3)

### Problem
`mapUtilityToFirmness()` in `build-negotiation-intent.ts` has 4 output values (0.25, 0.55, 0.75, 0.90), but the persona-renderer's `firmnessInstruction` only distinguishes 3 levels: `>=0.75` ("firm"), `>=0.55` ("moderate"), else ("warm"). This coarse mapping means firmness 0.55 and 0.74 produce the same instruction even though they represent meaningfully different positions.

### Files to Change

**File: `src/negotiation/intent/build-negotiation-intent.ts`**

Refine `mapUtilityToFirmness()` (lines 254–259) to output 5 levels:

```typescript
/**
 * Map a utility score (0–1) to a firmness value (0–1).
 * Higher firmness = more assertive LLM tone.
 *
 * Utility ≥ 80%  → firmness 0.15  (near acceptance, very soft)
 * Utility 65–80% → firmness 0.35  (comfortable zone, warm)
 * Utility 50–65% → firmness 0.55  (negotiating zone, moderate)
 * Utility 35–50% → firmness 0.75  (escalation zone, firm)
 * Utility < 35%  → firmness 0.90  (walk-away zone, very firm)
 */
export function mapUtilityToFirmness(utilityScore: number): number {
  if (utilityScore >= 0.8) return 0.15;
  if (utilityScore >= 0.65) return 0.35;
  if (utilityScore >= 0.5) return 0.55;
  if (utilityScore >= 0.35) return 0.75;
  return 0.9;
}
```

**File: `src/llm/persona-renderer.ts`**

Update the `firmnessInstruction` block in `buildInstruction()` (around lines 165–170) to use 5 levels:

```typescript
  const firmnessInstruction =
    intent.firmness >= 0.85
      ? "Be very firm. This is close to a final position."
      : intent.firmness >= 0.7
        ? "Be firm and clear. Hold your position."
        : intent.firmness >= 0.5
          ? "Be moderate — polite but direct."
          : intent.firmness >= 0.3
            ? "Be warm and collaborative. Show flexibility."
            : "Be very warm and accommodating. We're close to agreement.";
```

### Acceptance Criteria
- Utility 0.85 → firmness 0.15 → "Be very warm and accommodating" (near accept)
- Utility 0.70 → firmness 0.35 → "Be warm and collaborative"
- Utility 0.55 → firmness 0.55 → "Be moderate — polite but direct"
- Utility 0.40 → firmness 0.75 → "Be firm and clear"
- Utility 0.25 → firmness 0.90 → "Be very firm"
- The mapping is still deterministic (pure function, no randomness)
- Existing tests that assert specific firmness values will need updating for the new thresholds

---

## Implementation Order

Execute in this exact order:

1. **Fix 4 (H5)** — Firmness granularity in `build-negotiation-intent.ts` + `persona-renderer.ts` (foundation, no deps)
2. **Fix 2 (H3)** — Commercial position expansion in `build-negotiation-intent.ts` (uses the new firmness thresholds)
3. **Fix 3 (M3)** — Vendor movement signal across 3 files (new NegotiationIntent field)
4. **Fix 1 (M2)** — Proximity-accept in `conversation-service.ts` (highest risk, do last)

---

## Scope Boundaries — DO NOT TOUCH

- Do NOT modify `decide.ts` — engine logic is untouched. Proximity-accept is a conversation-level strategy, not an engine change
- Do NOT modify `validate-llm-output.ts` or `fallback-templates.ts` — Sprint 1 and 2 handled those
- Do NOT modify `phrasing-history.ts` — Sprint 2 handled that
- Do NOT modify the LLM system prompt (lines 38–73 in persona-renderer.ts) — only the instruction builder
- Do NOT add external dependencies
- The `NegotiationIntent` interface gets ONE new optional field (`vendorMovement`) — that's the only interface change in this sprint

---

## Testing Guidance

```typescript
// Fix 1 (M2) — proximity-accept:
// Vendor at $49,500, max_acceptable = $50,000, round 6, terms OK
// → decision.action should become "ACCEPT"
// Vendor at $48,000, max_acceptable = $50,000, round 6
// → stays COUNTER (gap 4% > 2%)

// Fix 2 (H3) — commercial position rotation:
selectCommercialPosition("COUNTER", 0.80, "formal", 1)  // → high_firmness[0]
selectCommercialPosition("COUNTER", 0.80, "formal", 2)  // → high_firmness[1]
selectCommercialPosition("COUNTER", 0.80, "formal", 4)  // → wraps to high_firmness[0]

// Fix 3 (M3) — vendor movement:
// prev = $60,000, current = $55,000 → vendorMovement = "significant" (8.3% drop)
// prev = $55,000, current = $54,000 → vendorMovement = "minor" (1.8%)
// prev = $55,000, current = $55,000 → vendorMovement = undefined

// Fix 4 (H5) — firmness granularity:
mapUtilityToFirmness(0.85)  // → 0.15
mapUtilityToFirmness(0.70)  // → 0.35
mapUtilityToFirmness(0.55)  // → 0.55
mapUtilityToFirmness(0.40)  // → 0.75
mapUtilityToFirmness(0.25)  // → 0.90
```
