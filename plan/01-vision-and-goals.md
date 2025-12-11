# AgentHub - Vision and Goals (Simplified)

## Vision Statement

AgentHub is a **simple, self-hosted tool** for running multiple AI agents with isolated data access. Each agent is a YAML config file with its own system prompt, tools, and data folder.

Think of it as **apps for AI** - your finance agent can't see your research notes, your work agent can't access personal data.

**Start in 5 minutes. Extend in 5 lines of Python.**

## The Problem We're Solving

**One AI assistant sees all your data.** That's risky.

- You ask ChatGPT about finances, then about work projects
- Everything mixes in one conversation history
- No boundaries between personal, work, and sensitive data

**We need data isolation without complexity.**

## Core Goals (MVP)

### 1. **Dead Simple Setup**
```bash
git clone https://github.com/you/agenthub
cd agenthub
cp .env.example .env  # Add your API keys
pip install -r requirements.txt
./agenthub chat finance "How much did I spend?"
```

### 2. **One Config File Per Agent**
```yaml
# agents/finance.yml
id: finance
model:
  provider: ollama
  model: llama3
system_prompt: "You are a finance assistant..."
data_dir: data/finance
tools: [calculator, file_reader]
```

### 3. **Folder-Based Data Isolation**
```
data/
  finance/        ← Finance agent's sandbox
  research/       ← Research agent's sandbox
  personal/       ← Personal agent's sandbox
```
Each agent only accesses its folder. Simple as that.

### 4. **Git-Safe by Default**
- YAML configs → committed ✅
- `.env` secrets → gitignored ✅
- `data/` folders → gitignored ✅

### 5. **Extensible Without Complexity**
Add a new tool:
```python
# tools/my_tool.py
def my_tool(arg):
    return "result"
```

Register it:
```yaml
# agents/my-agent.yml
tools:
  - type: python
    module: tools.my_tool
    function: my_tool
```

## Use Cases

**Finance Agent**
```bash
./agenthub chat finance "How much did I spend on groceries?"
# Only has access to data/finance/ folder
```

**Research Agent**
```bash
./agenthub chat research "Summarize these papers"
# Only has access to data/research/ folder
```

**Work Agent**
```bash
./agenthub chat work "Draft a status update"
# Only has access to data/work/ folder
```

**Zero cross-contamination.** Each agent is isolated.

## Success Criteria

### MVP (Week 1)
- [ ] CLI working: `./agenthub chat <agent> "<message>"`
- [ ] 3 example agents: generic, finance, research
- [ ] Folder-based data isolation working
- [ ] Works with Ollama (local LLM)
- [ ] Secrets in `.env`, configs in Git

### Phase 2 (Week 2-3)
- [ ] Add OpenAI/Anthropic support
- [ ] Custom Python tool system
- [ ] Session history (simple JSON files)
- [ ] 10+ example agent templates

### Phase 3 (Month 2)
- [ ] Optional web UI (if users ask for it)
- [ ] Agent marketplace/sharing
- [ ] Better tool docs

**Ship fast. Add features when users demand them.**

## Non-Goals

- ❌ Enterprise features (multi-user, RBAC, audit logs)
- ❌ Complex orchestration (keep it simple)
- ❌ Building an AI framework (use LiteLLM/LangChain)
- ❌ Web UI first (CLI first, UI later if needed)
- ❌ Mobile apps

## Why This Matters

**For power users:**
- Run AI locally (Ollama) for privacy
- Separate work, personal, and sensitive data
- Git-based workflow (configs in, secrets out)

**For developers:**
- Fork and customize in Python
- Share agent templates publicly
- Build tools as simple Python functions

**For teams:**
- Share agent configs, not secrets
- Everyone has their own `.env` file

## Principles

1. **Simple beats complex** - If you can't explain it in 3 sentences, simplify
2. **CLI first, UI later** - Terminal is faster to build and debug
3. **Local first** - Ollama before OpenAI
4. **Files over databases** - Until you need a database
5. **Copy-paste beats abstraction** - Until you need DRY
