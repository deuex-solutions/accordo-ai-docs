# Comprehensive Analysis: Utility Score & BATNA in Accordo AI and Industry

> **Date:** February 2026
> **Scope:** Full codebase analysis (frontend + backend) and industry research

---

## Table of Contents

1. [Part 1: Utility Score in Accordo AI](#part-1-utility-score-in-accordo-ai)
   - [1.1 How It's Calculated](#11-how-its-calculated)
   - [1.2 Individual Parameter Utilities](#12-individual-parameter-utilities)
   - [1.3 How It Drives Decisions (INSIGHTS Mode)](#13-how-it-drives-decisions-insights-mode)
   - [1.4 Counter-Offer Generation](#14-counter-offer-generation)
   - [1.5 Parameter Weights (Deal Wizard Step 4)](#15-parameter-weights-deal-wizard-step-4)
   - [1.6 Frontend Visualization](#16-frontend-visualization)
   - [1.7 Explainability & Audit Trail](#17-explainability--audit-trail)
   - [1.8 Advanced Features](#18-advanced-features)
2. [Part 2: BATNA in Accordo AI](#part-2-batna-in-accordo-ai)
   - [2.1 Definition & Storage](#21-definition--storage)
   - [2.2 How BATNA Is Set](#22-how-batna-is-set)
   - [2.3 How BATNA Influences Decisions](#23-how-batna-influences-decisions)
   - [2.4 BATNA Display (Frontend)](#24-batna-display-frontend)
   - [2.5 BATNA in Smart Defaults](#25-batna-in-smart-defaults)
3. [Part 3: Industry Research](#part-3-industry-research)
   - [3.1 Multi-Attribute Utility Theory (MAUT)](#31-multi-attribute-utility-theory-maut)
   - [3.2 Constructing Utility Functions](#32-constructing-utility-functions)
   - [3.3 How Procurement Platforms Use Utility Scoring](#33-how-procurement-platforms-use-utility-scoring)
   - [3.4 BATNA in B2B Procurement](#34-batna-in-b2b-procurement)
   - [3.5 BATNA vs ZOPA](#35-batna-vs-zopa)
   - [3.6 Combining Utility Score and BATNA](#36-combining-utility-score-and-batna)
   - [3.7 Game-Theoretic Models](#37-game-theoretic-models)
   - [3.8 AI Negotiation Bots: Utility + BATNA](#38-ai-negotiation-bots-utility--batna)
   - [3.9 Decision Frameworks for Automated Negotiation](#39-decision-frameworks-for-automated-negotiation)
   - [3.10 Concession Strategies](#310-concession-strategies)
   - [3.11 Real-World Case Studies](#311-real-world-case-studies)
   - [3.12 Accordo vs Industry Comparison](#312-accordo-vs-industry-comparison)
4. [Sources](#sources)

---

## Part 1: Utility Score in Accordo AI

### 1.1 How It's Calculated

The system uses a **weighted multi-attribute utility model** with 12+ parameters:

```
Total Utility = Σ (Parameter_Utility × Parameter_Weight / 100)
```

**Core engine files:**

| File | Purpose |
|------|---------|
| `src/modules/chatbot/engine/utility.ts` | Legacy 2-parameter model (price + payment terms) |
| `src/modules/chatbot/engine/weightedUtility.ts` | Advanced multi-parameter model (12+ params) |
| `src/modules/chatbot/engine/parameterUtility.ts` | 6 utility calculation types |
| `src/modules/chatbot/engine/deliveryUtility.ts` | Delivery-specific utility calculations |
| `src/modules/chatbot/engine/decide.ts` | Decision engine using utility thresholds |
| `src/modules/chatbot/engine/types.ts` | Type definitions and default constants |

#### Legacy Simple Utility (Two-Parameter Model)

**File:** `engine/utility.ts` (lines 51-139)

```typescript
// Price: Lower is better for buyer
function priceUtility(config: NegotiationConfig, price: number) {
  const { target, max_acceptable } = config.parameters.total_price;
  if (price <= target) return 1;
  if (price >= max_acceptable) return 0;
  return 1 - (price - target) / (max_acceptable - target);
}

// Payment Terms: Longer is better for buyer
// Net 30 = 1.0, Net 60 = 0.7, Net 90 = 0.5 (configurable)
function termsUtility(config: NegotiationConfig, terms: string | number | null): number

// Combined Total
function totalUtility(config: NegotiationConfig, offer: Offer) {
  const wP = config.parameters.total_price.weight;   // e.g., 60%
  const wT = config.parameters.payment_terms.weight;  // e.g., 40%
  return clamp01(priceUtility * wP + termsUtility * wT);
}
```

#### Advanced Weighted Utility (Multi-Parameter Model)

**File:** `engine/weightedUtility.ts` (lines 40-1083)

Supports 12+ parameters from the Deal Wizard with normalization:

```typescript
// If weights don't sum to 100, scale result
totalUtility = totalUtility × (100 / totalWeight)
// Final result clamped to [0, 1] range
```

### 1.2 Individual Parameter Utilities

**File:** `engine/parameterUtility.ts` — Six utility calculation types:

| Type | Calculation | Example |
|------|-------------|---------|
| **Linear (lower_better)** | `1 - (value - target) / range` | Price: $95 offer with $90 target, $100 max = 0.5 utility |
| **Linear (higher_better)** | `(value - min) / range` | Discount: 10% offered, 5% min, 15% max = 0.5 utility |
| **Stepped** | Predefined mappings for discrete values | Net 30 = 0.5, Net 60 = 0.75, Net 90 = 1.0 |
| **Date** | Proximity-based utility | Earlier delivery = higher utility |
| **Percentage** | Specialized for % values | Advance payment: lower % = higher utility |
| **Boolean/Binary** | Threshold-based | Certification match = 1, no match = 0 |

**Parameter-specific calculations (weightedUtility.ts):**

| Parameter | Lines | Direction | Calculation |
|-----------|-------|-----------|-------------|
| Target Unit Price | 832-893 | Lower is better | Linear toward target price |
| Max Acceptable Price | 832-893 | Threshold | Within budget = 1, outside = 0 |
| Volume Discount | 832-893 | Higher is better | Percentage-based linear |
| Advance Payment Limit | 895-920 | Lower is better | Percentage-based |
| Payment Terms Days | 895-920 | Longer is better | Stepped with custom min/max |
| Delivery Date | 922-948 | Earlier is better | Date proximity |
| Warranty Period | 950-1000 | Longer is better | Linear |
| Quality Standards | 950-1000 | Must match | Binary certification matching |

### 1.3 How It Drives Decisions (INSIGHTS Mode)

**File:** `engine/decide.ts` (lines 36-626)

#### Decision Threshold Zones

```
Utility Zone Mapping (Default Thresholds):
┌─────────────────────────────────────────────────────────┐
│ 0%           30%           50%           70%         100%│
├─────────────┼─────────────┼─────────────┼─────────────┤
│  WALK_AWAY  │  ESCALATE   │  COUNTER    │   ACCEPT    │
│    < 30%    │  30%-50%    │  50%-70%    │   ≥ 70%     │
└─────────────┴─────────────┴─────────────┴─────────────┘
```

#### Dynamic Thresholds by Priority

**File:** `engine/weightedUtility.ts` (lines 737-756)

| Priority | Accept | Escalate | Walk Away | Strategy |
|----------|--------|----------|-----------|----------|
| **HIGH** (Maximize Savings) | 75% | 55% | 35% | Aggressive, strict thresholds |
| **MEDIUM** (Fair Deal) | 70% | 50% | 30% | Balanced approach |
| **LOW** (Quick Close) | 65% | 45% | 25% | Relaxed, faster closure |

#### Decision Logic Flow (lines 497-625)

```
Step 1: Check for Missing Data
  → If price or terms missing → ASK_CLARIFY

Step 2: Check Budget Violation
  → If price > max_acceptable:
    → Round < 10: COUNTER with calculated price
    → Round ≥ 10: WALK_AWAY

Step 3: Apply Utility-Based Decisions
  → if utility >= acceptThreshold → ACCEPT
  → else if utility < walkawayThreshold:
    → Round ≥ 10 AND vendor rigid AND not exploring → WALK_AWAY
    → Otherwise → COUNTER
  → else if utility < escalateThreshold:
    → Round ≥ 10 AND stalled 3+ rounds → ESCALATE
    → Otherwise → COUNTER
  → else (counter zone) → COUNTER
```

### 1.4 Counter-Offer Generation

**Function:** `calculateDynamicCounter()` — `engine/decide.ts` (lines 248-370)

#### Price Aggressiveness by Priority

```
Counter Price = Target + (Price Range × Aggressiveness Factor)

Priority HIGH:   15% of range (stays closest to target)
Priority MEDIUM: 40% of range (moderate concessions)
Priority LOW:    55% of range (moves toward vendor quickly)
```

**Dynamic adjustments per round:**
- Round adjustment: +2% per round, capped at +10%
- Concession bonus: Based on vendor's previous concessions
- Emphasis adjustment: +/- 5-10% based on vendor focus area

**Example calculation:**
```
Target: $90, Max Acceptable: $100, Range: $10

Priority MEDIUM (40%):
Counter = $90 + ($10 × 0.40) = $94

Round 3 adjustment: +2% per round = +2%
Counter = $94 + ($10 × 0.02) = $94.20
```

#### Payment Terms Selection
- HIGH priority: Best terms for buyer (Net 90 = longest payment)
- MEDIUM/LOW: Incrementally better terms (Net 30 → Net 45 → Net 60)

### 1.5 Parameter Weights (Deal Wizard Step 4)

**File:** `engine/weightedUtility.ts` (lines 150-347, 669-677)

**Default AI-Suggested Weights:**

```
targetUnitPrice:           35%
maxAcceptablePrice:        20%
volumeDiscountExpectation: 10%
advancePaymentLimit:        5%
deliveryDate:              15%
warrantyPeriod:            10%
qualityStandards:           5%
─────────────────────────────
Total:                     100%
```

**Weight Application:**
```typescript
// Each parameter's contribution
contribution = utility × (weight / 100)

// Example:
// Price utility: 0.75, Weight: 35%
// Contribution: 0.75 × 0.35 = 0.2625

// Normalization if weights don't sum to 100:
totalUtility = totalUtility × (100 / totalWeight)
```

**Impact on Decisions:**
- If user weights price heavily (60%): Price becomes dominant driver
- If user weights delivery heavily (30%): Delivery becomes critical
- Thresholds remain the same (70%/50%/30%) regardless of weight distribution

### 1.6 Frontend Visualization

#### Components Overview

| Component | File | Key Features |
|-----------|------|--------------|
| **Unified Utility Bar** | `chatbot/sidebar/UnifiedUtilityBar.tsx` | Linear gradient bar with threshold zones, animated dot indicator |
| **Weighted Utility Bar** | `chatbot/sidebar/WeightedUtilityBar.tsx` | Circular progress (0-100%), per-parameter breakdown, sparkline trend |
| **Decision Threshold Gauge** | `chatbot/sidebar/DecisionThresholdZones.tsx` | Semicircular gauge with animated needle |
| **Convergence Chart** | `chatbot/sidebar/ConvergenceChart.tsx` | Line chart: vendor vs PM price over rounds |
| **Decision Badge** | `chatbot/chat/DecisionBadge.tsx` | Color-coded action badge with utility score |

#### Unified Utility Bar

```
0%          30%         50%        70%       100%
|-----------|-----------|----------|---------|
  Walk Away   Escalate    Counter    Accept
   (Red)      (Orange)     (Blue)    (Green)

   Current: 65% → Dot positioned at 65% on gradient
```

**Color coding:**
- Excellent (≥ 80%): Green (#22C55E)
- Good (60-80%): Blue (#3B82F6)
- Warning (40-60%): Yellow (#EAB308)
- Critical (< 40%): Red (#EF4444)

#### Weighted Utility Bar (Parameter Breakdown)

```
Weighted Utility: 65%

Price:        ████████████████████░░░░░░░░░░ 45%  (Blue)
Payment:      █████████████░░░░░░░░░░░░░░░░░ 32%  (Purple)
Delivery:     ███████░░░░░░░░░░░░░░░░░░░░░░░ 15%  (Green)
Contract:     ███░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 8%  (Amber)

Trend: ↑ +3% (Sparkline showing 5-10 historical values)
Recommendation: Counter ↔
```

#### Convergence Chart

```
$105 ┤              ╲         (Vendor Offer - Red)
$100 ┤           ╲╲╲╲╲╲╲╲
 $95 ┤        ╱╱╱           (PM Counter - Blue)
 $90 ┤     ╱╱╱
 $85 ┤   ╱╱
     └────────────────────────
       R1  R2  R3  R4  R5

     Status: Converging ✓
```

#### Decision Badge

```
┌─────────────────┬──────────────┐
│ ✓ Accept        │ 🔢 75%       │    → Green background
│ ↔ Counter       │ 🔢 58%       │    → Blue background
│ ↑ Escalate      │ 🔢 42%       │    → Orange background
│ ✕ Walk Away     │ 🔢 25%       │    → Red background
│ ? Ask Clarify   │              │    → Yellow background
└─────────────────┴──────────────┘
```

### 1.7 Explainability & Audit Trail

**Function:** `computeExplainability()` — `engine/utility.ts` (lines 146-204)

Every decision includes a detailed breakdown:

```typescript
{
  vendorOffer: { total_price: 95, payment_terms: "Net 60" },
  utilities: {
    priceUtility: 0.75,
    termsUtility: 0.80,
    weightedPrice: 0.45,    // 0.75 × 0.60
    weightedTerms: 0.32,    // 0.80 × 0.40
    total: 0.77,            // 77%
  },
  decision: { action: "ACCEPT", reasons: [...], counterOffer: null },
  configSnapshot: { weights, thresholds, priceConfig, termOptions },
}
```

### 1.8 Advanced Features

#### Vendor Emphasis Detection (`engine/preferenceDetector.ts`)

```
vendorEmphasis: 'price-focused' | 'terms-focused' | 'balanced' | 'unknown'

Dynamic Counter Strategy:
- Price-focused vendor → Offer higher price, push harder on terms
- Terms-focused vendor → Push harder on price, be flexible on terms
- Balanced vendor → Use standard priority-based calculation
```

#### Behavioral Signals
- `concessionVelocity`: Average price concession per round
- `convergenceRate`: How fast offers are meeting
- `isConverging`: True if gap decreasing
- `isStalling`: True if utility unchanged for 3+ rounds

#### MESO (Multi-Equitable Strategic Options) (`engine/types.ts`, lines 192-223)

When utility stalls, the system presents 3 distinct offers:
- Each offer has the same total utility for the buyer
- Different attribute combinations let the vendor reveal preferences
- Tracked via `MesoSelectionRecord` to learn vendor priorities

#### Utility History Tracking

```typescript
interface UtilityHistoryRecord {
  round: number;
  utility: number;     // 0-1
  timestamp: Date;
}
// Stored in NegotiationState.utilityHistory
// Used for sparkline visualization and stall detection
```

### 1.9 End-to-End Example Flow

```
Step 1: Vendor submits offer
├─ Total Price: $95
├─ Payment Terms: "Net 60"
└─ Delivery: 5 days

Step 2: Calculate individual utilities
├─ Price Utility = 0.50  (target $90, max $100: 1 - ($95-$90)/($100-$90))
├─ Terms Utility = 0.75  (interpolated for Net 60)
└─ Delivery Utility = 0.95 (5 days, target date met)

Step 3: Apply weights (user configured)
├─ Price weight: 35%  → 0.50 × 0.35 = 0.175
├─ Terms weight: 20%  → 0.75 × 0.20 = 0.150
├─ Delivery weight: 15% → 0.95 × 0.15 = 0.143
└─ Other params: 30% → 0.80 × 0.30 = 0.240

Total Utility = 0.175 + 0.150 + 0.143 + 0.240 = 0.708 (71%)

Step 4: Apply thresholds (MEDIUM priority)
├─ 71% >= 70% (accept threshold)? YES
└─ Action: ACCEPT ✓

Step 5: Display to PM
├─ Unified Bar → 71% in green "Accept" zone
├─ Weighted Bar → Price 50%, Terms 75%, Delivery 95%
├─ Badge → "✓ Accept | 71%"
└─ Reason: "Utility 71% >= accept threshold 70%"
```

---

## Part 2: BATNA in Accordo AI

### 2.1 Definition & Storage

**Database:**
- **Model:** Requisition (`src/models/requisition.ts`, line 73)
- **Column:** `batna: DataTypes.DOUBLE` (line 147)
- **Type:** `number | null`
- **Semantic meaning:** "Minimum acceptable price" / "Target price for negotiation"

**Related fields on Requisition:**
- `maxDiscount: DataTypes.DOUBLE` — Maximum discount allowed
- `discountedValue: DataTypes.DOUBLE` — Current discounted value calculation

**Type definitions:**

```typescript
// Backend (src/types/index.ts, lines 110-116)
interface UserPreferences {
  batna?: number;
  maxDiscount?: number;
  maxPrice?: number;
  priceWeight?: number;
  deliveryWeight?: number;
}

// Frontend (src/types/chatbot.ts, lines 688-693)
negotiationLimits?: {
  batna: number | null;
  maxDiscount: number | null;
  discountedValue: number | null;
};
```

### 2.2 How BATNA Is Set

BATNA is **user-provided** (manually entered), **not AI-calculated**.

**Frontend entry point:** `src/components/Requisition/NegotiationParameters.tsx`

```tsx
// Line 136
<label>BATNA (Best Alternative To a Negotiated Agreement)</label>

// Line 138-147
<input type="number" name="batna" min={0} step={0.01} placeholder="Enter BATNA value" />

// Line 149 (helper text)
"Target price for negotiation. This is the minimum acceptable price."
```

**Validation (lines 73-76):**
```typescript
if (formData.batna && Number(formData.batna) <= 0) {
  toast.error("BATNA must be greater than 0");
  return;
}
```

**Backend update:** Saved via `PUT /requisition/update/:id` as an allowed numeric field (`requisition.service.ts`, line 398).

**BATNA is static** — it does not change during negotiation. It remains fixed at the requisition level.

### 2.3 How BATNA Influences Decisions

BATNA's influence is **indirect**, feeding through the utility calculation chain:

```
BATNA → informs max_acceptable price parameter
       → price utility function (prices at/above max_acceptable = 0% utility)
       → weighted total utility
       → threshold comparison
       → ACCEPT / COUNTER / WALK_AWAY decision
```

**The `decideNextMove` function does NOT contain explicit BATNA checks.** Instead:

1. BATNA conceptually shapes the `max_acceptable` price in the negotiation config
2. The utility calculation uses `max_acceptable`: `priceUtility = 1 - (price - target) / (max_acceptable - target)`
3. Utility thresholds (70%, 50%, 30%) drive the actual decisions

**Relationship to walkaway threshold:**
- **Walkaway Threshold:** A utility score percentage (default 30%) that triggers WALK_AWAY
- **BATNA:** An absolute price target (in currency units)
- They are conceptually distinct but related — BATNA helps define the utility function that the walkaway threshold evaluates

### 2.4 BATNA Display (Frontend)

**Requisition Parameters component** (`NegotiationParameters.tsx`, lines 129-245):

When both BATNA and Discounted Value are populated, a status summary appears:

```
┌──────────────────────────────────────────┐
│  Negotiation Status                       │
│                                           │
│  BATNA Target:      $95,000              │
│  Current Value:     $92,000              │
│  Difference:        -$3,000              │
│                                           │
│  ✓ Target achieved! Current value is     │
│    at or below BATNA.                    │
└──────────────────────────────────────────┘
```

**BATNA is NOT displayed in the active negotiation room sidebar.** The sidebar focuses on:
- Parameter-level utility scores
- Threshold zones (accept, counter, escalate, walkaway)
- Convergence charts

### 2.5 BATNA in Smart Defaults

**Function:** `getSmartDefaultsService()` — `chatbot.service.ts` (lines 1044-1201)

BATNA is **retrieved, not computed** by smart defaults:

```typescript
// Lines 1157-1160: Simply passes through user-entered value
const batna = requisition.batna || null;
const maxDiscount = requisition.maxDiscount || null;
const discountedValue = requisition.discountedValue || null;

// Lines 1188-1193: Returned in response
negotiationLimits: { batna, maxDiscount, discountedValue }
```

Smart defaults DO compute price suggestions independently:
- `targetUnitPrice = averageTarget` (from requisition product data)
- `maxAcceptablePrice = averageTarget × 1.2` (20% above target)

These are calculated from product/vendor history, **not derived from BATNA**.

### 2.6 Implementation Summary

| Aspect | Details | File Reference |
|--------|---------|----------------|
| Storage | DOUBLE column in Requisitions table | `requisition.ts:73, 147` |
| Type | `number \| null` | `types/index.ts:111`, `chatbot.ts:690` |
| Entry | Manual form input | `NegotiationParameters.tsx:138-147` |
| Validation | Must be > 0 | `NegotiationParameters.tsx:73-76` |
| Smart Defaults | Retrieved, not computed | `chatbot.service.ts:1157-1160` |
| Decision Making | Indirect via `max_acceptable` price | `utility.ts:56`, `decide.ts:456-457` |
| Walkaway Threshold | Separate utility-based (30%) | `decide.ts:387` |
| Frontend Display | Negotiation Parameters only | `NegotiationParameters.tsx:129-245` |
| Negotiation Room | Not displayed in sidebar | N/A |
| Static/Dynamic | Static (set once) | Does not change during negotiation |

---

## Part 3: Industry Research

### 3.1 Multi-Attribute Utility Theory (MAUT)

MAUT is the foundational mathematical framework used across procurement platforms. The core formula:

```
U(x) = Σ(i=1 to n) [ w_i × u_i(x_i) ]
```

Where:
- `U(x)` = total utility score for a supplier offer `x`
- `w_i` = weight assigned to attribute `i` (where Σ w_i = 1.0)
- `u_i(x_i)` = individual utility function for attribute `i`
- `n` = number of evaluation criteria

**Validity condition:** The additive form requires *preferential independence* among attributes. When this doesn't hold, a multiplicative form is used.

### 3.2 Constructing Utility Functions

Each attribute has a single-attribute utility function normalizing raw values to [0, 1]:

**Price / Cost (inversely proportional):**
```
u_price(x) = (x_max - x) / (x_max - x_min)
```

**Delivery Lead Time (inversely proportional):**
```
u_delivery(x) = (T_max - x) / (T_max - T_min)
```

**Quality (directly proportional):**
```
u_quality(x) = (x - Q_min) / (Q_max - Q_min)
```

**Payment Terms (directly proportional for buyer):**
```
u_payment(x) = (x - P_min) / (P_max - P_min)
```

**Typical weighted scoring model:**

| Attribute | Weight | Score (1-5) | Weighted Score |
|-----------|--------|-------------|----------------|
| Price | 0.40 | 4 | 1.60 |
| Quality | 0.25 | 5 | 1.25 |
| Delivery | 0.20 | 3 | 0.60 |
| Payment Terms | 0.10 | 4 | 0.40 |
| Sustainability | 0.05 | 3 | 0.15 |
| **Total** | **1.00** | | **4.00** |

**Weight determination methods:**
- **Direct assignment:** Stakeholders assign importance percentages
- **Pairwise comparisons (AHP):** Criteria compared against each other systematically
- **CRITIC method:** Correlation analysis between criteria to reduce subjectivity
- **Swing weights (SMARTS):** Rank criteria by "swing" from worst to best value

### 3.3 How Procurement Platforms Use Utility Scoring

| Platform | Approach |
|----------|----------|
| **SAP Ariba** | Multilevel weighted scoring, graphical TCO formula builder, total cost auctions beyond price |
| **Coupa** | Multi-agent AI "Navi" leveraging $8T in anonymized spend data for benchmarking |
| **Keelvar** | Sourcing optimization exploring thousands of award scenarios with combinatorial optimization; "elasticity thresholds" and "circuit breakers" for human escalation |
| **GEP SMART** | AI/ML for automated supplier selection, analyzing historical spending patterns and performance metrics |
| **Jaggaer** | Agentic AI suggesting follow-up questions, adapting criteria weights based on emerging priorities |
| **Fairmarkit** | AI agents for tail spend: demand management, intelligent supplier matching, automated negotiations |
| **Ivalua** | Intelligent Virtual Assistant (IVA) and generative AI modules across procurement workflows |

### 3.4 BATNA in B2B Procurement

#### Definition

**BATNA** (Best Alternative to a Negotiated Agreement) — defined by the Harvard Program on Negotiation as "the most realistic thing you will do if you do not reach an agreement today."

In procurement, BATNA manifests as:
- Switching to an alternative qualified supplier
- Bringing the capability in-house (make vs. buy)
- Accepting a standing offer from another vendor
- Postponing the purchase
- Redesigning the product to eliminate the need

#### Calculation

```
BATNA Value = Value of Best Alternative - Switching Costs - Risk Premium

Reservation Price = BATNA Value + Transaction-Specific Adjustments
```

**Example:** Component priced at $10/unit, alternative supplier offers $10.50/unit with $0.20/unit switching costs → BATNA value = $10.70/unit → Reservation price = $10.70.

#### Pre-Negotiation BATNA Assessment (Best Practice)

1. Request competitive quotes from 2+ suppliers
2. Validate quality (samples, QA)
3. Calculate switching costs (tooling, shipping, requalification)
4. Prepare transition plan (e.g., phased 30/70 split)
5. Quantify BATNA monetarily
6. Set reservation price (BATNA adjusted for risk)

#### Dynamic BATNA Adjustment

Modern practice treats BATNA as dynamic:
- Updated as new market data arrives
- If competing offer falls through → BATNA weakens → reservation price shifts
- If new alternatives emerge → BATNA strengthens
- Automated systems recalculate each round based on: elapsed time, market intelligence, parallel negotiation outcomes, opponent concession patterns

### 3.5 BATNA vs ZOPA

**ZOPA (Zone of Possible Agreement)** = range where buyer's and seller's reservation prices overlap:

```
ZOPA exists when: Buyer's Reservation Price > Seller's Reservation Price
ZOPA Range = [Seller's Reservation Price, Buyer's Reservation Price]
```

**Example:**
- Buyer willing to pay $2,500-$3,000 (reservation = $3,000)
- Seller will accept $2,750-$3,250 (reservation = $2,750)
- ZOPA = $2,750 to $3,000

**No overlap** (buyer max < seller min) = **NOPA** → rational parties walk to BATNAs.

### 3.6 Combining Utility Score and BATNA

Modern platforms integrate both in a layered architecture:

```
Layer 1 — Value Mapping:
  Define utility function across all terms using MAUT

Layer 2 — BATNA as Reservation Utility:
  Convert BATNA to utility score (e.g., if BATNA alternative yields 0.65 utility,
  agent won't accept offers below 0.65)

Layer 3 — Dynamic Threshold:
  threshold(t) = RU + (aspiration - RU) × f(t)
  Starts above reservation utility, decays toward it over time

Layer 4 — Decision Logic:
  IF offer_utility >= aspiration_level    → ACCEPT
  ELSE IF offer_utility >= reservation    → COUNTER (with concession)
  ELSE IF offer_utility < reservation     → WALK AWAY
```

### 3.7 Game-Theoretic Models

#### Nash Bargaining Solution

Identifies the agreement maximizing the product of both parties' utility gains over their BATNAs:

```
Maximize: (U_buyer(x) - U_buyer(BATNA_buyer)) × (U_seller(x) - U_seller(BATNA_seller))

Subject to:
  U_buyer(x) >= U_buyer(BATNA_buyer)
  U_seller(x) >= U_seller(BATNA_seller)
```

This yields the Pareto-optimal, symmetric fair outcome.

#### Rubinstein Alternating-Offers Model

For multi-round negotiations with time discounting:

```
Player 1's equilibrium share = (1 - d2) / (1 - d1 × d2)

Where d = e^(-r × Δt)  (discount factor from time preferences)
```

Key insight: The more patient player captures a larger share. As both become very patient, the solution converges to Nash bargaining.

### 3.8 AI Negotiation Bots: Utility + BATNA

#### Pactum AI Architecture

- **Contract Space:** Sum of all negotiable terms, each with associated Value Functions (utility)
- **Contract Sampling:** Millions of contracts sampled to find optimal counter-offers
- **MESOs:** Multiple Equivalent Simultaneous Offers — same utility for buyer, different attribute combos
- **Minimum Acceptance Thresholds:** Pre-configured reservation values
- **Dynamic Pricing:** Initial offers adjusted based on real-time acceptance patterns
- **Behavioral Layer:** Leverages "world's largest database of behavioral negotiation learnings"
- **Core philosophy:** "80% of human-negotiated contracts never reach the Pareto optimal line"
- **Result:** Average 11% increase in deal value

#### MIA-RVelous Algorithm (Academic, 2024)

```
EU_rv(π) = Σ(i=1 to D) [ u_i × a_i × Π(j=1 to i-1)(1 - a_j) ] + rv × Π(j=1 to D)(1 - a_j)

Where:
  u_i = agent's utility for bid i
  a_i = opponent's estimated acceptance probability for bid i
  rv  = reservation value (BATNA utility)
  D   = deadline (maximum rounds)
```

Higher reservation values encourage riskier (more ambitious) bidding. Bids with utility below `rv` are filtered out entirely.

### 3.9 Decision Frameworks for Automated Negotiation

#### Standard Accept / Counter / Walk Away

```
Given: offer_utility, threshold T(t), reservation_value RV

IF offer_utility >= T(t):
    → ACCEPT

ELSE IF offer_utility >= RV:
    → COUNTER (concede from current aspiration)

ELSE IF offer_utility < RV:
    → WALK AWAY to BATNA
```

**Advanced conditions (from ANAC research):**
- Is current offer better than any previous offer?
- Is expected utility of future offers lower than current?
- Does accepting now yield higher expected utility than continuing?

### 3.10 Concession Strategies

#### Time-Dependent Concession

```
target_utility(t) = U_min + (U_max - U_min) × (1 - (t/T_max)^(1/e))

Where:
  U_min = reservation utility (BATNA)
  U_max = aspiration utility (ideal outcome)
  t     = current time/round
  T_max = deadline
  e     = concession parameter
```

| Strategy | Parameter `e` | Behavior |
|----------|---------------|----------|
| **Boulware** | 0 < e < 1 | Minimal early concessions; sharp drop near deadline. Most effective in competitive settings. |
| **Linear** | e = 1 | Steady, constant-rate concessions throughout. |
| **Conceder** | e > 1 | Rapid early concessions; flattens near deadline. |

#### Behavior-Dependent (Tit-for-Tat)

```
next_concession = γ × opponent_last_concession

Where γ = adjustment parameter:
  γ = 1  → mirror exactly
  γ < 1  → concede less than opponent
  γ > 1  → concede more than opponent
```

#### Multi-Parameter Trade-Off (Pareto-Seeking)

1. Maintain total utility constant
2. Increase utility on issues valued by opponent
3. Decrease utility on issues less valued by opponent
4. Result: Move toward Pareto frontier without reducing own utility

### 3.11 Real-World Case Studies

#### Pactum AI — Walmart and Maersk

| Metric | Result |
|--------|--------|
| Supplier agreement rate | 68% of suppliers reached deals with AI |
| Savings | Average 3% on negotiated deals |
| Supplier preference | 75% preferred the bot over human negotiators |
| Speed | Deals in days vs weeks |
| Use case | Replenishment terms for low-margin, frequently purchased items |

(Documented in HBR Case Study TB0756)

#### Keelvar — Autonomous Sourcing

- Machine-to-machine protocols (not conversational)
- Explores thousands of award scenarios with combinatorial optimization
- "Elasticity thresholds" and "circuit breakers" for human escalation
- Dynamic demand pooling ("Implicit Consortia") across organizations

#### Fairmarkit — Boeing Tail Spend

- 10x more sourcing events per FTE
- ~$40k savings per buyer per week
- 90%+ bids below benchmark pricing
- 3 minutes per sourcing event
- Serves Amazon, BP, Goodyear, Nestle in 70+ countries

#### eMoldino — Manufacturing Procurement

- Early payment discount optimization: 15% savings
- AI-based price comparisons: 20% reduction in overcharging
- Predictive risk scoring: 5% premium reduction
- **Total: 40% cost savings**

#### McKinsey Maturity Model (2025)

| Stage | Description | AI Role |
|-------|-------------|---------|
| **Assisted** | AI as copilot; flags opportunities | Human decides everything |
| **Semi-Autonomous** | AI adjusts within preset limits | Human controls critical decisions |
| **Fully Autonomous** | AI manages within guardrails | Human oversight by exception |

Estimated **25-40% efficiency gain** through AI transformation.

### 3.12 Accordo vs Industry Comparison

| Dimension | Industry Best Practice | Accordo Current State | Gap |
|-----------|----------------------|----------------------|-----|
| **Utility Model** | Additive MAUT (5-15 params) | Implemented (12+ params) | None |
| **BATNA Expression** | Converted to reservation utility | Stored as raw price, indirect influence | BATNA not directly converted to utility threshold |
| **Decision Logic** | Accept/Counter/Walk Away via utility vs dynamic threshold | Implemented with fixed thresholds per priority | Thresholds are per-priority but not time-dependent |
| **Concession Strategy** | Time-dependent (Boulware/Linear/Conceder) + behavior-dependent | Priority-based aggressiveness + vendor emphasis detection | No formal concession decay function |
| **Pareto Seeking** | MESOs, trade-off strategies | MESO system exists | Implemented |
| **Dynamic BATNA** | Updated each round based on market data | Static (set once on requisition) | BATNA doesn't adapt during negotiation |
| **Maturity Stage** | Semi-autonomous → Fully autonomous | Semi-autonomous (INSIGHTS mode) | On track |
| **Explainability** | Audit trail, parameter breakdown | Full explainability with utility breakdown | Strong |
| **Behavioral Analysis** | Opponent modeling, emphasis detection | Vendor emphasis detection, rigidity tracking | Implemented |

---

## Sources

### Procurement Platforms
- [SAP Ariba Supplier Evaluation Weighting and Scoring](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/af9ef57f504840d2b81be8667206d485/b68ae77f93a846f192d3ef53cf2bf060.html)
- [SAP Ariba Sourcing Total Cost Definition](https://learning.sap.com/learning-journeys/introducing-projects-within-sap-ariba-sourcing/identifying-the-key-attributes-of-total-cost-events_c5887cf6-b7c1-4f5c-ad36-fc502ba89362)
- [Keelvar Autonomous Negotiation Agents](https://www.keelvar.com/knowledge-hub/autonomous-negotiation-agents-the-end-of-the-chatbot-era-in-sourcing)
- [Keelvar Sourcing Optimization Software](https://www.keelvar.com/sourcing-optimization-software)
- [GEP Autonomous Negotiation Agents for Savings](https://www.gep.com/blog/technology/autonomous-negotiation-agents)
- [Jaggaer Agentic AI in Sourcing and Supplier Management](https://www.jaggaer.com/blog/agentic-ai-sourcing-supplier-management)
- [Fairmarkit AI Agents for Autonomous Sourcing](https://www.fairmarkit.com/)
- [Fairmarkit 2025 Gartner Hype Cycle Recognition](https://www.businesswire.com/news/home/20250723006360/en/Fairmarkit-Recognized-in-Three-Categories-of-the-2025-Gartner-Hype-Cycle)

### Pactum AI
- [Pactum Platform Components](https://pactum.com/the-required-components-of-autonomous-negotiations-platform-pactum-platform/)
- [Pactum Understanding Autonomous Negotiations](https://pactum.com/understanding-autonomous-negotiations/)
- [Pactum Understanding Agentic AI in Procurement](https://pactum.com/understanding-agentic-ai-in-procurement-how-autonomous-ai-has-been-transforming-supplier-deals/)
- [HBR Case Study: Pactum at Walmart and Maersk (TB0756)](https://store.hbr.org/product/pactum-s-ai-in-contract-negotiations-walmart-and-maersk/TB0756)
- [Fortune: Pactum Helping Walmart and Maersk Negotiate](https://fortune.com/2021/04/27/a-i-startup-pactum-negotiating-software-saving-walmart-money/)

### BATNA and Negotiation Theory
- [Harvard PON: How to Find Your BATNA](https://www.pon.harvard.edu/daily/batna/translate-your-batna-to-the-current-deal/)
- [KARRASS: Complete Guide to BATNA, ZOPA, and Reservation Point](https://www.karrass.com/blog/batna)
- [KARRASS: Reservation Price vs BATNA](https://www.karrass.com/blog/reservation-price-vs-batna)
- [Aligned Negotiation: Mastering the Bargaining Range (ZOPA)](https://www.alignednegotiation.com/zopa-bargaining-range)
- [Harvard Business School: Understanding ZOPA](https://online.hbs.edu/blog/post/understanding-zopa)
- [Corporate Finance Institute: BATNA Definition](https://corporatefinanceinstitute.com/resources/valuation/what-is-batna/)

### Game Theory and Academic Research
- [Nash Bargaining Solution — Game Theory](https://www.vaia.com/en-us/explanations/business-studies/managerial-economics/nash-bargaining/)
- [Rubinstein Bargaining Model — Wikipedia](https://en.wikipedia.org/wiki/Rubinstein_bargaining_model)
- [MIT OCW: Nash Bargaining Solution](https://ocw.mit.edu/courses/6-254-game-theory-with-engineering-applications-spring-2010/)
- [MIA-RVelous: Optimal Concessions with Reservation Value (arXiv 2024)](https://arxiv.org/html/2404.19361v1)
- [Introduction to Automated Negotiation (arXiv 2025)](https://arxiv.org/pdf/2511.08659)
- [Concession-Based Classification of Negotiation Strategies](https://homepages.cwi.nl/~baarslag/pub/Towards_a_quantitative_concession-based_classification_method_of_negotiation_strategies.pdf)
- [Automated Configuration of Negotiation Strategies (arXiv 2020)](https://arxiv.org/pdf/2004.00094)

### ANAC Competition
- [ANAC 2025 — 16th Competition](https://web.tuat.ac.jp/~katfuji/ANAC2025/)
- [ANAC at AAMAS 2025](https://www.ifaamas.org/Proceedings/aamas2025/pdfs/p3000.pdf)
- [Large-Scale AI Negotiation Competition (arXiv 2025)](https://arxiv.org/pdf/2503.06416)
- [ANAC at TU Delft](https://ii.tudelft.nl/nego/node/7)

### MAUT and Scoring
- [MAUT in Practice — D-Sight](https://www.d-sight.com/utility-maut)
- [Multi-Attribute Utility Theory — ResearchGate](https://www.researchgate.net/publication/278655188_Multi-Attribute_Utility_Theory)
- [Supplier Selection Using Weighted Utility Additive Method](https://www.researchgate.net/publication/269664551_Supplier_Selection_Using_Weighted_Utility_Additive_Method)
- [NC State: Supplier Evaluation Metrics](https://scm.ncsu.edu/scm-articles/article/performance-measurements-and-metrics-an-analysis-of-supplier-evaluation)

### Industry Analysis
- [MIT CTL: How AI Is Reshaping Supplier Negotiations](https://ctl.mit.edu/news/how-ai-reshaping-supplier-negotiations)
- [McKinsey: Redefining Procurement in the Era of Agentic AI](https://www.mckinsey.com/capabilities/operations/our-insights/redefining-procurement-performance-in-the-era-of-agentic-ai)
- [eMoldino: AI-Powered Supplier Negotiation 40% Cost Savings](https://www.emoldino.com/how-ai-powered-supplier-negotiation-increased-cost-savings-by-40-real-case-study/)
- [Boeing and Fairmarkit — Supply Chain Management Review](https://www.scmr.com/article/boeing-artificial-intelligence-procurement-pilot)
