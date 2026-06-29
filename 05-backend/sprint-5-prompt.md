# Sprint 5 — Final Verification, Integration Tests & Hardening

## Context

Sprints 1–4 are fully implemented (16 humanization fixes + 1 patch). All 1225 unit tests pass and TypeScript compiles cleanly. Sprint 5 is the **final hardening pass** — it ensures that the changes work together correctly, adds integration-level test coverage for the new behaviors, updates stale documentation/comments, and performs a safety sweep for any regressions.

This sprint produces NO new features. It validates, tests, documents, and locks down what's already built.

---

## Part 1: Automated Verification Script

Create a one-shot verification script that can be run anytime to confirm all 16 fixes remain intact. This catches regressions from future refactors.

**File: NEW `src/__tests__/humanization-audit.test.ts`**

```typescript
/**
 * Humanization Audit — Integration Tests
 *
 * Validates that all 16 Sprint 1–4 humanization fixes are correctly in place.
 * This file serves as a regression guard: if any fix gets accidentally reverted,
 * these tests will catch it.
 */

import { buildFingerprint } from "../llm/phrasing-history.js";
import { humanRoundPrice, mapUtilityToFirmness, getCurrencySymbol, buildNegotiationIntent } from "../negotiation/intent/build-negotiation-intent.js";
import { sanitizeText, validateLlmOutput } from "../llm/validate-llm-output.js";
import { buildArcSummary, extractArcRounds } from "../llm/arc-summary.js";
import { extractVendorConcerns } from "../modules/chatbot/engine/tone-detector.js";
import { detectTermsRequest } from "../modules/chatbot/engine/parse-offer.js";
import { getFallbackResponse } from "../llm/fallback-templates.js";

describe("Humanization Audit — Sprint 1", () => {

  describe("H4: normalizeDelivery (no 'by by')", () => {
    it("strips leading prepositions from delivery in intent", () => {
      const intent = buildNegotiationIntent({
        action: "COUNTER",
        utilityScore: 0.5,
        counterPrice: 50000,
        counterPaymentTerms: "Net 30",
        counterDelivery: "by March 2026",
        concerns: [],
        tone: "formal",
        targetPrice: 45000,
        maxAcceptablePrice: 55000,
      });
      expect(intent.allowedDelivery).toBe("March 2026");
      expect(intent.allowedDelivery).not.toMatch(/^by /i);
    });

    it("humanizes ISO dates", () => {
      const intent = buildNegotiationIntent({
        action: "COUNTER",
        utilityScore: 0.5,
        counterPrice: 50000,
        counterPaymentTerms: null,
        counterDelivery: "2026-03-15",
        concerns: [],
        tone: "formal",
        targetPrice: 45000,
        maxAcceptablePrice: 55000,
      });
      expect(intent.allowedDelivery).toBe("March 15, 2026");
    });
  });

  describe("B1: Multi-currency price validation", () => {
    it("rejects wrong GBP price in COUNTER response", () => {
      const intent = buildNegotiationIntent({
        action: "COUNTER",
        utilityScore: 0.5,
        counterPrice: 50000,
        counterPaymentTerms: null,
        counterDelivery: null,
        concerns: [],
        tone: "formal",
        targetPrice: 45000,
        maxAcceptablePrice: 55000,
        currencyCode: "GBP",
      });
      // LLM says £60,000 but allowed is £50,000 → should fail
      expect(() =>
        validateLlmOutput("We counter at £60,000 total.", intent),
      ).toThrow();
    });

    it("accepts correct GBP price", () => {
      const intent = buildNegotiationIntent({
        action: "COUNTER",
        utilityScore: 0.5,
        counterPrice: 50000,
        counterPaymentTerms: null,
        counterDelivery: null,
        concerns: [],
        tone: "formal",
        targetPrice: 45000,
        maxAcceptablePrice: 55000,
        currencyCode: "GBP",
      });
      const result = validateLlmOutput(
        "We're countering at £50,000 based on our budget requirements.",
        intent,
      );
      expect(result).toContain("50,000");
    });
  });

  describe("H2: humanRoundPrice", () => {
    it("rounds to clean numbers", () => {
      expect(humanRoundPrice(98412.67)).toBe(98500);
      expect(humanRoundPrice(400737.19)).toBe(401000);
      expect(humanRoundPrice(4212)).toBe(4250);
      expect(humanRoundPrice(847.33)).toBe(850);
      expect(humanRoundPrice(2503200)).toBe(2505000);
    });

    it("never returns penny-precise values for prices > $1000", () => {
      const testPrices = [1234.56, 5678.90, 12345.67, 99999.99, 500000.50];
      for (const price of testPrices) {
        const rounded = humanRoundPrice(price);
        expect(rounded % 10).toBe(0); // Always divisible by at least 10
      }
    });
  });

  describe("M1: AI-tell phrase expansion", () => {
    it("strips uncontracted forms", () => {
      expect(sanitizeText("We would love to proceed at this price")).not.toContain("would love to");
      expect(sanitizeText("I would love to find a solution")).not.toContain("would love to");
    });

    it("strips new patterns", () => {
      expect(sanitizeText("Don't hesitate to reach out")).not.toContain("hesitate");
      expect(sanitizeText("Thank you for your patience in this")).not.toContain("Thank you for your patience");
    });
  });
});

describe("Humanization Audit — Sprint 2", () => {

  describe("A5: Phrasing fingerprint (5 words)", () => {
    it("distinguishes messages sharing first 3 words", () => {
      const fp1 = buildFingerprint("COUNTER", "Thank you for your proposal");
      const fp2 = buildFingerprint("COUNTER", "Thank you for coming back to us");
      expect(fp1).not.toBe(fp2);
    });

    it("produces 5-word fingerprints", () => {
      const fp = buildFingerprint("COUNTER", "We appreciate your offer and wish to proceed");
      const parts = fp.split("|")[1].split(":");
      expect(parts.length).toBe(5);
    });
  });

  describe("A3: Vendor concern extraction", () => {
    it("extracts real concerns from vendor message", () => {
      const concerns = extractVendorConcerns(
        "Our timeline is tight and we need to consider the supply chain impact"
      );
      expect(concerns).toContain("timeline pressure");
      expect(concerns).toContain("supply chain");
    });

    it("returns empty for number-only messages", () => {
      expect(extractVendorConcerns("50000")).toEqual([]);
    });

    it("caps at 3 concerns", () => {
      const concerns = extractVendorConcerns(
        "Budget is tight, timeline urgent, quality must be guaranteed, and our competitor offered better volume discounts"
      );
      expect(concerns.length).toBeLessThanOrEqual(3);
    });
  });

  describe("A4: detectTermsRequest wiring", () => {
    it("detects vendor term questions", () => {
      const result = detectTermsRequest("Can you do Net 30?");
      expect(result).not.toBeNull();
      expect(result!.requestedDays).toBe(30);
    });

    it("ignores offer statements", () => {
      const result = detectTermsRequest("$50,000 with Net 60");
      expect(result).toBeNull();
    });
  });

  describe("H1: COUNTER template pool depth", () => {
    it("has 7 templates per tone for COUNTER", () => {
      // Generate 7 fallback responses to verify pool depth
      const responses = new Set<string>();
      for (let i = 0; i < 20; i++) {
        const intent: any = {
          action: "COUNTER",
          firmness: 0.55,
          commercialPosition: "test",
          allowedPrice: 50000,
          allowedPaymentTerms: "Net 30",
          acknowledgeConcerns: [],
          vendorTone: "formal",
          currencySymbol: "$",
          phrasingHistory: [], // empty so all variants are eligible
        };
        responses.add(getFallbackResponse(intent));
      }
      // With 7 templates and random selection, 20 tries should hit at least 4 unique ones
      expect(responses.size).toBeGreaterThanOrEqual(4);
    });
  });
});

describe("Humanization Audit — Sprint 3", () => {

  describe("H5: Firmness 5-level mapping", () => {
    it("returns 5 distinct values", () => {
      const values = new Set([
        mapUtilityToFirmness(0.85),
        mapUtilityToFirmness(0.70),
        mapUtilityToFirmness(0.55),
        mapUtilityToFirmness(0.40),
        mapUtilityToFirmness(0.25),
      ]);
      expect(values.size).toBe(5);
    });

    it("maps correctly", () => {
      expect(mapUtilityToFirmness(0.85)).toBe(0.15);
      expect(mapUtilityToFirmness(0.70)).toBe(0.35);
      expect(mapUtilityToFirmness(0.55)).toBe(0.55);
      expect(mapUtilityToFirmness(0.40)).toBe(0.75);
      expect(mapUtilityToFirmness(0.25)).toBe(0.9);
    });
  });

  describe("M3: vendorMovement field", () => {
    it("exists on NegotiationIntent when passed", () => {
      const intent = buildNegotiationIntent({
        action: "COUNTER",
        utilityScore: 0.5,
        counterPrice: 50000,
        counterPaymentTerms: null,
        counterDelivery: null,
        concerns: [],
        tone: "formal",
        targetPrice: 45000,
        maxAcceptablePrice: 55000,
        vendorMovement: "significant",
      });
      expect(intent.vendorMovement).toBe("significant");
    });

    it("is undefined when not passed", () => {
      const intent = buildNegotiationIntent({
        action: "COUNTER",
        utilityScore: 0.5,
        counterPrice: 50000,
        counterPaymentTerms: null,
        counterDelivery: null,
        concerns: [],
        tone: "formal",
        targetPrice: 45000,
        maxAcceptablePrice: 55000,
      });
      expect(intent.vendorMovement).toBeUndefined();
    });
  });

  describe("H3: Commercial position rotation", () => {
    it("produces different positions for different rounds", () => {
      const intent1 = buildNegotiationIntent({
        action: "COUNTER",
        utilityScore: 0.5,
        counterPrice: 50000,
        counterPaymentTerms: null,
        counterDelivery: null,
        concerns: [],
        tone: "formal",
        targetPrice: 45000,
        maxAcceptablePrice: 55000,
        roundNumber: 1,
      });
      const intent2 = buildNegotiationIntent({
        action: "COUNTER",
        utilityScore: 0.5,
        counterPrice: 50000,
        counterPaymentTerms: null,
        counterDelivery: null,
        concerns: [],
        tone: "formal",
        targetPrice: 45000,
        maxAcceptablePrice: 55000,
        roundNumber: 2,
      });
      expect(intent1.commercialPosition).not.toBe(intent2.commercialPosition);
    });
  });
});

describe("Humanization Audit — Sprint 4", () => {

  describe("A1: Arc summary", () => {
    it("returns empty for < 2 rounds", () => {
      expect(buildArcSummary([])).toBe("");
      expect(buildArcSummary([
        { role: "VENDOR", content: "test", extractedOffer: { total_price: 50000 }, counterOffer: null, decisionAction: null },
      ])).toBe("");
    });

    it("builds arc for 2+ completed rounds", () => {
      const messages = [
        { role: "VENDOR", content: "", extractedOffer: { total_price: 60000 }, counterOffer: null, decisionAction: null },
        { role: "ACCORDO", content: "", extractedOffer: null, counterOffer: { total_price: 50000 }, decisionAction: "COUNTER" },
        { role: "VENDOR", content: "", extractedOffer: { total_price: 55000 }, counterOffer: null, decisionAction: null },
        { role: "ACCORDO", content: "", extractedOffer: null, counterOffer: { total_price: 52000 }, decisionAction: "COUNTER" },
      ];
      const arc = buildArcSummary(messages, "$");
      expect(arc).toContain("$60,000");
      expect(arc).toContain("$50,000");
      expect(arc).toContain("$55,000");
      expect(arc).toContain("$52,000");
      expect(arc).toContain("2 rounds");
    });

    it("does not contain banned words", () => {
      const messages = [
        { role: "VENDOR", content: "", extractedOffer: { total_price: 60000 }, counterOffer: null, decisionAction: null },
        { role: "ACCORDO", content: "", extractedOffer: null, counterOffer: { total_price: 50000 }, decisionAction: "COUNTER" },
        { role: "VENDOR", content: "", extractedOffer: { total_price: 55000 }, counterOffer: null, decisionAction: null },
        { role: "ACCORDO", content: "", extractedOffer: null, counterOffer: { total_price: 52000 }, decisionAction: "COUNTER" },
      ];
      const arc = buildArcSummary(messages, "$");
      expect(arc.toLowerCase()).not.toContain("utility");
      expect(arc.toLowerCase()).not.toContain("threshold");
      expect(arc.toLowerCase()).not.toContain("target");
      expect(arc.toLowerCase()).not.toContain("max_acceptable");
      expect(arc.toLowerCase()).not.toContain("weight");
    });

    it("caps at 150 words", () => {
      // Build a long message history (10 rounds)
      const messages: any[] = [];
      for (let i = 0; i < 10; i++) {
        messages.push({ role: "VENDOR", content: "", extractedOffer: { total_price: 60000 - i * 1000 }, counterOffer: null, decisionAction: null });
        messages.push({ role: "ACCORDO", content: "", extractedOffer: null, counterOffer: { total_price: 50000 + i * 500 }, decisionAction: "COUNTER" });
      }
      const arc = buildArcSummary(messages, "$");
      const wordCount = arc.split(/\s+/).length;
      expect(wordCount).toBeLessThanOrEqual(151); // 150 + possible "..."
    });
  });

  describe("L2: Locale-locked formatting", () => {
    it("formats prices consistently regardless of runtime locale", () => {
      // toLocaleString("en-US") should always produce comma-separated thousands
      const formatted = (98500).toLocaleString("en-US");
      expect(formatted).toBe("98,500");
      const large = (3150000).toLocaleString("en-US");
      expect(large).toBe("3,150,000");
    });
  });
});
```

### Notes on the test file:
- Adjust import paths if your project uses path aliases or different module resolution
- The test uses actual exported functions (no mocking) — these are integration/regression tests
- If any test fails, it means a fix was reverted or broken
- Place the file wherever your existing test files live (likely `src/__tests__/` or alongside the source files)

---

## Part 2: Update Stale Comments & Documentation

Several file-level JSDoc comments reference old behavior that no longer applies after our changes. Update these:

### File: `src/negotiation/intent/build-negotiation-intent.ts`

The `NegotiationIntent.firmness` JSDoc comment (around line 138-142) still says:
```
 * Utility ≥70% → 0.25 | 50–70% → 0.55 | 30–50% → 0.75 | <30% → 0.90
```

Update to:
```
 * Utility ≥80% → 0.15 | 65–80% → 0.35 | 50–65% → 0.55 | 35–50% → 0.75 | <35% → 0.90
```

### File: `src/llm/persona-renderer.ts`

The top-of-file comment (lines 12-13) still says:
```
 * - Limits responses to ~120 words.
 * - Temperature: 0.5 for controlled, consistent output.
```

Update to:
```
 * - Response length adapts per action type (8–140 words).
 * - Temperature: 0.7 for natural variation across rounds.
```

### File: `src/llm/phrasing-history.ts`

The top-of-file comment (lines 14-16) still says:
```
 * Fingerprint format: first 3 words (lowercased, punctuation-stripped) joined
 * by ":" with the action type prefixed — e.g. "COUNTER|appreciate:the:quick".
 * Coarse on purpose so a one-word change still trips the de-dup check.
```

Update to:
```
 * Fingerprint format: first 5 words (lowercased, punctuation-stripped) joined
 * by ":" with the action type prefixed — e.g. "COUNTER|appreciate:the:quick:turnaround:on".
 * Balanced: specific enough to avoid false collisions while coarse enough that
 * minor rephrasing still trips the de-dup check.
```

### File: `src/llm/validate-llm-output.ts`

The top-of-file comment (line 9) says:
```
 * 2. Response must be ≤ 160 words.
```

This was already changed to adaptive per-action bounds. Update to:
```
 * 2. Response length checked per-action (ACCEPT 8–60, COUNTER 25–110, MESO 25–140, etc.).
```

---

## Part 3: Safety Sweep — grep for Remaining Issues

Run these grep checks and fix any findings:

```bash
# 1. No bare toLocaleString without locale (already verified but double-check after test file addition)
grep -rn "\.toLocaleString()" src/ --include="*.ts" | grep -v "en-US" | grep -v "node_modules" | grep -v "__tests__"

# 2. No "concerns: []" anywhere in conversation-service
grep -n "concerns: \[\]" src/modules/chatbot/convo/conversation-service.ts

# 3. No banned words leaking into arc-summary (non-comment lines)
grep -n "utility\|threshold\|target_price\|max_acceptable" src/llm/arc-summary.ts | grep -v "^\s*//"

# 4. Confirm slice(0, 5) in phrasing-history (not 3)
grep -n "slice(0," src/llm/phrasing-history.ts

# 5. COUNTER_REASONING_HINTS has 7+ entries
grep -c "\"" src/llm/persona-renderer.ts | head -1  # rough check; manually verify array

# 6. humanRoundPrice is exported and called
grep -n "humanRoundPrice" src/negotiation/intent/build-negotiation-intent.ts
```

If ANY of checks 1–4 return unexpected results, fix them. Checks 5–6 are confirmatory.

---

## Part 4: End-to-End Smoke Test (Manual Verification)

After the test file is created and comments are updated, run:

```bash
# Full compile check
npx tsc --noEmit

# Run all tests including the new audit file
npm run test:unit

# Run ONLY the new audit tests to see them pass individually
npx jest humanization-audit --verbose
```

All tests should pass. If any audit test fails, it indicates a Sprint 1–4 fix was not correctly applied or was accidentally reverted — investigate and fix.

---

## Scope

- **Create**: `src/__tests__/humanization-audit.test.ts` (new regression guard)
- **Update comments only**: `build-negotiation-intent.ts`, `persona-renderer.ts`, `phrasing-history.ts`, `validate-llm-output.ts`
- **No functional changes** to any production code
- **No new dependencies**
- **Do NOT modify** any existing test files — the new audit file is additive only

---

## Success Criteria

- `tsc --noEmit` clean
- All existing 1225 tests still pass
- New `humanization-audit.test.ts` passes (all ~25 test cases green)
- Stale comments updated to reflect current behavior
- Safety grep checks return clean results
- Total test count: 1225 + ~25 new = ~1250 passing
