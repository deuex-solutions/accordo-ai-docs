# Sprint 1 Implementation Prompt — Quick Wins (Day 1)

## Context

You are working on the **Accordo-AI** backend — an AI-powered procurement negotiation platform. The system has a CONVERSATION mode pipeline where vendor messages come in, a deterministic engine decides the action (ACCEPT/COUNTER/WALK_AWAY/ESCALATE/MESO/ASK_CLARIFY), and an LLM renders a human-sounding response. A validator checks the LLM output, and fallback templates fire if validation fails.

This sprint contains 5 quick-win fixes that address production bugs and formatting issues in the negotiation response pipeline. All changes are in:

```
Accordo-ai-backend/src/
```

---

## Fix 1: H4 — "by by" Double-Word Bug (RICE: 80)

### Problem
When `allowedDelivery` already starts with "by" (e.g., `"by March 2026"`), the fallback templates concatenate `"delivery by ${i.allowedDelivery}"` producing `"delivery by by March 2026"`. The `sanitizeText()` function in `validate-llm-output.ts` catches some duplicate prepositions via regex on line 256, but the templates should not produce them in the first place.

### Root Cause
`allowedDelivery` is passed through raw from `conversation-service.ts` → `build-negotiation-intent.ts` → templates/LLM without stripping leading prepositions.

### Files to Change

**File: `src/negotiation/intent/build-negotiation-intent.ts`**

Add a helper function before the main `buildNegotiationIntent` function (around line 285):

```typescript
/**
 * Strip leading prepositions from delivery strings to prevent "by by" or "on on"
 * when templates prepend their own preposition.
 * E.g. "by March 2026" → "March 2026", "on 15th Jan" → "15th Jan"
 */
function normalizeDelivery(delivery: string | null | undefined): string | undefined {
  if (!delivery) return undefined;
  // Strip leading "by", "on", "within", "before", "after" (case-insensitive)
  return delivery.replace(/^\s*(by|on|within|before|after)\s+/i, "").trim() || undefined;
}
```

Then in the main function body (around line 402-403), change:

```typescript
// BEFORE:
if (counterDelivery) {
  intent.allowedDelivery = counterDelivery;
}

// AFTER:
if (counterDelivery) {
  intent.allowedDelivery = normalizeDelivery(counterDelivery);
}
```

### Acceptance Criteria
- `"by March 2026"` input → `allowedDelivery` = `"March 2026"`
- `"within 2 weeks"` input → `allowedDelivery` = `"2 weeks"`
- `"15 days"` input (no preposition) → `allowedDelivery` = `"15 days"` (unchanged)
- `""` or `null` → `undefined`
- Templates that say `delivery by ${i.allowedDelivery}` now produce `"delivery by March 2026"` (not "delivery by by March 2026")

---

## Fix 2: B1 — Multi-Currency Price Validation (RICE: 48) ⚠️ SAFETY CRITICAL

### Problem
`extractPrices()` in `validate-llm-output.ts` only matches the `$` symbol. For deals in GBP (£), EUR (€), or INR (₹), the validator cannot detect prices at all. This means:
- COUNTER responses with wrong prices pass validation unchecked
- ACCEPT responses with hallucinated prices are not caught
- MESO rogue-price checks are bypassed

### Root Cause
All regex patterns in `extractPrices()` hardcode `\$` instead of matching all supported currency symbols.

### File to Change

**File: `src/llm/validate-llm-output.ts`**

Replace the entire `extractPrices()` function (lines 144–194) with:

```typescript
/**
 * Extract all numeric price values mentioned in text.
 * Handles: $98,000 | £98,000 | €98K | ₹98.5K | 98 thousand | $98000 | A$5,000
 * Supports all currency symbols defined in the system: $ £ € ₹ A$
 *
 * Returns an array of numeric values (in full base currency units).
 */
function extractPrices(text: string): number[] {
  const prices: number[] = [];

  // Currency symbol pattern — covers $, £, €, ₹, and A$ (Australian dollar)
  const SYM = `(?:\\$|£|€|₹|A\\$)`;

  // Match patterns like $98,000 | £98,000.00 | €98000 | ₹1,50,000 (Indian grouping)
  const basePattern = new RegExp(`${SYM}\\s*([\\d,]+(?:\\.\\d{1,2})?)`, "g");
  let match;
  while ((match = basePattern.exec(text)) !== null) {
    const value = parseFloat(match[1].replace(/,/g, ""));
    if (!isNaN(value) && value > 100) {
      prices.push(value);
    }
  }

  // Match patterns like $98K | £98.5k | ₹98.5K
  const kPattern = new RegExp(`${SYM}\\s*([\\d.]+)\\s*[kK]\\b`, "g");
  while ((match = kPattern.exec(text)) !== null) {
    const value = parseFloat(match[1]) * 1000;
    if (!isNaN(value) && value > 100) {
      prices.push(value);
    }
  }

  // Match patterns like $1.5M | £2M | €1.2M
  const mPattern = new RegExp(`${SYM}\\s*([\\d.]+)\\s*[mM]\\b`, "g");
  while ((match = mPattern.exec(text)) !== null) {
    const value = parseFloat(match[1]) * 1_000_000;
    if (!isNaN(value) && value > 100) {
      prices.push(value);
    }
  }

  // Match "98 thousand" or "1.5 million" (language patterns, currency-agnostic)
  const wordPattern = /([\d.]+)\s+thousand\b/gi;
  while ((match = wordPattern.exec(text)) !== null) {
    const value = parseFloat(match[1]) * 1000;
    if (!isNaN(value) && value > 100) {
      prices.push(value);
    }
  }
  const millionPattern = /([\d.]+)\s+million\b/gi;
  while ((match = millionPattern.exec(text)) !== null) {
    const value = parseFloat(match[1]) * 1_000_000;
    if (!isNaN(value) && value > 100) {
      prices.push(value);
    }
  }

  // Also match bare lakh/crore for INR contexts
  const lakhPattern = /([\d.]+)\s+lakh\b/gi;
  while ((match = lakhPattern.exec(text)) !== null) {
    const value = parseFloat(match[1]) * 100_000;
    if (!isNaN(value) && value > 100) {
      prices.push(value);
    }
  }
  const crorePattern = /([\d.]+)\s+crore\b/gi;
  while ((match = crorePattern.exec(text)) !== null) {
    const value = parseFloat(match[1]) * 10_000_000;
    if (!isNaN(value) && value > 100) {
      prices.push(value);
    }
  }

  // Deduplicate
  return [...new Set(prices)];
}
```

### Acceptance Criteria
- `"Our counter is £400,000"` → extracts `[400000]`
- `"We propose ₹31,50,000"` → extracts `[3150000]` (Indian grouping, commas stripped)
- `"The price is €98.5K"` → extracts `[98500]`
- `"A$5,000 total"` → extracts `[5000]`
- `"2.5 lakh"` → extracts `[250000]`
- `"1.2 crore"` → extracts `[12000000]`
- All existing `$`-based tests still pass unchanged
- COUNTER validation now catches wrong £/€/₹ prices that previously slipped through

---

## Fix 3: L1 — Human-Readable Delivery Dates (RICE: 40)

### Problem
Delivery dates reach the LLM and templates as raw ISO strings or database formats (e.g., `"2026-03-15"`, `"2026-03-15T00:00:00Z"`). A real procurement manager would write "March 15, 2026" or "mid-March 2026" — not an ISO date.

### Root Cause
No date formatting applied anywhere in the pipeline before the value reaches `allowedDelivery`.

### File to Change

**File: `src/negotiation/intent/build-negotiation-intent.ts`**

Add a date formatting helper near the `normalizeDelivery` function (from Fix 1):

```typescript
/**
 * Convert ISO-style dates to human-readable format for vendor-facing messages.
 * "2026-03-15" → "March 15, 2026"
 * "2026-03-15T00:00:00Z" → "March 15, 2026"
 * "Q2 2026" → "Q2 2026" (already readable, pass through)
 * "4-6 weeks" → "4-6 weeks" (not a date, pass through)
 */
function humanizeDeliveryDate(delivery: string): string {
  // Try to match ISO date patterns: YYYY-MM-DD or YYYY-MM-DDTHH:mm:ssZ
  const isoMatch = delivery.match(/^(\d{4})-(\d{2})-(\d{2})/);
  if (isoMatch) {
    const [, year, month, day] = isoMatch;
    const months = [
      "January", "February", "March", "April", "May", "June",
      "July", "August", "September", "October", "November", "December",
    ];
    const monthName = months[parseInt(month, 10) - 1];
    const dayNum = parseInt(day, 10);
    return `${monthName} ${dayNum}, ${year}`;
  }
  // Pass through anything that's already human-readable
  return delivery;
}
```

Then update `normalizeDelivery` to also humanize dates:

```typescript
function normalizeDelivery(delivery: string | null | undefined): string | undefined {
  if (!delivery) return undefined;
  // First: humanize any ISO dates
  let normalized = humanizeDeliveryDate(delivery.trim());
  // Then: strip leading prepositions to prevent "by by"
  normalized = normalized.replace(/^\s*(by|on|within|before|after)\s+/i, "").trim();
  return normalized || undefined;
}
```

### Acceptance Criteria
- `"2026-03-15"` → `"March 15, 2026"`
- `"2026-03-15T00:00:00Z"` → `"March 15, 2026"`
- `"by 2026-06-01"` → `"June 1, 2026"` (preposition stripped + date humanized)
- `"4-6 weeks"` → `"4-6 weeks"` (pass through)
- `"Q2 2026"` → `"Q2 2026"` (pass through)
- `"mid-March"` → `"mid-March"` (pass through)

---

## Fix 4: H2 — Human-Rounded Pricing (RICE: 38)

### Problem
The engine in `decide.ts` rounds counter-prices to 2 decimal places (`Math.round(counterPrice * 100) / 100`), producing prices like `£400,737.19` or `$98,412.67`. Real procurement managers never quote to the penny — they round to clean numbers ($98,500 or $400,000).

### Root Cause
Line 322 of `decide.ts` applies machine precision. No human-rounding step exists downstream.

### File to Change

**File: `src/negotiation/intent/build-negotiation-intent.ts`**

Add a human-rounding utility near the top of the file (after the currency symbol map):

```typescript
/**
 * Round a price to a clean, human-sounding number.
 * Procurement managers don't quote to the penny.
 *
 * Rules:
 * - < $1,000:      round to nearest $10       (e.g. $847.33 → $850)
 * - $1,000–$9,999: round to nearest $50       (e.g. $4,212 → $4,200)
 * - $10K–$99,999:  round to nearest $500      (e.g. $98,412 → $98,500)
 * - $100K–$999K:   round to nearest $1,000    (e.g. $400,737 → $401,000)
 * - $1M+:          round to nearest $5,000    (e.g. $2,503,200 → $2,505,000)
 *
 * Always rounds UP (ceil) to avoid accidentally going below target.
 */
export function humanRoundPrice(price: number): number {
  if (price <= 0) return price;

  let step: number;
  if (price < 1_000) step = 10;
  else if (price < 10_000) step = 50;
  else if (price < 100_000) step = 500;
  else if (price < 1_000_000) step = 1_000;
  else step = 5_000;

  return Math.ceil(price / step) * step;
}
```

Then apply it in the main function body where `allowedPrice` is set (around line 398):

```typescript
// BEFORE:
intent.allowedPrice = resolved;

// AFTER:
intent.allowedPrice = resolved != null ? humanRoundPrice(resolved) : undefined;
```

**IMPORTANT**: Make sure the `resolveAllowedPrice` bounds-clamping still works correctly. The flow is: engine produces raw price → `resolveAllowedPrice` clamps to [target, max] → `humanRoundPrice` rounds to clean number. Since `humanRoundPrice` rounds UP, verify it doesn't push above `maxAcceptablePrice`. Add a final clamp:

```typescript
// After human-rounding, ensure we haven't exceeded max
if (resolved != null) {
  let humanPrice = humanRoundPrice(resolved);
  // If rounding pushed above max, round DOWN to nearest step instead
  if (maxAcceptablePrice != null && humanPrice > maxAcceptablePrice) {
    humanPrice = maxAcceptablePrice; // Fall back to exact max
  }
  intent.allowedPrice = humanPrice;
}
```

Replace the simple `intent.allowedPrice = resolved;` with the block above.

### Acceptance Criteria
- `$98,412.67` → `$98,500`
- `£400,737.19` → `£401,000`
- `₹31,250` → `₹31,300` (nearest 500 since < 100K)  
  Wait — ₹31,250 is in the $10K–$99,999 range → nearest 500 → `₹31,500`
- `$4,212` → `$4,250` (nearest $50)
- `$847.33` → `$850` (nearest $10)
- `$2,503,200` → `$2,505,000`
- Never exceeds `maxAcceptablePrice` after rounding
- Export `humanRoundPrice` so it can be unit-tested

---

## Fix 5: M1 — AI-Tell Phrase Gap in Validator (RICE: 36)

### Problem
The `AI_TELL_PHRASES` regex array in `validate-llm-output.ts` catches `"we'd love to"` but misses the uncontracted form `"we would love to"`. Same for `"I'd love to"` / `"I would love to"`. The LLM sometimes outputs the full form, which slips through. Additionally, there are two AI-tell phrases in the fallback templates themselves (lines 114 and 276 of `fallback-templates.ts`) that should be cleaned.

### Files to Change

**File: `src/llm/validate-llm-output.ts`**

Replace the `AI_TELL_PHRASES` array (lines 123–132) with an expanded version:

```typescript
// AI-tell phrases — performative-helpful filler that reads as LLM output.
// Stripped silently; the surrounding sentence usually reads fine without them.
const AI_TELL_PHRASES: RegExp[] = [
  /\bwe('| woul)?d love to\b/gi,
  /\bI('| woul)?d love to\b/gi,
  /\bwe would love to\b/gi,
  /\bI would love to\b/gi,
  /\bthis better aligns with our needs\b/gi,
  /\blet us know your thoughts\b/gi,
  /\bfeel free to\b/gi,
  /\bI hope this helps\b/gi,
  /\bI'?m happy to discuss\b/gi,
  /\blooking forward to hearing\b/gi,
  /\bdon'?t hesitate to\b/gi,
  /\bwe appreciate your understanding\b/gi,
  /\bplease don'?t hesitate\b/gi,
  /\bwe'?re confident (that |this )?(will|can)\b/gi,
  /\bthank you for your patience\b/gi,
];
```

**Note on regex consolidation**: The first two patterns `we('| woul)?d love to` already cover the contracted form (`we'd`) AND the uncontracted (`we would`). The explicit `we would love to` and `I would love to` lines are kept as belts-and-suspenders since the `('| woul)?d` pattern is slightly tricky. Alternatively, you can simplify to just:

```typescript
/\bwe('d| would) love to\b/gi,
/\bI('d| would) love to\b/gi,
```

Pick whichever is clearer. The important thing is both contracted AND uncontracted forms are caught.

**File: `src/llm/fallback-templates.ts`**

In the `friendly` COUNTER templates (line 114), the phrase `"We'd love to meet somewhere in the middle"` will already be caught by `sanitizeText()` (which runs on fallback output at line 390). However, to be explicit and avoid the strip leaving awkward artifacts:

Line 114 — change the template from:
```typescript
(i) =>
  `Really appreciate your proposal! We'd love to meet somewhere in the middle, our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. What do you think?`,
```

To:
```typescript
(i) =>
  `Really appreciate your proposal. We think there's a fair middle ground here, our counter is ${i.currencySymbol}${i.allowedPrice?.toLocaleString()}${i.allowedPaymentTerms ? ` with ${i.allowedPaymentTerms}` : ""}${i.allowedDelivery ? `, delivery ${i.allowedDelivery}` : ""}. What do you think?`,
```

Also check line 276 (in WALK_AWAY or MESO friendly templates) — if it contains "We'd love to move forward", replace with `"We'd like to move forward"` or similar neutral phrasing.

### Acceptance Criteria
- `"We would love to work with you on this"` → stripped to `"work with you on this"` (or sentence cleaned)
- `"I'd love to find a solution"` → stripped
- `"We'd love to meet in the middle"` → stripped
- `"Don't hesitate to reach out"` → stripped
- `"Thank you for your patience"` → stripped
- All existing stripped phrases still work
- Fallback templates at lines 114 no longer contain AI-tell phrases pre-sanitization

---

## Implementation Order

Execute these in this exact order (each builds on the previous):

1. **Fix 1 (H4)** — Add `normalizeDelivery()` in `build-negotiation-intent.ts`
2. **Fix 3 (L1)** — Add `humanizeDeliveryDate()` in the same file (combines with Fix 1's function)
3. **Fix 4 (H2)** — Add `humanRoundPrice()` in the same file and apply to `allowedPrice`
4. **Fix 2 (B1)** — Replace `extractPrices()` in `validate-llm-output.ts`
5. **Fix 5 (M1)** — Expand `AI_TELL_PHRASES` in `validate-llm-output.ts` + fix template line 114

---

## Testing Guidance

After implementing all 5 fixes, verify with these quick checks:

```typescript
// Fix 1 + Fix 3 combined:
normalizeDelivery("by 2026-03-15")       // → "March 15, 2026"
normalizeDelivery("by March 2026")        // → "March 2026"
normalizeDelivery("4-6 weeks")            // → "4-6 weeks"
normalizeDelivery(null)                   // → undefined

// Fix 4:
humanRoundPrice(98412.67)                 // → 98500
humanRoundPrice(400737.19)                // → 401000
humanRoundPrice(4212)                     // → 4250
humanRoundPrice(847.33)                   // → 850

// Fix 2:
extractPrices("Our offer is £400,000")    // → [400000]
extractPrices("Counter: ₹31,50,000")      // → [3150000]
extractPrices("Price: €98.5K")            // → [98500]

// Fix 5:
sanitizeText("We would love to proceed at $5,000")  // → "proceed at $5,000"
sanitizeText("Don't hesitate to reach out")         // → "reach out"
```

---

## Scope Boundaries — DO NOT TOUCH

- Do NOT modify `decide.ts` line 322 (`Math.round(...)`). We keep the engine's precision; rounding happens downstream in `build-negotiation-intent.ts`.
- Do NOT modify `conversation-service.ts` in this sprint.
- Do NOT modify `persona-renderer.ts` in this sprint.
- Do NOT change the `NegotiationIntent` interface fields — only the values flowing through them.
- Do NOT add new npm dependencies. All changes use built-in JS/TS only.
- Do NOT modify any tests UNLESS they explicitly test the changed behavior (e.g., a test asserting `allowedPrice` equals the raw engine output to the penny would need updating).
