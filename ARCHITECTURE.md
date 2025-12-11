# AgentHub Architecture

## Overview

AgentHub is **~500 lines of Python** that provides data-isolated AI agents.

**Core idea:** Each agent gets a folder. Simple as that.

---

## High-Level Design

```
User Command
     │
     ├─→ CLI (agenthub.py)
     │       │
     │       └─→ AgentRunner (runner.py)
     │               │
     │               ├─→ Load config (agents/*.yml)
     │               ├─→ Init LLM (Ollama/OpenAI/Anthropic)
     │               ├─→ Load tools (calculator, file_ops, etc.)
     │               ├─→ Run conversation loop
     │               └─→ Execute tools in sandbox (data/agent/)
     │
     └─→ Response
```

**That's it.** No orchestrators, no registries, no mappers.

---

## Components

### 1. CLI (`agenthub.py` - ~50 lines)

Simple command-line interface.

```python
./agenthub chat finance "How much did I spend?"
./agenthub list
./agenthub reset finance
```

Parses command → Creates AgentRunner → Calls `runner.chat()` → Prints response.

### 2. AgentRunner (`runner.py` - ~200 lines)

Core logic. Does everything:

1. Load agent config from YAML
2. Initialize LLM client (Ollama/OpenAI/Anthropic)
3. Load tools (built-in + custom)
4. Run conversation loop
5. Execute tool calls
6. Manage conversation history

**Key methods:**
- `__init__(agent_config_path)` - Set up agent
- `chat(message)` - Process message and return response
- `reset()` - Clear conversation history

### 3. Built-in Tools (`tools/*.py` - ~100 lines)

Simple Python functions:
- `calculator.py` - Safe math evaluation
- `file_ops.py` - Read/write files (scoped to data_dir)
- `web_search.py` - Search the web

### 4. Agent Configs (`agents/*.yml`)

One YAML file per agent. Everything in one place:

```yaml
id: finance
model:
  provider: ollama
  model: llama3
system_prompt: "You are a finance assistant..."
data_dir: data/finance
tools: [calculator, file_reader, file_writer]
```

### 5. Data Folders (`data/*`)

Each agent gets a folder:

```
data/
  finance/
    .history.json       # Conversation history
    transactions.json   # User data
  research/
    .history.json
    notes.md
```

---

## Data Flow

```
1. User: "How much did I spend?"
   ↓
2. CLI parses command
   ↓
3. AgentRunner loads finance.yml
   ↓
4. Initialize Ollama with llama3
   ↓
5. Send message to LLM with tools
   ↓
6. LLM responds: "I need to read transactions.json"
   ↓
7. Execute file_reader("transactions.json")
   - Validate: Is path within data/finance/? ✅
   - Read: data/finance/transactions.json
   - Return contents
   ↓
8. Send tool result back to LLM
   ↓
9. LLM responds: "You spent $450.23"
   ↓
10. Save to .history.json and return to user
```

**Total time:** 1-3 seconds

---

## Security Model

**Three rules:**

1. **Secrets in `.env`**
   - Never in Git
   - Loaded at runtime
   - Never sent to LLM

2. **Data in folders**
   - Each agent gets `data/{agent-id}/`
   - Tools can't access outside

3. **Path validation**
   - No ".." (directory traversal)
   - No absolute paths
   - All paths relative to data_dir

---

## File Structure

```
AgentHub/
├── agenthub.py              # CLI (~50 lines)
├── runner.py                # AgentRunner (~200 lines)
├── tools/                   # Built-in tools (~100 lines)
│   ├── __init__.py
│   ├── calculator.py
│   ├── file_ops.py
│   └── web_search.py
├── agents/                  # Agent configs (Git ✅)
│   ├── finance.yml
│   ├── research.yml
│   └── work.yml
├── data/                    # Agent data (Git ❌)
│   ├── finance/
│   ├── research/
│   └── work/
├── .env                     # Secrets (Git ❌)
├── .env.example             # Template (Git ✅)
├── requirements.txt
└── README.md
```

**Total:** ~350 lines of Python + YAML configs

---

## Dependencies

**Core:**
- Python 3.10+
- LangChain (LLM abstraction)
- PyYAML (config parsing)
- python-dotenv (secret loading)

**LLM Providers:**
- langchain-community (Ollama)
- langchain-openai (OpenAI)
- langchain-anthropic (Anthropic)

**That's it.** No databases, no web frameworks, no complex deps.

---

## Why So Simple?

**We removed:**
- ❌ Agent Registry - Just glob `agents/*.yml`
- ❌ MCP Registry - Tools are Python functions
- ❌ Data Scope Mapper - Just use folders
- ❌ Secrets Resolver - Just use `os.getenv()`
- ❌ Session Manager - Just use JSON files
- ❌ API Server - Just use CLI (add later if needed)
- ❌ Database - Just use JSON/text files
- ❌ Vector stores - Add later if needed

**Result:**
- 10+ components → 3 components
- ~5000 lines → ~500 lines
- Weeks to MVP → Weekend to MVP

---

## Comparison: Old vs New

| Aspect | Old Design | New Design |
|--------|------------|------------|
| **Components** | 10+ | 3 |
| **Config Files** | agents/*.yml + mcp/*.yml + data_scopes.yml | agents/*.yml only |
| **Code** | ~5000 lines | ~500 lines |
| **Storage** | Multiple SQLite databases | JSON files + folders |
| **Complexity** | High | Low |
| **Time to MVP** | Weeks | Weekend |
| **Learning Curve** | Steep | Gentle |

---

## Design Principles

1. **Simple beats complex**
   - If you can't explain it in 3 sentences, simplify

2. **CLI first, UI later**
   - Terminal is faster to build and debug
   - Add web UI only if users ask

3. **Local first**
   - Ollama before OpenAI
   - Privacy by default

4. **Files over databases**
   - JSON files until you need a database
   - Easier to debug and backup

5. **Copy-paste beats abstraction**
   - DRY when you have 3+ copies
   - Not before

---

## Extension Points

**Adding a new tool:**
```python
# tools/my_tool.py
def my_function(arg: str) -> str:
    return "result"
```

```yaml
# agents/my-agent.yml
tools:
  - type: python
    module: tools.my_tool
    functions: [my_function]
```

**Adding a new LLM provider:**
```python
# In runner.py, add to _init_llm()
elif provider == 'custom':
    from custom_llm import CustomLLM
    return CustomLLM(...)
```

**Adding persistence:**
```python
# Change _load_history() and _save_history()
# to use SQLite instead of JSON
```

---

## What We're NOT Building (MVP)

- ❌ Web UI (CLI first)
- ❌ Multi-user auth (single user)
- ❌ Real-time collaboration
- ❌ Agent marketplace (file sharing is enough)
- ❌ Complex orchestration
- ❌ Vector stores (add when needed)
- ❌ RAG (add when needed)
- ❌ Agent-to-agent communication

**Add these only when users demand them.**

---

## Future Enhancements

**When users ask for them:**

1. **Web UI** (~500 lines)
   - FastAPI backend
   - React frontend
   - WebSocket for streaming

2. **Vector stores** (~200 lines)
   - ChromaDB integration
   - RAG for long-term memory

3. **Agent marketplace**
   - Public agent templates
   - One-click install

4. **Multi-user**
   - JWT auth
   - User-specific `.env`

**But not now. MVP first.**

---

## Performance

**Typical execution:**
- Load config: <1ms
- Init LLM client: ~100ms
- LLM call: 1-3s (depends on model)
- Tool execution: 10-500ms
- Save history: <10ms

**Total:** 1-4 seconds per message

**Optimization opportunities (later):**
- Cache LLM client
- Lazy load tools
- Stream responses
- Parallel tool execution

---

## Testing Strategy

**Manual testing (MVP):**
```bash
# Test each agent
./agenthub chat finance "Hello"
./agenthub chat research "Hello"

# Test tools
./agenthub chat finance "Calculate 2 + 2"
./agenthub chat finance "Read transactions.json"

# Test isolation
# Finance agent can't read research data
```

**Automated tests (later):**
- Unit tests for tools
- Integration tests for AgentRunner
- E2E tests for CLI

---

## Deployment

**Development:**
```bash
git clone repo
cd agenthub
pip install -r requirements.txt
cp .env.example .env
./agenthub chat finance "Hello"
```

**Production:**
Same as development! Run on your laptop, server, or Raspberry Pi.

**Optional Docker:**
```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "agenthub.py"]
```

---

## Summary

**AgentHub in 3 sentences:**

1. Each agent is a YAML config with its own data folder
2. AgentRunner loads config, inits LLM, runs conversation loop
3. Tools are Python functions that validate paths before execution

**That's the entire architecture.**

---

## References

- **Detailed design:** See `plan/` folder
  - [Vision and Goals](plan/01-vision-and-goals.md)
  - [Architecture Overview](plan/02-architecture-overview.md)
  - [Component Specs](plan/03-component-specs.md)
  - [Execution Flow](plan/04-execution-flow.md)
  - [Security Principles](plan/05-security-principles.md)

- **Code:** Coming soon (MVP in progress)

---

**Philosophy: Do less, better.**

Start coding → Ship MVP → Get feedback → Iterate
