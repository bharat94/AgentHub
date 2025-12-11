# AgentHub - Security Principles (Simplified)

## Core Security Model

**Three simple rules:**
1. **Secrets in `.env`** - Never in Git
2. **Data in folders** - Each agent gets its own
3. **Path validation** - No directory traversal

That's it. No complex security architecture.

---

## 1. Git Safety

### Problem
How do you share agent configs without leaking API keys?

### Solution
Separate secrets from config.

**What's committed:**
```
✅ agents/*.yml          # Agent configs (no secrets)
✅ tools/*.py            # Tool implementations
✅ .env.example          # Template
✅ .gitignore            # Ensures .env is ignored
```

**What's gitignored:**
```
❌ .env                  # API keys and tokens
❌ data/                 # Private data
❌ **/.history.json      # Conversation history
```

**.gitignore:**
```gitignore
# Secrets
.env
.env.*

# Private data
data/

# Logs
*.log
```

**.env.example (template):**
```bash
# Copy this to .env and fill in your keys

# LLM Providers
OPENAI_API_KEY=your_key_here
ANTHROPIC_API_KEY=your_key_here
OLLAMA_BASE_URL=http://localhost:11434

# Custom APIs
STOCKS_API_KEY=your_key_here
```

**Result:** You can commit your entire agent library to GitHub without exposing secrets.

---

## 2. Data Isolation

### Problem
How do you prevent finance agent from reading research data?

### Solution
Each agent gets a folder. Tools can't access outside that folder.

**Directory structure:**
```
data/
  finance/              # Finance agent's sandbox
    transactions.json
    .history.json
  research/             # Research agent's sandbox
    notes.md
    .history.json
  work/                 # Work agent's sandbox
    projects/
    .history.json
```

**Enforcement:**
```python
class AgentRunner:
    def __init__(self, agent_config_path):
        # Each agent gets its own data directory
        self.data_dir = Path(self.config['data_dir'])

        # All file operations are relative to this
        self.data_dir.mkdir(parents=True, exist_ok=True)

# File read tool
def read_file(self, path: str) -> str:
    full_path = self.data_dir / path

    # Validate path
    if ".." in str(path):
        raise ValueError("Path traversal not allowed")

    if path.is_absolute():
        raise ValueError("Absolute paths not allowed")

    # Ensure within data_dir
    try:
        full_path.resolve().relative_to(self.data_dir.resolve())
    except ValueError:
        raise PermissionError("Access denied")

    return full_path.read_text()
```

**Attack scenarios and defenses:**

| Attack | Code | Result |
|--------|------|--------|
| Directory traversal | `../research/notes.md` | ❌ Blocked by ".." check |
| Absolute path | `/etc/passwd` | ❌ Blocked by absolute path check |
| Symlink escape | `ln -s /etc data/link` | ❌ Blocked by resolve() check |
| Valid relative | `transactions.json` | ✅ Allowed |

---

## 3. Secret Management

### Problem
LLMs can't see API keys, but tools need them.

### Solution
Secrets resolved at runtime, never sent to LLM.

**Bad (❌):**
```python
# Sending secret to LLM
system_prompt = f"Use this API key: {api_key}"
llm.chat([{"role": "system", "content": system_prompt}])
```

**Good (✅):**
```python
# Secret used only for auth, not sent to LLM
llm = OpenAI(api_key=api_key)  # In HTTP header
response = llm.chat(messages)  # Secret not in messages
```

**Implementation:**
```python
# Load secrets from .env
from dotenv import load_dotenv
load_dotenv()

# Reference secrets by name in config
# agents/finance.yml
model:
  provider: openai
  api_key_env: OPENAI_API_KEY  # ← Name, not value

# Resolve at runtime
api_key = os.getenv(config['model']['api_key_env'])
if not api_key:
    raise ValueError(f"Missing {api_key_env} in .env")

# Use for authentication only
llm = OpenAI(api_key=api_key)
```

**Secrets are:**
- ✅ Loaded from `.env`
- ✅ Used for API authentication
- ✅ Never sent in LLM messages
- ✅ Never logged

**Secrets are NOT:**
- ❌ Hardcoded in configs
- ❌ Committed to Git
- ❌ Sent to LLM
- ❌ Visible in error messages

---

## 4. Input Validation

### Problem
What if the LLM tries malicious tool calls?

### Solution
Validate everything.

**Path validation:**
```python
def validate_path(path: str):
    """Ensure path is safe."""
    # No directory traversal
    if ".." in str(path):
        raise ValueError("Path traversal not allowed")

    # Must be relative
    if Path(path).is_absolute():
        raise ValueError("Absolute paths not allowed")

    # No null bytes
    if "\x00" in path:
        raise ValueError("Invalid path characters")

    return path
```

**Tool argument validation:**
```python
def _execute_tool(self, tool_call):
    """Execute tool with validation."""
    tool_name = tool_call.name
    tool_args = tool_call.args

    # Validate tool exists
    if tool_name not in self.tools:
        return f"Error: Unknown tool '{tool_name}'"

    # Validate arguments
    try:
        # Type checking, range validation, etc.
        validated_args = validate_tool_args(tool_name, tool_args)
    except ValueError as e:
        return f"Error: Invalid arguments: {e}"

    # Execute with validated args
    try:
        result = self.tools[tool_name](**validated_args)
    except Exception as e:
        return f"Error: {str(e)}"

    return result
```

**Expression evaluation (calculator):**
```python
def calculate(expression: str) -> float:
    """Safely evaluate math expressions."""
    import ast
    import operator

    # Only allow math operators
    allowed_ops = {
        ast.Add: operator.add,
        ast.Sub: operator.sub,
        ast.Mult: operator.mul,
        ast.Div: operator.truediv,
    }

    def eval_node(node):
        if isinstance(node, ast.Num):
            return node.n
        elif isinstance(node, ast.BinOp):
            op = allowed_ops.get(type(node.op))
            if not op:
                raise ValueError("Unsupported operation")
            return op(eval_node(node.left), eval_node(node.right))
        else:
            raise ValueError("Invalid expression")

    tree = ast.parse(expression, mode='eval')
    return eval_node(tree.body)
```

---

## 5. What We DON'T Do (MVP)

**Not worried about (yet):**
- ❌ Multi-user authentication - single user MVP
- ❌ Role-based access control - trust your agents
- ❌ Audit logging - add later if needed
- ❌ Rate limiting - not needed locally
- ❌ Encryption at rest - OS handles this
- ❌ Network security - running locally

**Why:**
- AgentHub runs on **your machine**
- You control the agent configs
- You control the tools
- You're not exposing to internet

**When to add:**
- Multi-user: Add auth when you want multiple users
- Audit logs: Add when you need compliance
- Rate limits: Add when using cloud APIs with limits
- Encryption: Add if storing sensitive regulated data

---

## 6. Threat Model

### What We Protect Against

| Threat | Mitigation |
|--------|------------|
| Secrets leaked in Git | `.env` is gitignored |
| Agent reads wrong data | Folder-based isolation + path validation |
| Directory traversal | ".." and absolute path checks |
| Malicious tool calls | Input validation on all args |
| API key in logs | Never log secrets |
| LLM sees API keys | Secrets used for auth only, not in messages |

### What We DON'T Protect Against (MVP)

| Threat | Why Not (Yet) |
|--------|---------------|
| Malicious agent configs | Trust model: you write the configs |
| Compromised Python packages | Use virtual env, review deps |
| Host OS compromise | Outside scope: secure your machine |
| Network MITM | Running locally, no network exposure |
| Side-channel attacks | Not relevant for local tool |

---

## 7. Security Checklist

### For Users

**Setup:**
- [ ] Never commit `.env` file
- [ ] Use `.env.example` as template
- [ ] Use strong, unique API keys
- [ ] Don't share your `.env` file

**Usage:**
- [ ] Review agent configs before running
- [ ] Only add tools you trust
- [ ] Don't run untrusted agent configs
- [ ] Keep your data in `data/` folder

**Maintenance:**
- [ ] Rotate API keys regularly
- [ ] Back up `.env` securely (password manager)
- [ ] Update dependencies (`pip install -U`)

### For Developers

**When adding tools:**
- [ ] Validate all inputs
- [ ] Sanitize all outputs
- [ ] Never log secrets
- [ ] Use scoped file access
- [ ] Handle errors gracefully

**When adding features:**
- [ ] Keep secrets in `.env`
- [ ] Never send secrets to LLM
- [ ] Validate paths before file ops
- [ ] Use least privilege principle

---

## 8. Example Attacks and Defenses

### Attack: Directory Traversal

**Attempt:**
```
LLM: file_reader("../../../etc/passwd")
```

**Defense:**
```python
if ".." in path:
    raise ValueError("Path traversal not allowed")
# Result: ❌ Blocked
```

### Attack: Absolute Path

**Attempt:**
```
LLM: file_reader("/etc/passwd")
```

**Defense:**
```python
if Path(path).is_absolute():
    raise ValueError("Absolute paths not allowed")
# Result: ❌ Blocked
```

### Attack: Symlink Escape

**Setup:**
```bash
ln -s /etc data/finance/link
```

**Attempt:**
```
LLM: file_reader("link/passwd")
```

**Defense:**
```python
full_path.resolve().relative_to(self.data_dir.resolve())
# Result: ❌ ValueError, blocked
```

### Attack: Code Injection in Calculator

**Attempt:**
```
LLM: calculator("__import__('os').system('rm -rf /')")
```

**Defense:**
```python
# Only AST nodes for math operations allowed
if not isinstance(node, (ast.Num, ast.BinOp)):
    raise ValueError("Invalid expression")
# Result: ❌ Blocked
```

---

## 9. Secure Defaults

**Principle: Secure by default, opt-in to relax.**

```python
# Default: No tools
tools: []

# Explicit: Add specific tools
tools: [calculator, file_reader]

# Default: No data access
data_dir: data/empty

# Explicit: Give specific folder
data_dir: data/finance

# Default: Only relative paths
# Absolute paths rejected automatically

# Default: Secrets from .env only
# Never from configs
```

---

## 10. Future Security Enhancements

**When you need more:**

1. **Multi-user mode:**
   ```python
   # Add user auth
   auth:
     type: jwt
     allowed_users: ["user@example.com"]
   ```

2. **Audit logging:**
   ```python
   # Log all tool calls
   logger.audit(f"Agent {agent_id} called {tool_name}")
   ```

3. **Rate limiting:**
   ```python
   # Limit API calls
   @rate_limit(max=10, per=60)
   def chat(message):
       ...
   ```

4. **Encryption at rest:**
   ```python
   # Encrypt sensitive data
   data = encrypt(content, key=os.getenv("ENCRYPTION_KEY"))
   ```

**But for MVP: Keep it simple. Add these when users ask for them.**

---

## Summary

**Security principles (in order of importance):**

1. **Secrets in `.env`** - Never in Git ✅
2. **Folder isolation** - Each agent gets a sandbox ✅
3. **Path validation** - No directory traversal ✅
4. **Input validation** - Validate all tool args ✅
5. **Secure defaults** - Opt-in to permissions ✅

**Not needed (yet):**
- ❌ Multi-user auth
- ❌ Audit logs
- ❌ Rate limiting
- ❌ Encryption at rest

**Philosophy:** Start simple. Add security features when users need them.
