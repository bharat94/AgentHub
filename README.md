# AgentHub

**Simple, self-hosted AI agents with data isolation.**

Run multiple AI agents, each with its own data folder and tools. Your finance agent can't see your research notes. Your work agent can't access personal data. **Zero cross-contamination.**

```bash
# Setup in 5 minutes
git clone https://github.com/yourusername/AgentHub.git
cd AgentHub
cp .env.example .env  # Add your API keys
pip install -r requirements.txt

# Chat with an agent
./agenthub chat finance "How much did I spend this month?"
./agenthub chat research "Summarize these papers"
./agenthub chat work "Draft a status update"
```

---

## Why AgentHub?

**One AI assistant sees all your data. That's risky.**

- You ask ChatGPT about finances, then about work projects
- Everything mixes in one conversation history
- No boundaries between personal, work, and sensitive data

**AgentHub gives each agent its own sandbox:**

```
data/
  finance/       ← Finance agent's files (isolated)
  research/      ← Research agent's files (isolated)
  work/          ← Work agent's files (isolated)
```

Each agent only accesses its folder. Simple as that.

---

## Quick Start

### 1. Install

```bash
git clone https://github.com/yourusername/AgentHub.git
cd AgentHub

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Set up secrets
cp .env.example .env
# Edit .env and add your API keys
```

### 2. Create Your First Agent

```yaml
# agents/my-agent.yml
id: my-agent
model:
  provider: ollama  # Start with local LLM (free!)
  model: llama3
system_prompt: "You are a helpful assistant."
data_dir: data/my-agent
tools:
  - calculator
  - file_reader
  - file_writer
```

### 3. Chat

```bash
./agenthub chat my-agent "Hello!"
```

That's it!

---

## Example Agents

### Finance Agent

```yaml
# agents/finance.yml
id: finance
name: Finance Assistant
model:
  provider: ollama
  model: llama3
system_prompt: |
  You are a personal finance assistant. Help track expenses,
  analyze spending, and provide budgeting advice.
data_dir: data/finance
tools:
  - calculator
  - file_reader
  - file_writer
```

**Usage:**
```bash
# Add some transaction data
echo '[{"date": "2025-01-10", "amount": 45.50, "category": "groceries"}]' > data/finance/transactions.json

# Ask about spending
./agenthub chat finance "How much did I spend on groceries?"
```

### Research Agent

```yaml
# agents/research.yml
id: research
name: Research Assistant
model:
  provider: ollama
  model: llama3
system_prompt: |
  You are a research assistant. Help organize notes, summarize papers,
  and track research progress.
data_dir: data/research
tools:
  - file_reader
  - file_writer
  - web_search
```

**Usage:**
```bash
./agenthub chat research "Summarize the papers in my folder"
```

---

## Key Features

### 🔒 Data Isolation
Each agent gets its own folder. Finance agent can't read research notes.

### 🔐 Git-Safe
Configs go in Git ✅
Secrets go in `.env` ❌ (gitignored)
Data goes in `data/` ❌ (gitignored)

### 🧩 Simple & Extensible
- One YAML file per agent
- Add tools in 5 lines of Python
- ~500 lines of code total

### 🌐 Works with Any LLM
- **Local**: Ollama (free, private)
- **Cloud**: OpenAI, Anthropic

---

## CLI Commands

```bash
# List agents
./agenthub list

# Chat with an agent
./agenthub chat <agent> "<message>"

# Reset conversation history
./agenthub reset <agent>

# Show agent info
./agenthub info <agent>

# Create new agent
./agenthub new <agent>
```

---

## Adding Custom Tools

### Python Function Tool

```python
# tools/my_tools.py
def analyze_text(text: str) -> dict:
    """Count words, sentences, etc."""
    words = len(text.split())
    sentences = text.count('.') + text.count('!') + text.count('?')
    return {"words": words, "sentences": sentences}
```

**Register in agent config:**
```yaml
tools:
  - calculator
  - type: python
    module: tools.my_tools
    functions: [analyze_text]
```

### HTTP API Tool

```yaml
tools:
  - type: http
    name: get_weather
    url: https://api.weather.com/current/{city}
    headers:
      api-key_env: WEATHER_API_KEY
```

---

## Project Structure

```
AgentHub/
├── agenthub.py              # CLI entry point
├── runner.py                # Core AgentRunner class
├── tools/                   # Built-in tools
│   ├── calculator.py
│   ├── file_ops.py
│   └── web_search.py
├── agents/                  # Agent configs (commit these)
│   ├── finance.yml
│   ├── research.yml
│   └── work.yml
├── data/                    # Agent data (gitignored)
│   ├── finance/
│   ├── research/
│   └── work/
├── .env                     # Secrets (gitignored)
├── .env.example             # Template (commit this)
├── requirements.txt
└── README.md
```

---

## Configuration

### Agent Config (agents/*.yml)

```yaml
id: my-agent               # Unique ID
name: My Agent             # Display name
description: What it does  # Optional

# LLM Configuration
model:
  provider: ollama         # ollama | openai | anthropic
  model: llama3            # Model name
  # For cloud providers:
  # api_key_env: OPENAI_API_KEY
  temperature: 0.7         # Optional
  max_tokens: 2000         # Optional

# System Instructions
system_prompt: |
  You are a helpful assistant...

# Data Isolation
data_dir: data/my-agent    # Agent's sandbox

# Tools
tools:
  - calculator             # Built-in tools
  - file_reader
  - file_writer

# Authorization (simple)
allowed_users: ["*"]       # Everyone
```

### Environment Variables (.env)

```bash
# LLM API Keys
OPENAI_API_KEY=sk-proj-...
ANTHROPIC_API_KEY=sk-ant-...

# Local LLM
OLLAMA_BASE_URL=http://localhost:11434

# Custom Tool APIs
STOCKS_API_KEY=...
WEATHER_API_KEY=...
```

---

## Security

**Three simple rules:**
1. **Secrets in `.env`** - Never in Git ✅
2. **Data in folders** - Each agent gets its own ✅
3. **Path validation** - No directory traversal ✅

**What's safe to commit:**
- ✅ `agents/*.yml` (agent configs)
- ✅ `tools/*.py` (tool implementations)
- ✅ `.env.example` (template)

**What's gitignored:**
- ❌ `.env` (your API keys)
- ❌ `data/` (private data)
- ❌ `**/.history.json` (conversation history)

---

## Roadmap

### MVP (Week 1) - 🚧 In Progress
- [x] Design simplified architecture
- [ ] Implement AgentRunner (~200 lines)
- [ ] Implement CLI (~50 lines)
- [ ] Add built-in tools (calculator, file ops, web search)
- [ ] Create 3 example agents
- [ ] Test with Ollama

### Phase 2 (Week 2-3)
- [ ] Add OpenAI/Anthropic support
- [ ] Improve error messages
- [ ] Add session management
- [ ] Create 10+ agent templates
- [ ] Write comprehensive docs

### Phase 3 (Month 2)
- [ ] Optional web UI (if users want it)
- [ ] Agent marketplace/templates
- [ ] Plugin system for tools
- [ ] Community contributions

---

## FAQ

**Q: Do I need to run anything in the cloud?**
A: No! Use Ollama for fully local, private AI.

**Q: Can I use OpenAI/Claude?**
A: Yes! Just add your API key to `.env` and change the provider in agent config.

**Q: How do I share my agents?**
A: Commit the `agents/*.yml` files to Git. Others copy `.env.example` to `.env` and add their own API keys.

**Q: Is my data private?**
A: Yes. Data stays in `data/` folder on your machine (gitignored). With Ollama, everything runs locally.

**Q: Can agents talk to each other?**
A: Not in MVP. Each agent is isolated. We may add this later if users want it.

**Q: How do I add a new tool?**
A: Write a Python function in `tools/`, then register it in your agent's YAML config. See "Adding Custom Tools" above.

---

## Contributing

We welcome contributions! Please:
1. Fork the repo
2. Create a feature branch
3. Make your changes
4. Submit a pull request

### Ideas for Contributions
- New built-in tools
- Example agent templates
- Better error messages
- Documentation improvements
- Bug fixes

---

## License

MIT License - see [LICENSE](./LICENSE) for details.

---

## Acknowledgments

- Inspired by [Claude Code](https://claude.com/claude-code)
- Built with [LangChain](https://python.langchain.com/)
- Powered by [Ollama](https://ollama.ai/) for local LLMs

---

## Support

- **Issues**: [GitHub Issues](https://github.com/yourusername/AgentHub/issues)
- **Discussions**: [GitHub Discussions](https://github.com/yourusername/AgentHub/discussions)

---

**Start in 5 minutes. Build agents in 5 lines.**

```bash
git clone https://github.com/yourusername/AgentHub.git
cd AgentHub && cp .env.example .env
pip install -r requirements.txt
./agenthub chat finance "Hello!"
```
