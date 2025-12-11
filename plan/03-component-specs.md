# AgentHub - Component Specifications (Simplified)

## Overview

AgentHub has **3 core components**:
1. **AgentRunner** - Loads config, runs LLM conversation loop
2. **CLI** - Command-line interface
3. **Tools** - Built-in functions agents can use

That's it. No registries, no mappers, no orchestrators.

---

## 1. Agent Configuration (YAML)

### Purpose
Define everything about an agent in one file.

### Location
`agents/{agent-id}.yml`

### Schema

```yaml
# Minimal config
id: my-agent
model:
  provider: ollama
  model: llama3
system_prompt: "You are a helpful assistant."
data_dir: data/my-agent

# Full config
id: finance
name: Finance Assistant
description: Personal finance management

model:
  provider: openai              # ollama | openai | anthropic
  model: gpt-4
  api_key_env: OPENAI_API_KEY   # Reference to .env
  temperature: 0.7
  max_tokens: 2000

system_prompt: |
  You are a personal finance assistant with access to transaction data.
  Help the user track expenses, analyze spending, and manage budgets.

data_dir: data/finance          # Agent's data sandbox

tools:
  - calculator                  # Built-in tools
  - file_reader
  - file_writer
  - web_search

  # Custom Python functions
  - type: python
    module: tools.finance_tools
    functions: [analyze_transactions, get_balance]

  # HTTP APIs
  - type: http
    name: stock_price
    url: https://api.stocks.com/price/{symbol}
    headers:
      auth_env: STOCKS_API_KEY

allowed_users: ["*"]            # Simple auth
```

### Validation

Simple Python validation:
```python
required_fields = ['id', 'model', 'system_prompt', 'data_dir']
for field in required_fields:
    if field not in config:
        raise ValueError(f"Missing required field: {field}")

if config['model']['provider'] not in ['ollama', 'openai', 'anthropic']:
    raise ValueError(f"Unknown provider: {config['model']['provider']}")
```

---

## 2. AgentRunner Class

### Purpose
Core logic for running an agent: load config, init LLM, execute conversation loop.

### File
`runner.py` (~200-300 lines)

### Interface

```python
class AgentRunner:
    def __init__(self, agent_config_path: str):
        """
        Initialize agent from config file.
        - Load YAML config
        - Create data directory
        - Initialize LLM client
        - Load tools
        - Load conversation history
        """

    def chat(self, message: str) -> str:
        """
        Process user message and return response.
        Handles tool calls automatically.
        """

    def reset(self):
        """Clear conversation history."""

    def _load_config(self, path: str) -> dict:
        """Load and validate agent config."""

    def _init_llm(self):
        """Initialize LLM client based on provider."""

    def _load_tools(self) -> dict:
        """Load tools from config."""

    def _execute_tool(self, tool_call) -> str:
        """Execute a tool call."""

    def _load_history(self) -> list:
        """Load conversation history from file."""

    def _save_history(self):
        """Save conversation history to file."""
```

### Example Usage

```python
from runner import AgentRunner

# Initialize agent
runner = AgentRunner("agents/finance.yml")

# Chat
response = runner.chat("How much did I spend on groceries?")
print(response)

# Continue conversation
response = runner.chat("What about restaurants?")
print(response)

# Reset
runner.reset()
```

### Error Handling

```python
# Missing .env variable
if not api_key:
    raise ValueError(
        f"Missing {api_key_env} in .env file. "
        f"Please add it to your .env file."
    )

# Agent not found
if not agent_path.exists():
    raise FileNotFoundError(
        f"Agent '{agent_id}' not found. "
        f"Available agents: {list_agent_ids()}"
    )

# Tool execution error
try:
    result = tool_func(**args)
except Exception as e:
    result = f"Error executing tool: {str(e)}"
```

---

## 3. CLI Interface

### Purpose
Command-line interface for interacting with agents.

### File
`agenthub.py` (~50-100 lines)

### Commands

```bash
# List available agents
$ agenthub list
finance
research
work

# Chat with an agent
$ agenthub chat finance "How much did I spend?"
You spent $450.23 on groceries last month.

# Continue conversation (uses history)
$ agenthub chat finance "What about restaurants?"
You spent $230.50 on restaurants.

# Reset conversation
$ agenthub reset finance
Conversation history cleared.

# Create new agent
$ agenthub new my-agent
Created agents/my-agent.yml

# Show agent info
$ agenthub info finance
ID: finance
Name: Finance Assistant
Model: gpt-4 (OpenAI)
Data: data/finance/
Tools: calculator, file_reader, file_writer
```

### Implementation

```python
#!/usr/bin/env python3
import sys
from pathlib import Path
from dotenv import load_dotenv
from runner import AgentRunner

def main():
    load_dotenv()

    if len(sys.argv) < 2:
        print_help()
        sys.exit(1)

    command = sys.argv[1]

    if command == "list":
        cmd_list()
    elif command == "chat":
        cmd_chat(sys.argv[2:])
    elif command == "reset":
        cmd_reset(sys.argv[2])
    elif command == "new":
        cmd_new(sys.argv[2])
    elif command == "info":
        cmd_info(sys.argv[2])
    else:
        print(f"Unknown command: {command}")
        sys.exit(1)

def cmd_chat(args):
    if len(args) < 2:
        print("Usage: agenthub chat <agent> <message>")
        sys.exit(1)

    agent_id = args[0]
    message = " ".join(args[1:])

    runner = AgentRunner(f"agents/{agent_id}.yml")
    response = runner.chat(message)
    print(response)
```

---

## 4. Built-in Tools

### Purpose
Provide common functionality to all agents.

### Location
`tools/` directory

### Available Tools

#### Calculator
```python
# tools/calculator.py
def calculate(expression: str) -> float:
    """
    Safely evaluate math expressions.
    Example: calculate("2 + 2 * 3") -> 8.0
    """
    import ast
    import operator

    allowed_ops = {
        ast.Add: operator.add,
        ast.Sub: operator.sub,
        ast.Mult: operator.mul,
        ast.Div: operator.truediv,
        ast.Pow: operator.pow,
    }

    def eval_expr(node):
        if isinstance(node, ast.Num):
            return node.n
        elif isinstance(node, ast.BinOp):
            return allowed_ops[type(node.op)](
                eval_expr(node.left),
                eval_expr(node.right)
            )
        else:
            raise ValueError("Invalid expression")

    tree = ast.parse(expression, mode='eval')
    return eval_expr(tree.body)
```

#### File Reader (Scoped)
```python
# tools/file_ops.py
from pathlib import Path

def read_file(path: Path) -> str:
    """
    Read file from agent's data directory.
    Path must be relative and within data_dir.
    """
    # Prevent directory traversal
    if ".." in str(path) or path.is_absolute():
        raise ValueError("Invalid path: must be relative and within data_dir")

    if not path.exists():
        raise FileNotFoundError(f"File not found: {path}")

    return path.read_text()

def write_file(path: Path, content: str):
    """Write file to agent's data directory."""
    if ".." in str(path) or path.is_absolute():
        raise ValueError("Invalid path: must be relative and within data_dir")

    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(content)

def list_files(dir_path: Path) -> list[str]:
    """List files in directory."""
    if not dir_path.exists():
        return []
    return [str(f.relative_to(dir_path)) for f in dir_path.rglob("*") if f.is_file()]
```

#### Web Search
```python
# tools/web_search.py
import requests

def search(query: str, num_results: int = 5) -> str:
    """
    Search the web and return results.
    Uses DuckDuckGo or similar free API.
    """
    # Implementation using DuckDuckGo API or similar
    # Return formatted results as string
    pass
```

---

## 5. Data Isolation

### Purpose
Ensure each agent can only access its own data folder.

### Implementation

**In AgentRunner:**
```python
def __init__(self, agent_config_path: str):
    self.config = self._load_config(agent_config_path)
    self.data_dir = Path(self.config['data_dir'])
    self.data_dir.mkdir(parents=True, exist_ok=True)
```

**In Tools:**
```python
def read_file(path: str) -> str:
    # path is relative to data_dir
    full_path = self.data_dir / path

    # Validate path is within data_dir
    try:
        full_path.resolve().relative_to(self.data_dir.resolve())
    except ValueError:
        raise PermissionError("Access denied: path outside data_dir")

    return full_path.read_text()
```

**Directory Structure:**
```
data/
  finance/
    .history.json      # Conversation history (auto-created)
    transactions.json  # User data
    notes.txt
  research/
    .history.json
    papers/
    notes.md
  work/
    .history.json
    projects/
```

---

## 6. Secret Management

### Purpose
Keep API keys and tokens out of Git.

### .env File

```bash
# .env (gitignored)
# LLM Providers
OPENAI_API_KEY=sk-proj-...
ANTHROPIC_API_KEY=sk-ant-...

# Custom Tool APIs
STOCKS_API_KEY=abc123
WEATHER_API_KEY=xyz789
```

### .env.example File

```bash
# .env.example (committed)
# Copy this to .env and fill in your keys

# LLM Providers
OPENAI_API_KEY=your_key_here
ANTHROPIC_API_KEY=your_key_here

# Custom Tool APIs
STOCKS_API_KEY=your_key_here
WEATHER_API_KEY=your_key_here
```

### Usage in Code

```python
import os
from dotenv import load_dotenv

load_dotenv()

# Read secret
api_key = os.getenv("OPENAI_API_KEY")
if not api_key:
    raise ValueError("Missing OPENAI_API_KEY in .env")

# Use secret (never log or send to LLM)
llm = OpenAI(api_key=api_key)
```

---

## 7. Conversation History

### Purpose
Maintain context across multiple messages.

### Storage

**File:** `data/{agent}/.history.json`

**Format:**
```json
[
  {
    "role": "system",
    "content": "You are a finance assistant..."
  },
  {
    "role": "user",
    "content": "How much did I spend?"
  },
  {
    "role": "assistant",
    "content": "You spent $450.23..."
  }
]
```

### Implementation

```python
def _load_history(self) -> list:
    """Load conversation history."""
    if self.history_file.exists():
        import json
        with open(self.history_file) as f:
            return json.load(f)

    # Start fresh with system prompt
    return [{"role": "system", "content": self.config['system_prompt']}]

def _save_history(self):
    """Save conversation history."""
    import json
    with open(self.history_file, 'w') as f:
        json.dump(self.messages, f, indent=2)

def reset(self):
    """Clear conversation history."""
    self.messages = [{"role": "system", "content": self.config['system_prompt']}]
    self._save_history()
```

---

## 8. Custom Tools

### Purpose
Let users add their own tools easily.

### Python Functions

**Create tool:**
```python
# tools/finance_tools.py
def analyze_transactions(file_path: str) -> dict:
    """Analyze transactions from JSON file."""
    import json
    with open(file_path) as f:
        transactions = json.load(f)

    total = sum(t['amount'] for t in transactions)
    categories = {}
    for t in transactions:
        cat = t['category']
        categories[cat] = categories.get(cat, 0) + t['amount']

    return {
        'total': total,
        'by_category': categories
    }
```

**Register in agent config:**
```yaml
tools:
  - type: python
    module: tools.finance_tools
    functions: [analyze_transactions]
```

**Usage by LLM:**
The LLM can now call:
```json
{
  "tool_name": "analyze_transactions",
  "args": {"file_path": "transactions.json"}
}
```

### HTTP APIs

**Register API in agent config:**
```yaml
tools:
  - type: http
    name: get_weather
    url: https://api.weather.com/current/{city}
    method: GET
    headers:
      api-key_env: WEATHER_API_KEY
```

**Implementation:**
```python
def _create_http_tool(self, config: dict):
    """Create HTTP API wrapper."""
    def http_tool(**kwargs):
        import requests
        url = config['url'].format(**kwargs)
        headers = {}

        # Add auth header if specified
        if 'headers' in config:
            for key, value in config['headers'].items():
                if key.endswith('_env'):
                    # Resolve from .env
                    headers[key[:-4]] = os.getenv(value)
                else:
                    headers[key] = value

        response = requests.get(url, headers=headers)
        return response.json()

    return http_tool
```

---

## Summary

| Component | Lines of Code | Complexity |
|-----------|---------------|------------|
| AgentRunner | ~200 | Medium |
| CLI | ~50 | Low |
| Built-in Tools | ~100 | Low |
| **Total** | **~350** | **Low** |

**Keep it simple. Ship it fast.**
