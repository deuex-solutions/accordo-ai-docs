x# Accordo-AI: Chatbot & Negotiation Engine — Technical Architecture Document

> **Audience**: CTO / Technical Investor
> **Version**: 1.0 — March 2026
> **Scope**: Complete chatbot system — decision engine, LLM integration, PM interface, vendor portal, MESO negotiation

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [High-Level Chat Flow — End to End](#3-high-level-chat-flow--end-to-end)
4. [Negotiation Decision Engine](#4-negotiation-decision-engine)
5. [Utility Scoring System](#5-utility-scoring-system)
6. [Two-Phase Messaging Architecture](#6-two-phase-messaging-architecture)
7. [LLM Integration & Safety Boundary](#7-llm-integration--safety-boundary)
8. [MESO — Multiple Equivalent Simultaneous Offers](#8-meso--multiple-equivalent-simultaneous-offers)
9. [Vendor Portal Flow](#9-vendor-portal-flow)
10. [PM (Procurement Manager) Interface Flow](#10-pm-procurement-manager-interface-flow)
11. [Behavioral Analysis & Adaptive Strategy](#11-behavioral-analysis--adaptive-strategy)
12. [Offer Parsing & Multi-Currency Support](#12-offer-parsing--multi-currency-support)
13. [Data Model & Persistence](#13-data-model--persistence)
14. [API Endpoint Reference](#14-api-endpoint-reference)
15. [Security & Auditability](#15-security--auditability)
16. [Error Recovery & Fault Tolerance](#16-error-recovery--fault-tolerance)
17. [Frontend Component Architecture](#17-frontend-component-architecture)
18. [Performance & Scalability](#18-performance--scalability)
19. [Technology Stack](#19-technology-stack)

---

## 1. Executive Summary

Accordo-AI is a B2B procurement negotiation platform where an AI-powered Procurement Manager (PM) autonomously negotiates with vendors on behalf of buyers. The system combines a **deterministic decision engine** (utility-based scoring with configurable thresholds) with **LLM-powered natural language generation** — separated by a hard security boundary that prevents the LLM from ever accessing commercial strategy data.

### Key Differentiators

| Capability | Description |
|---|---|
| **Deterministic Decision Engine** | Weighted multi-parameter utility scoring (price, payment terms, delivery, warranty, quality) with threshold-based actions (ACCEPT, COUNTER, ESCALATE, WALK_AWAY) |
| **Hard LLM Boundary** | LLM generates natural language but never sees utility scores, weights, thresholds, target prices, or negotiation config |
| **Two-Phase Messaging** | Vendor message saved instantly (~100ms); AI response generated asynchronously (5-15s) — feels like a real chat |
| **MESO Negotiation** | Multiple Equivalent Simultaneous Offers — 3 options with equal utility but different trade-offs, enabling preference learning |
| **Behavioral Analysis** | Momentum tracking, concession velocity, convergence rate, stall detection — strategy adapts in real-time |
| **Multi-Currency** | Live FX conversion (USD, INR, EUR, GBP, AUD) with regional number format parsing (Indian lakhs, EU decimals) |
| **Two Negotiation Modes** | INSIGHTS (fully deterministic, no LLM) and CONVERSATION (deterministic decisions + LLM rendering) |

### How It Works — 30-Second Overview

```
Buyer configures deal (price targets, weights, thresholds)
  → Vendor receives email link → Opens portal → Submits quote
    → AI parses offer (regex, currency conversion)
      → Decision engine scores utility (0-100%)
        → If COUNTER: calculates optimal counter-offer
          → LLM renders human-like response (never sees strategy)
            → Validator checks output (banned words, price bounds)
              → Fallback template if LLM fails (vendor never knows)
                → Typing delay simulated server-side
                  → Response delivered to vendor
```

---

## 2. System Architecture Overview

### High-Level System Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Frontend — React 19 / Vite / TypeScript                  │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐   │
│  │ PM Interface  │  │Vendor Portal │  │ Deal Wizard  │  │Utility Sidebar│   │
│  │(Negotiation  │  │ (VendorChat) │  │ (4-Step      │  │ (Real-time    │   │
│  │    Room)     │  │              │  │   Config)    │  │   Scoring)    │   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘   │
│         │POST /messages   │POST /vendor     │POST /deals       │GET /util  │
└─────────┼─────────────────┼─────────────────┼──────────────────┼───────────┘
          │                 │                 │                  │
          ▼                 ▼                 ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Backend — Node.js / Express / TypeScript                  │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                        REST API Layer                                │   │
│  │                   (Controllers + Routes)                             │   │
│  └────────────────────────────┬─────────────────────────────────────────┘   │
│                               │                                             │
│  ┌────────────────────────────┼──────────────────────────────────────────┐  │
│  │              Negotiation Engine                                       │  │
│  │                                                                       │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐               │  │
│  │  │Offer Parser │  │Decision Eng. │  │ Utility Scorer │               │  │
│  │  │(Regex + FX) │  │ (decide.ts)  │  │(Weighted 5-Par)│               │  │
│  │  └─────────────┘  └──────────────┘  └────────────────┘               │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐               │  │
│  │  │ Behavioral  │  │MESO Generator│  │Counter Calc.   │               │  │
│  │  │  Analyzer   │  │(3 Equal-Util)│  │(Priority-Based)│               │  │
│  │  └─────────────┘  └──────────────┘  └────────────────┘               │  │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                               │                                             │
│                          Decision                                           │
│                               ▼                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │             LLM Pipeline (Hard Boundary)                             │   │
│  │                                                                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │   │
│  │  │ Intent Build.│  │Persona Rend. │  │Output Valid.  │               │   │
│  │  │(Negotiation  │  │(LLM Call,    │  │(Banned Words, │               │   │
│  │  │   Intent)    │  │  Temp 0.5)   │  │   Prices)     │               │   │
│  │  │  ** HARD **  │  └──────────────┘  │  ** HARD **   │               │   │
│  │  └──────────────┘                    └───────┬───────┘               │   │
│  │  ┌──────────────┐  ┌──────────────┐          │ Fail                 │   │
│  │  │Typing Delay  │  │Fallback Tmpl │◄─────────┘                     │   │
│  │  │(Server-Side) │  │(5+ Var/Actn) │                                 │   │
│  │  └──────────────┘  └──────────────┘                                 │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────┐      ┌────────────────────────────────────────────┐   │
│  │   Data Layer     │      │          External Services                  │   │
│  │                  │      │                                              │   │
│  │ ┌──────────────┐ │      │ ┌────────────┐  ┌──────────┐  ┌──────────┐ │   │
│  │ │ PostgreSQL   │ │      │ │ OpenAI API │  │  Ollama  │  │Currency  │ │   │
│  │ └──────────────┘ │      │ │(gpt-3.5-t) │  │(Local LLM│  │API (FX)  │ │   │
│  │ ┌──────────────┐ │      │ └────────────┘  │ Fallback)│  └──────────┘ │   │
│  │ │ Sequelize    │ │      │ ┌────────────┐  └──────────┘               │   │
│  │ │ Models (32T) │ │      │ │Email Svc   │                             │   │
│  │ └──────────────┘ │      │ │ (AWS SES)  │                             │   │
│  └──────────────────┘      │ └────────────┘                             │   │
│                            └────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Backend Directory Structure

```
Accordo-ai-backend/src/
├── modules/chatbot/           # Core chatbot module
│   ├── chatbot.routes.ts      # Express router (all endpoints)
│   ├── chatbot.controller.ts  # Request handlers
│   ├── chatbot.service.ts     # Business logic orchestrator (~1200 lines)
│   ├── chatbot.repo.ts        # Database queries
│   ├── engine/                # Deterministic decision engine
│   │   ├── decide.ts          # Core decision logic (~1110 lines)
│   │   ├── utility.ts         # Legacy 2-param utility
│   │   ├── weightedUtility.ts # New 5-param weighted utility
│   │   ├── parseOffer.ts      # Regex offer extraction
│   │   ├── offerAccumulator.ts# Multi-message offer merging
│   │   ├── meso.ts            # MESO option generation
│   │   ├── toneDetector.ts    # Vendor tone classification
│   │   ├── behavioralAnalyzer.ts # Momentum & convergence
│   │   ├── stallDetector.ts   # Stall detection
│   │   ├── preferenceDetector.ts # Vendor preference learning
│   │   └── types.ts           # Type definitions (~400 lines)
│   ├── convo/                 # CONVERSATION mode orchestration
│   │   ├── conversationService.ts  # Pipeline controller
│   │   └── processConversationTurn.ts
│   └── vendor/                # Vendor simulation (testing)
├── negotiation/intent/        # Hard boundary layer
│   └── buildNegotiationIntent.ts
├── llm/                       # LLM pipeline
│   ├── personaRenderer.ts     # LLM call (only entry point)
│   ├── validateLlmOutput.ts   # Output validation
│   └── fallbackTemplates.ts   # Deterministic fallbacks
├── delivery/
│   └── simulateTypingDelay.ts # Server-side typing delay
├── metrics/
│   └── logNegotiationStep.ts  # Audit logging
└── models/                    # Sequelize models (32 tables)
```

---

## 3. High-Level Chat Flow — End to End

### Complete Negotiation Lifecycle

```
 PM          Frontend      Backend API     Decision Eng.    LLM Pipeline      DB          Vendor Portal    VENDOR
 │              │              │               │               │               │               │              │
 │              │              │               │               │               │               │              │
 │ === PHASE 1: DEAL SETUP ========================================================================           │
 │              │              │               │               │               │               │              │
 │──Create Deal─►              │               │               │               │               │              │
 │ (4-step wiz) │              │               │               │               │               │              │
 │              │──POST /deals─►               │               │               │               │              │
 │              │ (config,     │               │               │               │               │              │
 │              │  weights,    │               │               │               │               │              │
 │              │  thresholds) │               │               │               │               │              │
 │              │              │───────────────────────────────────────────────►│               │              │
 │              │              │  Create ChatbotDeal + NegotiationConfig       │               │              │
 │              │              │───────────────────────────────────────────────────────────────►│              │
 │              │              │  Email with unique portal link                │               │──────────────►
 │              │              │               │               │               │               │              │
 │ === PHASE 2: VENDOR SUBMITS QUOTE ==============================================================           │
 │              │              │               │               │               │               │              │
 │              │              │               │               │               │               │◄─────────────│
 │              │              │               │               │               │               │ Opens portal │
 │              │              │◄──GET /vendor-chat/:token─────────────────────────────────────│              │
 │              │              │───────────────────────────────────────────────────────────────►│              │
 │              │              │               │               │               │               │◄─────────────│
 │              │              │               │               │               │               │ Types quote: │
 │              │              │               │               │               │               │"$48K,Net60,  │
 │              │              │◄──POST /vendor-message-instant (Phase 1)──────────────────────│  20 days"    │
 │              │              │───────────────────────────────────────────────►│ Save msg      │              │
 │              │              │──────────────────────────────────────────────────────────────►│              │
 │              │              │  Vendor message confirmed (~100ms)            │               │              │
 │              │              │◄──POST /pm-response-async (Phase 2)───────────────────────────│              │
 │              │              │               │               │               │               │              │
 │ === PHASE 3: AI PROCESSING =====================================================================           │
 │              │              │               │               │               │               │              │
 │              │              │──parseOffer───►               │               │               │              │
 │              │              │ ("$48K,Net60,  │               │               │               │              │
 │              │              │   20 days")   │               │               │               │              │
 │              │              │◄──────────────│               │               │               │              │
 │              │              │ {total_price:  │               │               │               │              │
 │              │              │  48000,        │               │               │               │              │
 │              │              │  payment: Net60│               │               │               │              │
 │              │              │  delivery: 20} │               │               │               │              │
 │              │              │──decideNext───►               │               │               │              │
 │              │              │  Move()       │               │               │               │              │
 │              │              │               │──Calc utility─┤               │               │              │
 │              │              │               │  (48% COUNTER)│               │               │              │
 │              │              │               │──Calc counter─┤               │               │              │
 │              │              │               │  $47K, Net 70 │               │               │              │
 │              │              │◄──────────────│               │               │               │              │
 │              │              │ Decision{COUNTER, $47K, 0.48} │               │               │              │
 │              │              │               │               │               │               │              │
 │              │              │   ┌─── if CONVERSATION Mode ─────────────┐    │               │              │
 │              │              │   │                           │           │    │               │              │
 │              │              │───┼──buildNegotiationIntent──►│           │    │               │              │
 │              │              │   │    (Intent has NO weights, │           │    │               │              │
 │              │              │   │     thresholds, or config) │           │    │               │              │
 │              │              │   │                           │──render───┤    │               │              │
 │              │              │   │                           │  Message  │    │               │              │
 │              │              │   │                           │ (Temp 0.5)│    │               │              │
 │              │              │   │                           │──validate─┤    │               │              │
 │              │              │   │                           │  Output   │    │               │              │
 │              │              │   │    ┌─ if Pass ──────────┐ │           │    │               │              │
 │              │              │◄──┼────│ "Thank you...      │ │           │    │               │              │
 │              │              │   │    │  counter at $47K"  │ │           │    │               │              │
 │              │              │   │    └────────────────────┘ │           │    │               │              │
 │              │              │   │    ┌─ if Fail ──────────┐ │           │    │               │              │
 │              │              │   │    │ getFallbackTemplate │ │           │    │               │              │
 │              │              │◄──┼────│ (silent, vendor    │ │           │    │               │              │
 │              │              │   │    │  never knows)      │ │           │    │               │              │
 │              │              │   │    └────────────────────┘ │           │    │               │              │
 │              │              │   │                           │──typing───┤    │               │              │
 │              │              │   │                           │  delay    │    │               │              │
 │              │              │   │                           │ (6-12s)   │    │               │              │
 │              │              │   └──────────────────────────────────────┘    │               │              │
 │              │              │   ┌─── if INSIGHTS Mode ────┐ │               │               │              │
 │              │              │───┼──generateAccordoResp────►│ │               │               │              │
 │              │              │◄──┼──Template response───────│ │               │               │              │
 │              │              │   └─────────────────────────┘ │               │               │              │
 │              │              │               │               │               │               │              │
 │              │              │───────────────────────────────────────────────►│               │              │
 │              │              │  Save PM message + decision + explainability  │               │              │
 │              │              │──────────────────────────────────────────────────────────────►│              │
 │              │              │  PM response + typing delay                   │               │              │
 │              │              │               │               │               │               │──────────────►
 │              │              │               │               │               │               │Display counter│
 │              │              │               │               │               │               │              │
 │ === PHASE 4: NEGOTIATION ROUNDS ================================================================           │
 │              │              │               │               │               │               │              │
 │              │              │   ┌─ Loop: Until ACCEPT / WALK_AWAY / ESCALATE ───────────┐   │              │
 │              │              │   │                           │               │            │   │              │
 │              │              │   │  Vendor counter or accept │               │            │   │◄─────────────│
 │              │              │◄──┼──Two-phase message flow───────────────────────────────┼───│              │
 │              │              │───┼──Re-evaluate utility ────►               │            │   │              │
 │              │              │◄──┼──Decision (COUNTER/ACCEPT/ESCALATE/WALK)──            │   │              │
 │              │              │───┼──Response──────────────────────────────────────────────┼──►│              │
 │              │              │   └───────────────────────────────────────────────────────┘   │              │
 │              │              │               │               │               │               │              │
 │ === PHASE 5: MESO (After Round 5) ==============================================================           │
 │              │              │               │               │               │               │              │
 │              │              │──generateMeso─►               │               │               │              │
 │              │              │  Options()    │               │               │               │              │
 │              │              │◄──────────────│               │               │               │              │
 │              │              │ 3 offers, ~equal utility,     │               │               │              │
 │              │              │ different trade-offs          │               │               │              │
 │              │              │──────────────────────────────────────────────────────────────►│              │
 │              │              │ MESO options + "Others" button│               │               │              │
 │              │              │               │               │               │               │              │
 │              │              │   ┌─ if Vendor selects MESO option ───────────────────────┐   │              │
 │              │              │◄──┼──POST /meso/select (optionId)──────────────────────────┼──│              │
 │              │              │───┼──────────────────────────────────────────►│ ACCEPTED   │   │              │
 │              │              │   └───────────────────────────────────────────────────────┘   │              │
 │              │              │   ┌─ if Vendor clicks "Others" ──────────────────────────┐    │              │
 │              │              │◄──┼──POST /meso/others (custom price + terms)──────────────┼──│◄─────────────│
 │              │              │   │  4 more text rounds, then next MESO cycle │            │   │              │
 │              │              │   └───────────────────────────────────────────────────────┘   │              │
 │              │              │               │               │               │               │              │
 │ === PHASE 6: TERMINAL STATE ====================================================================           │
 │              │              │               │               │               │               │              │
 │              │              │───────────────────────────────────────────────►│               │              │
 │              │              │ Update deal status (ACCEPTED/WALKED_AWAY/ESCALATED)           │              │
 │              │              │ Update contract status + Capture vendor profile               │              │
 │◄─────────────│◄─────────────│ Toast notification + status update            │               │              │
 │              │              │──────────────────────────────────────────────────────────────►│──────────────►
 │              │              │               │               │               │ Completion    │              │
 │              │              │               │               │               │   screen      │              │
```

### Two-Mode Processing Pipeline

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Vendor  │    │ Parse Offer  │    │Decision Eng. │    │Utility Score │
│  Message │───►│(Regex + FX)  │───►│ (decide.ts)  │───►│ (Weighted)   │
└──────────┘    └──────────────┘    └──────────────┘    └──────┬───────┘
                                                               │
                                                               ▼
                                                    ┌─────────────────────┐
                                                    │     Action?         │
                                                    └──┬───┬───┬───┬───┬─┘
                    ┌──────────────┘    │   │   │   └──────────────┐
                    ▼                  ▼   │   ▼                  ▼
            ┌──────────────┐  ┌────────┐   │ ┌──────────┐  ┌───────────┐
            │   COUNTER    │  │ ACCEPT │   │ │WALK_AWAY │  │ASK_CLARIFY│
            │(Calc Counter)│  │Terminal│   │ │ Terminal  │  │Request    │
            └──────┬───────┘  └────────┘   │ └──────────┘  │More Info  │
                   │                       ▼               └─────┬─────┘
                   │               ┌──────────┐                  │
                   │               │ ESCALATE │                  │
                   │               │ Terminal  │                  │
                   │               └──────────┘                  │
                   │                                             │
                   └────────────────────┬────────────────────────┘
                                        ▼
                                 ┌──────────────┐
                                 │    Mode?     │
                                 └───┬──────┬───┘
                                     │      │
                    INSIGHTS ────────┘      └──────── CONVERSATION
                         │                                  │
                         ▼                                  ▼
                ┌────────────────┐                 ┌────────────────┐
                │ Template-Based │                 │ Build Intent   │
                │   Response     │                 │(Hard Boundary) │
                └───────┬────────┘                 │  ** HARD **    │
                        │                          └───────┬────────┘
                        │                                  ▼
                        │                          ┌────────────────┐
                        │                          │  LLM Call      │
                        │                          │  (Temp 0.5)    │
                        │                          └───────┬────────┘
                        │                                  ▼
                        │                          ┌────────────────┐
                        │                     ┌────│Validate Output │────┐
                        │                     │    │(Banned Words)  │    │
                        │                  Pass    └────────────────┘  Fail
                        │                     │                         │
                        │                     │                  ┌──────┴───────┐
                        │                     │                  │  Fallback    │
                        │                     │                  │  Template    │
                        │                     │                  └──────┬───────┘
                        │                     │                         │
                        ▼                     ▼                         ▼
                      ┌─────────────────────────────────────────────────┐
                      │              Typing Delay (Server-Side)         │
                      └──────────────────────┬──────────────────────────┘
                                             ▼
                                    ┌────────────────┐
                                    │  Save to DB    │
                                    │  + Return      │
                                    └────────────────┘
```

---

## 4. Negotiation Decision Engine

The decision engine (`decide.ts`, ~1110 lines) is the core of the system. It is **fully deterministic** — given the same inputs, it always produces the same output. No LLM is involved in decision-making.

### Decision Flow

```
┌───────────────────────┐
│ Vendor Offer Received │
└──────────┬────────────┘
           ▼
┌───────────────────────┐
│ Parse Offer           │
│ (Price, Terms, Deliv.)│
└──────────┬────────────┘
           ▼
     ┌───────────┐
     │  Offer    │
     │ Complete? │
     └──┬─────┬──┘
     No │     │ Yes
        ▼     │
┌────────────┐│
│Merge with  ││
│Prev Offers ││
└─────┬──────┘│
      ▼       │
 ┌─────────┐  │
 │ Still   │  │
 │Incompl? │  │
 └──┬───┬──┘  │
 Yes│   │No   │
    ▼   └──┐  │
┌────────┐ │  │
│ASK_    │ │  │
│CLARIFY │ │  │
└────────┘ │  │
           ▼  ▼
     ┌──────────────────┐
     │Calculate Weighted│
     │    Utility       │
     └────────┬─────────┘
              ▼
     ┌──────────────────┐
     │  Total Utility   │
     │ Score (0-100%)   │
     └────────┬─────────┘
              ▼
      ┌──────────────┐
      │Utility >= 70%│──── Yes ───► ┌────────┐
      └──────┬───────┘              │ ACCEPT │  (GREEN)
          No │                      └────────┘
             ▼
      ┌──────────────┐
      │Utility >= 50%│──── Yes ───► ┌─────────┐
      └──────┬───────┘              │ COUNTER │  (BLUE)
          No │                      │(Calc    │
             ▼                      │ Offer)  │───────────────────┐
      ┌──────────────┐              └─────────┘                   │
      │Utility >= 30%│                   ▲                        │
      └──┬────────┬──┘                   │ No                     │
      Yes│     No │              ┌───────┴──────┐                 │
         ▼        │              │  Stalled     │                 │
    ┌─────────┐   │              │  3+ Rounds?  │                 │
    │ Stalled │   │              └──────┬───────┘                 │
    │3+ Rnds? │   │                  Yes│                         │
    └─┬────┬──┘   │                    ▼                          │
   Yes│ No │      │            ┌───────────┐                     │
      ▼    └──────┼───────────►│ COUNTER   │  (BLUE)             │
┌──────────┐      │            └───────────┘                     │
│ ESCALATE │      ▼                                              │
│ (ORANGE) │ ┌────────────────┐                                  │
└──────────┘ │Round >= 10 &   │                                  │
             │Vendor Rigid?   │                                  │
             └──┬──────────┬──┘                                  │
             Yes│       No │                                     │
                ▼          └──────────────►┌─────────┐           │
          ┌───────────┐                    │ COUNTER │           │
          │ WALK_AWAY │                    │  (BLUE) │           │
          │   (RED)   │                    └─────────┘           │
          └───────────┘                                          │
                                                                 │
              ┌──────────────────────────────────────────────────┘
              │
              ▼
       ┌─────────────┐
       │Deal Priority?│
       └──┬────┬────┬─┘
     HIGH │  MED│ LOW│
          ▼    ▼    ▼
     ┌──────┐┌──────┐┌──────┐
     │ 15%  ││ 40%  ││ 55%  │
     │toward││toward││toward│
     │vendor││vendor││vendor│
     │(Firm)││(Bal.)││(Flex)│
     └──┬───┘└──┬───┘└──┬───┘
        └───────┼────────┘
                ▼
       ┌────────────────┐
       │+ 2% per round  │
       │  (max +10%)    │
       └───────┬────────┘
               ▼
       ┌────────────────┐
       │+ Concession    │
       │ Bonus (if      │
       │ vendor yielding│
       └───────┬────────┘
               ▼
       ┌────────────────────┐
       │ Final Counter-Offer│
       │(Price+Terms+Deliv.)│
       └────────────────────┘
```

### Threshold Configuration by Priority

| Priority | Accept (>=) | Counter | Escalate (<) | Walk Away (<) | Max Rounds | Counter Aggressiveness |
|---|---|---|---|---|---|---|
| **HIGH** | 75% | 55-75% | 55% (if stalled) | 35% (round 10+) | 10 | 15% toward vendor |
| **MEDIUM** | 70% | 50-70% | 50% (if stalled) | 30% (round 10+) | 6 | 40% toward vendor |
| **LOW** | 65% | 45-65% | 45% (if stalled) | 25% (round 10+) | 4 | 55% toward vendor |

### Safety Guards — Why the Engine Never Walks Away Too Early

1. **Minimum Round Rule**: WALK_AWAY only triggers after round 10, even if utility is below threshold
2. **Stall Requirement**: ESCALATE requires 3+ consecutive rounds with no vendor movement
3. **Preference Exploration Protection**: During MESO cycles, engine will not walk away (vendor is actively engaged)
4. **Convergence Override**: If vendor is converging (positive momentum), rounds auto-extend past softMax

---

## 5. Utility Scoring System

### Five-Parameter Weighted Model

The utility score (0-100%) represents how well a vendor's offer meets the buyer's configured preferences.

```
┌─── Vendor Offer ────────────────┐     ┌── Parameter Utilities (0-1) ────────┐
│                                  │     │                                      │
│  Price:    $48,000               │     │  Price Utility                       │
│  Terms:    Net 60                │     │  = 1 - (48K-45K)/(50K-45K) = 0.40   │
│  Delivery: 20 days              │     │  Terms Utility                       │
│  Warranty: 12 months            │     │  = Net60 mapping = 0.60             │
│  Quality:  ISO 9001             │     │  Delivery Utility                    │
│                                  │     │  = date proximity = 0.70            │
└─────────────┬────────────────────┘     │  Warranty Utility                    │
              │                          │  = 12/24 months = 0.50              │
              └────────────►             │  Quality Utility                     │
                                         │  = ISO match = 1.00                 │
                                         └──────────────┬───────────────────────┘
                                                        │
                                                        ▼
┌─── Weights (from Wizard Step 4) ─┐     ┌── Contributions ────────────────────┐
│                                   │     │                                     │
│  Price:    35%  ─────────────────────►  │  0.40 x 0.35 = 0.140               │
│  Terms:    25%  ─────────────────────►  │  0.60 x 0.25 = 0.150               │
│  Delivery: 20%  ─────────────────────►  │  0.70 x 0.20 = 0.140               │
│  Warranty: 10%  ─────────────────────►  │  0.50 x 0.10 = 0.050               │
│  Quality:  10%  ─────────────────────►  │  1.00 x 0.10 = 0.100               │
│                                   │     │                                     │
└───────────────────────────────────┘     └──────────────┬──────────────────────┘
                                                         │
                                                         ▼
                                          ┌──────────────────────────────────────┐
                                          │       Total Utility                  │
                                          │       = 0.580 = 58%                  │
                                          │       ---> COUNTER Zone              │
                                          └──────────────────────────────────────┘
```

### Price Utility Calculation

```
Given:
  target_price    = $45,000  (buyer's desired price)
  max_acceptable  = $50,000  (buyer's ceiling)
  vendor_price    = $48,000  (vendor's offer)

Price Utility = 1 - (vendor_price - target_price) / (max_acceptable - target_price)
             = 1 - (48,000 - 45,000) / (50,000 - 45,000)
             = 1 - 3,000 / 5,000
             = 1 - 0.60
             = 0.40 (40%)

Interpretation:
  - 1.0 = vendor meets target price exactly
  - 0.0 = vendor at maximum acceptable price
  - < 0  = vendor above maximum (clamped to 0)
  - > 1  = vendor below target (clamped to 1, bonus)
```

### Payment Terms Utility Mapping

```
Canonical Scale (Buyer prefers longer terms):
  Net 30  → 0.20  (worst for buyer — pays fastest)
  Net 45  → 0.40
  Net 60  → 0.60
  Net 75  → 0.80
  Net 90  → 1.00  (best for buyer — pays latest)

Non-standard terms (e.g., Net 52):
  → Linear interpolation between nearest standard values
  → Flagged as non-standard in UI
```

### Threshold Zones Visualization

```
 0%          30%         50%         70%         100%
 |-----------|-----------|-----------|-----------|
 | WALK_AWAY | ESCALATE  |  COUNTER  |  ACCEPT   |
 |   (RED)   |  (ORANGE) |  (BLUE)   |  (GREEN)  |
 |-----------|-----------|-----------|-----------|
                                  ●
                          Current: 58% → COUNTER
```

---

## 6. Two-Phase Messaging Architecture

This is the **core UX innovation** — it makes AI negotiation feel as responsive as a real chat.

### The Problem with Traditional AI Chat

```
Traditional: User sends message → Waits 5-15s for AI → Sees everything at once
Result: Feels sluggish, user stares at a spinner, poor UX
```

### Accordo's Two-Phase Solution

```
 User         Frontend        Backend          LLM
  │              │               │               │
  │              │               │               │
  │  ═══ Phase 1: Instant Feedback (~100ms) ════════
  │              │               │               │
  │──Clicks      │               │               │
  │  "Send"─────►│               │               │
  │              │──POST /vendor─►               │
  │              │  msg-instant  │               │
  │              │               │──Save vendor──┤
  │              │               │  msg to DB    │
  │              │◄──{vendorMsg,─│               │
  │              │   deal} ~100ms│               │
  │              │               │               │
  │              │──Display msg──┤               │
  │              │  IMMEDIATELY  │               │
  │              │──Show typing──┤               │
  │              │  indicator    │               │
  │              │               │               │
  │  ═══ Phase 2: Async AI Response (5-15s) ════════
  │              │               │               │
  │              │──POST /pm-───►│               │
  │              │  resp-async   │               │
  │              │               │──Parse offer──┤
  │              │               │  (regex)      │
  │              │               │──Calc utility─┤
  │              │               │──Decide action┤
  │              │               │──Generate ────┼──────────────►│
  │              │               │  response     │               │
  │              │               │  (if CONVO)   │◄──Validated───│
  │              │               │               │   response    │
  │              │               │──Simulate ────┤               │
  │              │               │  typing delay │               │
  │              │◄──{pmMessage,─│               │               │
  │              │  decision,    │               │               │
  │              │  delayMs}     │               │               │
  │              │               │               │               │
  │              │──Hide typing──┤               │               │
  │              │  indicator    │               │               │
  │              │──Display PM───┤               │               │
  │              │  response w/  │               │               │
  │              │  decision     │               │               │
  │              │  badge        │               │               │
  │              │               │               │               │
  │  User sees: Instant message + natural typing delay + AI response
  │  Feels like chatting with a real person
```

### Timeout & Fallback

```
┌─────────────────────────┐
│Phase 2: Request PM Resp.│
└────────────┬────────────┘
             ▼
     ┌───────────────┐
     │   Race:       │
     │ LLM vs 5s    │
     │  Timeout     │
     └───┬───────┬───┘
         │       │
  LLM <5s    Timeout >5s
         │       │
         ▼       ▼
┌──────────┐  ┌──────────────────────┐
│ Validate │  │POST /pm-response-    │
│  Output  │  │       fallback       │
└──┬────┬──┘  └──────────┬───────────┘
   │    │                │
 Pass  Fail              │
   │    │                │
   │    ▼                ▼
   │  ┌──────────────────────┐
   │  │  Use Fallback        │
   │  │  Template             │
   │  └──────────┬───────────┘
   │             │
   ▼             ▼
┌──────────────────────────────────────┐
│  Deliver Response                    │
│  (Vendor never knows if LLM         │
│   or template)                       │
└──────────────────────────────────────┘
```

### Server-Side Typing Delays

All typing delays are enforced server-side (not frontend timers), ensuring consistent human-like pacing regardless of actual LLM response time.

| Action | Delay Range | Rationale |
|---|---|---|
| **COUNTER** | 6-12s | Complex thinking — reviewing terms, calculating |
| **MESO** | 8-15s | Preparing multiple options — more "thought" needed |
| **ACCEPT** | 3-6s | Quick decision — internal approval |
| **WALK_AWAY** | 2-4s | Brief deliberation — firm decision |
| **ESCALATE** | 4-8s | Moderate deliberation — consulting internally |
| **ASK_CLARIFY** | 2-4s | Quick question — natural follow-up |

---

## 7. LLM Integration & Safety Boundary

### The Hard Boundary — Most Critical Architectural Decision

The system treats the LLM as an **untrusted text generator**. The LLM's only job is to express a pre-determined decision in natural language. It has zero influence on what price to offer, when to accept, or whether to walk away.

```
┌══════════════════════════════════════════════════════════════════════════┐
║                  TRUSTED ZONE — Decision Engine                         ║
║                                                                         ║
║  ┌─────────────────────────┐   ┌─────────────────────────┐              ║
║  │ NegotiationConfig       │   │ Utility Calculation      │              ║
║  │ - target: $45K          │   │ - priceUtility: 0.40     │              ║
║  │ - max: $50K             │──►│ - termsUtility: 0.60     │              ║
║  │ - weights: {price: 60%, │   │ - totalUtility: 0.48     │              ║
║  │   terms: 40%}           │   └────────────┬────────────┘              ║
║  │ - accept_threshold: 70% │                │                           ║
║  └─────────────────────────┘                ▼                           ║
║                                ┌─────────────────────────┐              ║
║                                │ Decision Output          │              ║
║                                │ - action: COUNTER        │              ║
║                                │ - counterPrice: $47,000  │              ║
║                                │ - counterTerms: Net 70   │              ║
║                                └────────────┬────────────┘              ║
║                                             │                           ║
╚═════════════════════════════════════════════╪════════════════════════════╝
                                              │
                     ┌────────────────────────┼────────────────────────┐
                     │        HARD BOUNDARY                            │
                     │     buildNegotiationIntent()                    │
                     └────────────────────────┼────────────────────────┘
                                              │
╔═════════════════════════════════════════════╪════════════════════════════╗
║                  UNTRUSTED ZONE — LLM Pipeline                          ║
║                                             │                           ║
║                                             ▼                           ║
║                     ┌─────────────────────────────────────┐             ║
║                     │ NegotiationIntent                    │             ║
║                     │ - action: COUNTER                    │             ║
║                     │ - allowedPrice: $47,000              │             ║
║                     │ - firmness: 0.55                     │             ║
║                     │ - vendorTone: friendly                │             ║
║                     └────────────────┬────────────────────┘             ║
║                                      ▼                                  ║
║                     ┌─────────────────────────────────────┐             ║
║                     │ LLM Call (Temperature 0.5)           │             ║
║                     └────────────────┬────────────────────┘             ║
║                                      ▼                                  ║
║                     ┌─────────────────────────────────────┐             ║
║                     │ "Thank you for your offer.           │             ║
║                     │  We counter at $47,000               │             ║
║                     │  with Net 70 terms."                 │             ║
║                     └─────────────────────────────────────┘             ║
║                                                                         ║
╚═════════════════════════════════════════════════════════════════════════╝
```

### What the LLM Receives vs. What It Never Sees

| LLM Receives (NegotiationIntent) | LLM NEVER Sees |
|---|---|
| `action`: COUNTER | Utility weights (60% price, 40% terms) |
| `allowedPrice`: $47,000 (only for COUNTER) | Thresholds (70% accept, 50% escalate) |
| `firmness`: 0.55 (moderate) | Target price ($45,000) |
| `vendorTone`: friendly | Max acceptable price ($50,000) |
| `weakestParameter`: "price" | Raw utility scores (0.48) |
| `dealTitle`, `vendorName`, `category` | Behavioral signals (momentum, convergence) |
| | Vendor profile / historical data |
| | Concession velocity calculations |
| | Anchor price or strategy parameters |

### Firmness Mapping (Utility → Tone)

The `firmness` value (0-1) guides the LLM's communication tone without revealing the underlying utility score:

| Utility Score | Firmness | LLM Instruction | Example Tone |
|---|---|---|---|
| >= 70% | 0.25 (soft) | "Be warm and collaborative" | "We're really close to a deal..." |
| 50-70% | 0.55 (moderate) | "Be professional but firm" | "We appreciate this, but need..." |
| 30-50% | 0.75 (firm) | "Be direct and assertive" | "We need significant improvement..." |
| < 30% | 0.90 (very firm) | "Be non-negotiable" | "This is well below what we can accept..." |

### Output Validation — Defense in Depth

Every LLM response passes through validation before reaching the vendor:

```
┌──────────────────┐
│ LLM Response Text│
└────────┬─────────┘
         ▼
┌──────────────────┐
│  Length           │
│  20-500 chars?   │
└───┬──────────┬───┘
    │ No       │ Yes
    ▼          ▼
┌────────┐  ┌──────────────────┐
│VALIDATN│  │ Contains banned  │
│ FAIL   │  │    words?        │
└───┬────┘  └───┬──────────┬───┘
    │           │ Yes      │ No
    │           ▼          ▼
    │      ┌────────┐  ┌──────────────────┐
    │◄─────│VALIDATN│  │ Price within     │
    │      │ FAIL   │  │ [target, max]    │
    │      └────────┘  │   +/- 0.5%?      │
    │                  └───┬──────────┬───┘
    │                      │ No      │ Yes
    │                      ▼         ▼
    │                 ┌────────┐  ┌──────────────────┐
    │◄────────────────│VALIDATN│  │ Word count       │
    │                 │ FAIL   │  │   <= 160?        │
    │                 └────────┘  └───┬──────────┬───┘
    │                                 │ No      │ Yes
    │                                 ▼         ▼
    │                            ┌────────┐  ┌──────────────────────┐
    │◄───────────────────────────│VALIDATN│  │ VALIDATION PASS      │
    │                            │ FAIL   │  │ ---> Deliver to      │
    │                            └────────┘  │      vendor          │
    │                                        └──────────────────────┘
    │
    ▼
┌──────────────────────────┐
│ Silent Fallback           │
│ ---> Use deterministic   │
│      template            │
└───────────┬──────────────┘
            ▼
┌──────────────────────────┐
│ Deliver to vendor         │
│ (vendor never knows)      │
└──────────────────────────┘
```

**Banned Keywords** (LLM response rejected if any appear):
`utility`, `algorithm`, `score`, `threshold`, `model`, `AI`, `LLM`, `neural`, `weight`, `parameter`, `optimization`, `machine learning`, `calculation`, `formula`, `percentage`, `basis points`

### Fallback Templates — Zero-Downtime Guarantee

If the LLM fails, times out, or produces invalid output, the system silently uses a pre-written template. There are 5+ humanized variations per action type, selected by vendor tone:

```
COUNTER (Friendly tone):
  "Thank you for your offer. We'd like to propose $47,000 with Net 70 terms
   and delivery within 20 days. We believe this works well for both sides."

COUNTER (Firm tone):
  "We've reviewed your proposal carefully. Our counter: $47,000, Net 70,
   delivery by March 20. These terms reflect the value we need from this deal."

ACCEPT:
  "Thank you — we're pleased to accept your offer of $46,500 with Net 60 terms.
   We look forward to finalizing the contract."
```

---

## 8. MESO — Multiple Equivalent Simultaneous Offers

MESO is a negotiation technique from behavioral economics. Instead of making one counter-offer, the system presents 3 options with **approximately equal utility** but **different trade-off profiles**. This reveals vendor preferences without asking directly.

### MESO Generation Logic

```
                    ┌──────────────────────┐
                    │ Round >= 5 AND       │
                    │ Deal still           │
                    │ NEGOTIATING?         │
                    └───┬──────────────┬───┘
                     Yes│              │No
                        ▼              ▼
               ┌────────────┐  ┌────────────────┐
               │Generate 3  │  │Continue Normal │
               │  Offers    │  │Text Negotiation│
               └─────┬──────┘  └────────────────┘
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
 ┌──────────────┐┌──────────────┐┌──────────────┐
 │  Offer 1:    ││  Offer 2:    ││  Offer 3:    │
 │PRICE-FOCUSED ││TERMS-FOCUSED ││  BALANCED    │
 │              ││              ││              │
 │- Best price  ││- Medium price││- Medium price│
 │  (-2.5%)     ││- Best terms  ││- Medium terms│
 │- Medium terms││  (Net 90)    ││- Fastest     │
 │- Fast deliv. ││- Medium deliv││  delivery    │
 │- Min warranty││- Std warranty││- Extended    │
 └──────┬───────┘└──────┬───────┘│  warranty    │
        │               │        └──────┬───────┘
        └───────────────┼───────────────┘
                        ▼
              ┌────────────────────┐
              │ All 3 utilities    │
              │ within 2% of      │◄───┐
              │ each other?        │    │
              └───┬────────────┬───┘    │
               Yes│            │No      │
                  ▼            ▼        │
    ┌────────────────┐  ┌───────────┐   │
    │Present to      │  │Adjust     │   │
    │Vendor +        │  │prices to  │───┘
    │"Others" Button │  │equalize   │
    └───────┬────────┘  │utilities  │
            │           └───────────┘
            ▼
     ┌──────────────┐
     │   Vendor     │
     │   Action?    │
     └──┬────────┬──┘
        │        │
  Selects     Clicks
  Option     "Others"
        │        │
        ▼        ▼
┌────────────┐┌────────────────────┐
│Learn Pref. ││ Custom Counter     │
│+ ACCEPT    ││ ---> 4 more rounds │
│   Deal     ││ ---> Next MESO     │
│  (GREEN)   ││    cycle (BLUE)    │
└────────────┘└────────────────────┘
```

### Preference Learning from Selection

When a vendor selects a MESO option, the system infers what they value most:

| Vendor Selects | Inferred Preference | Weight Adjustment |
|---|---|---|
| Offer 1 (Price-focused) | Price-sensitive | `price_weight += 10%` |
| Offer 2 (Terms-focused) | Cash-flow driven | `terms_weight += 10%` |
| Offer 3 (Balanced) | Delivery/quality priority | `delivery_weight += 10%` |

This inferred preference shapes future MESO cycles (if vendor clicks "Others" and continues negotiating).

### MESO Lifecycle State Machine

```
                           ┌────────────────────────┐
                           │                        │
                  ┌────────▼───────────┐            │
      ──────────►│NORMAL_NEGOTIATION  │            │
     Round 1-5   │                    │            │
                  └────────┬───────────┘            │
                           │ Round >= 5             │
                           ▼                        │
                  ┌────────────────────┐            │
              ┌──►│MESO_PRESENTATION  │            │
              │   └──┬─────────────┬──┘            │
              │      │             │                │
              │  Selects       Clicks               │
              │  option       "Others"              │
              │      │             │                │
              │      ▼             ▼                │
              │ ┌──────────┐ ┌───────────┐          │
              │ │  DEAL    │ │OTHERS_FORM│          │
              │ │ ACCEPTED │ └─────┬─────┘          │
              │ └──────────┘       │                │
              │                    │ Submit custom   │
              │                    │ offer           │
              │                    ▼                │
              │           ┌──────────────┐          │
              │           │ POST_OTHERS  │          │
              │           └──┬────────┬──┘          │
              │              │        │             │
              │     After 4  │  Stall detected     │
              │     text     │  (3+ rounds)        │
              │     rounds   │        │             │
              │              │        ▼             │
              └──────────────┘ ┌──────────────┐     │
                               │STALL_QUESTION│     │
                               └──┬────────┬──┘     │
                                  │        │        │
                       "Yes, this │   "No, I can    │
                        is final" │    adjust"      │
                                  │        │        │
                                  ▼        └────────┘
                          ┌──────────────┐
                          │ FINAL_MESO   │
                          └──┬────────┬──┘
                             │        │
                      Selects     No selection
                      option      + timeout
                             │        │
                             ▼        ▼
                     ┌──────────┐ ┌──────────┐
                     │  DEAL    │ │ ESCALATED│
                     │ ACCEPTED │ │          │
                     └──────────┘ └──────────┘
```

---

## 9. Vendor Portal Flow

The vendor portal is a **public, unauthenticated** interface accessed via a unique token link sent by email.

### Vendor Experience — Step by Step

```
┌───────────────────────────────────┐
│ Vendor receives email             │
│ with unique portal link           │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│Opens /vendor-chat/:token          │
│ (No login required)               │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│ Portal loads deal context          │
│ (RFQ details, PM's requirements)  │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐◄──────────────────────────┐
│ Text-based negotiation            │                           │
│ (Rounds 1-5)                      │                           │
└────────────────┬──────────────────┘                           │
                 ▼                                              │
┌───────────────────────────────────┐                           │
│ Vendor types offer:               │                           │
│ "$48K, Net 60, 20 days"           │                           │
└────────────────┬──────────────────┘                           │
                 ▼                                              │
┌───────────────────────────────────┐                           │
│ Message appears instantly         │                           │
│ (Phase 1: ~100ms)  (BLUE)        │                           │
└────────────────┬──────────────────┘                           │
                 ▼                                              │
┌───────────────────────────────────┐                           │
│ Typing indicator:                 │                           │
│"Procurement Manager is responding"│                           │
└────────────────┬──────────────────┘                           │
                 ▼                                              │
┌───────────────────────────────────┐                           │
│ AI PM response appears            │                           │
│ with counter-offer                │                           │
└────────────────┬──────────────────┘                           │
                 ▼                                              │
          ┌──────────────┐                                      │
          │ Round >= 5?  │                                      │
          └──┬────────┬──┘                                      │
          No │        │ Yes                                     │
             └────────┼─────────────────────────────────────────┘
                      │                                    (No)
                      ▼
        ┌──────────────────────────┐
        │ MESO Options Displayed   │
        └─────────────┬────────────┘
                      ▼
        ┌──────────────────────────┐
        │ 3 Offer Cards:           │
        │ - Price-Focused          │
        │ - Terms-Focused          │
        │ - Balanced               │
        └─────────────┬────────────┘
                      ▼
        ┌──────────────────────────┐
        │ Text input DISABLED      │
        │"Select an offer above    │
        │ or click Others"         │
        └─────────────┬────────────┘
                      ▼
               ┌──────────────┐
               │Vendor Action │
               └──┬────────┬──┘
                  │        │
           Select      Click
           Option     "Others"
                  │        │
                  ▼        ▼
     ┌──────────────┐  ┌──────────────────────┐
     │Deal ACCEPTED │  │ Custom form:          │
     │Show complet- │  │ Total Price +         │
     │ion screen    │  │ Payment Days          │
     │   (GREEN)    │  └──────────┬────────────┘
     └──────────────┘             │
                                  ▼
                       ┌──────────────────┐
                       │Submit custom     │
                       │offer             │
                       └────────┬─────────┘
                                ▼
                       ┌──────────────────┐
                       │4 more text       │──────► back to MESO
                       │rounds            │
                       └──────────────────┘
```

### Vendor Portal Input State Machine

```
                    ┌────────────────────────────┐
     Portal ────────►        text                │ Normal Composer visible
     opens          │  (Normal text input)       │
                    └──┬───────────┬──────────┬──┘
                       │           │          │
                  Send message  MESO opts  Deal terminal
                  (rounds 1-5)  arrive      state
                       │           │          │
                       │           ▼          ▼
                       │  ┌──────────────┐   [END]
                       │  │  disabled    │ MESO cards shown,
                       │  │             │ input disabled
                       │  └──┬───────┬──┘
                       │     │       │
                       │  Select   Click
                       │  MESO    "Others"
                       │  option     │
                       │     │       ▼
                       │     │  ┌──────────────┐
                       │     │  │ others_form  │ Price + terms
                       │     │  │              │ form visible
                       │     │  └──┬────────┬──┘
                       │     │     │        │
                       │     │  Cancel    Submit
                       │     │     │     (triggers
                       │     │     │    PM response)
                       │     │     │        │
                       └─────┼─────┼────────┘
                             │     │
                             │     └──────► back to "disabled"
                             │
                             └──────► Deal accepted (text → END)
```

---

## 10. PM (Procurement Manager) Interface Flow

### Navigation Hierarchy

```
┌─────────────────────────────────────────────────┐
│ Level 1: RequisitionListPage                     │
│ /chatbot/requisitions                            │
│ ─────────────────────────────                    │
│ All RFQs with deal counts                        │
│ Search, filter, sort, archive                    │
└──────────┬──────────────────────────────┬────────┘
           │ Click RFQ card              │ New Deal
           ▼                              ▼
┌──────────────────────────┐    ┌───────────────────────────────┐
│Level 2: RequisitionDeals │    │ Deal Wizard (4 Steps)          │
│/chatbot/requisitions/    │    │ /chatbot/requisitions/         │
│  :rfqId                  │    │  deals/new                     │
│──────────────────────    │    │                                │
│All vendor deals for RFQ  │◄───│                                │
│Status filter, summary    │    │ ┌──────────┐  ┌──────────┐    │
│         modal            │    │ │Step 1:   │  │Step 2:   │    │
└──────────┬───────────────┘    │ │Basic Info│─►│Commerc.  │    │
           │ Click deal card    │ │Title,Vend│  │Price,Qty,│    │
           ▼                    │ │Mode,Prior│  │Terms,Del.│    │
┌──────────────────────────┐    │ └──────────┘  └─────┬────┘    │
│Level 3: NegotiationRoom  │    │                     ▼         │
│/chatbot/requisitions/    │    │ ┌──────────┐  ┌──────────┐    │
│  :rfqId/vendors/:vendorId│    │ │Step 3:   │  │Step 4:   │    │
│  /deals/:dealId          │    │ │Contract &│◄─│Parameter │    │
│──────────────────────────│    │ │SLA, Warr.│  │Weights   │    │
│Live negotiation chat +   │    │ │Penalties │  │Importance│    │
│utility sidebar           │    │ │Quality   │  │Dist(100%)│    │
└──────────────────────────┘    │ └──────────┘  └──────────┘    │
                                └───────────────────────────────┘
```

### NegotiationRoom Layout (Viewport-Locked)

```
┌──────────────────────────────────────────────────────────────────┐
│  ← Back   Acme Corp - Electronics   ● NEGOTIATING   Round 5/8  │
│                                                    ↻ Refresh  ⟲ │
├──────────────────────────────────────────────────────────────────┤
│  [ Chat ]  [ Summary ]                                          │
├────────────────────────────────────┬─────────────────────────────┤
│                                    │  UTILITY SIDEBAR            │
│  ┌──────────────────────┐          │                             │
│  │ Vendor (10:15 AM)    │          │  ┌─────────────────────┐    │
│  │ We offer $48,000     │          │  │ ■■■■■■■■●■■■■■■■■■  │    │
│  │ Net 60, 20 days      │          │  │    58% → COUNTER    │    │
│  └──────────────────────┘          │  └─────────────────────┘    │
│                                    │                             │
│         ┌──────────────────────┐   │  Price Parameters           │
│         │ Accordo PM (10:15)   │   │  ├ Target: $45,000         │
│         │ Thank you. We        │   │  ├ Max: $50,000            │
│         │ counter at $47,000   │   │  └ Utility: 40% ■■■■□□    │
│         │ ┌────────────────┐   │   │                             │
│         │ │ COUNTER        │   │   │  Payment Terms              │
│         │ │ $47K|Net70|20d │   │   │  ├ Range: 30-90 days       │
│         │ └────────────────┘   │   │  └ Utility: 60% ■■■■■■□   │
│         └──────────────────────┘   │                             │
│                                    │  AI Reasoning               │
│  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐       │  ├ R5: COUNTER (58%)       │
│  │ ● ● ● Accordo typing...│       │  ├ R4: COUNTER (52%)       │
│  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘       │  └ R3: COUNTER (45%)       │
│                                    │                             │
├────────────────────────────────────┤                             │
│ Type your message...        Send ▸ │                             │
└────────────────────────────────────┴─────────────────────────────┘
```

### Real-Time Sidebar Updates

```
 NegotiationRoom   useDealActions    Backend API     Utility Sidebar
       │                │                │                │
       │                │                │                │
       │  ═══ On message sent or received ═══════════════════
       │                │                │                │
       │                │──GET /deals/   │                │
       │                │  :id/utility──►│                │
       │                │◄───────────────│                │
       │                │ {totalUtility: │                │
       │                │  0.58,         │                │
       │                │  paramUtils,   │                │
       │                │  thresholds}   │                │
       │                │                │                │
       │                │──GET /deals/   │                │
       │                │  :id/behavioral►                │
       │                │◄───────────────│                │
       │                │ {momentum: 0.4,│                │
       │                │  isConverging, │                │
       │                │  strategy}     │                │
       │                │                │                │
       │◄───────────────│                │                │
       │ Updated utility│                │                │
       │ + behavioral   │                │                │
       │                │                │                │
       │──Props update──────────────────────────────────►│
       │  triggers re-render             │                │
       │                │                │                │
       │                │                │  UnifiedUtilityBar
       │                │                │  animates dot to 58%
       │                │                │                │
       │                │                │  AI Reasoning adds
       │                │                │  new decision entry
       │                │                │                │
       │                │                │  Parameter sections
       │                │                │  update utility bars
       │                │                │                │
       │                │                │  Round dividers show
       │                │                │  strategy color:
       │                │                │  Green=Converging
       │                │                │  Yellow=Stalling
       │                │                │  Red=Diverging
```

---

## 11. Behavioral Analysis & Adaptive Strategy

### Signals Tracked Per Round

```
┌──────────────────────────────────────────────────────────────────────┐
│                       Behavioral Signals                              │
│                                                                       │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────┐ ┌──────┐ │
│  │ Momentum   │ │ Concession │ │Convergence │ │  Stall   │ │Senti-│ │
│  │ (-1 to +1) │ │ Velocity   │ │   Rate     │ │Detection │ │ment  │ │
│  │            │ │ ($/round)  │ │ (%/round)  │ │(boolean) │ │(pos/ │ │
│  │Is vendor   │ │How fast is │ │Is gap betw.│ │No progr. │ │ neu/ │ │
│  │moving      │ │vendor drop-│ │offers      │ │for 3+    │ │ neg) │ │
│  │toward us?  │ │ping price? │ │closing?    │ │rounds?   │ │      │ │
│  └─────┬──────┘ └─────┬──────┘ └──────┬─────┘ └────┬─────┘ └──┬───┘ │
└────────┼──────────────┼───────────────┼─────────────┼──────────┼─────┘
         │              │               │             │          │
         ▼              ▼               ▼             ▼          ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     Strategy Adaptations                              │
│                                                                       │
│  ┌────────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────┐ │
│  │ Aggressiveness │ │Concession    │ │  Round       │ │   Early    │ │
│  │  Adjustment    │ │  Bonus       │ │ Extension    │ │ Escalation │ │
│  │                │ │              │ │              │ │            │ │
│  │+15% if converg.│ │Extra flex if │ │Auto-extend   │ │If stalling │ │
│  │-10% if diverg. │ │vendor yielding│ │if convergence│ │at 60% of  │ │
│  │                │ │              │ │> 10%/round   │ │softMax     │ │
│  └────────────────┘ └──────────────┘ └──────────────┘ └────────────┘ │
│                                                                       │
│  Momentum ──► Aggressiveness     Concession Vel. ──► Concession Bonus │
│  Sentiment ──► Aggressiveness    Convergence Rate ──► Round Extension │
│                                  Stall Detection ──► Early Escalation │
└──────────────────────────────────────────────────────────────────────┘
```

### Dynamic Round Management

```
┌──────────────────────────────┐
│ Deal Starts                   │
│ softMax = 6, hardMax = 12     │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ Rounds 1-6                    │
│ (Normal negotiation)          │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ Round 6: softMax reached      │
└──────────────┬───────────────┘
               ▼
       ┌───────────────┐
       │ Convergence   │
       │ Rate > 10%?   │
       └──┬─────────┬──┘
       Yes│         │No
          ▼         ▼
┌──────────────┐ ┌──────────────┐
│ Auto-extend  │ │  Stalled     │
│ to softMax+2 │ │  3+ rounds?  │
└──────┬───────┘ └──┬────────┬──┘
       │         Yes│     No │
       │            ▼        ▼
       │    ┌──────────┐ ┌──────────┐
       │    │ ESCALATE │ │ Continue │
       │    │(Human PM │ │  (with   │
       │    │ reviews) │ │ caution) │
       │    │ (ORANGE) │ └──────────┘
       │    └──────────┘      ▲
       │                      │
       ▼                      │
  ┌──────────┐                │
  │ hardMax  │                │
  │ reached? │                │
  └──┬────┬──┘                │
  Yes│    │No                 │
     ▼    └───────────────────┘
┌──────────────┐
│ Force        │
│ terminal     │
│ action (RED) │
└──────────────┘
```

---

## 12. Offer Parsing & Multi-Currency Support

### Regex-Based Offer Extraction

The parser handles diverse vendor message formats:

| Vendor Message | Extracted Offer |
|---|---|
| `"We offer $50,000"` | `{total_price: 50000}` |
| `"₹45 lakhs, Net 60"` | `{total_price: 4500000, payment_terms: "Net 60"}` |
| `"29k net45 within 30 days"` | `{total_price: 29000, payment_terms: "Net 45", delivery_days: 30}` |
| `"€29.000 and 60 day terms"` | `{total_price: 29000, payment_terms: "Net 60"}` (EU format) |
| `"Can deliver ASAP"` | `{delivery_days: 2}` |
| `"early March delivery"` | `{delivery_date: "2026-03-10"}` (approximated) |

### Multi-Currency Conversion Flow

```
┌───────────────────────────────┐
│ Vendor message:                │
│ "₹37,50,000 Net 45"           │
└──────────────┬────────────────┘
               ▼
┌───────────────────────────────┐
│ Detect currency: INR          │
│ (₹ symbol)                    │
└──────────────┬────────────────┘
               ▼
       ┌───────────────┐
       │ Requisition   │
       │  currency?    │
       └──┬─────────┬──┘
          │         │
     Same (INR)  Different (USD)
          │         │
          ▼         ▼
┌──────────┐ ┌──────────────────────────────────┐
│Use as-is:│ │ Live FX Conversion                │
│₹37,50,000│ │ ₹37,50,000 / 83.5 = $44,910      │
└──────┬───┘ └──────────────┬───────────────────┘
       │                    ▼
       │            ┌───────────────┐
       │            │ FX Rate       │
       │            │  Cached?      │
       │            └──┬─────────┬──┘
       │            Yes│      No │
       │          (< 1hr)        │
       │               │        ▼
       │               │  ┌──────────────┐
       │               │  │ Fetch live   │
       │               │  │  rate        │
       │               │  └──┬────────┬──┘
       │               │  Succ│    Fail│
       │               │     ▼        ▼
       │        ┌──────┴──┐ ┌──────┐ ┌───────────────┐
       │        │Use cache│ │Use   │ │Use fallback   │
       │        │  rate   │ │live  │ │rate (hardcoded)│
       │        └────┬────┘ │rate  │ └───────┬───────┘
       │             │      └──┬───┘         │
       │             │         │             │
       └─────────────┼─────────┼─────────────┘
                     └─────────┼─────────────┘
                               ▼
              ┌──────────────────────────────────┐
              │ Store in offer:                   │
              │ {total_price: 44910,              │
              │  meta: {                          │
              │    currency_detected: 'INR',      │
              │    currency_converted: true,       │
              │    original_price: 3750000         │
              │  }}                                │
              └──────────────────────────────────┘
```

### Regional Number Format Support

| Format | Region | Example | Parsed Value |
|---|---|---|---|
| Indian lakhs | India | `1,50,000` | 150,000 |
| Indian with ₹ | India | `₹37,50,000` | 3,750,000 |
| EU thousands (dot) | Europe | `29.000` | 29,000 |
| EU decimal (comma) | Europe | `1.234,56` | 1,234.56 |
| US standard | US | `29,000.50` | 29,000.50 |
| K shorthand | Global | `50k` or `50K` | 50,000 |
| M shorthand | Global | `1.5M` | 1,500,000 |

---

## 13. Data Model & Persistence

### Core Entity Relationship

```
┌──────────────────┐     ┌──────────────────────────────────────────────────┐
│   REQUISITION    │     │                 CHATBOT_DEAL                      │
│──────────────────│     │──────────────────────────────────────────────────│
│ id          (PK) │     │ id                   (PK, uuid)                  │
│                  │     │ title                (string)                    │
│                  │     │ status               (enum: NEGOTIATING |        │
└────────┬─────────┘     │                       ACCEPTED | WALKED_AWAY |   │
         │               │                       ESCALATED)                 │
         │  1:N          │ round                (int)                       │
         └──────────────►│ mode                 (enum: INSIGHTS |           │
                         │                       CONVERSATION)              │
┌──────────────────┐     │ negotiation_config_json  (jsonb)                 │
│     VENDOR       │     │ latest_offer_json        (jsonb)                 │
│──────────────────│     │ latest_vendor_offer      (jsonb)                 │
│ id          (PK) │     │ convo_state_json         (jsonb)                 │
│                  │     │ latest_utility           (decimal)               │
│                  │  1:N│ latest_decision_action   (string)                │
└────────┬─────────┘     │ requisition_id      (FK)                        │
         └──────────────►│ vendor_id           (FK)                        │
                         │ user_id             (FK)                         │
┌──────────────────┐     │ contract_id         (FK)                        │
│      USER        │     │ archived_at         (timestamp)                 │
│──────────────────│  1:N│                                                  │
│ id          (PK) ├────►│                                                  │
│                  │     └────────┬───────────────────────┬─────────────────┘
└──────────────────┘              │                       │
                                  │ 1:N                   │ 1:N
┌──────────────────┐              │                       │
│    CONTRACT      │   1:1        │                       │
│──────────────────├─────────────►│                       │
│ id          (PK) │              │                       │
└──────────────────┘              │                       │
                                  ▼                       ▼
              ┌──────────────────────────┐  ┌──────────────────────────┐
              │    CHATBOT_MESSAGE        │  │      MESO_ROUND          │
              │──────────────────────────│  │──────────────────────────│
              │ id              (PK,uuid)│  │ id           (PK, uuid)  │
              │ deal_id         (FK)     │  │ deal_id      (FK)        │
              │ role  (enum: VENDOR |    │  │ round_number (int)       │
              │   ACCORDO | SYSTEM)      │  │ options_presented (jsonb)│
              │ content         (text)   │  │ selected_option_id (str) │
              │ extracted_offer (jsonb)  │  │ inferred_prefs   (jsonb) │
              │ engine_decision (jsonb)  │  │ is_final     (boolean)   │
              │ decision_action (string) │  │ created_at   (timestamp) │
              │ utility_score  (decimal) │  └──────────────────────────┘
              │ counter_offer   (jsonb)  │
              │ explainability_json(jsonb)│
              │ round           (int)    │
              │ created_at   (timestamp) │
              └──────────────────────────┘
```

### NegotiationConfig (JSONB — Stored in Deal)

```typescript
{
  parameters: {
    total_price: {
      weight: 0.35,               // From wizard Step 4
      direction: 'minimize',
      anchor: 38250,              // target × 0.85
      target: 45000,              // targetUnitPrice × quantity
      max_acceptable: 50000,      // maxPrice × quantity
      concession_step: 833        // (max - target) / maxRounds
    },
    payment_terms: {
      weight: 0.25,
      options: ['Net 30', 'Net 60', 'Net 90'],
      utility: { 'Net 30': 0.2, 'Net 60': 0.6, 'Net 90': 1.0 }
    },
    delivery_date: { weight: 0.20, ... },
    warranty:      { weight: 0.10, ... },
    quality:       { weight: 0.10, ... }
  },
  accept_threshold: 0.70,
  escalate_threshold: 0.50,
  walkaway_threshold: 0.30,
  max_rounds: 6,
  priority: 'MEDIUM',
  currency: 'USD',
  dynamicRounds: {
    softMaxRounds: 6,
    hardMaxRounds: 12,
    autoExtendEnabled: true
  }
}
```

---

## 14. API Endpoint Reference

### Deal Lifecycle

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/chatbot/requisitions/:rfqId/vendors/:vendorId/deals` | Create deal with full config |
| `GET` | `/api/chatbot/requisitions/:rfqId/vendors/:vendorId/deals/:dealId` | Get deal + messages |
| `GET` | `/api/chatbot/.../deals/:dealId/config` | Get negotiation config |
| `GET` | `/api/chatbot/.../deals/:dealId/utility` | Get weighted utility breakdown |
| `GET` | `/api/chatbot/.../deals/:dealId/summary` | Get deal summary |
| `POST` | `/api/chatbot/.../deals/:dealId/reset` | Reset deal (clear messages) |
| `POST` | `/api/chatbot/.../deals/:dealId/archive` | Archive deal |

### Two-Phase Messaging

| Method | Endpoint | Purpose | Latency |
|---|---|---|---|
| `POST` | `.../deals/:dealId/vendor-message-instant` | Save vendor message (Phase 1) | ~100ms |
| `POST` | `.../deals/:dealId/pm-response-async` | Generate PM response (Phase 2) | 5-15s |
| `POST` | `.../deals/:dealId/pm-response-fallback` | Fallback PM response (timeout) | ~200ms |

### MESO Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `.../deals/:dealId/meso/select` | Vendor selects MESO option → auto-accept |
| `POST` | `.../deals/:dealId/meso/others` | Vendor submits custom counter |
| `POST` | `.../deals/:dealId/final-offer/confirm` | "Is this your final offer?" response |

### Vendor Portal (Public — No Auth)

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/api/vendor-chat/:token` | Load deal (PM targets stripped) |
| `POST` | `/api/vendor-chat/:token/submit-quote` | Submit initial quote |
| `POST` | `/api/vendor-chat/:token/send-message` | Send vendor message |
| `POST` | `/api/vendor-chat/:token/pm-response` | Get AI PM response |
| `POST` | `/api/vendor-chat/:token/meso/select` | Select MESO option |
| `POST` | `/api/vendor-chat/:token/others` | Submit custom counter |

### Supporting Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/api/chatbot/requisitions` | List requisitions with deal counts |
| `GET` | `.../requisitions/:rfqId/deals` | Get all deals for RFQ |
| `GET` | `.../requisitions/:rfqId/vendors` | Get vendors for RFQ |
| `GET` | `.../vendors/:vendorId/smart-defaults` | Historical data for deal wizard |
| `POST` | `.../vendors/:vendorId/drafts` | Save wizard draft |

---

## 15. Security & Auditability

### Authentication & Authorization

```
┌──────────────────┐
│ Incoming Request  │
└────────┬─────────┘
         ▼
  ┌──────────────┐
  │ Route Type?  │
  └──┬────────┬──┘
     │        │
  Public   Protected
     │        │
     ▼        ▼
┌──────────────┐  ┌──────────────────────┐
│Token-Based   │  │JWT Authentication    │
│/vendor-chat/ │  │Authorization:        │
│  :token      │  │  Bearer <token>      │
│(Unique per   │  └──────────┬───────────┘
│ contract)    │             ▼
└──────┬───────┘  ┌──────────────────────┐
       │          │Decode JWT            │
       │          │{userId, userType,    │
       │          │ companyId}           │
       │          └──────────┬───────────┘
       │                     ▼
       │          ┌──────────────────────┐
       │          │Check RBAC Permissions│
       │          │role --> permissions   │
       │          │  --> module access   │
       │          └──────────┬───────────┘
       │                     │
       ▼                     ▼
┌──────────────┐  ┌──────────────────────┐
│Verify token  │  │  Request Proceeds    │
│exists + maps │  └──────────────────────┘
│to active     │             ▲
│contract      │             │
└──────┬───────┘             │
       ▼                     │
┌──────────────┐             │
│Strip PM      │             │
│targets from  │─────────────┘
│response      │
│(vendor never │
│ sees config) │
│  ** HARD **  │
└──────────────┘
```

### Audit Trail — What Gets Logged

| Layer | What's Logged | What's NOT Logged |
|---|---|---|
| **Negotiation Step Logger** | action, firmness, round, counterPrice, vendorTone, dealId, fromLlm | LLM prompts/responses, utility scores, vendor messages, PII |
| **Message Persistence** | Full message content, extracted offer, decision, utility, explainability | Intermediate calculations, rejected LLM outputs |
| **API Usage Logger** | LLM model, token count, latency, cost | Prompt content, response content |
| **Request Logger** | HTTP method, path, status, duration | Request body, response body |

### Data Isolation (Vendor Portal)

The vendor portal endpoint strips all PM-side data before returning:

- Vendor **sees**: Deal title, messages, MESO options, status
- Vendor **never sees**: Target price, max price, utility scores, weights, thresholds, behavioral analysis, PM strategy, decision reasons, config

---

## 16. Error Recovery & Fault Tolerance

### Multi-Layer Failsafe Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  Layer 1: LLM Call                                                 │
│  ┌─────────────────┐     ┌─────────────────┐     ┌──────────────┐ │
│  │ Try OpenAI      │──►  │ Try Ollama      │──►  │  Fallback    │ │
│  │ (gpt-3.5-turbo) │Fail │ (Local LLM)     │Fail │  Template    │ │
│  └─────────────────┘     └─────────────────┘     │(Deterministic│ │
│                                                   └──────┬───────┘ │
│                                                          │         │
│  Layer 2: Output Validation                              │         │
│  ┌─────────────────┐                                     │         │
│  │ Validate LLM    │                                     │         │
│  │ Output          │                                     │         │
│  │                 │── Banned words ─────────────────────►│         │
│  │                 │── Wrong price ──────────────────────►│         │
│  │                 │── Too long ─────────────────────────►│         │
│  └─────────────────┘                                     │         │
│                                                          │         │
│  Layer 3: Timeout Handling                               │         │
│  ┌─────────────────┐     ┌──────────────────────┐        │         │
│  │ 5s Client       │──►  │POST /pm-response-    │───────►│         │
│  │ Timeout         │     │      fallback         │        │         │
│  └─────────────────┘     └──────────────────────┘        │         │
│                                                          │         │
│  Layer 4: Currency Conversion                            │         │
│  ┌─────────────────┐     ┌─────────────────┐     ┌──────┴───────┐ │
│  │ Live FX Rate    │──►  │ Cached Rate     │──►  │ Hardcoded    │ │
│  │                 │Fail │ (< 1 hour old)  │Exp. │ Fallback Rate│ │
│  └─────────────────┘     └─────────────────┘     └──────────────┘ │
│                                                                    │
└───────────────────────────────────┬────────────────────────────────┘
                                    ▼
                     ┌───────────────────────────────┐
                     │  Response Delivered            │
                     │  Vendor never knows            │
                     │  which path was used  (GREEN)  │
                     └───────────────────────────────┘
```

### Failure Scenarios & Recovery

| Failure | Impact | Recovery |
|---|---|---|
| LLM timeout | No response within 4s | Silent fallback to template |
| LLM banned word | Response mentions "algorithm" | Silent fallback to template |
| LLM wrong price | Response says "$42K" when counter is $47K | Silent fallback to template |
| Both LLMs down | OpenAI + Ollama unavailable | Template always works (zero LLM dependency) |
| Currency API down | Can't convert ₹ → $ | Cached rate (1hr) → hardcoded fallback |
| DB write fails | Message not saved | Error returned to frontend, user retries |
| Offer parse fails | Can't extract price from message | ASK_CLARIFY action, vendor prompted for clarity |

---

## 17. Frontend Component Architecture

### Component Tree (Negotiation)

```
┌─────────────────────────────────────────────────────────────────────┐
│ App (Router)                                                         │
└──┬───────────────────────────────────────────────────────────────┬──┘
   │                                                               │
   ▼                                                               ▼
┌──────────────────────────────────────────────────────┐   ┌──────────────┐
│ DashBoardLayout (Sidebar + Content)                   │   │ VendorChat   │
└──┬──────────┬──────────────┬──────────────────────┬──┘   │(Public Portal│
   │          │              │                      │      └──┬───┬───┬──┘
   ▼          ▼              ▼                      ▼         │   │   │
┌────────┐ ┌────────┐ ┌───────────────┐ ┌────────────────┐   │   │   │
│Requis. │ │Requis. │ │Negotiation    │ │NewDealWizard   │   │   │   │
│List    │ │Deals   │ │Room           │ │(4-Step Config) │   │   │   │
│Page    │ │Page    │ │(Level 3: Live)│ │                │   │   │   │
│(Lvl 1) │ │(Lvl 2) │ └──┬──┬──┬──┬──┘ └──┬──┬──┬──┬───┘   │   │   │
└────────┘ └────────┘    │  │  │  │        │  │  │  │        │   │   │
                         │  │  │  │        │  │  │  │        │   │   │
              ┌──────────┘  │  │  └──┐     │  │  │  │        │   │   │
              ▼             │  │     ▼     ▼  ▼  ▼  ▼        │   │   │
        ┌──────────┐        │  │  ┌──────┐ S1 S2 S3 S4       │   │   │
        │useDeal   │        │  │  │Side- │                    │   │   │
        │Actions   │        │  │  │bar   │                    │   │   │
        │Hook      │        │  │  └──┬──┬┴───┐                │   │   │
        │(State+API│        │  │     │  │    │                │   │   │
        └──────────┘        │  │     ▼  ▼    ▼                │   │   │
                            │  │  ┌────┐┌──┐┌──────┐          │   │   │
              ┌─────────────┘  │  │Unif││AI││Param │          │   │   │
              ▼                │  │Util││Re││Sectn │          │   │   │
        ┌──────────┐           │  │Bar ││as││(Accrd│          │   │   │
        │ChatTrans-│           │  └────┘│on││ ion) │          │   │   │
        │cript     │           │        │TL│└──────┘          │   │   │
        │(Msg List)│           │        └──┘                  │   │   │
        └──┬───────┘           │                              │   │   │
           ▼                   ▼                              ▼   ▼   ▼
     ┌──────────┐       ┌──────────┐                   ┌────┐┌──┐┌────┐
     │Message   │       │Composer  │                   │Chat││ME││Othr│
     │Bubble    │       │(Text In) │                   │Tran││SO││Form│
     │(Per Msg) │       └──────────┘                   │scrp││Op││    │
     └──┬────┬──┘                                      └────┘│ts│└────┘
        │    │                                               └──┘
        ▼    ▼                                         ┌──────────┐
   ┌──────┐┌──────┐                                    │Composer  │
   │Decis.││Offer │                                    └──────────┘
   │Badge ││Card  │
   │(Color││(Cntr)│
   └──────┘└──────┘
```

### State Management Pattern

The frontend uses **no global state library** (no Redux, no Zustand). State is managed through:

1. **Custom Hooks** (`useDealActions`, `useConversation`) — encapsulate API calls + state
2. **React Context** — only for theme (dark/light mode)
3. **Component-local state** — UI state (tabs, modals, filters)
4. **Derived state** — `useMemo` for expensive calculations (reasoning timeline, adaptive config)

This keeps the architecture simple and avoids over-engineering while maintaining clean separation of concerns.

---

## 18. Performance & Scalability

### Frontend Optimizations

| Optimization | Technique | Impact |
|---|---|---|
| Viewport-locked layout | `h-screen overflow-hidden` | No browser reflow on messages |
| Independent scroll areas | Chat + sidebar scroll separately | Smooth UX |
| Smart auto-scroll | Only scroll if user near bottom (100px threshold) | Respects user reading position |
| Memoized calculations | `useMemo` for reasoning timeline, config | Prevents re-computation |
| Conditional polling | Poll only when `status === 'NEGOTIATING'` | Reduces API calls |
| Lazy-loaded Summary tab | Fetched on tab click, not on mount | Faster initial load |
| Two-phase messaging | Phase 1 is ~100ms | Instant perceived responsiveness |

### Backend Optimizations

| Optimization | Technique | Impact |
|---|---|---|
| Rate limiting | 100 req/15min per IP | DoS protection |
| Event loop protection | `toobusy-js` → 503 if lag detected | Server stability |
| Currency caching | 1-hour cache with fallback rates | Reduces external API calls |
| Connection pooling | Sequelize pool (default 5 connections) | DB efficiency |
| Server-side delay | Typing delay on server, not client | Consistent UX across devices |
| Auto-migration | Runs on startup | Zero-downtime deploys |
| Graceful shutdown | SIGTERM handler | Clean connection cleanup |

---

## 19. Technology Stack

### Backend

| Layer | Technology | Purpose |
|---|---|---|
| Runtime | Node.js 20+ | Server runtime |
| Framework | Express.js | HTTP server |
| Language | TypeScript (ES Modules) | Type safety |
| Database | PostgreSQL | Primary data store |
| ORM | Sequelize 6 | Database abstraction |
| LLM (Primary) | OpenAI API (gpt-3.5-turbo) | Natural language generation |
| LLM (Fallback) | Ollama (local, qwen3) | Self-hosted LLM fallback |
| Auth | JWT (jsonwebtoken) | Token-based authentication |
| Email | Nodemailer + AWS SES | Vendor notifications |
| Logging | Winston + daily rotation | Structured logging |
| Testing | Vitest (441+ unit tests) | Test framework |
| API Docs | Swagger / OpenAPI | Auto-generated docs |
| Security | Helmet, CORS, bcrypt | HTTP security |

### Frontend

| Layer | Technology | Purpose |
|---|---|---|
| Framework | React 19 | UI library |
| Build Tool | Vite 5.4 | Bundler + dev server |
| Language | TypeScript 5.9 (strict) | Type safety |
| Routing | React Router v7 | Client-side routing |
| Styling | Tailwind CSS 3.4 | Utility-first CSS |
| HTTP Client | Axios | API communication |
| Charts | Chart.js + react-chartjs-2 | Data visualization |
| Forms | react-hook-form + Zod | Form state + validation |
| PDF | jsPDF + jspdf-autotable | Client-side PDF generation |
| Notifications | react-hot-toast | Toast messages |
| Testing | Vitest + Testing Library | Component testing |

---

*Document generated March 2026 — Accordo-AI v2.0 (Feb 2026 Refactor)*
