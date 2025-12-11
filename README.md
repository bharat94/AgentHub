# AgentHub

A self-hosted, privacy-first AI agent orchestration platform that enables you to run multiple specialized AI agents with strict data isolation, secret management, and Git-safe configuration.

## Overview

AgentHub is your personal AI operating system where each agent is like an app with specific permissions, tools, and data access. Built for privacy, security, and ease of use.

## Key Features

- **🔒 Privacy-First**: Self-hosted, your data never leaves your machine
- **🎯 Data Isolation**: Each agent only accesses its designated data scopes
- **🔐 Secret Management**: API keys and tokens safely stored in `.env`, never committed to Git
- **🔧 Extensible**: Easy to add new agents, tools, and MCP servers via YAML configs
- **🌐 Multi-LLM Support**: Works with OpenAI, Anthropic, Ollama, and more
- **🛠️ MCP Integration**: Model Context Protocol for standardized tool integration

## Architecture

AgentHub consists of:
- **Agent Orchestrator**: Core execution engine
- **Agent Registry**: Loads and validates agent configs
- **MCP Registry**: Manages external tool integrations
- **Data Scope Mapper**: Enforces data isolation
- **Secrets Resolver**: Secure secret management

## Project Structure

```
AgentHub/
├── plan/                           # Design documents (you are here)
│   ├── 01-vision-and-goals.md     # Project vision and objectives
│   ├── 02-architecture-overview.md # System architecture
│   ├── 03-component-specs.md      # Component specifications
│   ├── 04-execution-flow.md       # Data flow and execution
│   └── 05-security-principles.md  # Security and isolation
│
├── agents/                         # Agent configurations (coming soon)
├── mcp/                           # MCP server configs (coming soon)
├── backend/                       # Backend server code (coming soon)
├── frontend/                      # Web UI (coming soon)
└── data/                          # Private data stores (gitignored)
```

## Getting Started

### Prerequisites

- Node.js 18+ or Python 3.10+
- API keys for your chosen LLM provider (OpenAI, Anthropic, etc.)

### Installation

```bash
# Clone the repository
git clone https://github.com/yourusername/AgentHub.git
cd AgentHub

# Copy environment template
cp .env.example .env

# Add your API keys to .env
nano .env

# Install dependencies (coming soon)
# npm install
# or
# pip install -r requirements.txt

# Start the server (coming soon)
# npm run dev
# or
# python -m agenthub.server
```

## Documentation

All design documents are in the [`plan/`](./plan) directory:

- **[Vision and Goals](./plan/01-vision-and-goals.md)** - What we're building and why
- **[Architecture Overview](./plan/02-architecture-overview.md)** - System design and components
- **[Component Specs](./plan/03-component-specs.md)** - Detailed specifications
- **[Execution Flow](./plan/04-execution-flow.md)** - How requests flow through the system
- **[Security Principles](./plan/05-security-principles.md)** - Security and isolation design

## Current Status

🚧 **Early Development** - We're currently in the design and planning phase. The architecture is finalized and implementation is starting soon.

### Roadmap

- [x] Phase 0: Design and Architecture
- [ ] Phase 1: MVP (Core functionality)
- [ ] Phase 2: Polish and UX
- [ ] Phase 3: Scale and Collaboration
- [ ] Phase 4: Advanced Features

## Example: Finance Assistant Agent

```yaml
# agents/finance-assistant.yml
id: finance-assistant
name: Finance Assistant
description: Personal finance tracking and budget management

model:
  provider: openai
  model_name: gpt-4
  secret_ref: OPENAI_API_KEY

tools:
  mcp_servers:
    - finance_mcp

data_scopes:
  - finances          # Only access to finances.db

auth:
  allowed_users:
    - "*"            # All authenticated users
```

## Why AgentHub?

### For Individuals
- Full control over your AI interactions
- Privacy for sensitive data (finance, health, personal notes)
- Use any LLM (local or cloud)
- Cost optimization (use cheap models for simple tasks)

### For Teams
- Shared agent configurations without sharing secrets
- Git-based workflow
- Easy onboarding
- Audit trail for compliance

### For Developers
- Open-source friendly architecture
- Shareable agent templates
- Easy to extend and customize

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](./CONTRIBUTING.md) (coming soon) for details.

## License

MIT License - see [LICENSE](./LICENSE) for details.

## Security

Found a security issue? Please email security@example.com instead of opening a public issue.

## Acknowledgments

- Inspired by [Claude Code](https://claude.com/claude-code) and the [Model Context Protocol](https://modelcontextprotocol.io)
- Built with love for the AI community

## Contact

- GitHub Issues: [https://github.com/yourusername/AgentHub/issues](https://github.com/yourusername/AgentHub/issues)
- Discussions: [https://github.com/yourusername/AgentHub/discussions](https://github.com/yourusername/AgentHub/discussions)

---

**Note**: This is an early-stage project. The architecture is solid, and implementation is in progress. Star the repo to follow our progress!
