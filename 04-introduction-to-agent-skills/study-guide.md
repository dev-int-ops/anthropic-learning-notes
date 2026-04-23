# Introduction to Agent Skills

*This guide was created after completing the Introduction to Agent Skills course on Anthropic's learning platform. It summarizes and organizes the key concepts in one place, with AI assistance used to structure and format the content, to help learners grasp key concepts quickly.*

---

## Table of Contents

1. [What Are Skills?](#1-what-are-skills)
2. [Where Skills Live](#2-where-skills-live)
3. [Skills vs. Other Claude Code Customization Options](#3-skills-vs-other-claude-code-customization-options)
4. [Creating a Skill from Scratch](#4-creating-a-skill-from-scratch)
5. [How Skill Matching Works](#5-how-skill-matching-works)
6. [Skill Priority Hierarchy](#6-skill-priority-hierarchy)
7. [Advanced Skill Configuration](#7-advanced-skill-configuration)
8. [Sharing & Distributing Skills](#8-sharing--distributing-skills)
9. [Skills & Subagents](#9-skills--subagents)
10. [Troubleshooting Skills](#10-troubleshooting-skills)
11. [Quick Reference Cheatsheet](#11-quick-reference-cheatsheet)

---

## 1. What Are Skills?

Skills are **reusable markdown files** that teach Claude Code how to handle specific tasks automatically — without you repeating instructions every single time.

### The Core Problem They Solve

Every time you:
- Explain your team's coding standards
- Re-describe how you want PR feedback structured
- Remind Claude of your preferred commit message format

…you are repeating yourself. **Skills fix this.**

### Key Characteristics

- A skill is a **folder of instructions** Claude Code can discover and apply when relevant
- Each skill lives in a file called `SKILL.md` with a **name** and **description** in its frontmatter
- Claude uses the **description** to decide when to activate a skill — this is the matching engine
- Skills load **on demand**, not on every conversation (unlike `CLAUDE.md`)
- When you find yourself explaining the same thing to Claude repeatedly → **that's a skill waiting to be written**

### Mental Model

Think of skills like an expert consultant on call. They don't sit in every meeting (they're not in context all the time), but when the topic matches their specialty, they're brought in automatically.

---

## 2. Where Skills Live

| Location | Path | Scope |
|---|---|---|
| **Personal skills** | `~/.claude/skills/` | Follow you across all projects |
| **Project skills** | `.claude/skills/` (repo root) | Shared with anyone who clones the repo |
| **Windows personal** | `C:/Users/<your-user>/.claude/skills/` | Same as personal, Windows path |

### Rules

- **Personal skills** = your individual preferences: commit style, documentation format, how you like code explained
- **Project skills** = team standards: brand guidelines, fonts, colors, review checklists
- Project skills are **committed to version control** — the whole team shares them automatically via Git

---

## 3. Skills vs. Other Claude Code Customization Options

This is one of the most important distinctions in the course. Using the wrong tool leads to unnecessary complexity.

### Side-by-Side Comparison

| Feature | `CLAUDE.md` | **Skills** | Subagents | Hooks | MCP Servers |
|---|---|---|---|---|---|
| **When it activates** | Every conversation, always | On demand, when matched | Explicitly delegated | On events (file save, tool call) | Externally, via tools |
| **Best for** | Always-on project standards | Task-specific expertise | Isolated delegated work | Automated side effects | External integrations |
| **Context window impact** | Always loaded | Only when activated | Separate context | N/A | N/A |
| **Trigger mechanism** | Automatic | Semantic match to request | Explicit delegation | Event-driven | Tool call |

### When to Use Each

**Use `CLAUDE.md` for:**
- Project-wide standards that always apply ("always use TypeScript strict mode")
- Hard constraints ("never modify the database schema")
- Framework preferences and coding style that apply everywhere

**Use Skills for:**
- Task-specific expertise (PR review, commit messages, debugging checklist)
- Knowledge that is only relevant sometimes
- Detailed procedures that would clutter every conversation

**Use Subagents when:**
- You want to delegate a task to a separate execution context
- You need different tool access than the main conversation
- You want isolation between delegated work and your main context

**Use Hooks for:**
- Operations that should run on every file save (e.g., auto-linting)
- Validation before specific tool calls
- Automated side effects of Claude's actions

**Use MCP Servers for:**
- Connecting to external tools and integrations entirely (different category)

### Practical Setup Example (Real Project)

A complete setup might look like:
```
CLAUDE.md          → Always-on: "Use TypeScript strict mode, never touch DB schema"
.claude/skills/    → PR review checklist, commit format, onboarding guide
.claude/agents/    → Isolated frontend-reviewer subagent
hooks              → Auto-lint on every file save
MCP                → Connects to Jira, GitHub, Slack
```

---

## 4. Creating a Skill from Scratch

### Step 1 — Create the directory

The **directory name must match the skill name**:

```bash
mkdir -p ~/.claude/skills/pr-description
```

For a project skill:
```bash
mkdir -p .claude/skills/pr-description
```

### Step 2 — Create the `SKILL.md` file

```markdown
---
name: pr-description
description: Writes pull request descriptions. Use when creating a PR, writing a PR, or when the user asks to summarize changes for a pull request.
---

When writing a PR description:

1. Run `git diff main...HEAD` to see all changes on this branch
2. Write a description following this format:

## What
One sentence explaining what this PR does.

## Why
Brief context on why this change is needed.

## Changes
- Bullet points of specific changes made
- Group related changes together
- Mention any files deleted or renamed
```

### Step 3 — Restart Claude Code

Skills load at startup. **Always restart your session** after creating or editing a skill.

### Step 4 — Verify

Ask Claude: *"What skills are available?"* — your skill should appear in the list.

### Step 5 — Test

Make some changes on a branch and say: *"Write a PR description for my changes."*

Claude will:
1. Indicate it's using the PR description skill
2. Check your `git diff`
3. Write the description following your template — **same format every time**

---

## 5. How Skill Matching Works

Understanding this makes you much better at writing effective skills.

### The Matching Flow

```
Claude Code starts
        ↓
Scans 4 locations for skills
        ↓
Loads ONLY name + description (not full content)
        ↓
You send a request
        ↓
Claude compares your message against all skill descriptions
(Semantic matching — intent-based, not keyword-only)
        ↓
Match found → Claude prompts you to confirm loading the skill
        ↓
You confirm → Full SKILL.md content loads into context
        ↓
Claude follows the skill's instructions
```

### Key Details

- **Only names and descriptions load at startup** — this is intentional. It keeps your context window clean. A PR review checklist shouldn't be in context when you're debugging.
- **Semantic matching** means the intent must overlap, not the exact words. "Explain what this function does" can match "explain code with visual diagrams."
- **You get a confirmation prompt** before the full skill loads — this keeps you aware of what context Claude is pulling in.

---

## 6. Skill Priority Hierarchy

When two skills share the same name, this order determines which wins:

```
1. Enterprise  (highest — managed settings, always wins)
2. Personal    (~/.claude/skills/)
3. Project     (.claude/skills/ in repo)
4. Plugins     (lowest priority)
```

### Practical Implications

- If your company has an enterprise `code-review` skill and you also have a personal `code-review`, **the enterprise one wins every time**
- This lets organizations enforce mandatory standards while individuals can still customize — as long as names don't clash
- **Best practice: use descriptive, specific names** to avoid conflicts. Instead of `review`, use `frontend-review` or `api-security-review`

---

## 7. Advanced Skill Configuration

### All Frontmatter Fields

| Field | Required | Notes |
|---|---|---|
| `name` | ✅ Yes | Lowercase, numbers, hyphens only. Max 64 characters. Must match directory name. |
| `description` | ✅ Yes | Max 1,024 characters. **Most important field** — Claude's matching is based entirely on this. |
| `allowed-tools` | ❌ Optional | Restricts which tools Claude can use while the skill is active |
| `model` | ❌ Optional | Specifies which Claude model to use for this skill |

### Writing Effective Descriptions

A good description answers **two questions**:
1. What does the skill do?
2. When should Claude use it?

❌ **Weak:** `"Helps with docs"`  
✅ **Strong:** `"Writes and reviews technical documentation. Use when creating READMEs, API docs, onboarding guides, or when the user asks to document a module, function, or workflow."`

**Tips:**
- Add trigger phrases that match how you *actually* phrase requests
- Test with variations: "profile this," "why is this slow?," "make this faster" — if a variation fails to trigger, add those keywords
- Think about synonyms and related phrases users would naturally say

### Restricting Tools with `allowed-tools`

Use this for **read-only or security-sensitive workflows** where you don't want Claude editing files:

```markdown
---
name: codebase-onboarding
description: Helps new developers understand how the system works.
allowed-tools: Read, Grep, Glob, Bash
model: sonnet
---
```

With `allowed-tools` set, Claude can **only** use those listed tools without asking permission — no editing, no writing. If you omit `allowed-tools`, no restrictions apply.

### Progressive Disclosure (Keeping Skills Efficient)

The challenge: skills share Claude's context window. Stuffing everything into one 2,000-line `SKILL.md` wastes tokens and is hard to maintain.

**Solution — Progressive Disclosure:** Keep essential instructions in `SKILL.md` and put detailed reference material in separate files that Claude reads *only when needed*.

**Recommended directory structure:**
```
.claude/skills/
  my-skill/
    SKILL.md              ← Core instructions (keep under 500 lines)
    references/
      architecture-guide.md
      api-reference.md
    scripts/
      validate-env.sh
      transform-data.py
    assets/
      template.html
```

**In `SKILL.md`, link conditionally:**
```markdown
If the user asks about system architecture, read references/architecture-guide.md.
If they're asking where to add a component, skip that file entirely.
```

This is like having a **table of contents in context** rather than the entire book.

**Rule of thumb:** Keep `SKILL.md` under **500 lines**. If you exceed that, split content into reference files.

### Using Scripts Efficiently

Scripts in your skill directory can **run without loading their contents into context**. Only the *output* consumes tokens — not the script code itself.

The key instruction in `SKILL.md`: tell Claude to **run** the script, not **read** it.

Best for:
- Environment validation
- Data transformations that need to be consistent
- Operations that are more reliable as tested code than as generated code

---

## 8. Sharing & Distributing Skills

### Method 1 — Commit to Repository (Simplest)

Place skills in `.claude/skills/` and commit them. Anyone who clones the repo gets the skills automatically. Updates propagate through normal Git pull.

**Best for:**
- Team coding standards
- Project-specific workflows
- Skills that reference your codebase structure

### Method 2 — Plugins (Broader Distribution)

Plugins let you distribute skills across multiple repositories and to the wider community via marketplaces.

In your plugin project, create a `skills/` directory with the same structure as `.claude/skills/`. Publish to a marketplace; others install it into Claude Code.

**Best for:** Skills that aren't project-specific and could be useful to the broader community.

### Method 3 — Enterprise Managed Settings (Org-Wide)

Administrators deploy skills organization-wide via managed settings. These take the **highest priority** and override everything else with the same name.

```json
"strictKnownMarketplaces": [
  {
    "source": "github",
    "repo": "acme-corp/approved-plugins"
  },
  {
    "source": "npm",
    "package": "@acme-corp/compliance-plugins"
  }
]
```

**Best for:** Mandatory standards, security requirements, compliance workflows — anything that *must* be consistent across the organization.

---

## 9. Skills & Subagents

### The Gotcha Everyone Hits

> **Subagents do NOT automatically see your skills.**

When you delegate a task to a subagent, it starts with a **fresh, clean context**. Skills from the main conversation are not inherited.

### Two Important Distinctions

| Agent Type | Can Access Skills? |
|---|---|
| Built-in agents (Explorer, Plan, Verify) | ❌ Never |
| Custom subagents (defined in `.claude/agents/`) | ✅ Yes, but only if explicitly listed |

### How to Give a Custom Subagent Skills

Create an agent file in `.claude/agents/` (use `/agents` command in Claude Code to do this interactively). Add a `skills` field in the frontmatter:

```markdown
---
name: frontend-security-accessibility-reviewer
description: "Use this agent when you need to review frontend code for accessibility..."
tools: Bash, Glob, Grep, Read, WebFetch, WebSearch, Skill
model: sonnet
color: blue
skills: accessibility-audit, performance-check
---
```

When you delegate to this subagent, it has both `accessibility-audit` and `performance-check` loaded from the start and applies them to every review.

> **Note:** For subagents, skills load **at startup** — not on demand like in the main conversation.

### When This Pattern Shines

- You want isolated task delegation with specific expertise baked in
- Different subagents need different skills (frontend reviewer vs. backend reviewer)
- You want to enforce standards in delegated work without relying on prompts

---

## 10. Troubleshooting Skills

### Start Here: Use the Skills Validator

Before debugging anything else, run the **agent skills verifier** (install via `uv`):

```bash
# Navigate to your skill directory, then run the validator
# It catches structural problems before you waste time debugging
```

### Problem: Skill Doesn't Trigger

**Cause:** Almost always the description — not enough semantic overlap with your request.

**Fix:**
- Compare your description against how you actually phrase requests
- Add trigger phrases users would naturally say
- Test with variations ("profile this," "why is this slow?," "make this faster")
- If any variation fails to trigger → add those keywords to the description

### Problem: Skill Doesn't Load / Doesn't Appear

**Check:**
- `SKILL.md` must be **inside a named directory** — not sitting at the `skills/` root
- File name must be exactly `SKILL.md` — all caps on "SKILL", lowercase "md"
- YAML frontmatter syntax must be valid

**Diagnose:**
```bash
claude --debug
# Look for messages mentioning your skill name — loading errors appear here
```

### Problem: Wrong Skill Gets Used

**Cause:** Descriptions are too similar between skills.

**Fix:** Make descriptions more distinct and specific. Being specific also prevents conflicts with other similar-sounding skills.

### Problem: Skill Priority Conflict (Personal Skill Being Ignored)

**Cause:** An enterprise or higher-priority skill has the same name.

**Fix:**
1. Rename your skill to something more distinct (easiest)
2. Talk to your admin about adjusting the enterprise skill

### Problem: Plugin Skills Not Appearing

**Fix:** Clear the cache → restart Claude Code → reinstall the plugin. If still missing, the plugin structure is probably wrong — run the validator.

### Problem: Skill Loads but Fails at Runtime

| Cause | Fix |
|---|---|
| Missing dependencies | Ensure packages are installed; document dependencies in skill description |
| Permission issues | Run `chmod +x` on any scripts your skill references |
| Path separators | Use forward slashes everywhere, even on Windows |

---

## 11. Quick Reference Cheatsheet

### Create a Skill

```bash
# Personal skill (all projects)
mkdir -p ~/.claude/skills/my-skill
touch ~/.claude/skills/my-skill/SKILL.md

# Project skill (this repo only)
mkdir -p .claude/skills/my-skill
touch .claude/skills/my-skill/SKILL.md
```

### Minimal `SKILL.md` Template

```markdown
---
name: my-skill
description: What it does and when to use it. Use when [specific phrases].
---

Instructions for Claude go here.
```

### Full `SKILL.md` Template

```markdown
---
name: my-skill
description: Detailed description of what this skill does and when Claude should use it. Include trigger phrases.
allowed-tools: Read, Grep, Glob, Bash
model: sonnet
---

## Overview
Brief summary of the skill's purpose.

## Instructions
Step-by-step instructions Claude should follow.

## Reference
For detailed architecture info, read references/architecture-guide.md only when asked about system design.
```

### Decision Tree

```
Do I need this behavior in EVERY conversation?
  → Yes → CLAUDE.md
  → No  → Does it activate on a task/request?
              → Yes → Skill
              → No  → Is it event-triggered (file save, tool call)?
                          → Yes → Hook
                          → No  → Does it need an isolated context?
                                      → Yes → Subagent
                                      → No  → MCP Server
```

### Golden Rules

1. **Skill not triggering?** → Improve the description, add trigger phrases
2. **Skill not loading?** → Check path structure and file name (`SKILL.md` exactly)
3. **Wrong skill used?** → Make descriptions more distinct
4. **Shadow by enterprise skill?** → Rename yours
5. **Subagent not using your skill?** → Add `skills:` field to the agent frontmatter
6. **Keep `SKILL.md` under 500 lines** → Use progressive disclosure for the rest
7. **Restart Claude Code** after every create/edit/delete

---

*Course: [Anthropic SkillJar — Introduction to Agent Skills](https://anthropic.skilljar.com/introduction-to-agent-skills)*
