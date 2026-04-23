# Claude Code in Action

*This guide was created after completing the Claude Code in Action course on Anthropic's learning platform. It summarizes and organizes the key concepts in one place, with AI assistance used to structure and format the content, to help learners grasp key concepts quickly.*

---

## Table of Contents

1. [What Is a Coding Assistant?](#1-what-is-a-coding-assistant)
2. [Claude Code in Action — Key Demos](#2-claude-code-in-action--key-demos)
3. [Setup & Installation](#3-setup--installation)
4. [Adding Context](#4-adding-context)
5. [Making Changes](#5-making-changes)
6. [Controlling Context](#6-controlling-context)
7. [Custom Commands](#7-custom-commands)
8. [Extending Claude Code with MCP Servers](#8-extending-claude-code-with-mcp-servers)
9. [GitHub Integration](#9-github-integration)
10. [Hooks — Introduction & Architecture](#10-hooks--introduction--architecture)
11. [Hooks — Defining & Implementing](#11-hooks--defining--implementing)
12. [Hooks — Advanced Types](#12-hooks--advanced-types)
13. [Useful Hooks for Real Projects](#13-useful-hooks-for-real-projects)
14. [The Claude Code SDK](#14-the-claude-code-sdk)
15. [Quick-Reference Cheat Sheet](#15-quick-reference-cheat-sheet)

---

## 1. What Is a Coding Assistant?

A **coding assistant** is a tool that uses a language model to write code and complete development tasks.

### Core Process
1. Receives a task (e.g., fix a bug from an error message)
2. The language model gathers context (reads files, understands the codebase)
3. Formulates a plan to solve the issue
4. Takes action (updates files, runs tests)

### The Fundamental Limitation
Language models can **only process text input/output**. They cannot:
- Directly read files
- Run terminal commands
- Interact with external systems

This is why a **Tool Use System** is required.

### How the Tool Use System Works
| Step | Who Acts | What Happens |
|---|---|---|
| 1 | User | Sends query to the assistant |
| 2 | Assistant | Appends tool definitions to the query |
| 3 | Language Model | Returns a formatted tool-call response |
| 4 | Assistant | Executes the actual action (reads file, runs command) |
| 5 | Assistant | Sends result back to the language model |
| 6 | Language Model | Produces the final response |

### Why Claude Excels at Tool Use
- Superior at understanding tool functions and combining them for complex tasks
- Claude Code is **extensible** — easy to add new tools
- Better security: uses direct code search instead of indexing (avoids sending codebase to external servers)
- Tool use quality directly determines coding assistant effectiveness

---

## 2. Claude Code in Action — Key Demos

### Performance Optimization
- **Project:** Chalk JavaScript library (5th most downloaded JS package, 429M weekly downloads)
- **Process:** Used benchmarks, profiling tools, created todo lists, identified bottlenecks, implemented fixes
- **Result:** **3.9× throughput improvement**

### Data Analysis
- **Project:** Churn analysis on a video streaming platform CSV dataset
- **Process:** Used Jupyter notebooks, executed code cells iteratively, viewed results, customized successive analyses based on findings

### UI Automation (with Playwright MCP)
- Opened a browser, took screenshots, updated UI styling, iterated on design improvements

### Infrastructure Security Review
- Terraform-defined AWS infrastructure (DynamoDB + S3) shared with an external partner
- Developer added user email to Lambda function output
- Claude Code automatically detected **PII exposure risk** by analyzing the full infrastructure flow

### Key Principle
> Claude Code = a **flexible assistant** that grows with team needs through tool expansion, not fixed functionality.

---

## 3. Setup & Installation

### Installation Commands

| Platform | Command |
|---|---|
| macOS (Homebrew) | `brew install --cask claude-code` |
| macOS / Linux / WSL | `curl -fsSL https://claude.ai/install.sh \| bash` |
| Windows CMD | `curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd` |

After installation, run `claude` in your terminal. You will be prompted to authenticate on first run.

### Cloud Provider Setup
- **AWS Bedrock:** https://code.claude.com/docs/en/amazon-bedrock
- **Google Cloud Vertex:** https://code.claude.com/docs/en/google-vertex-ai

### Project Setup (Course Sample Project)
```bash
# Install Node.js first, then:
npm run setup      # Installs dependencies, sets up local SQLite database
npm run dev        # Start the project
```

Place your Anthropic API key in the `.env` file to enable full API-powered features.

---

## 4. Adding Context

Context management is **critical** — too much irrelevant information decreases Claude's performance.

### `/init` Command
- Analyzes your entire codebase on first run
- Creates a `Claude.md` file with: project summary, architecture overview, and key file references
- File contents are included in **every** request automatically

### Three Types of Claude.md Files

| Type | Scope | Committed to Source Control? |
|---|---|---|
| **Project level** | Shared with the whole team | ✅ Yes |
| **Local level** | Personal instructions only | ❌ No |
| **Machine level** | Global, applies to all projects | ❌ No |

### Memory Mode (`#` symbol)
- Use the `#` shortcut to **intelligently edit** Claude.md files using natural language
- Example: `# Always use async/await instead of promise chains`

### `@` Symbol — Targeted Context
- Mention specific files directly: `@src/db/schema.ts`
- Provides targeted context instead of letting Claude search broadly
- Best practice: reference critical files (like database schemas) in `Claude.md` so they're always available

### Goal
Provide **just enough** relevant information for Claude to complete tasks effectively — not more, not less.

---

## 5. Making Changes

### Screenshots for Precise Communication
- Take a screenshot of the UI area you want to change
- Paste with **Ctrl+V** (not Cmd+V on macOS — this is intentional)
- Describe the change you want; Claude will understand the visual context

### Plan Mode
**Trigger:** Press `Shift + Tab` twice (or once if already auto-accepting edits)

What it does:
- Reads more files across your project
- Creates a detailed, step-by-step implementation plan
- Shows exactly what it intends to do
- **Waits for your approval** before executing

**Best for:** Multi-step tasks, changes affecting multiple files, broad codebase understanding.

### Thinking Modes
Triggered by specific phrases in your message:

| Phrase | Reasoning Budget |
|---|---|
| `Think` | Basic reasoning |
| `Think more` | Extended reasoning |
| `Think a lot` | Comprehensive reasoning |
| `Think longer` | Extended time reasoning |
| `Ultrathink` | Maximum reasoning capability |

**Best for:** Complex logic, debugging difficult issues, algorithmic challenges.

### Plan Mode vs. Thinking Mode

| Mode | Handles | Use When |
|---|---|---|
| **Plan Mode** | Breadth | Multi-step, many files, wide codebase |
| **Thinking Mode** | Depth | Tricky logic, specific bugs, algorithms |

> ⚠️ **Cost note:** Both modes consume additional tokens. You can combine them for tasks requiring both breadth and depth.

### Git Integration
Claude Code can stage and commit changes, and write descriptive commit messages automatically.

---

## 6. Controlling Context

### Keyboard Shortcuts

| Action | Key | What It Does |
|---|---|---|
| **Stop/Interrupt** | `Escape` | Stops Claude mid-response so you can redirect |
| **Rewind** | `Escape` × 2 | Shows all previous messages; jump back to any earlier point |

### Escape + Memory (Powerful Combo)
When Claude repeatedly makes the same mistake:
1. Press `Escape` to stop the current response
2. Use `#` to add a memory: e.g., `# Never use var, always use const`
3. Continue — Claude won't repeat that mistake in this project

### Context Management Commands

| Command | What It Does | When to Use |
|---|---|---|
| `/compact` | Summarizes conversation history while preserving learned knowledge | Claude has learned a lot; conversation is long but knowledge is valuable |
| `/clear` | Completely wipes the conversation history | Switching to an entirely unrelated task |

### When to Use These Techniques
- Long-running conversations where context gets cluttered
- Task transitions where previous context may confuse Claude
- Claude repeatedly makes the same mistakes
- Complex projects requiring focus on specific components

---

## 7. Custom Commands

Custom commands let you automate repetitive workflows into a single slash command.

### Setup
```
.claude/
  commands/
    audit.md          → becomes /audit
    write_tests.md    → becomes /write_tests
```

> ⚠️ **Restart Claude Code** after creating new command files.

### Basic Command Example (`audit.md`)
```markdown
Run npm audit to find vulnerable installed packages.
Run npm audit fix to apply available updates.
Run the test suite to verify nothing broke.
```
Activated with: `/audit`

### Commands with Arguments
Use the `$ARGUMENTS` placeholder to accept dynamic input:

```markdown
Write comprehensive tests for: $ARGUMENTS

Testing conventions:
* Use Vitest with React Testing Library
* Place test files in a __tests__ directory alongside the source file
* Name test files as [filename].test.ts(x)
* Use @/ prefix for imports

Coverage:
* Test happy paths
* Test edge cases
* Test error states
```

Activated with: `/write_tests src/hooks/use-auth.ts`

Arguments can be any string: file paths, descriptions, instructions — anything that gives Claude context.

### Key Benefits
- **Automation** — Turn repetitive workflows into single commands
- **Consistency** — Same steps followed every time
- **Context** — Encode project conventions directly into commands
- **Flexibility** — Arguments make commands reusable across inputs

---

## 8. Extending Claude Code with MCP Servers

**MCP (Model Context Protocol) servers** extend Claude Code with new tools. They can run locally or remotely.

### Installing an MCP Server
Run this in your **terminal** (not inside Claude Code):
```bash
claude mcp add playwright npx @playwright/mcp@latest
#              ^name        ^start command
```

### Managing Permissions
By default, Claude asks for permission each time it uses an MCP tool. To auto-approve:

Edit `.claude/settings.local.json`:
```json
{
  "permissions": {
    "allow": ["mcp__playwright"],
    "deny": []
  }
}
```
Note the **double underscores** in `mcp__playwright`.

### Practical Example: UI Prompt Improvement with Playwright
1. Claude opens browser → navigates to `localhost:3000`
2. Generates a test UI component
3. Analyzes visual styling and code quality
4. Updates the generation prompt file with improvements
5. Tests the improved prompt with a new component

**Result:** Instead of generic purple-to-blue gradients, Claude produces:
- Warm sunset gradients (orange → pink → purple)
- Ocean depth themes (teal → emerald → cyan)
- Asymmetric designs and overlapping elements

### Other MCP Server Categories
- Database interactions
- API testing and monitoring
- File system operations
- Cloud service integrations
- Development tool automation

---

## 9. GitHub Integration

Claude Code can run inside **GitHub Actions**, acting as an automated team member.

### Setup
Run inside Claude Code:
```
/install-github-app
```
This walks you through:
1. Installing the Claude Code GitHub app
2. Adding your API key
3. Auto-generating a PR with two workflow files

### Default GitHub Actions

#### 1. Mention Action
Mention Claude in any issue or PR with `@claude`:
- Claude analyzes the request and creates a task plan
- Executes the task with full codebase access
- Posts results directly in the issue or PR

#### 2. Pull Request Review Action
On every new PR, Claude automatically:
- Reviews proposed changes
- Analyzes the impact of modifications
- Posts a detailed report on the PR

### Workflow Customization (`.github/workflows/`)

#### Adding Project Setup Steps
```yaml
- name: Project Setup
  run: |
    npm run setup
    npm run dev:daemon
```

#### Custom Instructions
```yaml
custom_instructions: |
  The project is already set up with all dependencies installed.
  The server is running at localhost:3000. Logs are in logs.txt.
  Use sqlite3 CLI to query the DB if needed.
  Use mcp__playwright tools to launch a browser and interact with the app.
```

#### MCP Server Configuration
```yaml
mcp_config: |
  {
    "mcpServers": {
      "playwright": {
        "command": "npx",
        "args": ["@playwright/mcp@latest", "--allowed-origins", "localhost:3000;cdn.tailwindcss.com;esm.sh"]
      }
    }
  }
```

#### Tool Permissions ⚠️
In GitHub Actions, **every tool must be explicitly listed** — no shortcuts:
```yaml
allowed_tools: "Bash(npm:*),Bash(sqlite3:*),mcp__playwright__browser_snapshot,mcp__playwright__browser_click,..."
```

### Best Practices
- Start with default workflows, then customize gradually
- Use `custom_instructions` to provide project-specific context
- Be explicit about tool permissions when using MCP servers
- Test with simple tasks before complex ones

---

## 10. Hooks — Introduction & Architecture

Hooks let you run **custom commands before or after** Claude executes a tool.

### Normal Flow (No Hooks)
```
User Query → Claude Model → Tool Call Decision → Tool Executes → Result → Response
```

### Flow with Hooks
```
User Query → Claude Model → Tool Call Decision
                                    ↓
                          [PreToolUse Hook runs]
                                    ↓
                            Tool Executes
                                    ↓
                          [PostToolUse Hook runs]
                                    ↓
                              Result → Response
```

### Hook Types

| Type | When It Runs | Can Block? |
|---|---|---|
| **PreToolUse** | Before tool executes | ✅ Yes (exit code 2) |
| **PostToolUse** | After tool executes | ❌ No |

### Additional Hook Types
| Type | Trigger |
|---|---|
| **Notification** | Claude needs permission OR has been idle 60+ seconds |
| **Stop** | Claude has finished responding |
| **SubagentStop** | A subagent (Task in UI) has finished |
| **PreCompact** | Before a compact operation (manual or automatic) |
| **UserPromptSubmit** | When user submits a prompt, before Claude processes it |
| **SessionStart** | When starting or resuming a session |
| **SessionEnd** | When a session ends |

### Hook Configuration Locations

| Scope | File | Description |
|---|---|---|
| Global | `~/.claude/settings.json` | Affects all projects |
| Project (shared) | `.claude/settings.json` | Committed to source control |
| Project (personal) | `.claude/settings.local.json` | Not committed |

You can also use the `/hooks` command inside Claude Code to configure hooks interactively.

---

## 11. Hooks — Defining & Implementing

### Hook Configuration Structure

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Grep",
        "hooks": [
          {
            "type": "command",
            "command": "node /absolute/path/to/hooks/read_hook.js"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "node /absolute/path/to/hooks/edit_hook.js"
          }
        ]
      }
    ]
  }
}
```

> ⚠️ Use **absolute paths** for scripts (security best practice to prevent path interception / binary planting attacks).

### Tool Call Data Structure (stdin JSON)
Your hook script receives this via `stdin`:
```json
{
  "session_id": "2d6a1e4d-6...",
  "transcript_path": "/Users/...",
  "hook_event_name": "PreToolUse",
  "tool_name": "Read",
  "tool_input": {
    "file_path": "/code/queries/.env"
  }
}
```

### Exit Codes

| Exit Code | Meaning |
|---|---|
| `0` | Allow the tool call to proceed |
| `2` | Block the tool call (**PreToolUse only**) |

When exiting with code `2`, anything written to **stderr** is sent to Claude as feedback explaining why it was blocked.

### Example: Blocking `.env` File Access

**`hooks/read_hook.js`**
```javascript
async function main() {
  const chunks = [];
  for await (const chunk of process.stdin) {
    chunks.push(chunk);
  }

  const toolArgs = JSON.parse(Buffer.concat(chunks).toString());

  // Support both "file_path" and "path" field names
  const readPath = toolArgs.tool_input?.file_path || toolArgs.tool_input?.path || "";

  if (readPath.includes('.env')) {
    console.error("You cannot read the .env file");
    process.exit(2);
  }
}

main();
```

**`.claude/settings.local.json`**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Grep",
        "hooks": [
          {
            "type": "command",
            "command": "node /absolute/path/to/hooks/read_hook.js"
          }
        ]
      }
    ]
  }
}
```

> ⚠️ **Restart Claude Code** after any hook changes.

### Debugging Hook Input (Log Helper)
Not sure what data your hook will receive? Add this temporary hook to log it:
```json
{
  "PostToolUse": [
    {
      "matcher": "*",
      "hooks": [
        {
          "type": "command",
          "command": "jq . > post-log.json"
        }
      ]
    }
  ]
}
```
This writes the raw stdin input to `post-log.json` so you can inspect the exact structure.

### `$PWD` Placeholder for Portability
Since absolute paths differ between machines, the course project uses a `settings.example.json` with `$PWD` placeholders. Running `npm run setup` executes `scripts/init-claude.js` which:
1. Replaces all `$PWD` placeholders with the actual machine path
2. Copies the file as `settings.local.json`

This lets teams share hook configurations while still using absolute paths.

---

## 12. Hooks — Advanced Types

The `stdin` input structure **varies by hook type and matched tool**. This can make writing hooks tricky.

### PostToolUse — `TodoWrite` Tool Example
```json
{
  "session_id": "9ecf22fa-...",
  "transcript_path": "<path>",
  "hook_event_name": "PostToolUse",
  "tool_name": "TodoWrite",
  "tool_input": {
    "todos": [
      { "content": "write a readme", "status": "pending", "priority": "medium", "id": "1" }
    ]
  },
  "tool_response": {
    "oldTodos": [],
    "newTodos": [
      { "content": "write a readme", "status": "pending", "priority": "medium", "id": "1" }
    ]
  }
}
```

### Stop Hook Example
```json
{
  "session_id": "af9f50b6-...",
  "transcript_path": "<path>",
  "hook_event_name": "Stop",
  "stop_hook_active": false
}
```

### Strategy for Unknown Input Structures
1. Add a wildcard `matcher: "*"` logging hook first
2. Inspect the `post-log.json` output
3. Build your real hook based on the confirmed structure

---

## 13. Useful Hooks for Real Projects

### Hook 1: TypeScript Type Checker

**Problem:** When Claude modifies a function signature, it often doesn't update all call sites throughout the project, creating type errors.

**Solution:** A PostToolUse hook that runs `tsc --noEmit` after every TypeScript file edit.

**Flow:**
```
Claude edits file → PostToolUse hook triggers → tsc --noEmit runs →
type errors found → errors fed back to Claude → Claude fixes call sites
```

**Configuration:**
```json
{
  "PostToolUse": [
    {
      "matcher": "Write|Edit|MultiEdit",
      "hooks": [
        {
          "type": "command",
          "command": "npx tsc --noEmit 2>&1 | head -50"
        }
      ]
    }
  ]
}
```

> Works for any typed language with a type checker. For untyped languages, substitute your test runner.

---

### Hook 2: Query Duplication Prevention

**Problem:** On complex multi-step tasks, Claude creates new SQL queries/functions instead of reusing existing ones, leading to a bloated, duplicated codebase.

**Solution:** A PostToolUse hook that launches a **second Claude instance** to review new code in the queries directory for duplicates.

**Flow:**
```
Claude edits ./queries/ → hook detects change →
secondary Claude instance launched via SDK →
reviews existing queries for duplicates →
if duplicate found: exit code 2 + feedback sent →
primary Claude removes duplicate and reuses existing function
```

**Implementation (TypeScript using Claude Code SDK):**
```typescript
import { query } from "@anthropic-ai/claude-code";

const reviewPrompt = `
  Review the changes just made to the queries directory.
  Compare them against all existing query files.
  If you find any duplicate functionality, report it clearly.
  Format: DUPLICATE FOUND: <new function> duplicates <existing function>
`;

for await (const message of query({ prompt: reviewPrompt })) {
  if (/* duplicate detected in message */) {
    console.error("Duplicate query found. Use the existing function instead.");
    process.exit(2);
  }
}
```

**Trade-offs:**

| Pro | Con |
|---|---|
| Cleaner codebase, less duplication | Extra time per edit |
| Consistent reuse of existing functions | Additional API cost |

> **Recommendation:** Only watch critical directories (e.g., `./queries`) to minimize overhead.

---

## 14. The Claude Code SDK

The Claude Code SDK provides a **programmatic interface** to Claude Code via TypeScript, Python, or CLI — with the same tools available in the terminal.

### Key Characteristics
- Same Claude Code functionality as the terminal version
- Inherits all settings from `.claude` directory in the same project
- **Read-only by default** (files, directories, grep)
- Write permissions must be explicitly enabled
- Shows raw conversation between local Claude Code and the language model

### Primary Use Case
Integration into **larger pipelines and workflows** — not standalone usage. Best for: helper commands, scripts, git hooks, CI/CD automation.

### Basic TypeScript Example
```typescript
import { query } from "@anthropic-ai/claude-code";

const prompt = "Look for duplicate queries in the ./src/queries dir";

for await (const message of query({ prompt })) {
  console.log(JSON.stringify(message, null, 2));
}
// The last message contains Claude's complete response
```

### Enabling Write Permissions
**Option A — Per-query:**
```typescript
for await (const message of query({
  prompt,
  options: {
    allowedTools: ["Edit", "Write"]
  }
})) {
  console.log(JSON.stringify(message, null, 2));
}
```

**Option B — Project-wide:**
Configure allowed tools in `.claude/settings.json`.

### Practical Applications
| Use Case | Description |
|---|---|
| Git hooks | Automatically review code changes before commit |
| Build scripts | Analyze and optimize code during builds |
| CI/CD pipelines | Run code quality checks automatically |
| Documentation generation | Auto-generate docs when files change |
| Duplicate detection | Review hooks (as shown in Section 13) |

---

## 15. Quick-Reference Cheat Sheet

### Keyboard Shortcuts
| Shortcut | Action |
|---|---|
| `Ctrl+V` | Paste screenshot into Claude |
| `Shift+Tab` × 2 | Enable Plan Mode |
| `Escape` | Stop/interrupt Claude mid-response |
| `Escape` × 2 | Rewind conversation |
| `#` | Add/edit memory (Claude.md) |
| `@filename` | Include specific file as context |

### Slash Commands
| Command | Purpose |
|---|---|
| `/init` | Analyze codebase, create Claude.md |
| `/compact` | Summarize conversation, keep knowledge |
| `/clear` | Wipe conversation history |
| `/hooks` | Interactively configure hooks |
| `/install-github-app` | Set up GitHub Actions integration |
| `/audit` | Example custom command |
| `/write_tests <path>` | Example custom command with args |

### Hook Exit Codes
| Code | Meaning |
|---|---|
| `0` | Allow operation |
| `2` | Block operation (PreToolUse only) |

### Common Tool Names for Hook Matchers
| Tool Name | What It Does |
|---|---|
| `Read` | Reads file contents |
| `Grep` | Searches within files |
| `Write` | Creates new files |
| `Edit` | Edits existing files |
| `MultiEdit` | Multiple edits in one call |
| `TodoWrite` | Writes to Claude's todo list |

### MCP Permission Format
```
mcp__<servername>__<toolname>
```
Example: `mcp__playwright__browser_snapshot`

---

*Notes compiled from the Anthropic SkillJar course: Claude Code in Action [https://anthropic.skilljar.com/claude-code-in-action](https://anthropic.skilljar.com/claude-code-in-action). All examples reflect course content and personal annotations.*
