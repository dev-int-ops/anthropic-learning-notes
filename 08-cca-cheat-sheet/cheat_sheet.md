# Claude Certified Architect — Exam Cheat Sheet

> **Exam Domain Weights**
> | Domain | Weight |
> |--------|--------|
> | Agent Architecture & Orchestration | 27% |
> | Claude Code Configuration | 20% |
> | Prompt Engineering & Structured Output | 20% |
> | Tool Design & MCP Integration | 18% |
> | Context Management & Reliability | 15% |

---

## 01 — AI Fluency: Framework & Foundations

### Core AI Concepts

| Concept | Key Detail |
|---------|-----------|
| **Tokens** | ~4 characters each; always account for token limits in prompts and context windows |
| **Determinism** | Outputs are probabilistic — same prompt can yield different results; use `temperature` + JSON schemas for consistency |
| **Context Window** | Max tokens the model processes at once — includes system prompt, history, tool definitions, and tool results |
| **LLMs** | Process tokens sequentially via pattern recognition; no persistent memory between requests (stateless) |

### Prompt Fundamentals

- **Few-shot Prompting** — 2–4 examples outperform text instructions; examples show exact decision logic and output format
- **System Prompt Priority** — Takes precedence over user messages; use for role, constraints, and behavior (not per-request task instructions)
- **Lost-in-the-Middle Effect** — Models reliably read start and end of input; **place critical info first or last**
- **Explicit Criteria** — Instead of "check comments for accuracy," write: `"Comment contradicts actual code behavior"` or `"References non-existent functions"`

### Key Limitations to Design Around

| Limitation | Mitigation |
|-----------|------------|
| **Hallucination** | Structured outputs, validation, RAG |
| **Context Accumulation** | Trim tool results aggressively (40 fields → 5 needed) |
| **Knowledge Cutoff** | Feb 2025 for current models; use tools for current data |
| **No Persistent State** | Send full conversation history on every request |

---

## 02 — Claude-101: API Fundamentals

### Required Request Fields

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "system": "You are a helpful assistant.",
  "messages": [...],
  "tools": [...],
  "tool_choice": {"type": "auto"}
}
```

**Available models:**
| Model | Best For | Speed | Cost |
|-------|----------|-------|------|
| `claude-opus-4-7` | Complex reasoning, multi-step tasks | Slower | Higher |
| `claude-sonnet-4-6` | Balanced performance — **most tasks** | Medium | Medium |
| `claude-haiku-4-5` | Speed-critical, simple tasks | Fast | Lower |

> 💡 **Default:** Use Sonnet. Upgrade to Opus for complex reasoning; downgrade to Haiku for latency-sensitive tasks.

### Message Roles

| Role | Purpose |
|------|---------|
| `user` | User messages |
| `assistant` | Model responses (must be in history) |
| `tool` | Tool call results (`tool_result` content blocks) |

### `stop_reason` Handling — Critical for Agents

| Value | Meaning | Action |
|-------|---------|--------|
| `end_turn` | Model finished | ✅ Show result to user |
| `tool_use` | Model called a tool | Execute and return result |
| `max_tokens` | Token limit hit | Response truncated — increase limit |
| `stop_sequence` | Stop sequence triggered | Handle per app logic |

> ⚠️ **`end_turn` is the ONLY reliable completion signal.** Never parse text (`"Task completed"`) or use iteration limits as primary stop condition.

### `tool_choice` Options

```json
{"type": "auto"}                          // Model decides (default)
{"type": "any"}                           // Model must call some tool
{"type": "tool", "name": "tool_name"}     // Force specific tool
```

---

## 03 — Claude Code in Action

### CLAUDE.md Hierarchy (highest → lowest specificity)

```
~/.claude/CLAUDE.md               ← User-level (personal only, NOT in VCS)
.claude/CLAUDE.md or CLAUDE.md    ← Project-level (team-shared, managed in VCS) ✅
subdirectory/CLAUDE.md            ← Directory-level (applies to files in that dir)
```

> ❌ **Common Mistake:** Placing project instructions at user-level → new team members don't receive them.

### Modularizing with `@path`

```markdown
# .claude/CLAUDE.md
Coding standards: @./standards/coding-style.md
Testing requirements: @./standards/testing-requirements.md
Project overview: @README.md
```
Maximum nesting depth: **5 levels**

### Conditional Rules with `.claude/rules/`

```yaml
# .claude/rules/api-conventions.md
---
paths: ["src/api/**/*"]
---
For API files: use async/await with explicit error handling.
Each endpoint must return a standard response wrapper.
```

### Custom Skills / Slash Commands

```yaml
# .claude/skills/analyze/SKILL.md
---
context: fork          # Isolate verbose output in subagent
allowed-tools: ["Read", "Grep", "Glob"]
argument-hint: "Path to the directory to analyze"
---
Analyze the code structure in the specified directory.
```

### Planning Mode vs Direct Execution

| Use Planning Mode When | Use Direct Execution When |
|------------------------|--------------------------|
| Changes affect 20+ files | Single-file fix with clear stack trace |
| Multiple plausible approaches | Obvious, unambiguous change |
| Architectural decisions needed | Adding one validation check |
| Unfamiliar codebase / library migrations | |

**Planning Mode Workflow:** Explore (Read/Grep/Glob, no changes) → Produce plan → User approves → Execute

### Claude Code CLI for CI/CD

```bash
# Non-interactive mode with structured output
claude -p "Analyze PR for security issues" \
  --output-format json \
  --json-schema '{"type":"object",...}'

# Resume prior session context
claude --resume investigation-auth-bug
```

> Use `/compact` when context fills up — ⚠️ risk: numeric values and dates may lose precision.

---

## 04 — Introduction to Agent Skills

### The Agentic Loop

```
Send request → Receive response → Check stop_reason
  ├── tool_use  → Execute tool → Append result → Repeat
  └── end_turn  → ✅ Task complete → Show result to user
```

**Anti-patterns:**
- ❌ Parsing text like `"Task completed"` as completion signal
- ❌ Using `max_iterations=5` as stop condition
- ✅ Only `stop_reason == "end_turn"` signals completion

### Hub-and-Spoke Architecture

```
        Coordinator
       /     |     \
      /      |      \
 Subagent1  Sub2   Sub3
 (search) (analysis)(synthesis)
```

**Coordinator responsibilities:** Decompose tasks → Select subagents → Delegate → Aggregate → Validate → Error handling → User communication

> ⚠️ **Critical:** Subagents have **isolated context**. The coordinator does NOT automatically pass history. All required context must be **explicitly provided** in the subagent prompt.

### The Task Tool

```python
# ❌ Bad — no context
Task: "Analyze the document"

# ✅ Good — full context provided
Task: "Analyze the following document.
Document: [full text]
Prior results: [search findings]
Output format: [schema]"
```

**Parallel execution:** Multiple `Task` calls in one response run concurrently.

### AgentDefinition + Least Privilege

```python
agent = AgentDefinition(
    name="customer_support",
    system_prompt="You are a customer support agent...",
    allowed_tools=["get_customer", "lookup_order", "process_refund"]
    # Only include tools the agent actually needs
)
```

### Hooks for Deterministic Behavior

| Attribute | Hooks | Prompt Instructions |
|-----------|-------|-------------------|
| **Guarantee** | 100% deterministic | ~90% (not guaranteed) |
| **Use for** | Financial ops, compliance, critical rules | General preferences, formatting |

```python
@hook("PreToolUse")
def enforce_refund_limit(tool_call):
    if tool_call.name == "process_refund" and tool_call.args.amount > 500:
        return redirect_to_escalation(tool_call)

@hook("PostToolUse")
def normalize_dates(tool_result):
    return normalized_result  # Convert Unix → ISO 8601
```

### Error Handling Categories

| Category | Example | Retryable | Action |
|----------|---------|-----------|--------|
| **Transient** | Timeout, 503, network failure | ✅ Yes | Retry with backoff |
| **Validation** | Invalid format, missing field | ❌ No | Modify and retry |
| **Business** | Policy violation, threshold exceeded | ❌ No | Explain; propose alternative |
| **Permission** | Access denied | ❌ No | Escalate |

**Structured Error Response (required pattern):**

```json
{
  "isError": true,
  "errorCategory": "transient",
  "isRetryable": true,
  "message": "Temporary timeout while calling the orders API",
  "attempted_query": "order_id=12345",
  "partial_results": null,
  "alternative_approaches": ["Try narrower query", "Use alternative source"]
}
```

### Escalation Patterns

```
✅ Explicit escalation triggers:
  - Customer explicitly requests manager
  - Policy gap: agent cannot proceed
  - Agent cannot make progress

❌ Unreliable escalation triggers:
  - Sentiment analysis ("customer is angry")
  - Model self-confidence ("I'm 60% sure")
  - Automatic classifiers without context
```

**Structured Handoff (provide human with full context):**

```json
{
  "customer_id": "CUST-12345",
  "issue_summary": "Damaged item, requests refund",
  "order_id": "ORD-67890",
  "actions_taken": ["Verified customer", "Offered replacement"],
  "customer_request": "Full refund",
  "recommended_action": "Approve refund",
  "escalation_reason": "Customer insisted on manager"
}
```

---

## 05 — Model Context Protocol (MCP)

### MCP Resource Types

| Type | Purpose |
|------|---------|
| **Tools** | Functions the agent can call (CRUD, API calls, commands) |
| **Resources** | Data the agent can read for context (docs, schemas, catalogs) |
| **Prompts** | Predefined templates for common tasks |

### MCP Server Configuration

```json
// .mcp.json — project-level (VCS-managed, team-shared) ✅
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_TOKEN": "${GITHUB_TOKEN}"}
    }
  }
}
```

> 🔑 **Secrets in environment variables** — never hardcode tokens. User-level config (`~/.claude.json`) for personal/experimental servers — NOT in VCS.

### MCP Error Handling

```json
// ✅ Good — structured with recovery info
{
  "isError": true,
  "content": {
    "errorCategory": "transient",
    "isRetryable": true,
    "message": "Temporary timeout while calling orders API",
    "retry_after_seconds": 5
  }
}

// ❌ Bad — no recovery information
{"isError": true, "content": "Operation failed"}
```

### Strong Tool Description Template

```
Tool: search_knowledge_base
Description:
  Search the team's internal knowledge base for documentation, runbooks, and FAQs.
  Returns ranked results with relevance scores.

  Input: query string (e.g., "database migration", "deploy process")
  Output: array of {"title", "url", "relevance_score", "snippet"}

  Edge cases:
  - Empty results: no matching documents (vs search failure)
  - Partial results: timeout after 3 seconds, returns partial

  Use instead of:
  - search_web: for internal-only information
  - search_github: for runbooks and docs not in repos
```

---

## 06 — Building with the Claude API

### JSON Schema Best Practices

| Rule | Pattern |
|------|---------|
| **Nullable fields** | `"type": ["string", "null"]` for optional data |
| **Enum with fallback** | Add `"other"` + `category_detail` field |
| **Uncertain classification** | Add `"unclear"` to enum values |
| **Required fields** | Only mark `required` if always available — overuse causes hallucination |

```json
{
  "category": {"type": "string", "enum": ["bug", "feature", "docs", "other"]},
  "category_detail": {"type": ["string", "null"], "description": "Details if category = other"},
  "severity": {"type": "string", "enum": ["critical", "high", "medium", "low", "unclear"]}
}
```

> ✅ JSON schemas fix **syntax** | ❌ JSON schemas do NOT fix **semantics** — validate business logic separately.

### Validation & Retry-with-Feedback Pattern

```
Step 1: Extract data from document
Step 2: Validate (check constraints)
Step 3: If error → Retry with context:
  {
    "original_document": "...",
    "previous_extraction": {"total": 150, "items": [75, 70]},
    "error": "total=150 but sum(items)=145. Re-check."
  }
```

**When retry works:** Format errors, arithmetic inconsistencies, structural errors  
**When retry won't help:** Information absent from source, required context in another document

### Self-Correction Pattern

```json
{
  "stated_total": "$150.00",
  "calculated_total": "$145.00",
  "conflict_detected": true,
  "line_items": [
    {"name": "Widget A", "price": 75.00},
    {"name": "Widget B", "price": 70.00}
  ]
}
```

### Multi-Pass Code Review (10+ files)

```
Pass 1 (Per-File):    Analyze each file for local issues → list per file
Pass 2 (Integration): Analyze cross-file dependencies → issues at module boundaries
```
> Single pass over many files causes inconsistent quality and missed bugs.

### Prompt Chaining for Complex Tasks

```
Step 1: Extract metadata from document A
Step 2: Extract data from document A
Step 3: Validate extracted data
Step 4: Enrich with external data
Step 5: Final output
```

---

## 07 — MCP: Advanced Topics

### Building Custom MCP Servers

```python
from mcp.server import Server

server = Server("my-server")

@server.tool()
def query_internal_database(query: str) -> str:
    """Query the team's internal project database."""
    return results

@server.resource()
def get_project_schema() -> str:
    """Returns the team's data schema."""
    return schema
```

> ✅ Build custom servers for unique, team-specific workflows | ❌ Use community servers for standard integrations (GitHub, Jira, Slack, Gmail)

### Prompt Caching with MCP Resources

```python
@server.resource(cache_control="ephemeral")     # Cache on first read
def get_large_schema() -> str: ...

@server.resource(cache_ttl_seconds=300)          # Cache 5 minutes
def get_api_documentation() -> str: ...
```

> Only available for resources **> 1024 tokens** | Saves up to **90%** on repeated request tokens

### Monitoring & Observability

```python
@server.hook("PostToolUse")
def log_tool_execution(tool_name: str, result: dict, duration_ms: int):
    metrics.record(
        tool_name=tool_name,
        success=(not result.get("isError")),
        duration_ms=duration_ms,
        error_category=result.get("errorCategory")
    )
```

Identify: tools never used (remove), high error rates (refine), performance bottlenecks.

---

## ✅ Quick Reference Checklists

### Before Every API Request
- Full conversation history included
- System prompt clearly defined
- Tool descriptions specific and non-overlapping
- `max_tokens` appropriate for response size
- `tool_choice` aligns with task requirements

### When Building Agents
- `stop_reason` checked — not text parsing
- Subagent prompts include full required context
- Hooks used for deterministic rules (not prompts)
- Error responses structured with recovery info
- Escalation rules explicit (not confidence-based)

### When Using Claude Code
- `CLAUDE.md` at **project-level** (not user-level) for team conventions
- Large changes use planning mode for approval
- CI/CD uses `-p` flag for non-interactive mode
- Skills include `context` and `allowed-tools` frontmatter
- `@path` imports modularize configuration

### When Integrating MCP
- `.mcp.json` at project root (version-controlled)
- Secrets in environment variables (`${TOKEN}`)
- Tool descriptions include edge cases and alternatives
- Error responses include `errorCategory` and `isRetryable`
- Resources preferred over repeated tool calls for static data

---

## ⚠️ Top 10 Rules for the Exam

1. **`stop_reason == "end_turn"`** is the ONLY reliable agent completion signal — never parse text or use iteration limits
2. **Send full history on every request** — the model has no persistent state between API calls
3. **Tool descriptions are the selection mechanism** — vague/overlapping descriptions cause misrouting
4. **Subagents have isolated context** — coordinator must explicitly provide all required context
5. **Hooks for 100% guarantees** (financial ops, compliance); prompts for ~90% probabilistic behavior
6. **Structured errors include recovery info** — generic `"failed"` prevents coordinator recovery
7. **JSON schemas fix syntax, not semantics** — validate business logic separately with retry-with-feedback
8. **Few-shot > text instructions** — examples demonstrate decision logic better than prose
9. **Critical info at start or end** — lost-in-the-middle effect makes middle context unreliable
10. **MCP project config in VCS** — prevents team members from missing shared integrations

---

## ❌ Common Mistakes to Avoid

| ❌ Mistake | ✅ Correct Approach |
|-----------|-------------------|
| Parsing model text as completion signal | Check `stop_reason` |
| Assuming model persistence between requests | Send full history every time |
| Vague, overlapping tool descriptions | Be specific with edge cases and alternatives |
| Passing model confidence as validation | Validate separately |
| Escalating based on sentiment | Use explicit, rule-based triggers |
| Generic error responses | Structure with `errorCategory` + recovery info |
| Placing project config at user-level | Use project-level `.claude/` or `.mcp.json` |
| Single pass over large PRs | Use multi-pass (per-file + integration) |
| `required` on all schema fields | Only mark `required` if always available |
| Subagent without full context | Explicitly provide all context in Task prompt |
