# AgentHub - Simplified Architecture

## Core Principle: Do Less, Better

AgentHub is **~500 lines of Python** that runs AI agents with isolated data access.

```
┌─────────────────────────────────────┐
│         CLI Interface               │
│   $ agenthub chat finance "..."    │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│      AgentRunner (Core Logic)       │
│  1. Load config (agents/X.yml)      │
│  2. Init LLM client                 │
│  3. Run conversation loop           │
│  4. Execute tools (in sandbox)      │
└──────────────┬──────────────────────┘
               │
       ┌───────┼───────┐
       │       │       │
  ┌────▼──┐ ┌─▼────┐ ┌▼─────────┐
  │ .env  │ │agents│ │  data/   │
  │secrets│ │*.yml │ │{agent}/  │
  └───────┘ └──────┘ └──────────┘
```

## File Structure

```
agenthub/
├── agenthub.py              # CLI entry point (~50 lines)
├── runner.py                # AgentRunner class (~200 lines)
├── tools/                   # Built-in tools
│   ├── __init__.py
│   ├── calculator.py
│   ├── file_ops.py
│   └── web_search.py
├── agents/                  # Agent configs (committed to Git)
│   ├── finance.yml
│   ├── research.yml
│   └── work.yml
├── data/                    # Agent data folders (gitignored)
│   ├── finance/
│   │   ├── transactions.json
│   │   └── notes.txt
│   ├── research/
│   │   └── papers/
│   └── work/
│       └── docs/
├── .env                     # Secrets (gitignored)
├── .env.example             # Template (committed)
├── requirements.txt         # Python deps
└── README.md
```

## Agent Configuration (YAML)

One file per agent, everything in one place:

```yaml
# agents/finance.yml
id: finance
name: Finance Assistant
description: Manage personal finances

# LLM Config
model:
  provider: ollama           # ollama | openai | anthropic
  model: llama3              # Model name
  # For cloud providers:
  # api_key_env: OPENAI_API_KEY  # Reference to .env variable
  temperature: 0.7
  max_tokens: 2000

# System Instructions
system_prompt: |
  You are a personal finance assistant. You have access to transaction
  data and can help with budgeting, expense tracking, and financial analysis.
  Be concise and accurate with numbers.

# Data Isolation
data_dir: data/finance       # This agent's sandbox

# Tools
tools:
  # Built-in tools
  - calculator
  - file_reader
  - file_writer

  # Custom Python functions
  - type: python
    module: tools.finance_tools
    functions:
      - parse_transaction
      - calculate_total

  # HTTP APIs (optional)
  - type: http
    name: get_stock_price
    url: https://api.example.com/stock/{symbol}
    headers:
      auth_env: STOCKS_API_KEY  # Reference to .env

# Authorization (simple)
allowed_users:
  - "*"  # Everyone, or list specific emails
```

## The AgentRunner (Core Logic)

**File: `runner.py` (~200 lines)**

```python
import os
import yaml
from pathlib import Path
from dotenv import load_dotenv

class AgentRunner:
    """
    Core agent execution engine.
    Loads config, initializes LLM, runs conversation loop.
    """

    def __init__(self, agent_config_path: str):
        """Load agent configuration"""
        self.config = self._load_config(agent_config_path)
        self.data_dir = Path(self.config['data_dir'])
        self.data_dir.mkdir(parents=True, exist_ok=True)

        # Initialize LLM client
        self.llm = self._init_llm()

        # Load tools
        self.tools = self._load_tools()

        # Load conversation history
        self.history_file = self.data_dir / ".history.json"
        self.messages = self._load_history()

    def _load_config(self, path: str) -> dict:
        """Load and validate agent config"""
        with open(path) as f:
            config = yaml.safe_load(f)
        # TODO: Validate schema
        return config

    def _init_llm(self):
        """Initialize LLM client based on provider"""
        provider = self.config['model']['provider']

        if provider == 'ollama':
            from langchain_community.llms import Ollama
            return Ollama(model=self.config['model']['model'])

        elif provider == 'openai':
            from langchain_openai import ChatOpenAI
            api_key_env = self.config['model']['api_key_env']
            api_key = os.getenv(api_key_env)
            if not api_key:
                raise ValueError(f"Missing {api_key_env} in .env")
            return ChatOpenAI(
                api_key=api_key,
                model=self.config['model']['model'],
                temperature=self.config['model'].get('temperature', 0.7)
            )

        elif provider == 'anthropic':
            from langchain_anthropic import ChatAnthropic
            api_key_env = self.config['model']['api_key_env']
            api_key = os.getenv(api_key_env)
            if not api_key:
                raise ValueError(f"Missing {api_key_env} in .env")
            return ChatAnthropic(
                api_key=api_key,
                model=self.config['model']['model']
            )

        else:
            raise ValueError(f"Unknown provider: {provider}")

    def _load_tools(self) -> dict:
        """Load tools defined in config"""
        tools = {}

        for tool_config in self.config.get('tools', []):
            if isinstance(tool_config, str):
                # Built-in tool name
                tool = self._get_builtin_tool(tool_config)
                tools[tool_config] = tool

            elif tool_config.get('type') == 'python':
                # Custom Python function
                module_name = tool_config['module']
                module = __import__(module_name, fromlist=[''])
                for func_name in tool_config['functions']:
                    func = getattr(module, func_name)
                    tools[func_name] = func

            elif tool_config.get('type') == 'http':
                # HTTP API wrapper
                tools[tool_config['name']] = self._create_http_tool(tool_config)

        return tools

    def _get_builtin_tool(self, name: str):
        """Get built-in tool by name"""
        from tools import calculator, file_reader, file_writer, web_search

        builtins = {
            'calculator': calculator.calculate,
            'file_reader': lambda path: file_reader.read_file(self.data_dir / path),
            'file_writer': lambda path, content: file_writer.write_file(
                self.data_dir / path, content
            ),
            'web_search': web_search.search,
        }

        if name not in builtins:
            raise ValueError(f"Unknown builtin tool: {name}")

        return builtins[name]

    def chat(self, user_message: str) -> str:
        """
        Main conversation loop.
        1. Add user message to history
        2. Send to LLM with tools
        3. Execute tool calls if needed
        4. Return final response
        """
        # Add user message
        self.messages.append({
            "role": "user",
            "content": user_message
        })

        # Conversation loop (handles tool calls)
        max_iterations = 10
        for i in range(max_iterations):
            # Call LLM
            response = self.llm.invoke(
                self.messages,
                tools=list(self.tools.values())
            )

            # If text response, we're done
            if hasattr(response, 'content') and response.content:
                self.messages.append({
                    "role": "assistant",
                    "content": response.content
                })
                self._save_history()
                return response.content

            # If tool calls, execute them
            if hasattr(response, 'tool_calls') and response.tool_calls:
                for tool_call in response.tool_calls:
                    tool_name = tool_call['name']
                    tool_args = tool_call['args']

                    # Execute tool
                    try:
                        tool_func = self.tools[tool_name]
                        result = tool_func(**tool_args)
                    except Exception as e:
                        result = f"Error: {str(e)}"

                    # Add tool result to messages
                    self.messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call['id'],
                        "content": str(result)
                    })

                # Continue loop to get final response
                continue

            # No response and no tool calls - shouldn't happen
            break

        return "Error: Max iterations reached"

    def _load_history(self) -> list:
        """Load conversation history"""
        if self.history_file.exists():
            import json
            with open(self.history_file) as f:
                return json.load(f)

        # Start with system prompt
        return [{
            "role": "system",
            "content": self.config['system_prompt']
        }]

    def _save_history(self):
        """Save conversation history"""
        import json
        with open(self.history_file, 'w') as f:
            json.dump(self.messages, f, indent=2)
```

## CLI Interface

**File: `agenthub.py` (~50 lines)**

```python
#!/usr/bin/env python3
import sys
from pathlib import Path
from dotenv import load_dotenv
from runner import AgentRunner

def main():
    # Load environment variables
    load_dotenv()

    # Parse command
    if len(sys.argv) < 2:
        print("Usage: agenthub <command> [args]")
        print("\nCommands:")
        print("  chat <agent> <message>   - Chat with an agent")
        print("  list                     - List available agents")
        print("  new <agent>              - Create new agent")
        sys.exit(1)

    command = sys.argv[1]

    if command == "list":
        list_agents()

    elif command == "chat":
        if len(sys.argv) < 4:
            print("Usage: agenthub chat <agent> <message>")
            sys.exit(1)

        agent_id = sys.argv[2]
        message = " ".join(sys.argv[3:])

        # Run agent
        agent_path = Path("agents") / f"{agent_id}.yml"
        if not agent_path.exists():
            print(f"Agent not found: {agent_id}")
            print(f"Available agents: {list_agent_ids()}")
            sys.exit(1)

        runner = AgentRunner(str(agent_path))
        response = runner.chat(message)
        print(response)

    elif command == "new":
        if len(sys.argv) < 3:
            print("Usage: agenthub new <agent>")
            sys.exit(1)

        agent_id = sys.argv[2]
        create_agent(agent_id)

    else:
        print(f"Unknown command: {command}")
        sys.exit(1)

def list_agents():
    """List all available agents"""
    agents_dir = Path("agents")
    if not agents_dir.exists():
        print("No agents directory found")
        return

    for agent_file in agents_dir.glob("*.yml"):
        print(f"  {agent_file.stem}")

def list_agent_ids() -> list:
    """Get list of agent IDs"""
    agents_dir = Path("agents")
    return [f.stem for f in agents_dir.glob("*.yml")]

def create_agent(agent_id: str):
    """Create new agent from template"""
    # TODO: Create agent YAML from template
    print(f"Creating agent: {agent_id}")

if __name__ == "__main__":
    main()
```

## Data Isolation

**Simple folder-based isolation:**

```python
# Each agent gets its own folder
data/
  finance/              # Finance agent's sandbox
    transactions.json
    budget.txt
  research/             # Research agent's sandbox
    papers/
    notes.md
  work/                 # Work agent's sandbox
    projects/
    tasks.json
```

**Enforcement:**
- `AgentRunner` initializes with `data_dir = Path(config['data_dir'])`
- All file operations are relative to `data_dir`
- Tools can't access files outside `data_dir`

**Example:**
```python
# In finance agent
file_reader("transactions.json")  # Reads data/finance/transactions.json

# Trying to access other agent's data
file_reader("../research/notes.md")  # Blocked by path validation
```

## Secret Management

**.env file (gitignored):**
```bash
# LLM API Keys
OPENAI_API_KEY=sk-proj-...
ANTHROPIC_API_KEY=sk-ant-...

# Tool API Keys
STOCKS_API_KEY=xyz123
WEATHER_API_KEY=abc456
```

**Agent config references secrets:**
```yaml
model:
  api_key_env: OPENAI_API_KEY  # Name, not value
```

**Resolution at runtime:**
```python
api_key = os.getenv(config['model']['api_key_env'])
llm = OpenAI(api_key=api_key)
```

**Secrets never:**
- Committed to Git
- Sent to LLM in messages
- Exposed in logs (sanitize)

## Technology Stack

**Core:**
- Python 3.10+
- LangChain (LLM abstraction)
- PyYAML (config parsing)
- python-dotenv (secret loading)

**LLM Providers:**
- Ollama (local, free)
- OpenAI (cloud, paid)
- Anthropic (cloud, paid)

**Optional:**
- FastAPI (if adding web UI later)
- Rich (pretty CLI output)
- Typer (better CLI args)

## Deployment

**Development:**
```bash
git clone repo
cd agenthub
cp .env.example .env
# Edit .env with your API keys
pip install -r requirements.txt
./agenthub.py chat finance "How much did I spend?"
```

**Production (self-hosted):**
```bash
# Same as development!
# Run on your laptop, server, or Raspberry Pi
```

**Optional: Docker**
```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "agenthub.py"]
```

## What We're NOT Building

- ❌ Complex orchestration system
- ❌ MCP abstraction layer
- ❌ Multi-user auth system
- ❌ Separate agent/MCP registries
- ❌ Database for sessions (use JSON files)
- ❌ Vector stores (add later if needed)
- ❌ Web UI first (CLI is MVP)

## Comparison: Before vs After

| Component | Old Design | New Design |
|-----------|------------|------------|
| **Backend** | 10+ components | 1 (AgentRunner) |
| **Config Files** | agents/*.yml + mcp/*.yml + data_scopes.yml | agents/*.yml only |
| **Lines of Code** | ~5000+ | ~500 |
| **Databases** | Multiple SQLite | None (JSON files) |
| **Dependencies** | Many | Few (LangChain, PyYAML, dotenv) |
| **Time to MVP** | Weeks | Weekend |

## Next Steps

1. **Implement `runner.py`** - Core agent logic (~200 lines)
2. **Implement `agenthub.py`** - CLI interface (~50 lines)
3. **Add 3 built-in tools** - calculator, file_ops, web_search (~100 lines)
4. **Create 3 example agents** - generic, finance, research
5. **Test with Ollama** - Local LLM first
6. **Write docs** - README with examples
7. **Ship v0.1** - Get feedback

**Start coding, not planning.**
