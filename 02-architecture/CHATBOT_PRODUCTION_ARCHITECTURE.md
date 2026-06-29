# Accordo-AI: Production Chatbot System Architecture

> **Audience**: Technical investors / CTO-level
> **Scope**: Production-deployed features on `main` branch
> **Last Updated**: March 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [Negotiation Modes](#3-negotiation-modes)
4. [End-to-End Chat Flow](#4-end-to-end-chat-flow)
5. [Decision Engine](#5-decision-engine)
6. [Utility Scoring System](#6-utility-scoring-system)
7. [Offer Parsing & Accumulation](#7-offer-parsing--accumulation)
8. [LLM Integration & Safety Boundary](#8-llm-integration--safety-boundary)
9. [MESO: Multiple Equivalent Simultaneous Offers](#9-meso-multiple-equivalent-simultaneous-offers)
10. [Behavioral Intelligence](#10-behavioral-intelligence)
11. [Vendor Portal (Public)](#11-vendor-portal-public)
12. [PM Interface (NegotiationRoom)](#12-pm-interface-negotiationroom)
13. [Two-Phase Messaging Architecture](#13-two-phase-messaging-architecture)
14. [Frontend Architecture](#14-frontend-architecture)
15. [API Layer & Route Structure](#15-api-layer--route-structure)
16. [Data Model](#16-data-model)
17. [Authentication & Security](#17-authentication--security)
18. [Cross-Deal Learning & Vendor Profiles](#18-cross-deal-learning--vendor-profiles)
19. [Error Recovery & Fallbacks](#19-error-recovery--fallbacks)
20. [Performance & Infrastructure](#20-performance--infrastructure)
21. [Tech Stack](#21-tech-stack)

---

## 1. Executive Summary

Accordo-AI is an **AI-powered procurement negotiation platform** that autonomously negotiates with vendors on behalf of procurement managers (PMs). The system combines a **deterministic decision engine** with **LLM-powered natural language rendering** to conduct human-like negotiations while maintaining strict business rule enforcement.

### What Makes It Unique

- **Deterministic decisions, human-like delivery**: The AI never improvises on business logic — all accept/counter/escalate/walk-away decisions are mathematically computed. The LLM only renders the message in natural language.
- **Multi-parameter utility scoring**: Price, payment terms, delivery, warranty, quality, and custom parameters — each with configurable weights summing to 100%.
- **MESO (Multiple Equivalent Simultaneous Offers)**: After round 5, the system presents vendors with 3 equally-valued but differently-structured offers, enabling preference learning.
- **Two-phase messaging**: Vendor messages appear instantly (~100ms); PM responses generate asynchronously (5–15s) with typing indicators — mimicking real human conversation pace.
- **Vendor portal**: Public, unauthenticated negotiation interface via unique token links — vendors never see PM targets, thresholds, or utility scores.
- **Cross-deal learning**: The system learns from past negotiations with the same vendor, adjusting strategy based on historical patterns.

### Key Metrics

| Metric | Value |
|--------|-------|
| Backend codebase | ~30,000+ lines TypeScript |
| Frontend codebase | ~15,000+ lines React/TypeScript |
| Decision engine | ~1,110 lines (core `decide.ts`) |
| Weighted utility system | ~34KB multi-parameter scoring |
| MESO generator | ~1,478 lines |
| Behavioral analyzer | ~18KB real-time tracking |
| API endpoints | 63+ routes |
| Database models | 32 Sequelize models |
| Supported currencies | USD, INR, EUR, GBP, AUD |

---

## 2. System Architecture Overview

```
+=====================================================================+
|                        ACCORDO-AI PLATFORM                          |
+=====================================================================+
|                                                                     |
|   +---------------------+              +----------------------+    |
|   | FRONTEND (React 19) |              |  VENDOR PORTAL       |    |
|   | Port 5001            |              |  (Public, no auth)   |    |
|   | Vite + TypeScript   |              |  /vendor-chat/:token |    |
|   +----------+----------+              +----------+-----------+    |
|              |                                    |                 |
|              | Authenticated (JWT)                | Unauthenticated |
|              |                                    | (uniqueToken)   |
|              +----------------+-------------------+                 |
|                               |                                     |
|                               v                                     |
|   +----------------------------------------------------------+    |
|   |              BACKEND API (Express/TypeScript)             |    |
|   |              Port 5002                                     |    |
|   +----------------------------------------------------------+    |
|   |                                                            |    |
|   |   +--------------------------------------------------+   |    |
|   |   |        CHATBOT SERVICE (Orchestrator)             |   |    |
|   |   |        chatbot.service.ts (~5,000+ lines)         |   |    |
|   |   +--------------------------------------------------+   |    |
|   |        |                              |                   |    |
|   |        v                              v                   |    |
|   |   +-----------+            +------------------+          |    |
|   |   | INSIGHTS  |            |  CONVERSATION    |          |    |
|   |   |   MODE    |            |     MODE         |          |    |
|   |   +-----------+            +------------------+          |    |
|   |        |                          |                       |    |
|   |        +------------+-------------+                       |    |
|   |                     |                                     |    |
|   |                     v                                     |    |
|   |   +--------------------------------------------------+   |    |
|   |   |           DECISION ENGINE                         |   |    |
|   |   |   parseOffer -> decide -> buildIntent             |   |    |
|   |   +--------------------------------------------------+   |    |
|   |                     |                                     |    |
|   |                     v                                     |    |
|   |   +--------------------------------------------------+   |    |
|   |   |          LLM RENDERING LAYER                      |   |    |
|   |   |   personaRenderer -> validateOutput -> fallback   |   |    |
|   |   +--------------------------------------------------+   |    |
|   |                     |                                     |    |
|   |                     v                                     |    |
|   |   +--------------------------------------------------+   |    |
|   |   |          DELIVERY LAYER                           |   |    |
|   |   |   simulateTypingDelay -> logNegotiationStep       |   |    |
|   |   +--------------------------------------------------+   |    |
|   |                     |                                     |    |
|   |                     v                                     |    |
|   |   +--------------------------------------------------+   |    |
|   |   |       PostgreSQL / Sequelize ORM                  |   |    |
|   |   |       32 models, full audit trail                 |   |    |
|   |   +--------------------------------------------------+   |    |
|   |                                                            |    |
|   +------------------------------------------------------------+    |
|                                                                     |
+=====================================================================+
```

---

## 3. Negotiation Modes

The platform supports two distinct negotiation modes, both sharing the same decision engine:

### INSIGHTS Mode (Deterministic)

```
Vendor Message
    |
    v
parseOffer()         -- Extract price, terms, delivery from text
    |
    v
decide()             -- Deterministic decision (action + firmness)
    |
    v
responseGenerator()  -- Template-based PM response (no LLM)
    |
    v
PM Response
```

- **No LLM involvement** — responses are generated from templates
- **Full explainability** — every decision has transparent reasoning
- **Used for**: High-volume automated negotiations, testing, deterministic workflows

### CONVERSATION Mode (LLM-Enhanced)

```
Vendor Message
    |
    v
parseOffer()                -- Extract price, terms, delivery
    |
    v
decide()                    -- Deterministic decision (same engine)
    |
    v
buildNegotiationIntent()    -- Map Decision -> NegotiationIntent
    |                          (HARD BOUNDARY: strips all internals)
    v
personaRenderer()           -- LLM generates natural language
    |                          (temp 0.5, static system prompt)
    v
validateLlmOutput()         -- Reject banned words, wrong prices, >160 words
    |
    +--[PASS]--> simulateTypingDelay() --> PM Response
    |
    +--[FAIL]--> getFallbackResponse() --> PM Response (silent, vendor unaware)
```

- **Same decisions** as INSIGHTS — LLM only renders the message
- **Human-like conversation** — natural tone, contextual acknowledgment
- **Silent fallback** — if LLM fails validation, a humanized template is used instead

---

## 4. End-to-End Chat Flow

### Complete Negotiation Lifecycle

```
DEAL CREATION                      NEGOTIATION                           RESOLUTION
==============                     ===========                           ==========

PM creates deal                    Vendor sends message                  Decision engine
via Deal Wizard                        |                                 determines outcome
     |                                 v                                      |
     v                            [Phase 1: Instant Save]                     v
Set parameters:                   Save vendor msg to DB                  +----------+
- Target price                    Return to frontend (~100ms)            | ACCEPT   |---> Deal closed
- Max acceptable price                 |                                 +----------+    (agreed price)
- Anchor price                        v                                       |
- Payment terms                   [Phase 2: Async PM Response]           +----------+
- Delivery date                   Parse offer -> Decide -> Render        | COUNTER  |---> Next round
- Warranty                        Generate PM message (5-15s)            +----------+    (new offer)
- Custom params                   Validate & deliver                          |
- Parameter weights                    |                                 +----------+
     |                                 v                                 | ESCALATE |---> PM notified
     v                            Frontend shows:                        +----------+    (human review)
Generate vendor link              1. Vendor msg (instant)                     |
(uniqueToken URL)                 2. Typing indicator                    +----------+
     |                            3. PM response (after delay)           | WALK_AWAY|---> Deal ended
     v                            4. Updated utility/behavior            +----------+    (no agreement)
Share with vendor                      |
                                       v
                                  Round increments
                                  Repeat until terminal
```

### Round-by-Round Progression

```
Round 1-2: Opening Phase
  - Vendor submits initial offer
  - PM anchors with counter-offer
  - Behavioral baseline established

Round 3-4: Preference Exploration
  - Vendor concession patterns analyzed
  - Emphasis detected (price-focused vs terms-focused)
  - Strategy adapts to vendor behavior

Round 5+: MESO Phase (if not converging)
  - 3 equivalent offers presented to vendor
  - Vendor preference inferred from selection
  - "Others" option allows custom counter

Round 6-8: Convergence or Escalation
  - Stall detection triggers firmness increase
  - Dynamic round extension if converging
  - Escalation if stalled beyond threshold

Terminal: Accept, Walk Away, or Escalate
  - ACCEPT: Utility >= accept threshold (~70-85%)
  - WALK_AWAY: Utility < walkaway threshold (~30%)
  - ESCALATE: Stuck between thresholds after max rounds
```

---

## 5. Decision Engine

The decision engine (`decide.ts`, ~1,110 lines) is the core of the negotiation logic. It is **fully deterministic** — given the same inputs, it always produces the same output.

### Decision Flow

```
                        VENDOR OFFER (parsed)
                              |
                              v
                    +-------------------+
                    | Calculate Utility |
                    | (weighted multi-  |
                    |  parameter score) |
                    +-------------------+
                              |
                    Utility Score (0.0 - 1.0)
                              |
              +---------------+----------------+
              |               |                |
              v               v                v
        Utility >= 70%   50% <= U < 70%   30% <= U < 50%    U < 30%
              |               |                |                |
              v               v                v                v
         +---------+    +---------+      +-----------+    +----------+
         | ACCEPT  |    | COUNTER |      | ESCALATE  |    | WALK_AWAY|
         +---------+    +---------+      +-----------+    +----------+
                              |
                              v
                    +-------------------+
                    | Generate Counter  |
                    | Offer             |
                    | - Concession step |
                    | - Behavioral adj  |
                    | - Convergence adj |
                    +-------------------+
```

### Threshold Zones

```
  WALK_AWAY       ESCALATE         COUNTER           ACCEPT
  (Red)           (Orange)         (Blue)            (Green)
  |               |                |                 |
  0%------------>30%------------>50%--------------->70%----------->100%
  |               |                |                 |
  Vendor offer    Vendor offer     Vendor offer      Vendor offer
  far from        concerning,      in negotiable     meets or exceeds
  acceptable      needs human      range, counter    PM's targets
                  review           with adjusted
                                   terms
```

### Adaptive Thresholds

Thresholds are not static — they adapt based on:

| Factor | Effect |
|--------|--------|
| Round progression | Accept threshold decreases slightly each round (PM becomes more flexible) |
| Convergence rate | Fast convergence → maintain thresholds; slow → soften |
| Stall detection | 2+ rounds same offer → increase firmness, then trigger MESO |
| Vendor behavior | Cooperative vendors → more generous counters; rigid → escalate faster |
| Dynamic rounds | If converging near softMax → extend; if stalled → hard stop |

### Decision Output

```typescript
Decision {
  action: 'ACCEPT' | 'COUNTER' | 'ESCALATE' | 'WALK_AWAY' | 'MESO' | 'ASK_CLARIFY'
  firmness: 0.0 - 1.0       // How firm the response should be
  offer?: {                  // For COUNTER actions
    total_price: number      // Always total contract value
    payment_terms: string
    delivery_date: Date
  }
  reasoning?: string         // Human-readable explanation
}
```

---

## 6. Utility Scoring System

### Multi-Parameter Weighted Utility

The system evaluates vendor offers across multiple parameters, each with configurable weights that must sum to 100%.

```
UTILITY CALCULATION
===================

Total Utility = SUM(parameter_utility * parameter_weight)

Example with 5 parameters:
+---------------------+--------+---------+---------+-------+---------------+
| Parameter           | Weight | Vendor  | Target  | Util  | Contribution  |
+---------------------+--------+---------+---------+-------+---------------+
| Unit Price          |  30%   | $95     | $90     |  85%  |    25.5%      |
| Payment Terms       |  25%   | Net 30  | Net 60  |  60%  |    15.0%      |
| Delivery Date       |  20%   | Apr 1   | Mar 15  |  70%  |    14.0%      |
| Warranty            |  15%   | 1 Year  | 1 Year  | 100%  |    15.0%      |
| Late Delivery Pen.  |  10%   | 0.5%/d  | 0.5%/d  | 100%  |    10.0%      |
+---------------------+--------+---------+---------+-------+---------------+
                                           Total Utility:      79.5%
                                           Decision Zone:       ACCEPT
```

### Parameter Utility Types

| Type | Calculation | Example |
|------|-------------|---------|
| `linear` | Linear interpolation between min/max | Unit price: $80–$120 range |
| `stepped` | Discrete utility per bracket | Payment terms: Net 15/30/45/60/90 |
| `date` | Days from target date | Delivery: closer = higher utility |
| `percentage` | Percentage match to target | Late penalty: 0.5% target |
| `boolean` | Binary 0 or 1 | Insurance included: yes/no |
| `binary` | Two-state with defined values | Warranty: standard vs extended |

### Parameter Direction

Each parameter has a direction indicating what's better:

- `lower_better` — Price (lower = more favorable)
- `higher_better` — Warranty duration (longer = better)
- `match_target` — Exact match preferred (penalty %)
- `closer_better` — Delivery date (closer to target = better)

### Price Model (Critical)

```
ALL PRICES ARE TOTAL CONTRACT VALUES
=====================================

Config Builder:
  targetUnitPrice ($90) x minOrderQuantity (100 units) = target ($9,000)
  maxUnitPrice ($120)   x minOrderQuantity (100 units) = max    ($12,000)
  anchorUnitPrice ($80) x minOrderQuantity (100 units) = anchor ($8,000)

Vendor always bids total contract price:
  "I can offer $9,500 for the full order"
  → total_price = $9,500 (stored directly)

Counter-offers are also total:
  PM counters with $9,200 → total contract value
```

---

## 7. Offer Parsing & Accumulation

### Multi-Format Offer Extraction (`parseOffer.ts`, ~597 lines)

The parser extracts structured offers from free-text vendor messages:

```
VENDOR MESSAGE                          PARSED OFFER
===============                         ============

"I can offer $9,500 for               total_price: 9500
 the full order with Net 30     -->    payment_terms: "Net 30"
 payment terms, delivery by            delivery_date: 2025-04-01
 April 1st 2025"
```

**Supported formats:**

| Format | Example | Parsed As |
|--------|---------|-----------|
| USD | $9,500 / USD 9500 / 9,500 dollars | 9500 |
| INR | Rs. 75,000 / INR 75000 / 75,000 rupees | 75000 |
| EUR | EUR 8,200 / 8.200 euros | 8200 |
| GBP | GBP 7,100 / £7,100 | 7100 |
| AUD | AUD 12,400 / A$12,400 | 12400 |
| Indian lakhs | 2.5 lakhs / 2,50,000 | 250000 |
| Indian crores | 1.2 crores / 1,20,00,000 | 12000000 |
| Payment terms | Net 30 / 30 days / net-30 / N30 | "Net 30" |
| Delivery dates | April 1 / 2025-04-01 / 01/04/2025 | Date object |
| Delivery days | "within 15 days" / "2 weeks" | 15 days |

### Offer Accumulation (Multi-Message Offers)

Vendors sometimes spread offers across multiple messages:

```
Message 1: "I can do $9,500"         --> Partial: { total_price: 9500 }
Message 2: "With Net 30 terms"       --> Accumulated: { total_price: 9500,
Message 3: "Delivery in 3 weeks"                       payment_terms: "Net 30",
                                                        delivery_days: 21 }
```

The `offerAccumulator.ts` merges partial offers:
- Tracks which components (price, terms, delivery) have been provided
- Merges new components into existing partial offer
- Resets accumulation when a complete new offer is detected
- `isOfferComplete()` checks if all required components present

---

## 8. LLM Integration & Safety Boundary

### The Hard Boundary: `buildNegotiationIntent.ts`

This is the critical security boundary between the deterministic decision engine and the LLM:

```
DECISION ENGINE OUTPUT              INTENT (What LLM receives)
======================              ==========================

  action: COUNTER                     action: COUNTER
  firmness: 0.7                       firmness: 0.7
  utility: 0.62           STRIPPED    allowedPrice: $9,200
  weights: {price: 30%}   -------->   (No utility scores)
  thresholds: {...}        STRIPPED    (No weights)
  targetPrice: $9,000      STRIPPED    (No thresholds)
  maxPrice: $12,000        STRIPPED    (No target/max prices)
  config: {...}            STRIPPED    (No config details)
```

**What the LLM receives:**

| Field | Description |
|-------|-------------|
| `action` | What to do (COUNTER, ACCEPT, etc.) |
| `firmness` | How firm (0.0 = flexible, 1.0 = rigid) |
| `allowedPrice` | Only for COUNTER — always within [target, max] |
| `dealTitle` | Title of the deal |
| `vendorName` | Vendor company name |
| `productCategory` | Product category |
| `vendorMessage` | The vendor's latest message |
| `vendorTone` | Detected tone (formal/casual/urgent/firm/friendly) |

**What the LLM NEVER sees:**

- Utility scores, parameter weights, or threshold values
- Target price, maximum acceptable price, or anchor price
- Negotiation configuration or wizard settings
- Behavioral analysis data (momentum, convergence, stalling)
- Internal reasoning or strategy signals

### LLM Rendering (`personaRenderer.ts`, ~200 lines)

```
Single LLM Entry Point
=======================

personaRenderer.renderNegotiationMessage(context)
  |
  +-- Static system prompt (same for all negotiations)
  +-- Temperature: 0.5 (consistent, semi-creative)
  +-- Input: NegotiationIntent + vendor message + deal context
  +-- Output: Natural language PM response
  |
  v
validateLlmOutput(response, intent)
  |
  +-- Check banned keywords (utility, threshold, score, etc.)
  +-- Verify price within allowed range (for COUNTER)
  +-- Enforce max word count (~160 words)
  |
  +--[PASS]--> Return LLM response
  +--[FAIL]--> Return fallback template (silent, vendor unaware)
```

### Fallback Templates (`fallbackTemplates.ts`, ~600+ lines)

When the LLM fails validation or times out, the system uses pre-written humanized templates:

- **5+ variations per action type** (ACCEPT, COUNTER, ESCALATE, WALK_AWAY, MESO, ASK_CLARIFY)
- **Tone-aware** — formal, casual, urgent, firm, friendly variants
- **Random selection** — avoids repetitive responses
- **Silent substitution** — vendor never knows LLM was skipped

---

## 9. MESO: Multiple Equivalent Simultaneous Offers

MESO is a negotiation technique that presents vendors with multiple offers of **equal total utility** but **different trade-off profiles**, enabling preference learning.

### When MESO Triggers

```
Round >= 5 AND (
  utility >= 70%        -- Close to agreement
  OR stallDetected      -- Same offers 2+ rounds
  OR converging slowly  -- Gap shrinking but slowly
)
```

### How MESO Options Are Generated

```
MESO GENERATION (meso.ts, ~1,478 lines)
========================================

Given: Total target utility = 75%

Option 1: "Best Price"              Option 2: "Balanced"              Option 3: "Best Terms"
+---------------------------+      +---------------------------+     +---------------------------+
| Price:    $9,000 (best)   |      | Price:    $9,300          |     | Price:    $9,600 (highest)|
| Payment:  Net 15 (short)  |      | Payment:  Net 30          |     | Payment:  Net 45 (longest)|
| Delivery: 30 days         |      | Delivery: 25 days         |     | Delivery: 20 days (fast) |
| Utility:  ~75%            |      | Utility:  ~75%            |     | Utility:  ~75%           |
+---------------------------+      +---------------------------+     +---------------------------+

All three options have approximately equal utility (~75%)
but with different emphasis on price vs. terms
```

### MESO Flow on Vendor Portal

```
Vendor receives 3 MESO option cards
         |
         +--[Select Option 1/2/3]--> Deal ACCEPTED at selected terms
         |
         +--[Click "Others"]--> OthersForm appears
                                    |
                                    v
                              Vendor enters:
                              - Custom total price
                              - Custom payment terms (1-180 days)
                                    |
                                    v
                              Backend evaluates "Others" offer
                                    |
                         +----------+----------+
                         |                     |
                    Acceptable?           Not acceptable?
                         |                     |
                         v                     v
                    New MESO options       Continue text
                    (4 more rounds)        negotiation
                         |                     |
                         v                     v
                    If still no            After 4 rounds:
                    agreement:             Final MESO
                    Final MESO             (no "Others")
                    (forces choice)        Must select one
```

### Preference Learning

When a vendor selects a MESO option, the system infers their priorities:

| Selection | Inference |
|-----------|-----------|
| Option 1 (Best Price) | Vendor is price-sensitive → future counters emphasize price |
| Option 2 (Balanced) | Vendor wants fair terms → maintain balanced approach |
| Option 3 (Best Terms) | Vendor values terms → offer better terms in exchange for price |

---

## 10. Behavioral Intelligence

The system continuously analyzes vendor behavior across multiple dimensions:

### Behavioral Analyzer (`behavioralAnalyzer.ts`, ~18KB)

```
BEHAVIORAL SIGNALS (tracked per round)
======================================

Concession Velocity
  How quickly is the vendor moving toward PM's position?
  +----+----+----+----+----+----+
  | R1 | R2 | R3 | R4 | R5 | R6 |
  +----+----+----+----+----+----+
  |$12K|$11K|$10K|$9.8K|$9.6K|$9.5K|  <-- Decreasing velocity (slowing)
  +----+----+----+----+----+----+

Convergence Rate
  Is the gap between vendor and PM offers shrinking?
  PM:     $9.0K --- $9.1K --- $9.2K --- $9.3K
  Vendor: $12K  --- $11K  --- $10K  --- $9.6K
  Gap:    $3.0K --- $1.9K --- $0.8K --- $0.3K  <-- Converging

Momentum (-1 to +1)
  Composite score: Are we winning or losing ground?
  -1.0 = Losing ground (vendor getting firmer)
   0.0 = Neutral
  +1.0 = Winning ground (vendor conceding rapidly)

Strategy Classification
  Based on momentum + convergence:
  - "Holding Firm"     -- Maintain current position
  - "Accelerating"     -- Push harder, vendor is yielding
  - "Matching Pace"    -- Mirror vendor's concession rate
  - "Final Push"       -- Near agreement, make final offer
```

### Stall Detection (`stallDetector.ts`)

```
STALL DETECTION
===============

Stall = Vendor repeats same offer for 2+ consecutive rounds

Round 4: Vendor offers $10,000
Round 5: Vendor offers $10,000    <-- STALL DETECTED (identical offer)
Round 6: Vendor offers $9,950     <-- Stall cleared (new offer)

On stall:
  1. Increase firmness by 15%
  2. Trigger "Is this your final offer?" question
  3. If still stalled after question: Trigger MESO
  4. If stalled through MESO: Escalate or Walk Away
```

### Preference Detection (`preferenceDetector.ts`, ~12KB)

```
VENDOR EMPHASIS TRACKING
========================

Track what vendor prioritizes in messages:

"I really need Net 60 payment terms"     --> terms-focused
"Can we discuss a lower price?"          --> price-focused
"Delivery by March is critical for us"   --> delivery-focused
"I'm flexible on everything except..."   --> detect exception

Emphasis affects:
  - Which MESO option emphasizes what
  - Counter-offer structure
  - Response acknowledgment
```

### Tone Detection (`toneDetector.ts`, ~11KB)

```
VENDOR TONE CLASSIFICATION
==========================

5 Tone Categories:
  formal    -- "We would like to propose..."
  casual    -- "Hey, how about..."
  urgent    -- "We need this resolved ASAP..."
  firm      -- "This is our best offer, take it or leave it"
  friendly  -- "We appreciate your partnership..."

Signal Detection:
  urgency:        Time pressure phrases, deadlines
  frustration:    Negative sentiment, impatience
  assertiveness:  Firm positions, ultimatums
  cooperativeness: Willingness to compromise

Use: Metadata only (LLM tone matching via NegotiationIntent)
```

---

## 11. Vendor Portal (Public)

The vendor portal is a **public, unauthenticated** negotiation interface accessible via unique token URL.

### Security Model

```
VENDOR PORTAL SECURITY
======================

URL: /vendor-chat/{uniqueToken}

What vendor CAN see:
  + Their own messages
  + PM (AI) responses
  + MESO option cards
  + Deal title and status
  + Current round number

What vendor CANNOT see:
  - PM's target price
  - Maximum acceptable price
  - Utility scores
  - Decision engine reasoning
  - Parameter weights
  - Threshold values
  - Behavioral analysis
  - Negotiation configuration
```

### Vendor Portal State Machine

```
                    +------------------+
                    | VENDOR ENTERS    |
                    | (via unique URL) |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    | TEXT INPUT MODE  |
                    | (Composer)       |
                    +--------+---------+
                             |
                    Vendor sends message
                             |
                    +--------v---------+
                    | TYPING INDICATOR |
                    | "PM is typing..." |
                    +--------+---------+
                             |
                    PM response received
                             |
              +--------------+--------------+
              |                             |
         Has MESO?                    No MESO
              |                             |
              v                             v
    +------------------+         +------------------+
    | DISABLED INPUT   |         | TEXT INPUT MODE  |
    | Show MESO cards  |         | (continue)       |
    +--------+---------+         +------------------+
              |
    +---------+---------+
    |                   |
  Select              Others
  Option 1/2/3        Button
    |                   |
    v                   v
  ACCEPTED       +------------------+
                 | OTHERS FORM     |
                 | Price + Terms   |
                 +--------+--------+
                          |
               Submit custom offer
                          |
              +-----------+-----------+
              |                       |
         New MESO?              Back to text
              |
              v
        DISABLED INPUT
        (new MESO cards)
```

---

## 12. PM Interface (NegotiationRoom)

The NegotiationRoom (`NegotiationRoom.tsx`, ~1,737 lines) is the main PM-facing interface for INSIGHTS mode.

### Layout Architecture

```
+------------------------------------------------------------------+
|  STICKY HEADER                                                    |
|  [<- Back]  Deal Title | Status Badge | Round X/Y  [Refresh][v]  |
+------------------------------------------------------------------+
|                                                                    |
|  +-------------------------------------+  +---------------------+ |
|  |  MAIN CHAT AREA (flex: 1)           |  |  SIDEBAR (w-80)     | |
|  |                                     |  |                     | |
|  |  [Chat] [Summary]  <-- Tab nav      |  |  Utility Bar        | |
|  |                                     |  |  +--------------+   | |
|  |  ---- Round 1 · Holding Firm ----   |  |  | 0%    79%     |   | |
|  |                                     |  |  |  |----*-----|   | |
|  |  [Vendor]: "I offer $10,000..."     |  |  +--------------+   | |
|  |                                     |  |                     | |
|  |  [Accordo]: "Thank you for your..." |  |  AI Reasoning       | |
|  |    [COUNTER] $9,200                 |  |  Timeline           | |
|  |                                     |  |  +- Round 1: 45% -+ | |
|  |  ---- Round 2 · Accelerating ----   |  |  |  COUNTER $9.2K | | |
|  |                                     |  |  +- Round 2: 62% -+ | |
|  |  [Vendor]: "How about $9,600?"      |  |  |  COUNTER $9.3K | | |
|  |                                     |  |  +-  Click for    -+ | |
|  |  [Accordo]: "We appreciate..."      |  |  |  detail modal  | | |
|  |    [COUNTER] $9,300                 |  |  +----------------+ | |
|  |                                     |  |                     | |
|  |  ---- Round 3 · Final Push ----     |  |  Accordion Sections | |
|  |                                     |  |  [v] Price Params   | |
|  |  [Vendor]: "Final offer: $9,400"    |  |  [>] Payment Terms  | |
|  |                                     |  |  [>] Delivery Terms | |
|  |  ... AI Negotiator is typing ...    |  |  [>] Contract & SLA | |
|  |                                     |  |  [>] Custom Params  | |
|  +-------------------------------------+  +---------------------+ |
|  |  [Type your message...        ] [Send]  |                      |
|  +------------------------------------------+                      |
+------------------------------------------------------------------+
```

### Key Features

| Feature | Description |
|---------|-------------|
| **Smart Auto-Scroll** | Only scrolls when user within 100px of bottom — respects reading history |
| **Round Dividers** | Visual separators showing round number, strategy, and convergence status |
| **Decision Badges** | Color-coded: ACCEPT (green), COUNTER (blue), ESCALATE (orange), WALK_AWAY (red) |
| **AI Reasoning Timeline** | Per-round breakdown with utility score, action, and reasoning points |
| **AI Reasoning Modal** | Expandable detail view with full reasoning, previous/next navigation |
| **Utility Bar** | Gradient bar (red→green) with threshold zones and current position indicator |
| **Parameter Accordion** | Collapsible sections showing weights, current values, and utility per parameter |
| **Status Notifications** | Toast notifications when deal status changes |
| **Read-Only Mode** | Completed deals shown with chat history but no input |
| **Deal Reset** | Double-confirmation destructive reset (returns to round 0) |
| **5s Polling** | Real-time updates while deal is in NEGOTIATING status |

---

## 13. Two-Phase Messaging Architecture

The two-phase messaging system is a core UX innovation that makes AI negotiations feel natural:

### Why Two Phases?

```
PROBLEM: Traditional approach
  Vendor sends message --> [Wait 5-15 seconds] --> Both messages appear
  (User sees nothing during processing — feels broken)

SOLUTION: Two-phase approach
  Phase 1: Vendor message saved instantly (~100ms)
  Phase 2: PM response generated async (5-15s with typing indicator)
  (User sees vendor message immediately, then "AI is typing...")
```

### Phase 1: Instant Save

```
Frontend                          Backend
--------                          -------
sendVendorMessageTwoPhase()
  |
  +--> POST /vendor-message-instant
  |         |
  |         +--> Save message to DB
  |         +--> Return vendorMessage
  |
  +--> Update local messages[] (instant)
  +--> Show vendor message in ChatTranscript
  +--> Set pmTyping = true
  +--> Show "AI Negotiator is typing..."
```

### Phase 2: Async PM Response

```
Frontend                          Backend
--------                          -------
  +--> POST /pm-response-async
  |    (with vendorMessageId)
  |         |
  |         +--> parseOffer(vendorMessage)
  |         +--> decide(offer, config, state)
  |         +--> buildNegotiationIntent(decision)
  |         +--> personaRenderer(intent)
  |         |      |
  |         |      +--> LLM generates response
  |         |      +--> validateLlmOutput()
  |         |      +--> (fallback if invalid)
  |         |
  |         +--> simulateTypingDelay()
  |         |      COUNTER: 6-12s
  |         |      MESO:    8-15s
  |         |      ACCEPT:  2-5s
  |         |
  |         +--> Save PM message to DB
  |         +--> Return pmMessage + decision + deal
  |
  +--> Update messages[] with PM response
  +--> Update deal status/round
  +--> Set pmTyping = false
  +--> Hide typing indicator
```

### 5-Second Timeout Fallback

```
Frontend races two promises:

Promise 1: PM response from LLM pipeline
Promise 2: 5-second timeout

If timeout wins:
  +--> POST /pm-response-fallback
  |         |
  |         +--> Use deterministic template
  |         +--> No LLM involved
  |         +--> Return immediately
  |
  +--> Show fallback PM response
  +--> Vendor never knows LLM was skipped
```

### Server-Side Typing Delays

| Action | Delay Range | Rationale |
|--------|-------------|-----------|
| COUNTER | 6–12 seconds | Simulates deliberation on counter-offer |
| MESO | 8–15 seconds | Complex multi-option evaluation |
| ACCEPT | 2–5 seconds | Quick positive decision |
| ESCALATE | 3–8 seconds | Moderate deliberation |
| WALK_AWAY | 2–6 seconds | Decisive action |
| ASK_CLARIFY | 1–3 seconds | Simple question |

---

## 14. Frontend Architecture

### Technology Stack

| Technology | Version | Purpose |
|------------|---------|---------|
| React | 19 | UI framework |
| Vite | 5.4 | Build tool & dev server |
| TypeScript | 5.9 (strict) | Type safety |
| Tailwind CSS | 3.4 | Utility-first styling |
| React Router | v7 | Client-side routing |
| Axios | — | HTTP client with interceptors |
| react-hot-toast | — | Toast notifications |
| react-hook-form | — | Form management |

### State Management

No global state library — the frontend uses **hooks + context**:

```
HOOKS ARCHITECTURE
==================

useDealActions (INSIGHTS mode)
  +-- deal, messages, config, context
  +-- loading, sending, pmTyping
  +-- canNegotiate, canSend, canReset
  +-- sendVendorMessageTwoPhase()
  +-- reset(), reload()

useConversation (CONVERSATION mode)
  +-- deal, messages, convoState, context
  +-- loading, sending, error
  +-- currentPhase, refusalCount, warningLevel
  +-- sendMessage(), startConversation()
```

### Navigation Hierarchy

```
/chatbot/requisitions
    |
    +--> RequisitionListPage
    |    Shows: All requisitions with deal counts
    |    Click: Navigate to requisition deals
    |
    +--> /chatbot/requisitions/:rfqId
    |    RequisitionDealsPage
    |    Shows: All vendor deals for this requisition
    |    Click: Navigate to negotiation room
    |
    +--> /chatbot/requisitions/:rfqId/vendors/:vendorId/deals/:dealId
         NegotiationRoom
         Shows: Full negotiation interface with chat + sidebar
```

### API Service Layer

Two axios instances with different auth models:

```
authApi (Authenticated)                  api (Public)
  +-- Auto-adds Bearer token             +-- No authentication
  +-- Auto-refresh on 401                +-- Used for vendor portal
  +-- Request queue during refresh       +-- vendorChatService methods
  +-- chatbotService methods             +-- uniqueToken-based access
```

### Component Tree (NegotiationRoom)

```
NegotiationRoom
  +-- Header (sticky)
  |     +-- BackButton
  |     +-- DealTitle + StatusBadge + RoundCounter
  |     +-- RefreshButton + DropdownMenu (Reset/Export)
  |
  +-- MainContent (flex row)
        +-- ChatColumn (flex: 1)
        |     +-- TabNavigation (Chat / Summary)
        |     +-- ChatTranscript
        |     |     +-- RoundDivider (per round)
        |     |     +-- MessageBubble (per message)
        |     |     |     +-- DecisionBadge
        |     |     |     +-- OfferCard
        |     |     +-- TypingIndicator
        |     +-- StatusBanner
        |     +-- Composer
        |
        +-- Sidebar (w-80, overflow-y-auto)
              +-- UnifiedUtilityBar
              +-- AiReasoningTimeline
              |     +-- RoundCard (clickable)
              |     +-- AiReasoningModal (on click)
              +-- CollapsibleSection: Price Parameters
              +-- CollapsibleSection: Payment Terms
              +-- CollapsibleSection: Delivery Terms
              +-- CollapsibleSection: Contract & SLA
              +-- CollapsibleSection: Custom Parameters
```

---

## 15. API Layer & Route Structure

### Route Organization

The backend uses **nested URL patterns** that encode the full resource hierarchy:

```
BASE: /api/chatbot

Requisition-scoped routes:
  GET    /requisitions                                    -- List all requisitions
  GET    /requisitions/:rfqId/deals                       -- Deals for requisition
  GET    /requisitions/:rfqId/vendors/:vendorId/deals/:dealId  -- Get deal
  POST   /requisitions/:rfqId/vendors/:vendorId/deals/:dealId/messages  -- Send message
  POST   .../deals/:dealId/vendor-message-instant         -- Phase 1: instant save
  POST   .../deals/:dealId/pm-response-async              -- Phase 2: LLM response
  POST   .../deals/:dealId/pm-response-fallback           -- Phase 2: fallback
  GET    .../deals/:dealId/config                         -- Get deal configuration
  GET    .../deals/:dealId/utility                        -- Get utility breakdown
  GET    .../deals/:dealId/behavioral                     -- Get behavioral analysis
  POST   .../deals/:dealId/reset                          -- Reset deal to round 0
  GET    .../deals/:dealId/summary                        -- Deal summary
  GET    /deals/:dealId/lookup                            -- Lookup deal by ID (shortcut)

Vendor Chat (public):
  GET    /api/vendor-chat/deal                            -- Get deal (vendor view)
  POST   /api/vendor-chat/message                         -- Send vendor message
  POST   /api/vendor-chat/pm-response                     -- Get PM response
  POST   /api/vendor-chat/meso/select                     -- Select MESO option
  POST   /api/vendor-chat/meso/others                     -- Submit custom offer
  POST   /api/vendor-chat/enter                           -- Enter chat from quote
```

### Controller Methods (`chatbot.controller.ts`, ~1,502 lines)

48 controller methods handling:
- Deal CRUD operations
- Message processing (both modes)
- Two-phase messaging endpoints
- Utility and behavioral data endpoints
- MESO operations
- Deal wizard configuration
- Export (PDF/CSV)
- Demo scenarios

---

## 16. Data Model

### Core Entities

```
+------------------+       +-------------------+       +------------------+
|   Requisition    |       |   ChatbotDeal     |       | ChatbotMessage   |
+------------------+       +-------------------+       +------------------+
| id               |<------| requisitionId     |<------| dealId           |
| title            |       | vendorCompanyId   |       | messageRole      |
| currency         |       | dealStatus        |       |   VENDOR/ACCORDO |
| status           |       |   NEGOTIATING     |       |   /SYSTEM        |
| approvalStatus   |       |   ACCEPTED        |       | content          |
| approvalLevel    |       |   WALKED_AWAY     |       | extractedOffer   |
+------------------+       |   ESCALATED       |       | engineDecision   |
                           | dealMode          |       | round            |
+------------------+       |   INSIGHTS        |       | createdAt        |
| VendorCompany    |       |   CONVERSATION    |       +------------------+
+------------------+       | round             |
| id               |<------| latestUtility     |       +------------------+
| name             |       | convoStateJson    |       | MesoRound        |
| uniqueToken      |       | wizardConfig      |       +------------------+
+------------------+       +-------------------+       | dealId           |
                                                        | options (JSON)   |
+------------------+       +-------------------+       | selectedOption   |
| User             |       | NegotiationRound  |       | othersSubmitted  |
+------------------+       +-------------------+       +------------------+
| id               |       | dealId            |
| email            |       | roundNumber       |       +------------------+
| userType         |       | vendorOffer       |       | VendorNeg.       |
|   admin          |       | pmCounter         |       |   Profile        |
|   customer       |       | utility           |       +------------------+
|   vendor         |       | decision          |       | vendorId         |
| approvalLevel    |       +-------------------+       | dealOutcomes     |
+------------------+                                    | patterns         |
                                                        | preferences      |
                                                        +------------------+
```

### Supporting Models (32 total)

| Category | Models |
|----------|--------|
| **Core negotiation** | ChatbotDeal, ChatbotMessage, NegotiationRound, MesoRound |
| **Business entities** | Requisition, VendorCompany, Company, Product, Project |
| **Auth & RBAC** | User, Role, RolePermission |
| **Templates** | ChatbotTemplate, ChatbotTemplateParameter |
| **Sessions** | ChatSession |
| **Documents** | Contract, PO (Purchase Order) |
| **Vendor intelligence** | VendorNegotiationProfile, Preference, PreferenceTracker |
| **Bid analysis** | VendorBid, BidComparison, BidActionHistory |
| **Vector/RAG** | DealEmbedding, MessageEmbedding, VectorMigrationStatus |
| **Audit** | EmailLog, ApiUsageLog, UserAction |

---

## 17. Authentication & Security

### Authentication Flow

```
LOGIN
=====
POST /api/auth/login
  Body: { email, password }
  Response: { data: { accessToken: "Bearer eyJ...", refreshToken: "..." } }

Note: accessToken includes "Bearer " prefix — frontend strips it for header use.

AUTHENTICATED REQUESTS
======================
Authorization: Bearer <token>

Token payload: { userId, userType, companyId, email }
userType: "admin" | "customer" | "vendor" (fallback: "customer")

TOKEN REFRESH
=============
On 401 response:
  1. Queue subsequent requests
  2. Refresh token via /api/auth/refresh
  3. Retry queued requests with new token
  4. If refresh fails → redirect to /sign-in
```

### Authorization Model

| Auth Method | Use Case | Header |
|-------------|----------|--------|
| JWT Bearer Token | PM frontend, admin operations | `Authorization: Bearer <token>` |
| API Key + Secret | Programmatic access | `apiKey` + `apiSecret` headers |
| Unique Token | Vendor portal (public) | URL parameter: `/vendor-chat/:token` |

### Security Boundaries

```
VENDOR PORTAL ISOLATION
=======================

Vendor sees:           PM sees (NegotiationRoom):
  - Chat messages        - Chat messages
  - MESO options         - MESO options
  - Deal status          - Deal status
  - Deal title           - Deal title
                         + Utility scores
                         + Parameter weights
                         + Threshold zones
                         + Behavioral analysis
                         + AI reasoning
                         + Strategy labels
                         + Convergence data
                         + Configuration
```

---

## 18. Cross-Deal Learning & Vendor Profiles

### Vendor Profile Service (`vendorProfileService.ts`)

Tracks vendor behavior across negotiations:

```
VENDOR PROFILE
==============
VendorNegotiationProfile {
  dealOutcomes:  Agreement rate, average final price, average rounds
  patterns:      Concession velocity, flexibility level, emphasis preference
  preferences:   Price-focused vs terms-focused, typical payment terms
}

Cross-Deal Learning:
  - When starting a new deal with a known vendor:
    1. Load vendor's historical profile
    2. Adjust anchor price based on past agreements
    3. Set initial firmness based on vendor's flexibility
    4. Anticipate emphasis preference

  - After deal completion:
    1. Update profile with outcome
    2. Track patterns (did they concede more on price or terms?)
    3. Record final agreement details
```

### Historical Analyzer (`historicalAnalyzer.ts`)

```
CROSS-DEAL INTELLIGENCE
=======================

findSimilarDeals(currentDeal)
  |
  +--> Match by: vendor, product category, price range
  |
  +--> Return: historical agreement prices, round counts,
  |            successful strategies
  |
  +--> adjustAnchorFromHistory()
       If vendor previously agreed at $9,500:
         → Start anchor closer to $9,500 (not $8,000)
         → Reduces unnecessary rounds
```

---

## 19. Error Recovery & Fallbacks

### Graceful Degradation Chain

```
LLM RESPONSE PIPELINE
=====================

Step 1: personaRenderer() calls LLM
  |
  +--[Success]--> Step 2: validateLlmOutput()
  |                   |
  |                   +--[Valid]--> Use LLM response
  |                   |
  |                   +--[Invalid]--> Step 3: getFallbackResponse()
  |                                        |
  +--[Timeout]--> Step 3: getFallbackResponse()
  |                        |
  +--[Error]--> Step 3: getFallbackResponse()
                           |
                           +--> 5+ humanized template variations
                           +--> Tone-aware selection
                           +--> Random variation (no repetition)
                           +--> Vendor never knows LLM was skipped
```

### Two-Phase Error Recovery

```
PHASE 1 FAILURE (instant save)
  --> Toast error to PM
  --> Don't proceed to Phase 2
  --> UI state preserved

PHASE 2 FAILURE (PM response)
  --> 5-second timeout triggers fallback
  --> Fallback uses deterministic template
  --> PM message still appears (just template-based)
  --> Deal state still updates correctly
```

### Frontend Error Handling

| Scenario | Handling |
|----------|----------|
| Invalid deal ID | Show error UI with back navigation |
| Network failure | Toast notification, retry button |
| Auth token expired | Auto-refresh, queue requests |
| Deal not found | Error state with helpful message |
| Terminal deal | Read-only mode, no input |

---

## 20. Performance & Infrastructure

### Response Times

| Operation | Target | Mechanism |
|-----------|--------|-----------|
| Vendor message save | ~100ms | Direct DB write, no processing |
| PM response (LLM) | 5–15s | Server-side delay + LLM generation |
| PM response (fallback) | <1s | Template-based, no LLM |
| Utility calculation | <50ms | In-memory computation |
| Behavioral analysis | <100ms | In-memory state analysis |
| MESO generation | <200ms | Algorithmic option generation |

### Polling & Real-Time Updates

```
Frontend polls every 5 seconds while deal.status === 'NEGOTIATING'
  |
  +--> GET /deals/:dealId  (lightweight status check)
  |
  +--> If status changed → update UI, show notification
  |
  +--> If ACCEPTED/WALKED_AWAY/ESCALATED → stop polling, show outcome
```

### LLM Infrastructure

```
PRIMARY: Ollama (self-hosted)
  |
  +--[Healthy]--> Use Ollama for generation
  |
  +--[Unhealthy]--> FALLBACK: OpenAI API
                         |
                         +--> Token counting: gpt-3.5-turbo
                         +--> Message truncation if too long
                         +--> Usage logged to ApiUsageLog
                         +--> Exponential backoff (3 retries)
```

### Caching

| Cache | TTL | Purpose |
|-------|-----|---------|
| Currency rates | 1 hour | FX conversion for multi-currency |
| Vendor profiles | Per-request | Cross-deal learning data |
| LLM health check | Per-request | Ollama/OpenAI availability |

---

## 21. Tech Stack

### Backend

| Component | Technology |
|-----------|-----------|
| Runtime | Node.js |
| Framework | Express.js |
| Language | TypeScript (ES Modules, `.js` extension convention) |
| Database | PostgreSQL |
| ORM | Sequelize |
| Authentication | JWT (access + refresh tokens) |
| LLM (primary) | Ollama (self-hosted) |
| LLM (fallback) | OpenAI API |
| Email | AWS SES via nodemailer |
| Logging | Winston |
| Testing | Vitest (441+ unit tests) |

### Frontend

| Component | Technology |
|-----------|-----------|
| Framework | React 19 |
| Build Tool | Vite 5.4 |
| Language | TypeScript 5.9 (strict mode) |
| Styling | Tailwind CSS 3.4 |
| Routing | React Router v7 |
| HTTP Client | Axios (with interceptors) |
| Notifications | react-hot-toast |
| Forms | react-hook-form |
| State Management | Hooks + Context (no global store) |

### Infrastructure

| Component | Technology |
|-----------|-----------|
| Backend Port | 5002 |
| Frontend Port | 5001 |
| Database | PostgreSQL |
| Email | AWS SES SMTP |
| LLM Hosting | Ollama (local) + OpenAI (cloud fallback) |

---

*This document covers the production-deployed features on the `main` branch of Accordo-AI as of March 2026. For documentation on upcoming features (llm-chat branch), see [CHATBOT_SYSTEM_ARCHITECTURE.md](./CHATBOT_SYSTEM_ARCHITECTURE.md).*
