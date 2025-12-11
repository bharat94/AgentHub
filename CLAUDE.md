# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AgentHub is a self-hosted, privacy-first AI agent orchestration platform. It enables running multiple specialized AI agents with strict data isolation, secret management, and Git-safe configuration.

**Current Status**: Early development - design and planning phase complete. No implementation code exists yet.

## Core Architecture Principles

### Three-Layer Security Model

1. **Public Configuration** (safe to commit to Git)
   - `agents/*.yml` - Agent definitions
   - `mcp/*.yml` - MCP server configurations
   - `config/*.yml` - System settings

2. **Local Secrets** (gitignored, never committed)
   - `.env` - API keys and tokens
   - All configs use `secret_ref` references, never hardcoded secrets

3. **Private Data** (gitignored, isolated per agent)
   - `data/*.db` - Per-agent SQLite databases
   - `data/vectors/` - Vector stores for RAG
   - Complete data scope isolation between agents

### Key Components

The system follows a layered backend-centric architecture:

- **Agent Orchestrator**: Core execution engine that runs agent conversations, enforces security policies, and manages the conversation loop
- **Agent Registry**: Loads and validates `agents/*.yml` files at startup
- **MCP Registry**: Loads `mcp/*.yml` files and initializes MCP server connections
- **Secrets Resolver**: Reads `.env` and resolves `secret_ref` at runtime - secrets never sent to LLMs
- **Data Scope Mapper**: Maps logical scopes (e.g., "finances") to physical datastores and enforces agent-level access control

### Data Isolation

Each agent explicitly declares allowed data scopes:
- Finance agent can only access `finances.db`
- Research agent can only access `research.db`
- No cross-agent data leakage - enforced by orchestrator

### MCP Integration

Uses Model Context Protocol for tool integration:
- Per-agent MCP allowlists in agent configs
- Separate credentials per MCP server (e.g., `notion_personal` vs `notion_work`)
- MCP configs reference secrets via `secret_ref`

## Technology Stack Considerations

**Backend Options**:
- Python + FastAPI (recommended for AI/ML ecosystem)
- TypeScript + Express/Fastify (for full-stack JS teams)

**Database**:
- SQLite for MVP (single file, local-first)
- PostgreSQL for scale (better concurrency)

**Vector Store**:
- ChromaDB (embedded, simpler)
- Qdrant (server-based, more features)

**LLM Integration**:
- Direct API calls (most control) or LangChain/LlamaIndex

## Agent Configuration Structure

Agents are defined in YAML with this structure:

```yaml
id: agent-slug
name: Display Name
model:
  provider: openai | anthropic | ollama
  model_name: gpt-4
  secret_ref: OPENAI_API_KEY  # Never hardcode secrets
tools:
  mcp_servers:
    - mcp_id  # References mcp/{mcp_id}.yml
data_scopes:
  - scope_name  # e.g., "finances", "research"
auth:
  allowed_users:
    - "*" | specific emails
```

## MCP Configuration Structure

MCP servers defined in YAML:

```yaml
id: mcp_slug
connection:
  type: http | stdio | sse
  url: http://localhost:8080  # For HTTP/SSE
  auth:
    type: bearer | api_key | none
    secret_ref: MCP_TOKEN  # Never hardcode
tools:
  - name: tool_name
    description: What the tool does
```

## Security Requirements

**CRITICAL - Never Commit**:
- `.env` files or any secrets
- `data/` directory (databases, vectors)
- `*.db` files
- `logs/` directory (may contain sensitive info)

**Always Use**:
- `secret_ref` references in YAML configs
- Data scope declarations for all agents
- Input validation for user messages
- Parameter validation for tool calls
- Path sanitization to prevent directory traversal

**Validation Rules**:
- Agent IDs must be lowercase alphanumeric + hyphens
- All `secret_ref` values must exist in `.env`
- All `mcp_servers` references must point to valid MCP configs
- All `data_scopes` must map to existing datastores

## Orchestrator Execution Flow

1. Load agent config from registry
2. Check user permissions (`auth.allowed_users`)
3. Initialize LLM client with secret from `.env`
4. Attach MCP tools (only those in agent's `tools.mcp_servers`)
5. Load session history from database
6. Start conversation loop:
   - Call LLM with messages and tools
   - If text response: save and stream to client
   - If tool calls: validate scope, execute, return results to LLM
7. Enforce data scope restrictions before any database access

## Design Documents

Full architecture and specifications are in `plan/`:
- `01-vision-and-goals.md` - Project vision and objectives
- `02-architecture-overview.md` - System design and layer breakdown
- `03-component-specs.md` - Detailed component specifications with schemas
- `04-execution-flow.md` - Data flow and execution diagrams
- `05-security-principles.md` - Security model and threat mitigations

Additionally, `ARCHITECTURE.md` contains comprehensive mermaid diagrams showing system architecture, component interactions, data flow, agent execution, security layers, and deployment models.

## Development Philosophy

**Optimize for**:
- Single-user/small-team deployment first
- Privacy and data isolation above all
- Git-safe configuration sharing
- Simple self-hosted deployment
- Scale when needed, not prematurely

**Security-First Approach**:
- Least privilege by default (agents have no permissions until explicitly granted)
- Defense in depth (multiple validation layers)
- Secrets never in Git, never sent to LLMs
- Explicit opt-in for all capabilities (data scopes, tools, users)
