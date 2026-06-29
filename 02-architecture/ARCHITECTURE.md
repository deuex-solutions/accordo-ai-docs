# Accordo AI Chatbot - System Architecture

**Complete system architecture documentation**

Last Updated: January 4, 2026

---

## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Component Overview](#component-overview)
3. [Data Flow Diagrams](#data-flow-diagrams)
4. [Database Schema Overview](#database-schema-overview)
5. [API Architecture](#api-architecture)
6. [Frontend Architecture](#frontend-architecture)
7. [Backend Module Structure](#backend-module-structure)
8. [Technology Stack](#technology-stack)

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                 │
│  ┌────────────────┐  ┌─────────────────┐  ┌────────────────────┐  │
│  │  Web Browser   │  │  Mobile Browser │  │  Desktop App       │  │
│  │  (React SPA)   │  │  (Responsive)   │  │  (Future)          │  │
│  └────────┬───────┘  └────────┬────────┘  └──────────┬─────────┘  │
└───────────┼──────────────────┼────────────────────────┼────────────┘
            │                  │                        │
            └──────────────────┴────────────────────────┘
                               │
                        HTTP/HTTPS (REST API)
                               │
┌──────────────────────────────┼────────────────────────────────────────┐
│                      API GATEWAY LAYER                                 │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Express.js Server (Port 8000)                                   │ │
│  │  - CORS, Helmet, Rate Limiting                                   │ │
│  │  - JWT Authentication Middleware                                 │ │
│  │  - Request Logging & Error Handling                              │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────┼────────────────────────────────────────┘
                                │
┌───────────────────────────────┼────────────────────────────────────────┐
│                       APPLICATION LAYER                                │
│  ┌──────────────────┐   ┌──────────────────┐   ┌─────────────────┐  │
│  │  Auth Module     │   │  Chatbot Module  │   │  Contract Module│  │
│  │  - Sign In/Up    │   │  - Deal Mgmt     │   │  - CRUD Ops     │  │
│  │  - JWT Tokens    │   │  - Messaging     │   │  - Status Mgmt  │  │
│  └──────────────────┘   │  - Decision Eng  │   └─────────────────┘  │
│                         │  - Vendor Sim    │                         │
│  ┌──────────────────┐   │  - Conversation  │   ┌─────────────────┐  │
│  │  Requisition     │   └──────────────────┘   │  Vendor Module  │  │
│  │  Module          │                          │  - Vendor CRUD  │  │
│  └──────────────────┘   ┌──────────────────┐   └─────────────────┘  │
│                         │  Email Service   │                         │
│                         │  - Notifications │                         │
│                         │  - Templates     │                         │
│                         └──────────────────┘                         │
└───────────────────────────────┼────────────────────────────────────────┘
                                │
┌───────────────────────────────┼────────────────────────────────────────┐
│                        BUSINESS LOGIC LAYER                            │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  Decision Engine (INSIGHTS Mode)                                 │ │
│  │  - Offer Parsing (Regex)                                         │ │
│  │  - Utility Calculation (Price + Terms)                           │ │
│  │  - Decision Algorithm (ACCEPT/COUNTER/WALK_AWAY)                 │ │
│  │  - Explainability Generation                                     │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  Conversation Manager (CONVERSATION Mode)                        │ │
│  │  - Intent Classification (GREET, OFFER, REFUSAL, etc.)          │ │
│  │  - State Machine (WAITING_FOR_OFFER → NEGOTIATING → TERMINAL)   │ │
│  │  - Refusal Handling (NO, LATER, CONFUSED)                        │ │
│  │  - Context Tracking                                              │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  Vendor Simulator                                                │ │
│  │  - Scenario Detection (HARD, SOFT, WALK_AWAY)                    │ │
│  │  - Policy Constraints (min price, concession step)               │ │
│  │  - Auto-Reply Generation                                         │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────┼────────────────────────────────────────┘
                                │
┌───────────────────────────────┼────────────────────────────────────────┐
│                        DATA ACCESS LAYER                               │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  Sequelize ORM (TypeScript)                                      │ │
│  │  - Models: Deal, Message, Template, Contract, User              │ │
│  │  - Associations: One-to-Many, Many-to-Many                       │ │
│  │  - Migrations: Version-controlled schema changes                 │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────┼────────────────────────────────────────┘
                                │
┌───────────────────────────────┼────────────────────────────────────────┐
│                        EXTERNAL SERVICES LAYER                         │
│  ┌────────────────┐   ┌─────────────────┐   ┌────────────────────┐  │
│  │  PostgreSQL    │   │  Ollama LLM     │   │  SMTP Server       │  │
│  │  (Database)    │   │  (llama3.1)     │   │  (Email)           │  │
│  │  Port: 5432    │   │  Port: 11434    │   │  Port: 587         │  │
│  └────────────────┘   └─────────────────┘   └────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Component Overview

### Frontend Components

**Technology**: React 19 + TypeScript + Vite

**Core Modules**:

1. **Authentication**
   - Sign In/Sign Up pages
   - JWT token management
   - Protected routes
   - Token refresh logic

2. **Chatbot UI**
   - DealsPage (list view)
   - NegotiationRoom (INSIGHTS mode)
   - ConversationRoom (CONVERSATION mode)
   - ArchivedDealsPage
   - TrashPage
   - SummaryPage

3. **Components**
   - ChatTranscript (message list)
   - Composer (message input)
   - MessageBubble (role-based styling)
   - DecisionBadge (action indicators)
   - OfferCard (offer display)
   - ExplainDrawer (decision breakdown)

4. **State Management**
   - Custom hooks (useDeal, useDealActions, useConversation)
   - React Context (ThemeContext)
   - Local/session storage

### Backend Components

**Technology**: Node.js + Express.js + TypeScript

**Core Modules**:

1. **Chatbot Module** (`src/modules/chatbot/`)
   - Controller (17 endpoint handlers)
   - Service (15 business logic functions)
   - Repository (12 database queries)
   - Routes (23 REST endpoints)
   - Validators (8 Joi schemas)

2. **Decision Engine** (`src/modules/chatbot/engine/`)
   - Offer parsing (regex-based)
   - Utility calculation (price + terms)
   - Decision algorithm (threshold-based)
   - Explainability generation

3. **Conversation Engine** (`src/modules/chatbot/convo/`)
   - Intent classification (10 types)
   - State machine (4 phases)
   - Refusal handling (4 types)
   - LLM reply generation

4. **Vendor Simulator** (`src/modules/chatbot/vendor/`)
   - Scenario detection (HARD/SOFT/WALK_AWAY)
   - Policy enforcement
   - Auto-reply generation

5. **Supporting Services**
   - LLM service (Ollama integration)
   - Email service (SMTP with retry)
   - Logger (Winston)
   - Error handler

---

## Data Flow Diagrams

### INSIGHTS Mode Flow

```
┌─────────────┐
│   Client    │
│  (Browser)  │
└──────┬──────┘
       │ POST /api/chatbot/deals/{id}/messages
       │ { content: "I can offer $95 with Net 45" }
       ▼
┌──────────────────────────────────────┐
│   Express.js API Gateway             │
│   - Validate JWT                     │
│   - Extract user context             │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│   Chatbot Controller                 │
│   - Validate request body            │
│   - Extract dealId from params       │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│   Chatbot Service                    │
│   processVendorTurn()                │
└──────┬───────────────────────────────┘
       │
       ├─────────────────────────────────┐
       │                                 │
       ▼                                 ▼
┌─────────────────┐            ┌────────────────────┐
│  Parse Offer    │            │  Get Deal Config   │
│  (Regex)        │            │  from Database     │
│  Extract:       │            │  - Price params    │
│  - unit_price   │            │  - Terms params    │
│  - payment_terms│            │  - Thresholds      │
└─────────┬───────┘            └─────────┬──────────┘
          │                              │
          └──────────┬───────────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  Calculate Utility   │
          │  - Price utility     │
          │  - Terms utility     │
          │  - Weighted sum      │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  Decision Algorithm  │
          │  IF utility >= 75:   │
          │    ACCEPT            │
          │  ELIF utility < 30:  │
          │    WALK_AWAY         │
          │  ELSE:               │
          │    COUNTER           │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  Generate Counter    │
          │  Offer (if COUNTER)  │
          │  - Calculate price   │
          │  - Select terms      │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  Create Messages     │
          │  1. Vendor message   │
          │  2. Accordo reply    │
          │  (both in database)  │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  Update Deal State   │
          │  - Increment round   │
          │  - Update status     │
          │  - Store latest offer│
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  Return to Client    │
          │  {                   │
          │    vendorMessage,    │
          │    accordoMessage,   │
          │    deal              │
          │  }                   │
          └──────────────────────┘
```

### CONVERSATION Mode Flow

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ POST /api/chatbot/conversation/deals/{id}/messages
       │ { content: "No, I can't share that now" }
       ▼
┌──────────────────────────────────────┐
│   Conversation Service               │
│   processConversationMessage()       │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│   Step 1: Get Conversation State     │
│   - phase: WAITING_FOR_OFFER         │
│   - refusalCount: 2                  │
│   - lastVendorOffer: null            │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│   Step 2: Classify Refusal           │
│   Pattern: /\b(no|can't)\b/          │
│   Result: REFUSAL (type: NO)         │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│   Step 3: Update State               │
│   refusalCount: 2 → 3                │
│   lastRefusalType: NO                │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│   Step 4: Determine Intent           │
│   refusalCount >= 3 → ESCALATE       │
│   Otherwise → HANDLE_REFUSAL         │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│   Step 5: Build Conversation History│
│   Format: [                          │
│     { role: "vendor", content: "..." },│
│     { role: "accordo", content: "..."}│
│   ]                                  │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│   Step 6: Generate Reply via LLM     │
│   Ollama API Call                    │
└──────┬───────────────────────────────┘
       │
       ├─────────────────────────────────┐
       │                                 │
       ▼                                 ▼
┌─────────────────┐            ┌────────────────────┐
│  LLM Success    │            │  LLM Failed        │
│  Validate reply │            │  Use fallback      │
│  Check keywords │            │  template          │
└─────────┬───────┘            └─────────┬──────────┘
          │                              │
          └──────────┬───────────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  Save Messages       │
          │  - Vendor message    │
          │  - Accordo reply     │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  Update Deal State   │
          │  - convoStateJson    │
          │  - status (if term.) │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │  Return to Client    │
          │  {                   │
          │    vendorMessage,    │
          │    accordoMessage,   │
          │    conversationState,│
          │    revealAvailable   │
          │  }                   │
          └──────────────────────┘
```

---

## Database Schema Overview

### Entity Relationship Diagram

```
┌─────────────────────────┐
│        Users            │
│─────────────────────────│
│ id (PK)                 │
│ name                    │
│ email                   │
│ password_hash           │
│ user_type               │
│ company_id (FK)         │
└─────────┬───────────────┘
          │
          │ 1:N
          │
          ▼
┌─────────────────────────┐
│   Contracts             │
│─────────────────────────│
│ id (PK)                 │
│ requisition_id (FK)     │
│ vendor_id (FK)          │
│ chatbot_deal_id (FK)    │──┐
│ status                  │  │
└─────────────────────────┘  │
                             │ 1:1
                             │
          ┌──────────────────┘
          │
          ▼
┌─────────────────────────────────────┐
│      chatbot_deals                  │
│─────────────────────────────────────│
│ id (PK, UUID)                       │
│ title                               │
│ counterparty                        │
│ status (NEGOTIATING/ACCEPTED/etc.)  │
│ round                               │
│ mode (INSIGHTS/CONVERSATION)        │
│ latest_offer_json (JSONB)           │
│ convo_state_json (JSONB)            │
│ requisition_id (FK, nullable)       │
│ contract_id (FK, nullable)          │
│ user_id (FK)                        │
│ vendor_id (FK, nullable)            │
│ archived_at                         │
│ deleted_at                          │
└──────────┬──────────────────────────┘
           │
           │ 1:N
           │
           ▼
┌─────────────────────────────────────┐
│     chatbot_messages                │
│─────────────────────────────────────│
│ id (PK, UUID)                       │
│ deal_id (FK)                        │
│ role (VENDOR/ACCORDO/SYSTEM)        │
│ content (TEXT)                      │
│ extracted_offer (JSONB)             │
│ engine_decision (JSONB)             │
│ decision_action                     │
│ utility_score                       │
│ counter_offer (JSONB)               │
│ explainability_json (JSONB)         │
│ created_at                          │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│    chatbot_templates                │
│─────────────────────────────────────│
│ id (PK, UUID)                       │
│ name                                │
│ description                         │
│ is_default                          │
└──────────┬──────────────────────────┘
           │
           │ 1:N
           │
           ▼
┌─────────────────────────────────────┐
│  chatbot_template_parameters        │
│─────────────────────────────────────│
│ id (PK)                             │
│ template_id (FK)                    │
│ config_json (JSONB)                 │
└─────────────────────────────────────┘
```

### Key Relationships

- **User → Deal**: One user can have many deals (1:N)
- **Contract → Deal**: One contract links to one deal (1:1)
- **Deal → Messages**: One deal has many messages (1:N)
- **Template → Parameters**: One template has many parameter sets (1:N)

---

## API Architecture

### RESTful Design Principles

**Base URL**: `http://localhost:8000/api`

**Authentication**: JWT Bearer Token in `Authorization` header

**Response Format**:
```json
{
  "success": true,
  "data": { ... },
  "message": "Optional success message"
}
```

**Error Format**:
```json
{
  "success": false,
  "error": "Error description",
  "details": { ... }
}
```

### Endpoint Organization

```
/api
├── /auth
│   ├── POST /sign-in
│   ├── POST /sign-up
│   └── POST /refresh
│
├── /chatbot
│   ├── /deals
│   │   ├── GET /                           # List deals
│   │   ├── POST /                          # Create deal
│   │   ├── GET /:dealId                    # Get deal
│   │   ├── GET /:dealId/config             # Get config
│   │   ├── POST /:dealId/messages          # Send message (INSIGHTS)
│   │   ├── POST /:dealId/reset             # Reset deal
│   │   ├── POST /:dealId/archive           # Archive deal
│   │   ├── POST /:dealId/unarchive         # Unarchive
│   │   ├── POST /:dealId/soft-delete       # Soft delete
│   │   ├── POST /:dealId/restore           # Restore
│   │   └── DELETE /:dealId/permanent       # Permanent delete
│   │
│   ├── /conversation
│   │   └── /deals/:dealId
│   │       ├── POST /start                 # Start conversation
│   │       ├── POST /messages              # Send message (CONVERSATION)
│   │       └── GET /explainability         # Get decision breakdown
│   │
│   └── /vendor
│       └── /deals/:dealId
│           └── POST /next                  # Auto-reply
│
├── /contracts
│   ├── GET /
│   ├── POST /
│   └── PATCH /:id/status
│
└── /requisitions
    ├── GET /
    └── POST /
```

---

## Frontend Architecture

### Component Hierarchy

```
App.tsx
├── ThemeProvider
│   └── Router
│       ├── Public Routes
│       │   ├── HomePage
│       │   ├── SignIn
│       │   └── SignUp
│       │
│       └── Protected Routes (DashboardLayout)
│           ├── Dashboard
│           ├── ChatLayout
│           │   ├── DealsPage
│           │   ├── NewDealPage
│           │   ├── NegotiationRoom (INSIGHTS)
│           │   │   ├── ChatTranscript
│           │   │   │   └── MessageBubble (with DecisionBadge)
│           │   │   ├── Composer (with scenario chips)
│           │   │   └── NegotiationConfigPanel
│           │   │
│           │   ├── ConversationRoom (CONVERSATION)
│           │   │   ├── ChatTranscript
│           │   │   │   └── ConversationMessageBubble
│           │   │   ├── Composer (clean)
│           │   │   └── ExplainDrawer
│           │   │
│           │   ├── ArchivedDealsPage
│           │   ├── TrashPage
│           │   └── SummaryPage
│           │       ├── OutcomeBanner
│           │       ├── MetricsCards
│           │       └── ExportButtons
│           │
│           ├── RequisitionManagement
│           ├── ContractManagement
│           └── VendorManagement
```

### State Management Strategy

**Global State**:
- Theme (ThemeContext)
- Authentication (localStorage + axios interceptors)

**Local State**:
- Component-level useState/useReducer
- Custom hooks for data fetching

**Server State**:
- Managed via API calls (no Redux/Zustand)
- Caching in custom hooks

---

## Backend Module Structure

### Chatbot Module Structure

```
src/modules/chatbot/
├── index.ts                    # Module entry point
├── chatbot.controller.ts       # 17 route handlers
├── chatbot.service.ts          # 15 business logic functions
├── chatbot.repo.ts             # 12 database queries
├── chatbot.routes.ts           # 23 REST endpoints
├── chatbot.validator.ts        # 8 Joi schemas
├── chatbot.configMapper.ts     # Requisition → Config mapper
│
├── engine/                     # Decision Engine (INSIGHTS)
│   ├── types.ts
│   ├── config.ts
│   ├── parseOffer.ts           # Regex-based offer extraction
│   ├── utility.ts              # Utility calculation functions
│   ├── decide.ts               # Decision algorithm
│   └── processVendorTurn.ts    # Demo turn processor
│
├── convo/                      # Conversation Engine
│   ├── types.ts
│   ├── conversationManager.ts  # Intent + state logic
│   ├── conversationService.ts  # 15-step pipeline
│   ├── conversationTemplates.ts# 56 response templates
│   ├── llamaReplyGenerator.ts  # LLM integration
│   └── processConversationTurn.ts
│
├── llm/
│   └── chatbotLlamaClient.ts   # Dedicated Ollama client
│
├── vendor/                     # Vendor Simulation
│   ├── types.ts
│   ├── vendorPolicy.ts         # Scenario policies
│   ├── scenarioDetector.ts     # Auto-detect behavior
│   └── vendorAgent.ts          # LLM-driven simulator
│
└── utils/
    └── errorRecovery.ts        # Retry + fallback logic
```

---

## Technology Stack

### Frontend Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Framework | React | 19.2.3 | UI library |
| Build Tool | Vite | 5.4.11 | Fast dev server + bundler |
| Language | TypeScript | 5.9.3 | Type safety |
| Routing | React Router | 7.4.0 | Client-side routing |
| Styling | Tailwind CSS | 3.4.15 | Utility-first CSS |
| Forms | react-hook-form | 7.54.2 | Form validation |
| Validation | Yup / Zod | 1.6.1 / 3.24.1 | Schema validation |
| HTTP Client | Axios | 1.7.9 | API requests |
| Charts | Chart.js | 4.4.7 | Data visualization |
| PDF Export | jsPDF | 3.0.4 | Client-side PDF generation |
| Date Utilities | date-fns | 4.1.0 | Date formatting |
| Icons | Lucide React | 0.485.0 | Icon library |
| Notifications | react-hot-toast | 2.4.1 | Toast messages |

### Backend Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| Runtime | Node.js | 18+ | JavaScript runtime |
| Framework | Express.js | Latest | Web framework |
| Language | TypeScript | 5.9.3 | Type safety |
| ORM | Sequelize | 6.37.7 | Database ORM |
| Database | PostgreSQL | 14+ | Relational database |
| Authentication | jsonwebtoken | Latest | JWT tokens |
| Validation | Joi | Latest | Request validation |
| Logger | Winston | Latest | Logging |
| Email | Nodemailer | Latest | Email sending |
| LLM | Ollama | Latest | Local LLM server |
| LLM Model | llama3.1 | Latest | Language model |
| HTTP Client | Axios | Latest | External API calls |

### Development Tools

| Tool | Purpose |
|------|---------|
| ESLint | Code linting |
| Prettier | Code formatting |
| ts-node-dev | TypeScript development |
| Sequelize CLI | Database migrations |
| Vitest | Unit testing (frontend) |
| Docker | Containerization |

---

## Deployment Architecture

### Production Setup

```
┌────────────────────────────────────────────────────────┐
│                    Load Balancer                        │
│                   (Nginx/CloudFlare)                    │
└────────────────────┬───────────────────────────────────┘
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
┌────────────────┐    ┌────────────────┐
│ Frontend       │    │ Frontend       │
│ (Static Files) │    │ (Static Files) │
│ Vercel/Netlify │    │ (CDN Cache)    │
└────────────────┘    └────────────────┘

          │
          ▼
┌────────────────────────────────────────────────────────┐
│                    API Load Balancer                    │
└────────────────────┬───────────────────────────────────┘
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
┌────────────────┐    ┌────────────────┐
│ Backend        │    │ Backend        │
│ (Node.js)      │    │ (Node.js)      │
│ EC2/Railway    │    │ EC2/Railway    │
└────────┬───────┘    └────────┬───────┘
         │                     │
         └──────────┬──────────┘
                    │
          ┌─────────┴──────────┬──────────┐
          │                    │          │
          ▼                    ▼          ▼
┌─────────────────┐  ┌─────────────┐  ┌──────────┐
│ PostgreSQL      │  │ Ollama LLM  │  │ SMTP     │
│ (RDS/Supabase)  │  │ (Dedicated) │  │ (SendGrid│
└─────────────────┘  └─────────────┘  └──────────┘
```

---

## Security Architecture

### Authentication Flow

```
1. User Sign In
   → POST /api/auth/sign-in
   → Backend validates credentials
   → Generate JWT access token (24h) + refresh token (7d)
   → Return tokens to client

2. Client stores tokens
   → localStorage: %accessToken%, %refreshToken%

3. Protected API Request
   → Client sends: Authorization: Bearer <accessToken>
   → Backend middleware validates token
   → Extract userId, userType from payload
   → Attach to req.context
   → Proceed to controller

4. Token Expired
   → Backend returns 401
   → Frontend intercepts
   → POST /api/auth/refresh with refreshToken
   → Get new accessToken
   → Retry original request

5. Refresh Token Expired
   → Redirect to /sign-in
   → User must login again
```

### Data Security

- **Passwords**: Hashed with bcrypt (10 rounds)
- **JWT**: Signed with HS256 algorithm
- **HTTPS**: Required in production
- **CORS**: Whitelist frontend origins
- **Rate Limiting**: 100 requests/15min per IP
- **SQL Injection**: Prevented by Sequelize parameterization
- **XSS**: Sanitized in frontend rendering

---

## Performance Considerations

### Backend Optimization

- Database connection pooling (max: 20)
- Lazy loading of associations
- Indexed columns (dealId, userId, status, created_at)
- LLM response caching (future)
- Pagination (default: 10 items/page)

### Frontend Optimization

- Code splitting by route
- Lazy loading components
- Debounced search inputs
- Optimistic UI updates
- Local state caching
- Tree-shaken dependencies

### LLM Optimization

- Timeout: 30s
- Max tokens: 500
- Temperature: 0.7
- Fallback templates for failures
- GPU acceleration (if available)

---

## Monitoring & Logging

### Logging Strategy

**Backend (Winston)**:
- Console logging (development)
- File logging (production)
  - `logs/app.log` (all levels)
  - `logs/error.log` (errors only)
- Log levels: error, warn, info, http, debug

**Frontend**:
- Browser console (development)
- Error tracking service (Sentry - production)

### Metrics to Track

- API response times
- LLM inference times
- Database query times
- Deal creation rate
- Message send rate
- Success/escalation ratio
- Average negotiation rounds
- User engagement metrics

---

## Scalability Considerations

### Horizontal Scaling

- Stateless backend (can run multiple instances)
- Load balancer distributes requests
- Shared PostgreSQL database
- Session data in JWT (no server-side sessions)

### Database Scaling

- Read replicas for listing queries
- Write master for mutations
- Partitioning by date (messages table)
- Archiving old deals

### LLM Scaling

- Dedicated Ollama server
- Multiple model instances
- Request queuing
- Timeout handling
- Fallback to templates

---

## See Also

- [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) - Detailed schema documentation
- [API_ENDPOINTS.md](./API_ENDPOINTS.md) - Complete API reference
- [FRONTEND_COMPONENTS.md](./FRONTEND_COMPONENTS.md) - Component documentation
- [BACKEND_MODULES.md](./BACKEND_MODULES.md) - Backend module reference
- [LLM_INTEGRATION.md](./LLM_INTEGRATION.md) - LLM setup and usage
