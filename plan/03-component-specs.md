# AgentHub - Component Specifications

## Overview

This document provides detailed specifications for each component in the AgentHub system.

---

## 1. Agent Configuration (YAML)

### Purpose
Define agent behavior, capabilities, and access controls in a Git-safe, declarative format.

### File Location
`agents/{agent-id}.yml`

### Schema

```yaml
# agents/finance-assistant.yml
id: finance-assistant
name: Finance Assistant
description: Personal finance tracking and budget management
version: 1.0.0

# LLM Configuration
model:
  provider: openai          # openai | anthropic | ollama | local
  model_name: gpt-4        # Model identifier
  secret_ref: OPENAI_API_KEY  # Reference to .env variable
  temperature: 0.7
  max_tokens: 2000

# System Prompt
system_prompt: |
  You are a personal finance assistant with access to budget data and
  transaction history. Help the user manage their finances, track expenses,
  and provide budgeting advice. Always maintain confidentiality.

# Tool Access
tools:
  builtin:
    - web_search
    - calculator
  mcp_servers:
    - finance_mcp         # Reference to mcp/finance.yml
  skills:
    - analyze_spending    # Reference to skills/analyze_spending.py

# Data Access Control
data_scopes:
  - finances             # Access to finances.db
  - vector.finances      # Access to finance-specific vector store

# Memory Configuration
memory:
  enabled: true
  short_term_messages: 50    # Keep last 50 messages in context
  long_term_enabled: true    # Use vector store for long-term memory

# Authorization
auth:
  allowed_users:
    - "*"                 # All authenticated users
  # OR specific users:
  # - user@example.com

# Metadata
tags:
  - finance
  - personal
  - private

created_at: 2025-01-15
updated_at: 2025-01-15
author: you@example.com
```

### Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique identifier (slug format) |
| `name` | string | ✅ | Display name for UI |
| `description` | string | ✅ | Brief description of agent purpose |
| `version` | string | ✅ | Semantic version |
| `model.provider` | enum | ✅ | LLM provider (openai, anthropic, ollama, etc.) |
| `model.model_name` | string | ✅ | Model identifier |
| `model.secret_ref` | string | ✅ | Environment variable name for API key |
| `model.temperature` | float | ❌ | Sampling temperature (0-1) |
| `model.max_tokens` | int | ❌ | Max response length |
| `system_prompt` | string | ✅ | Instructions for the agent |
| `tools.builtin` | array | ❌ | Built-in tools to enable |
| `tools.mcp_servers` | array | ❌ | MCP server IDs to attach |
| `tools.skills` | array | ❌ | Custom Python scripts |
| `data_scopes` | array | ✅ | Data access permissions |
| `memory.enabled` | bool | ❌ | Enable conversation memory |
| `auth.allowed_users` | array | ✅ | Who can use this agent |

### Validation Rules

1. `id` must be lowercase alphanumeric + hyphens
2. `model.provider` must be supported by the system
3. `model.secret_ref` must exist in `.env`
4. `tools.mcp_servers` must reference valid MCP configs
5. `data_scopes` must map to existing data stores
6. No secrets or API keys directly in YAML

---

## 2. MCP Configuration (YAML)

### Purpose
Define MCP (Model Context Protocol) server connections and tools.

### File Location
`mcp/{mcp-id}.yml`

### Schema

```yaml
# mcp/finance.yml
id: finance_mcp
name: Finance MCP Server
description: Tools for accessing personal finance data
version: 1.0.0

# MCP Server Connection
connection:
  type: http                    # http | stdio | sse
  url: http://localhost:8080    # For HTTP/SSE
  # OR for stdio:
  # command: python
  # args: ["-m", "finance_mcp_server"]

  # Authentication
  auth:
    type: bearer              # bearer | api_key | none
    secret_ref: FINANCE_MCP_TOKEN  # Reference to .env

# Tools Provided
tools:
  - name: get_account_balance
    description: Get current balance for a specific account

  - name: list_transactions
    description: List recent transactions with optional filters

  - name: analyze_spending
    description: Analyze spending patterns by category

# Resources (Optional)
resources:
  - name: budget_data
    uri: finance://budgets
    description: Access to budget configurations

# Metadata
tags:
  - finance
  - private

created_at: 2025-01-15
updated_at: 2025-01-15
```

### Connection Types

#### HTTP
```yaml
connection:
  type: http
  url: https://api.example.com/mcp
  auth:
    type: bearer
    secret_ref: MCP_API_TOKEN
```

#### Stdio (Local Process)
```yaml
connection:
  type: stdio
  command: npx
  args: ["-y", "@your/mcp-server"]
  env:
    MCP_CONFIG_PATH: /path/to/config
```

#### SSE (Server-Sent Events)
```yaml
connection:
  type: sse
  url: http://localhost:3000/sse
  auth:
    type: none
```

---

## 3. Secrets Configuration (.env)

### Purpose
Store sensitive credentials outside of Git.

### File Location
`.env` (in project root, gitignored)

### Format

```bash
# .env
# LLM Provider API Keys
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...

# Local LLM Configuration
OLLAMA_BASE_URL=http://localhost:11434

# MCP Server Tokens
FINANCE_MCP_TOKEN=secret-token-123
NOTION_PERSONAL_TOKEN=secret_...
NOTION_WORK_TOKEN=secret_...

# Database Credentials (if remote)
POSTGRES_URL=postgresql://user:pass@localhost/agenthub

# Application Secrets
JWT_SECRET=random-secret-for-auth
SESSION_SECRET=another-random-secret
```

### Security Best Practices

1. **Never commit `.env` to Git**
   - Add to `.gitignore`
   - Provide `.env.example` template

2. **Use strong, unique secrets**
   - Generate with `openssl rand -hex 32`
   - Different secrets per environment

3. **Rotate secrets regularly**
   - Especially if exposed or shared

4. **Use secret management tools** (production)
   - Vault, AWS Secrets Manager, etc.
   - Integrate via custom secret resolver

---

## 4. Agent Orchestrator

### Purpose
Core execution engine that runs agent conversations and enforces security policies.

### Responsibilities

1. **Agent Lifecycle**
   - Load agent configuration from registry
   - Initialize LLM client with secrets
   - Attach allowed MCP servers
   - Set up data scope connections

2. **Conversation Loop**
   - Receive user message
   - Load session history from database
   - Send conversation to LLM
   - Process LLM response (text or tool calls)
   - Execute tool calls via MCP or builtin tools
   - Return results to LLM
   - Save messages to database
   - Stream response to client

3. **Security Enforcement**
   - Verify agent permissions for user
   - Enforce data scope restrictions
   - Inject secrets only during tool execution
   - Never expose secrets to LLM
   - Validate all tool calls

4. **Error Handling**
   - Graceful degradation if MCP unavailable
   - Retry logic for LLM API calls
   - User-friendly error messages

### Pseudo-Code

```python
class AgentOrchestrator:
    def __init__(self, agent_registry, mcp_registry, secrets_resolver, data_scope_mapper):
        self.agent_registry = agent_registry
        self.mcp_registry = mcp_registry
        self.secrets = secrets_resolver
        self.data_scopes = data_scope_mapper

    async def run_agent(self, agent_id, session_id, user_message):
        # 1. Load agent config
        agent_config = self.agent_registry.get(agent_id)

        # 2. Check permissions
        if not self.check_auth(agent_config, current_user):
            raise PermissionError("User not authorized for this agent")

        # 3. Initialize LLM client
        llm_client = self.create_llm_client(
            provider=agent_config.model.provider,
            model=agent_config.model.model_name,
            api_key=self.secrets.get(agent_config.model.secret_ref)
        )

        # 4. Attach MCP tools
        tools = []
        for mcp_id in agent_config.tools.mcp_servers:
            mcp_config = self.mcp_registry.get(mcp_id)
            mcp_client = self.create_mcp_client(mcp_config)
            tools.extend(mcp_client.get_tools())

        # 5. Attach data scopes
        data_connections = {}
        for scope in agent_config.data_scopes:
            data_connections[scope] = self.data_scopes.get_connection(scope)

        # 6. Load session history
        session = self.load_session(session_id)
        messages = session.get_messages()

        # 7. Add user message
        messages.append({"role": "user", "content": user_message})

        # 8. Conversation loop
        while True:
            # Call LLM
            response = await llm_client.chat(
                messages=messages,
                tools=tools,
                system_prompt=agent_config.system_prompt
            )

            # If text response, we're done
            if response.type == "text":
                self.save_message(session_id, "assistant", response.content)
                return response.content

            # If tool calls, execute them
            if response.type == "tool_calls":
                for tool_call in response.tool_calls:
                    # Execute tool (with data scope restrictions)
                    result = await self.execute_tool(
                        tool_call,
                        data_connections,
                        agent_config.data_scopes
                    )

                    # Add tool result to conversation
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": result
                    })

                # Continue loop to get final response
                continue

    async def execute_tool(self, tool_call, data_connections, allowed_scopes):
        # Validate data scope access
        required_scope = tool_call.get_required_scope()
        if required_scope not in allowed_scopes:
            raise PermissionError(f"Tool requires scope '{required_scope}' not allowed for this agent")

        # Execute tool with data connection
        db = data_connections[required_scope]
        result = await tool_call.execute(db)
        return result
```

---

## 5. Agent Registry

### Purpose
Load, validate, and provide access to agent configurations.

### Interface

```python
class AgentRegistry:
    def __init__(self, agents_dir: str):
        """Load all agent configs from agents/ directory"""
        self.agents = {}
        self.load_agents(agents_dir)

    def load_agents(self, agents_dir: str):
        """Scan directory for .yml files and load them"""
        for file in glob(f"{agents_dir}/*.yml"):
            config = yaml.safe_load(open(file))
            self.validate(config)
            self.agents[config['id']] = AgentConfig(**config)

    def get(self, agent_id: str) -> AgentConfig:
        """Get agent config by ID"""
        if agent_id not in self.agents:
            raise NotFoundError(f"Agent '{agent_id}' not found")
        return self.agents[agent_id]

    def list(self) -> list[AgentConfig]:
        """List all available agents"""
        return list(self.agents.values())

    def validate(self, config: dict):
        """Validate agent config against schema"""
        # Check required fields
        # Validate secret_ref exists
        # Check MCP references are valid
        # etc.
```

---

## 6. MCP Registry

### Purpose
Manage MCP server connections and tool catalogs.

### Interface

```python
class MCPRegistry:
    def __init__(self, mcp_dir: str, secrets_resolver: SecretsResolver):
        self.mcp_configs = {}
        self.mcp_clients = {}
        self.secrets = secrets_resolver
        self.load_mcps(mcp_dir)

    def load_mcps(self, mcp_dir: str):
        """Load all MCP configs and initialize clients"""
        for file in glob(f"{mcp_dir}/*.yml"):
            config = yaml.safe_load(open(file))
            self.mcp_configs[config['id']] = config
            self.mcp_clients[config['id']] = self.create_client(config)

    def create_client(self, config: dict) -> MCPClient:
        """Create MCP client based on connection type"""
        if config['connection']['type'] == 'http':
            return HTTPMCPClient(
                url=config['connection']['url'],
                auth_token=self.secrets.get(config['connection']['auth']['secret_ref'])
            )
        elif config['connection']['type'] == 'stdio':
            return StdioMCPClient(
                command=config['connection']['command'],
                args=config['connection']['args']
            )
        # etc.

    def get(self, mcp_id: str) -> MCPClient:
        """Get MCP client by ID"""
        return self.mcp_clients[mcp_id]
```

---

## 7. Secrets Resolver

### Purpose
Load secrets from `.env` and resolve references.

### Interface

```python
class SecretsResolver:
    def __init__(self, env_file: str = ".env"):
        load_dotenv(env_file)  # Load .env into environment
        self.secrets = {}
        self.load_secrets()

    def load_secrets(self):
        """Load all environment variables"""
        self.secrets = dict(os.environ)

    def get(self, secret_ref: str) -> str:
        """Resolve secret reference"""
        if secret_ref not in self.secrets:
            raise ValueError(f"Secret '{secret_ref}' not found in .env")
        return self.secrets[secret_ref]

    def has(self, secret_ref: str) -> bool:
        """Check if secret exists"""
        return secret_ref in self.secrets
```

---

## 8. Data Scope Mapper

### Purpose
Map logical data scopes to physical database connections.

### Configuration

```yaml
# config/data_scopes.yml
scopes:
  finances:
    type: sqlite
    path: data/finances.db

  notes.research:
    type: sqlite
    path: data/research.db

  notes.personal:
    type: sqlite
    path: data/personal.db

  vector.finances:
    type: chroma
    path: data/vectors/finances
    collection: finance_docs
```

### Interface

```python
class DataScopeMapper:
    def __init__(self, config_file: str):
        self.scopes = yaml.safe_load(open(config_file))
        self.connections = {}

    def get_connection(self, scope: str):
        """Get database connection for scope"""
        if scope in self.connections:
            return self.connections[scope]

        config = self.scopes['scopes'][scope]

        if config['type'] == 'sqlite':
            conn = sqlite3.connect(config['path'])
        elif config['type'] == 'postgres':
            conn = psycopg2.connect(config['url'])
        elif config['type'] == 'chroma':
            conn = chromadb.PersistentClient(path=config['path'])

        self.connections[scope] = conn
        return conn
```

---

## 9. Session Manager

### Purpose
Manage chat sessions and message history.

### Database Schema

```sql
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    agent_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata JSON
);

CREATE TABLE messages (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL REFERENCES sessions(id),
    role TEXT NOT NULL,  -- user | assistant | tool
    content TEXT,
    tool_calls JSON,
    tool_results JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

CREATE INDEX idx_messages_session ON messages(session_id);
CREATE INDEX idx_sessions_user ON sessions(user_id);
```

### Interface

```python
class SessionManager:
    def create_session(self, agent_id: str, user_id: str) -> Session:
        """Create new chat session"""

    def get_session(self, session_id: str) -> Session:
        """Get existing session"""

    def add_message(self, session_id: str, role: str, content: str):
        """Add message to session"""

    def get_messages(self, session_id: str, limit: int = 100) -> list[Message]:
        """Get session message history"""
```

---

## 10. Built-in Tools

### Web Search
```python
async def web_search(query: str, num_results: int = 5) -> str:
    """Search the web and return results"""
```

### Calculator
```python
def calculate(expression: str) -> float:
    """Safely evaluate math expressions"""
```

### File Operations (Scoped)
```python
async def read_file(path: str, data_scope: str) -> str:
    """Read file within allowed data scope"""

async def write_file(path: str, content: str, data_scope: str):
    """Write file within allowed data scope"""
```

---

This covers the core components. Each can be extended with additional features as needed.
