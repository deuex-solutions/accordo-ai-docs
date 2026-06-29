# Accordo AI - Agentic Architecture

```mermaid
graph TD
    %% Define Styles
    classDef client fill:#3b82f6,stroke:#1e3a8a,stroke-width:2px,color:#fff;
    classDef api fill:#10b981,stroke:#047857,stroke-width:2px,color:#fff;
    classDef graph fill:#f59e0b,stroke:#b45309,stroke-width:2px,color:#fff;
    classDef agent fill:#8b5cf6,stroke:#5b21b6,stroke-width:1px,color:#fff;
    classDef db fill:#ef4444,stroke:#b91c1c,stroke-width:2px,color:#fff;
    classDef ext fill:#6b7280,stroke:#374151,stroke-width:2px,color:#fff;

    %% 1. Client Layer
    Frontend["React / Next.js Frontend"]:::client
    Email["Vendor Email (SMTP/IMAP)"]:::client

    %% 2. API & Orchestration Layer
    API["Backend API Layer (Express)"]:::api
    Orchestrator["LangGraph StateGraph Orchestrator<br/>(Controls Workflow & State)"]:::graph

    Frontend -->|REST / WebSockets| API
    API -->|REST / WebSockets| Frontend
    Email -->|Webhooks| API
    API -->|Invokes Graph with thread_id| Orchestrator
    Orchestrator -->|Returns Result| API

    %% 3. Persistence Layer
    PG_Business[("PostgreSQL<br/>Sequelize: Business Data")]:::db
    PG_Graph[("PostgresSaver<br/>LangGraph Checkpointer")]:::db
    VectorDB[("Vector DB<br/>Historical Deals")]:::db

    API --> PG_Business
    Orchestrator -->|State Snapshot / HITL| PG_Graph
    PG_Graph -->|Load State| Orchestrator

    %% 4. Agent Nodes (The Brains)
    subgraph Track_1 ["Track 1: Vatsal - Core Foundations"]
        Parse[OfferParsingAgent]:::agent
        Decide[DecisionAgent]:::agent
    end

    subgraph Track_2 ["Track 2: Yug - Intelligence & Voice"]
        Tone[ToneAnalysisAgent]:::agent
        Behavior[BehaviorAnalysisAgent]:::agent
        Concern[ConcernExtractionAgent]:::agent
        Respond[ResponseGenerationAgent]:::agent
        Validate[ValidationAgent]:::agent
    end

    subgraph Track_3 ["Track 3: Adarsh - Strategy & Math"]
        Profile[VendorProfilingAgent]:::agent
        Meso[MESOGenerationAgent]:::agent
        Stall[StallRecoveryAgent]:::agent
    end

    subgraph Track_4_5 ["Track 4 & 5 - RAG & Utilities"]
        Rag[RAGContextAgent]:::agent
        Search[VectorSearchAgent]:::agent
        Doc[DocumentGenAgent]:::agent
    end

    %% State flow within Graph
    Orchestrator --> Parse
    Parse --> Tone
    Parse --> Behavior
    Parse --> Concern
    Parse --> Profile
    Tone --> Decide
    Behavior --> Decide
    Concern --> Decide
    Profile --> Decide
    Decide -->|Counter / Meso| Meso
    Decide -->|Counter / Meso| Stall
    Meso --> Respond
    Stall --> Respond
    Respond --> Validate
    Validate -->|Pass / Fail Loop| Orchestrator
    
    %% RAG Integration
    Search --> VectorDB
    VectorDB --> Search
    Rag --> Search
    Rag --> Decide

    %% 5. External APIs
    LLM["LLM Services<br/>(OpenAI / Anthropic)"]:::ext
    Parse -->|API Calls| LLM
    Respond -->|API Calls| LLM
    Decide -->|API Calls| LLM
```

### Flow Explanation:
1. **Input:** A vendor message arrives (via Email or Frontend portal) and hits the **API Layer**.
2. **State Injection:** The API invokes the **LangGraph Orchestrator**, loading the persistent state via the **PostgresSaver Checkpointer**.
3. **Parsing & Intelligence:** The `OfferParsingAgent` extracts structural data. In parallel, `ToneAnalysis`, `BehaviorAnalysis`, `ConcernExtraction`, and `VendorProfiling` read the message to update the shared `NegotiationState`.
4. **Decision Engine:** The `DecisionAgent` evaluates all state signals to decide on an action (Accept, Counter, Escalate, Walk Away).
5. **Strategy formulation:** If countering, `MESOGenerationAgent` formulates Pareto-optimal options. 
6. **Voice & Safety:** `ResponseGenerationAgent` drafts the PM-persona email, which is vetted by the `ValidationAgent` before being dispatched.
7. **Human-in-the-Loop (HITL):** At any critical juncture (like an Escalation), the Graph halts, saving state to PostgreSQL, waiting for human intervention from the Frontend.
