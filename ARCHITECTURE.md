# AgentHub - Architecture Diagrams

This document provides visual representations of the AgentHub system architecture.

## Table of Contents
- [High-Level System Architecture](#high-level-system-architecture)
- [Component Interaction Diagram](#component-interaction-diagram)
- [Data Flow Diagram](#data-flow-diagram)
- [Agent Execution Flow](#agent-execution-flow)
- [Security Layers](#security-layers)

---

## High-Level System Architecture

```mermaid
graph TB
    subgraph Client["🖥️ Client Layer"]
        UI[Web UI<br/>React/Vue/Svelte]
    end

    subgraph Backend["⚙️ Backend Layer"]
        API[API Server<br/>FastAPI/Express]
        ORCH[Agent Orchestrator]
        AGENTREG[Agent Registry]
        MCPREG[MCP Registry]
        SECRETS[Secrets Resolver]
        DATASCOPE[Data Scope Mapper]
    end

    subgraph Storage["💾 Storage Layer"]
        ENV[.env<br/>Secrets]
        DB1[(finances.db)]
        DB2[(research.db)]
        DB3[(personal.db)]
        SESSIONS[(sessions.db)]
        VECTORS[Vector Stores]
    end

    subgraph External["🌐 External Services"]
        LLM1[OpenAI]
        LLM2[Anthropic]
        LLM3[Ollama]
        MCP1[Finance MCP]
        MCP2[Notion MCP]
        MCP3[Custom MCPs]
    end

    UI <-->|HTTP/WS| API
    API --> ORCH
    ORCH --> AGENTREG
    ORCH --> MCPREG
    ORCH --> SECRETS
    ORCH --> DATASCOPE

    SECRETS -.->|reads| ENV
    DATASCOPE --> DB1
    DATASCOPE --> DB2
    DATASCOPE --> DB3
    DATASCOPE --> SESSIONS
    DATASCOPE --> VECTORS

    ORCH <-->|API calls| LLM1
    ORCH <-->|API calls| LLM2
    ORCH <-->|local| LLM3
    ORCH <-->|tools| MCP1
    ORCH <-->|tools| MCP2
    ORCH <-->|tools| MCP3

    style Client fill:#e1f5ff
    style Backend fill:#fff4e1
    style Storage fill:#f0f0f0
    style External fill:#e8f5e9
```

---

## Component Interaction Diagram

```mermaid
sequenceDiagram
    participant User
    participant UI
    participant API
    participant Orchestrator
    participant AgentRegistry
    participant SecretsResolver
    participant DataScopeMapper
    participant LLM
    participant MCP

    User->>UI: Select "Finance Assistant"
    UI->>API: GET /agents
    API->>AgentRegistry: List agents
    AgentRegistry-->>API: Agent configs
    API-->>UI: Available agents

    User->>UI: Send message
    UI->>API: POST /sessions/{id}/messages
    API->>Orchestrator: Process message

    Orchestrator->>AgentRegistry: Get agent config
    AgentRegistry-->>Orchestrator: finance-assistant.yml

    Orchestrator->>SecretsResolver: Resolve OPENAI_API_KEY
    SecretsResolver-->>Orchestrator: sk-proj-...

    Orchestrator->>DataScopeMapper: Get finances.db connection
    DataScopeMapper-->>Orchestrator: DB connection

    Orchestrator->>LLM: Send conversation
    LLM-->>Orchestrator: Tool call: list_transactions

    Orchestrator->>MCP: Execute tool
    MCP-->>Orchestrator: Transaction data

    Orchestrator->>LLM: Tool result
    LLM-->>Orchestrator: Final response

    Orchestrator-->>API: Stream response
    API-->>UI: SSE stream
    UI-->>User: Display response
```

---

## Data Flow Diagram

```mermaid
flowchart TD
    START([User Message]) --> API[API Layer]
    API --> AUTH{Authenticated?}
    AUTH -->|No| REJECT[Reject Request]
    AUTH -->|Yes| LOAD[Load Agent Config]

    LOAD --> PERM{User Allowed?}
    PERM -->|No| REJECT
    PERM -->|Yes| RESOLVE[Resolve Secrets]

    RESOLVE --> SCOPE[Check Data Scopes]
    SCOPE --> INITLLM[Initialize LLM Client]
    INITLLM --> CONV[Start Conversation Loop]

    CONV --> LLM[Call LLM]
    LLM --> RESPONSE{Response Type?}

    RESPONSE -->|Text| SAVE[Save Message]
    SAVE --> STREAM[Stream to User]
    STREAM --> END([Done])

    RESPONSE -->|Tool Call| VALIDATE{Tool Scope Valid?}
    VALIDATE -->|No| ERROR[Return Error]
    ERROR --> CONV
    VALIDATE -->|Yes| EXECUTE[Execute Tool]
    EXECUTE --> RESULT[Get Result]
    RESULT --> CONV

    style START fill:#90EE90
    style END fill:#90EE90
    style REJECT fill:#FFB6C6
    style ERROR fill:#FFB6C6
```

---

## Agent Execution Flow

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> LoadingConfig: User selects agent
    LoadingConfig --> ValidatingAuth: Config loaded
    ValidatingAuth --> Unauthorized: Auth failed
    ValidatingAuth --> InitializingAgent: Auth success

    InitializingAgent --> ResolvingSecrets
    ResolvingSecrets --> AttachingTools
    AttachingTools --> MappingDataScopes
    MappingDataScopes --> Ready

    Ready --> ProcessingMessage: User sends message
    ProcessingMessage --> CallingLLM
    CallingLLM --> ProcessingResponse

    ProcessingResponse --> StreamingToUser: Text response
    ProcessingResponse --> ExecutingTools: Tool calls

    ExecutingTools --> ValidatingScope: Check permissions
    ValidatingScope --> ToolDenied: Scope not allowed
    ValidatingScope --> CallingTool: Scope allowed
    CallingTool --> CallingLLM: Return results

    ToolDenied --> CallingLLM: Return error
    StreamingToUser --> Ready: Complete

    Ready --> Idle: Session ends
    Unauthorized --> [*]

    note right of ResolvingSecrets
        Secrets from .env
        Never sent to LLM
    end note

    note right of ValidatingScope
        Data scope isolation
        enforced here
    end note
```

---

## Security Layers

```mermaid
graph TB
    subgraph Layer1["Layer 1: Git Safety"]
        YAML[Agent Configs<br/>YAML files]
        MCP[MCP Configs<br/>YAML files]
        GITIGNORE[.gitignore]

        YAML -.->|References| SECRETS
        MCP -.->|References| SECRETS
        GITIGNORE -.->|Excludes| SECRETS
    end

    subgraph Layer2["Layer 2: Runtime Isolation"]
        SECRETS[".env<br/>Secret Store"]
        RESOLVER[Secrets Resolver]
        ORCHESTRATOR[Orchestrator]

        SECRETS --> RESOLVER
        RESOLVER -.->|Injects at runtime| ORCHESTRATOR
    end

    subgraph Layer3["Layer 3: Data Isolation"]
        ORCHESTRATOR --> SCOPE_CHECK{Data Scope<br/>Validation}
        SCOPE_CHECK -->|Allowed| DB1[(finances.db)]
        SCOPE_CHECK -->|Allowed| DB2[(research.db)]
        SCOPE_CHECK -.->|Blocked| DB3[(personal.db)]
    end

    subgraph Layer4["Layer 4: Execution Safety"]
        SCOPE_CHECK --> TOOL_EXEC[Tool Execution]
        TOOL_EXEC --> PARAM_VALID{Parameter<br/>Validation}
        PARAM_VALID -->|Valid| SAFE_EXEC[Safe Execution]
        PARAM_VALID -.->|Invalid| REJECT[Reject]
    end

    style Layer1 fill:#e3f2fd
    style Layer2 fill:#fff3e0
    style Layer3 fill:#f3e5f5
    style Layer4 fill:#e8f5e9
    style REJECT fill:#ffcdd2
```

---

## Agent Isolation Architecture

```mermaid
graph LR
    subgraph FinanceAgent["Finance Agent"]
        FA_CONFIG[Config]
        FA_MODEL[GPT-4]
        FA_TOOLS[Finance MCP]
        FA_DATA[(finances.db)]
    end

    subgraph ResearchAgent["Research Agent"]
        RA_CONFIG[Config]
        RA_MODEL[Claude]
        RA_TOOLS[Web Search]
        RA_DATA[(research.db)]
    end

    subgraph PersonalAgent["Personal Agent"]
        PA_CONFIG[Config]
        PA_MODEL[Llama 3]
        PA_TOOLS[Notion Personal]
        PA_DATA[(personal.db)]
    end

    subgraph WorkAgent["Work Agent"]
        WA_CONFIG[Config]
        WA_MODEL[GPT-4]
        WA_TOOLS[Notion Work<br/>Slack]
        WA_DATA[(work.db)]
    end

    ORCHESTRATOR[Agent Orchestrator] --> FinanceAgent
    ORCHESTRATOR --> ResearchAgent
    ORCHESTRATOR --> PersonalAgent
    ORCHESTRATOR --> WorkAgent

    FA_DATA -.X.-|Isolated| RA_DATA
    RA_DATA -.X.-|Isolated| PA_DATA
    PA_DATA -.X.-|Isolated| WA_DATA
    WA_DATA -.X.-|Isolated| FA_DATA

    style FinanceAgent fill:#c8e6c9
    style ResearchAgent fill:#bbdefb
    style PersonalAgent fill:#f8bbd0
    style WorkAgent fill:#ffe0b2
```

---

## File Structure and Git Safety

```mermaid
graph TD
    ROOT[AgentHub Repository]

    ROOT --> SAFE[Safe to Commit ✅]
    ROOT --> UNSAFE[Never Commit ❌]

    SAFE --> AGENTS[agents/*.yml]
    SAFE --> MCPS[mcp/*.yml]
    SAFE --> CONFIG[config/*.yml]
    SAFE --> CODE[src/]
    SAFE --> DOCS[docs/]

    UNSAFE --> ENV[.env]
    UNSAFE --> DATA[data/]
    UNSAFE --> DBS[*.db files]
    UNSAFE --> LOGS[logs/]
    UNSAFE --> CACHE[cache/]

    AGENTS -.->|References| ENV
    MCPS -.->|References| ENV

    style SAFE fill:#c8e6c9
    style UNSAFE fill:#ffcdd2
    style ROOT fill:#e3f2fd
```

---

## Deployment Architecture

```mermaid
graph TB
    subgraph Local["💻 Local Development"]
        DEV_UI[Web UI<br/>localhost:3000]
        DEV_API[Backend<br/>localhost:8000]
        DEV_DB[(SQLite)]
    end

    subgraph Docker["🐳 Docker Compose"]
        DC_NGINX[Nginx]
        DC_BACKEND[Backend Container]
        DC_DB[(PostgreSQL)]
        DC_REDIS[(Redis)]

        DC_NGINX --> DC_BACKEND
        DC_BACKEND --> DC_DB
        DC_BACKEND --> DC_REDIS
    end

    subgraph K8s["☸️ Kubernetes"]
        INGRESS[Ingress]
        SVC_FE[Frontend Service]
        SVC_BE[Backend Service]
        PODS[Backend Pods<br/>Auto-scaling]
        PG[(PostgreSQL<br/>StatefulSet)]
        PV[Persistent Volumes<br/>for data/]

        INGRESS --> SVC_FE
        INGRESS --> SVC_BE
        SVC_BE --> PODS
        PODS --> PG
        PODS --> PV
    end

    style Local fill:#e1f5ff
    style Docker fill:#fff4e1
    style K8s fill:#e8f5e9
```

---

## Technology Stack

```mermaid
mindmap
  root((AgentHub))
    Frontend
      React/Vue/Svelte
      TailwindCSS
      Shadcn UI
      WebSocket
    Backend
      FastAPI/Express
      Python/TypeScript
      LangChain
      OpenAI SDK
    Database
      SQLite MVP
      PostgreSQL Scale
      ChromaDB Vectors
    Infrastructure
      Docker
      Kubernetes
      Nginx
      Redis
    External
      OpenAI
      Anthropic
      Ollama
      MCP Servers
```

---

## Summary

AgentHub's architecture is designed with three core principles:

1. **🔒 Security First**: Secrets never in Git, data scope isolation, runtime-only secret resolution
2. **🎯 Modularity**: Clean separation of concerns, easy to extend and customize
3. **🚀 Scalability**: Start simple (SQLite), scale when needed (PostgreSQL + K8s)

For detailed component specifications, see [Component Specs](./plan/03-component-specs.md).

For security details, see [Security Principles](./plan/05-security-principles.md).
