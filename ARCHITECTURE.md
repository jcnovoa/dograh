# Dograh AI — Architecture Overview

## System Context

Dograh is an open-source, self-hostable voice AI platform for building conversational agents with telephony and WebRTC support. Users define conversation flows as directed graphs (workflows), connect phone numbers, and Dograh orchestrates real-time STT → LLM → TTS pipelines.

---

## High-Level Architecture

```mermaid
graph TB
    subgraph Clients
        Browser[Browser / Dashboard]
        SDK_PY[Python SDK]
        SDK_TS[TypeScript SDK]
        MCP_Client[MCP AI Agent]
        Phone[Phone / SIP]
    end

    subgraph Edge["Edge / Ingress"]
        Cloudflared[Cloudflared Tunnel]
        Nginx[Nginx HTTPS Proxy]
        Coturn[Coturn TURN/STUN]
    end

    subgraph Frontend["Frontend (Next.js 15 — :3010)"]
        UI[Next.js App Router]
        ReactFlow[React Flow Workflow Builder]
        Zustand[Zustand State]
    end

    subgraph Backend["Backend (FastAPI — :8000)"]
        API[FastAPI REST API]
        MCP_Server[MCP Server]
        Pipecat[Pipecat Voice Pipeline]
        WorkflowEngine[Workflow Graph Engine]
        TelephonyService[Telephony Service]
        CampaignService[Campaign Service]
        TaskWorker[ARQ Task Worker]
    end

    subgraph Data["Data Layer"]
        Postgres[(PostgreSQL 17 + pgvector)]
        Redis[(Redis 7)]
        MinIO[(MinIO S3)]
    end

    subgraph External["External Services"]
        STT[STT Providers]
        LLM[LLM Providers]
        TTS[TTS Providers]
        Telephony[Telephony Providers]
    end

    %% Client connections
    Browser --> Nginx
    Browser --> UI
    SDK_PY --> API
    SDK_TS --> API
    MCP_Client --> MCP_Server
    Phone --> Telephony

    %% Edge routing
    Nginx --> UI
    Nginx --> API
    Cloudflared --> API
    Coturn --> Pipecat

    %% Frontend to Backend
    UI -->|REST + WebSocket| API

    %% Backend internal
    API --> WorkflowEngine
    API --> TelephonyService
    API --> CampaignService
    API --> TaskWorker
    WorkflowEngine --> Pipecat
    TelephonyService --> Pipecat
    CampaignService --> TaskWorker

    %% Data access
    API --> Postgres
    API --> Redis
    TaskWorker --> Redis
    TaskWorker --> MinIO
    Pipecat --> MinIO
    WorkflowEngine --> Postgres

    %% External integrations
    Pipecat --> STT
    Pipecat --> LLM
    Pipecat --> TTS
    TelephonyService --> Telephony
    Telephony --> Cloudflared
```

---

## Voice Call Flow (Core Loop)

```mermaid
sequenceDiagram
    participant Caller
    participant TelProvider as Telephony Provider<br/>(Twilio/Vonage/Telnyx/etc.)
    participant API as Dograh API
    participant Pipeline as Pipecat Pipeline
    participant STT as STT Provider
    participant LLM as LLM Provider
    participant TTS as TTS Provider
    participant DB as PostgreSQL
    participant S3 as MinIO (S3)

    Caller->>TelProvider: Dials / Answers
    TelProvider->>API: Webhook (call connected)
    API->>Pipeline: Start voice pipeline
    Pipeline->>DB: Load workflow definition

    loop Real-time Conversation
        Caller->>TelProvider: Speech audio
        TelProvider->>Pipeline: Audio stream
        Pipeline->>STT: Audio frames
        STT-->>Pipeline: Transcribed text
        Pipeline->>LLM: Transcript + node prompt + context
        LLM-->>Pipeline: Response + edge evaluation
        Pipeline->>TTS: Response text
        TTS-->>Pipeline: Synthesized audio
        Pipeline->>TelProvider: Audio stream
        TelProvider->>Caller: Agent speaks
    end

    Note over Pipeline: Edge condition met → transition node
    Pipeline->>DB: Save run record (transcript, context)
    Pipeline->>S3: Upload recording audio
    Pipeline->>API: Post-call webhooks & integrations
```

---

## Backend Architecture

```mermaid
graph TB
    subgraph Routes["API Routes (/api/v1)"]
        R_Telephony[telephony]
        R_Workflow[workflow]
        R_Campaign[campaign]
        R_User[user]
        R_Org[organization]
        R_Tools[tool]
        R_KB[knowledge_base]
        R_WebRTC[webrtc_signaling]
        R_Recording[workflow_recording]
        R_Reports[reports]
        R_Auth[auth]
        R_Embed[public_embed / public_agent]
        R_S3[s3_signed_url]
        R_TextChat[workflow_text_chat]
    end

    subgraph Services["Service Layer"]
        S_Pipecat[pipecat/]
        S_Workflow[workflow/]
        S_Telephony[telephony/]
        S_Campaign[campaign/]
        S_Integrations[integrations/]
        S_Auth[auth/]
        S_GenAI[gen_ai/]
        S_Pricing[pricing/]
        S_Storage[storage]
        S_Config[configuration/]
        S_WorkerSync[worker_sync/]
        S_KB[filesystem/ + knowledge_base]
    end

    subgraph DB_Layer["Data Access (db/)"]
        DB_Models[SQLAlchemy Models]
        DB_Clients[Domain Clients]
        DB_Alembic[Alembic Migrations]
    end

    subgraph Tasks["Background Tasks (ARQ)"]
        T_KB[knowledge_base_processing]
        T_Campaign[campaign_tasks]
        T_Integrations[run_integrations]
        T_S3[s3_upload]
    end

    subgraph MCP["MCP Server (/api/v1/mcp)"]
        MCP_Tools[Tools: workflow CRUD, docs search]
    end

    Routes --> Services
    Services --> DB_Layer
    Services --> Tasks
    Tasks --> DB_Layer
    S_WorkerSync -->|Redis Pub/Sub| S_Pipecat
```

---

## Pipecat Voice Pipeline Detail

```mermaid
graph LR
    subgraph Pipeline["Pipecat Pipeline Runtime"]
        PB[Pipeline Builder]
        SF[Service Factory]
        RE[Recording Engine]
        AM[Audio Mixer]
        PC[Pre-Call Fetch]
    end

    subgraph Realtime["Realtime Providers"]
        OpenAI_RT[OpenAI Realtime]
        Gemini_RT[Gemini Live]
        Grok_RT[Grok Realtime]
        Ultravox_RT[Ultravox]
    end

    subgraph Traditional["Traditional Pipeline (STT→LLM→TTS)"]
        STT_Svc[STT Service]
        LLM_Svc[LLM Service]
        TTS_Svc[TTS Service]
    end

    subgraph Engine["Workflow Engine"]
        WG[Workflow Graph]
        PE[Pipecat Engine]
        CT[Custom Tools]
        VE[Variable Extractor]
        CS[Context Summarizer]
    end

    PB --> SF
    SF --> Realtime
    SF --> Traditional
    PE --> WG
    PE --> CT
    PE --> VE
    PE --> CS
    Pipeline --> Engine
```

---

## Frontend Architecture

```mermaid
graph TB
    subgraph Pages["App Router Pages"]
        P_Workflow["workflow — Agent List + Builder"]
        P_Campaign["campaigns — Bulk Outbound"]
        P_Telephony["telephony-configurations"]
        P_Recordings["recordings"]
        P_Tools["tools"]
        P_Files["files — Knowledge Base"]
        P_Reports["reports"]
        P_Usage["usage"]
        P_APIKeys["api-keys"]
        P_Settings["settings"]
    end

    subgraph Components["Component Library"]
        C_Flow[flow/ — React Flow Nodes, Edges, Renderer]
        C_Workflow[workflow/ — Tables, Cards, Folders]
        C_Telephony[telephony/ — Config Forms]
        C_UI[ui/ — shadcn/ui Primitives]
        C_Filters[filters/ — FilterBuilder]
        C_Layout[layout/ — Sidebar, AppLayout]
    end

    subgraph State["State Management"]
        Zustand_Store[Zustand Stores]
        Zundo[Zundo — Undo/Redo History]
        Context[React Context Providers]
    end

    subgraph Client["API Client"]
        OpenAPI[Auto-generated from OpenAPI spec]
        AuthInterceptor["stackframe/stack Auth"]
    end

    Pages --> Components
    Pages --> State
    Pages --> Client
    Client -->|HTTP| Backend_API[FastAPI Backend]
    C_Flow --> Zustand_Store
    Zustand_Store --> Zundo
```

---

## Infrastructure & Deployment

```mermaid
graph TB
    subgraph Docker["Docker Compose Stack"]
        subgraph Core["Core Services"]
            API_Container["dograh-api FastAPI :8000"]
            UI_Container["dograh-ui Next.js :3010"]
        end

        subgraph Data_Stores["Data Stores"]
            PG["PostgreSQL 17 + pgvector :5432"]
            RD["Redis 7 :6379"]
            MN["MinIO :9000 / :9001"]
        end

        subgraph Networking["Networking & Tunnels"]
            CF["Cloudflared Tunnel to API"]
            CT_Server["Coturn TURN/STUN :3478"]
            NG["Nginx HTTPS :443 - remote profile"]
        end
    end

    subgraph Volumes["Persistent Volumes"]
        V_PG[postgres_data]
        V_Redis[redis_data]
        V_MinIO[minio-data]
        V_Tmp[shared-tmp]
    end

    API_Container --> PG
    API_Container --> RD
    API_Container --> MN
    UI_Container --> API_Container
    CF --> API_Container
    NG --> UI_Container
    NG --> API_Container
    CT_Server --> API_Container

    PG --> V_PG
    RD --> V_Redis
    MN --> V_MinIO
```

---

## Data Model (Core Entities)

```mermaid
erDiagram
    Organization ||--o{ User : "has members"
    Organization ||--o{ Workflow : "owns"
    Organization ||--o{ TelephonyConfiguration : "owns"
    Organization ||--o{ Campaign : "owns"
    Organization ||--o{ Integration : "owns"
    Organization ||--o{ APIKey : "owns"

    Workflow ||--o{ WorkflowDefinition : "versions"
    Workflow ||--o{ WorkflowRun : "executes"
    Workflow }o--o| Folder : "grouped in"

    WorkflowDefinition {
        json workflow_json
        string status
        int version_number
        json workflow_configurations
    }

    WorkflowRun {
        string mode
        string state
        string call_type
        json usage_info
        json cost_info
        json gathered_context
        json logs
    }

    TelephonyConfiguration ||--o{ TelephonyPhoneNumber : "has"
    TelephonyPhoneNumber }o--o| Workflow : "routes inbound to"

    Campaign ||--o{ WorkflowRun : "generates"
    Campaign }o--|| Workflow : "uses"

    WorkflowRun }o--o| Campaign : "belongs to"
    WorkflowRun }o--|| WorkflowDefinition : "ran version"
```

---

## Telephony Provider Architecture

```mermaid
graph TB
    subgraph Providers["Telephony Providers"]
        Twilio[Twilio]
        Vonage[Vonage]
        Telnyx[Telnyx]
        Plivo[Plivo]
        Cloudonix[Cloudonix]
        Vobiz[Vobiz]
        ARI[ARI / Asterisk]
    end

    subgraph Abstraction["Provider Abstraction Layer"]
        Base[Base Provider Interface]
        Factory[Provider Factory]
        Registry[Provider Registry]
        Transfer[Call Transfer Manager]
        Status[Status Processor]
    end

    Factory --> Registry
    Registry --> Providers
    Base --> Twilio
    Base --> Vonage
    Base --> Telnyx
    Base --> Plivo
    Base --> Cloudonix
    Base --> Vobiz
    Base --> ARI
    Transfer --> Base
    Status --> Base
```

---

## Key Design Decisions

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Voice pipeline | Pipecat (git submodule) | Open-source framework for real-time voice AI, supports multiple providers |
| Realtime vs Traditional | Both paths | Realtime (OpenAI/Gemini/Grok) for lowest latency; traditional STT→LLM→TTS for flexibility |
| Task queue | ARQ (Redis-based) | Lightweight, async-native, shares Redis with cache/pub-sub |
| Workflow versioning | Immutable definitions | Every publish creates a new WorkflowDefinition; runs reference the exact version used |
| Multi-tenancy | Organization-scoped | All resources filtered by organization_id; strict tenant isolation |
| Telephony abstraction | Provider pattern | Common interface with per-provider implementations; new providers added without core changes |
| Frontend state | Zustand + Zundo | Lightweight stores with built-in undo/redo for the workflow builder |
| API client | Auto-generated | `openapi-ts` generates typed client from backend OpenAPI spec |
| Public access | Cloudflared tunnel | Zero-config public URL for telephony webhooks without port forwarding |
| WebRTC NAT traversal | Coturn | Self-hosted TURN server for reliable browser-to-server audio |
| Worker sync | Redis pub/sub | Propagates config changes across multiple FastAPI worker processes |
