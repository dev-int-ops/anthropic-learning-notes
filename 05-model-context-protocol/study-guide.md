# Introduction to Model Context Protocol (MCP)

*This guide was created after completing the Introduction to Model Context Protocol course on Anthropic's learning platform. It summarizes and organizes the key concepts in one place, with AI assistance used to structure and format the content, to help learners grasp key concepts quickly.*

---

## Table of Contents
1. [What is MCP?](#1-what-is-mcp)
2. [The Problem MCP Solves](#2-the-problem-mcp-solves)
3. [Core Architecture](#3-core-architecture)
4. [How MCP Works — End-to-End Flow](#4-how-mcp-works--end-to-end-flow)
5. [Building an MCP Server](#5-building-an-mcp-server)
6. [Testing with the MCP Inspector](#6-testing-with-the-mcp-inspector)
7. [Building an MCP Client](#7-building-an-mcp-client)
8. [The Three Server Primitives](#8-the-three-server-primitives)
   - [Tools — Model-Controlled](#81-tools--model-controlled)
   - [Resources — App-Controlled](#82-resources--app-controlled)
   - [Prompts — User-Controlled](#83-prompts--user-controlled)
9. [Choosing the Right Primitive](#9-choosing-the-right-primitive)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. What is MCP?

**Model Context Protocol (MCP)** is a communication layer that gives Claude access to external context and tools — without requiring you to write all the integration code yourself.

> **Simple analogy:** Think of MCP as a way to *outsource* tool definitions and execution to specialized servers, rather than building and maintaining everything yourself.

### MCP vs. Plain Tool Use
| Aspect | Plain Tool Use | MCP |
|---|---|---|
| Who writes tool schemas? | You | MCP Server author |
| Who handles API calls? | You | MCP Server |
| Who maintains integrations? | You | MCP Server author |
| Claude's role | Same | Same — calls tools as usual |

> **Common misconception:** MCP and tool use are *not* the same thing. Tool use is *how Claude calls tools*. MCP is *who provides and runs those tools*. They are complementary, not interchangeable.

---

## 2. The Problem MCP Solves

### Without MCP
Imagine building a chat interface where users can query GitHub. GitHub has enormous functionality: repos, pull requests, issues, projects, etc. Without MCP, you would need to:

- Write JSON schemas for every tool
- Write handler functions for every API endpoint
- Test and maintain all that code yourself
- Repeat this for every external service you want to support

This is an enormous and ongoing maintenance burden.

### With MCP
An **MCP Server for GitHub** wraps all of that functionality and exposes it as a standardized set of tools. Your application connects to the MCP server instead of building everything from scratch.

---

## 3. Core Architecture

```
Your Application
      │
      ▼
  MCP Client  ◄──────────────────────────────────────────────────────┐
  (your server)                                                        │
      │                                                                │
      │  ListToolsRequest / CallToolRequest                            │
      ▼                                                                │
  MCP Server  ────── Wraps external API (e.g. GitHub) ───► GitHub API │
  (tool definitions + execution)                                       │
                                                                       │
                         Results flow back ─────────────────────────►─┘
```

### Components
- **MCP Client** — The communication bridge in *your* server. It handles the message exchange and protocol details so your application doesn't have to.
- **MCP Server** — A dedicated server that wraps an external service and exposes tools, prompts, and resources in a standardized way.

### Transport Options
MCP is **transport agnostic** — the client and server can communicate over:
- **Standard input/output** (most common; both on the same machine)
- **HTTP**
- **WebSockets**
- Other network protocols

### Key Message Types
| Message | Direction | Purpose |
|---|---|---|
| `ListToolsRequest` | Client → Server | "What tools do you have?" |
| `ListToolsResult` | Server → Client | Returns list of available tools |
| `CallToolRequest` | Client → Server | "Run this tool with these args" |
| `CallToolResult` | Server → Client | Returns result of the tool execution |
| `ReadResourceRequest` | Client → Server | "Give me this resource by URI" |
| `ReadResourceResult` | Server → Client | Returns resource data |

---

## 4. How MCP Works — End-to-End Flow

**Example query:** *"What repositories do I have?"*

```
Step 1:  User submits query to your server
Step 2:  Your server needs to know what tools exist
Step 3:  Your server asks the MCP Client for tool list
Step 4:  MCP Client sends ListToolsRequest → MCP Server returns ListToolsResult
Step 5:  Your server sends user query + tool list → Claude
Step 6:  Claude decides to call a tool (e.g. get_repos)
Step 7:  Your server asks MCP Client to run that tool
Step 8:  MCP Client sends CallToolRequest → MCP Server calls GitHub API
Step 9:  GitHub responds → MCP Server returns CallToolResult
Step 10: Your server sends tool result back to Claude
Step 11: Claude formulates a final answer
Step 12: Your server delivers Claude's response to the user
```

> Each component has a **clear, single responsibility**. The MCP Client abstracts away protocol complexity so your application logic stays clean.

---

## 5. Building an MCP Server

The **Python MCP SDK** (`mcp.server.fastmcp`) eliminates manual JSON schema writing. You define tools using **Python decorators and type hints** — the SDK auto-generates the proper schema Claude understands.

### Server Initialization
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("DocumentMCP", log_level="ERROR")
```

### Storing Data
```python
docs = {
    "deposition.md": "This deposition covers the testimony of Angela Smith, P.E.",
    "report.pdf": "The report details the state of a 20m condenser tower.",
    "financials.docx": "These financials outline the project's budget and expenditures",
    # ...
}
```

---

### 5.1 Defining Tools

Use the `@mcp.tool` decorator. Parameters use `Field` from Pydantic for descriptions.

#### Read Tool
```python
@mcp.tool(
    name="read_doc_contents",
    description="Read the contents of a document and return it as a string."
)
def read_document(
    doc_id: str = Field(description="Id of the document to read")
):
    if doc_id not in docs:
        raise ValueError(f"Doc with id {doc_id} not found")
    return docs[doc_id]
```

#### Edit Tool
```python
@mcp.tool(
    name="edit_document",
    description="Edit a document by replacing a string in the document content with a new string."
)
def edit_document(
    doc_id: str = Field(description="Id of the document that will be edited"),
    old_str: str = Field(description="The text to replace. Must match exactly, including whitespace."),
    new_str: str = Field(description="The new text to insert in place of the old text.")
):
    if doc_id not in docs:
        raise ValueError(f"Doc with id {doc_id} not found")
    docs[doc_id] = docs[doc_id].replace(old_str, new_str)
```

#### Benefits of the SDK Decorator Approach
- ✅ No manual JSON schema writing
- ✅ Type hints provide automatic validation
- ✅ Field descriptions help Claude understand how to use each parameter
- ✅ Error handling integrates naturally with Python exceptions
- ✅ Tools are auto-registered through decorators

---

### 5.2 Defining Resources

Resources expose **read-only data** — similar to GET endpoints in an HTTP server. They're ideal for fetching information rather than performing actions.

**Two types of resources:**

#### Direct Resources (static URI)
```python
@mcp.resource(
    "docs://documents",
    mime_type="application/json"
)
def list_docs() -> list[str]:
    return list(docs.keys())
```

#### Templated Resources (parameterized URI)
The SDK automatically parses URI parameters and passes them as keyword arguments.
```python
@mcp.resource(
    "docs://documents/{doc_id}",
    mime_type="text/plain"
)
def fetch_doc(doc_id: str) -> str:
    if doc_id not in docs:
        raise ValueError(f"Doc with id {doc_id} not found")
    return docs[doc_id]
```

#### MIME Types
| Data Type | MIME Type |
|---|---|
| Structured / JSON | `application/json` |
| Plain text | `text/plain` |
| Binary / PDF | `application/pdf` |

> The SDK auto-serializes return values — you don't need to manually convert to JSON strings.

---

### 5.3 Defining Prompts

Prompts are **pre-built, high-quality instruction templates** that clients can invoke. They encode expert prompt engineering so users don't have to.

> **Key insight:** A user *can* ask Claude to "reformat the report.pdf in markdown" directly — but they'll get *far better results* from a carefully tested, specialized prompt you've crafted as the server author.

```python
from mcp.server import base

@mcp.prompt(
    name="format",
    description="Rewrites the contents of the document in Markdown format."
)
def format_document(
    doc_id: str = Field(description="Id of the document to format")
) -> list[base.Message]:
    prompt = f"""
Your goal is to reformat a document to be written with markdown syntax.

The id of the document you need to reformat is:
<document_id>
{doc_id}
</document_id>

Add in headers, bullet points, tables, etc as necessary. Feel free to add in structure.
Use the 'edit_document' tool to edit the document. After the document has been reformatted...
"""
    return [base.UserMessage(prompt)]
```

Prompts return a `list[base.Message]` — you can include multiple user/assistant messages to create complex conversation flows.

#### Benefits of Server-Defined Prompts
- **Consistency** — Same high-quality result every time
- **Expertise** — Encode domain knowledge once; all clients benefit
- **Reusability** — Multiple apps can use the same prompts
- **Maintainability** — Update in one place to improve all clients

---

## 6. Testing with the MCP Inspector

The Python MCP SDK includes a **built-in browser-based inspector** for testing servers without a full app connection.

### Start the Inspector
```bash
mcp dev mcp_server.py
# Opens at: http://127.0.0.1:6274
```

### Inspector Workflow
1. Open the URL in your browser
2. Click **Connect** to initialize the server
3. Navigate to **Tools** → click "List Tools" to see all available tools
4. Select a tool → fill in parameters → click "Run Tool" → verify results
5. Navigate to **Resources** / **Resource Templates** to test resource endpoints
6. Navigate to **Prompts** to see prompt definitions and test variable interpolation

### Key Feature
The inspector **maintains server state between calls** — so an edit made with one tool call will be visible when you call the read tool next. This lets you verify complete workflows end-to-end.

---

## 7. Building an MCP Client

The client is what allows your application code to communicate with the MCP server. In most real projects, you build *either* a client *or* a server — not both. This course builds both to show how they connect.

### Client Architecture
```
Your App Code
     │
     ▼
MCP Client (custom class)          ← you write this
     │
     ▼
Client Session (MCP Python SDK)    ← SDK handles protocol
     │
     ▼
MCP Server
```

The custom class wraps the session to handle **resource management and cleanup** automatically.

### Core Client Methods

#### List Tools
```python
async def list_tools(self) -> list[types.Tool]:
    result = await self.session().list_tools()
    return result.tools
```

#### Call Tool
```python
async def call_tool(
    self, tool_name: str, tool_input: dict
) -> types.CallToolResult | None:
    return await self.session().call_tool(tool_name, tool_input)
```

#### Read Resource
```python
import json
from pydantic import AnyUrl

async def read_resource(self, uri: str) -> Any:
    result = await self.session().read_resource(AnyUrl(uri))
    resource = result.contents[0]

    if isinstance(resource, types.TextResourceContents):
        if resource.mimeType == "application/json":
            return json.loads(resource.text)

    return resource.text
```

> Response structure: `result.contents[0]` contains the content + MIME type. JSON is parsed; plain text is returned as-is.

#### List Prompts
```python
async def list_prompts(self) -> list[types.Prompt]:
    result = await self.session().list_prompts()
    return result.prompts
```

#### Get a Prompt (with variable interpolation)
```python
async def get_prompt(self, prompt_name, args: dict[str, str]):
    result = await self.session().get_prompt(prompt_name, args)
    return result.messages
```

Arguments are passed as a dict — e.g. `{"doc_id": "plan.md"}` — and the server interpolates them into the prompt template automatically.

### Testing the Client
```bash
uv run mcp_client.py    # Verifies connection + prints available tools
uv run main.py          # Full app — try: "What is the contents of the report.pdf document?"
```

---

## 8. The Three Server Primitives

This is the **most important conceptual section** of the course. Each primitive is controlled by a *different* part of the application stack.

---

### 8.1 Tools — Model-Controlled

> **Who decides when to use them:** Claude (the AI model)

Tools give Claude **new capabilities** it can invoke autonomously to complete tasks. Claude reads the tool descriptions, decides which tool to call, provides the arguments, and uses the result.

**Use tools when:** You want to give Claude the ability to *do* something — fetch data, run code, mutate state, call APIs.

**Example:** "Calculate the square root of 3 using JavaScript" → Claude decides to call a JavaScript execution tool.

---

### 8.2 Resources — App-Controlled

> **Who decides when to use them:** Your application code

Resources expose data your **app** fetches and uses — either to populate UI elements or to inject context into prompts *before* sending them to Claude.

**Use resources when:** You need to get data into your app for display or to augment a prompt. The data arrives *with* the initial message to Claude — no tool call needed.

**Two real-world uses in this course:**
1. Populating an `@document` **autocomplete** list in the UI
2. **Injecting document content** directly into the prompt when a user types `@filename`

> **Real-world analogy:** The "Add from Google Drive" feature in Claude's interface — your application controls what documents appear and handles injecting their content into the chat context.

---

### 8.3 Prompts — User-Controlled

> **Who decides when to use them:** The end user

Prompts are **predefined workflows** users can trigger through UI interactions — slash commands, buttons, menu selections, etc.

**Use prompts when:** You want to give users access to a carefully crafted, reusable workflow they can invoke on demand.

**Example:** Typing `/format` in the CLI → a pre-built, tested prompt is sent to Claude with the right instructions. The user doesn't have to know how to write that prompt themselves.

> **Real-world analogy:** The workflow buttons below Claude's chat input — predefined, optimized workflows users can start with a single click.

---

## 9. Choosing the Right Primitive

| Question | Answer → Use |
|---|---|
| Do I want Claude to autonomously decide when to use this? | **Tool** |
| Do I need data in my app for UI or to pre-load context? | **Resource** |
| Do I want users to trigger a predefined workflow? | **Prompt** |

```
    ┌─────────────────────────────────────────────┐
    │            Who controls the action?         │
    └─────────────────────────────────────────────┘
        │               │               │
      Claude        Your App        The User
        │               │               │
       TOOL          RESOURCE          PROMPT
  (new capability)   (data/context)   (workflow)
```

---

## 10. Key Takeaways

### The Big Idea
MCP shifts the **burden of tool integration** from your server to specialized, reusable MCP servers. You get to focus on application logic; someone else maintains the integrations.

### Who Authors MCP Servers?
Anyone — including service providers themselves. For example, AWS could publish an official MCP server with tools for all their services. You'd simply connect to it.

### MCP ≠ Tool Use
- **Tool use** = *how* Claude calls functions
- **MCP** = *who provides and executes* those functions
- They work together; neither replaces the other

### Resources vs. Tools for Data Fetching
| Approach | When data is fetched | Who triggers it |
|---|---|---|
| Tool (Claude fetches data) | *After* initial message, mid-conversation | Claude |
| Resource (app injects data) | *Before* initial message, as part of context | Your app |

Resources are more efficient when you *know in advance* what context Claude needs.

### The Flow in One Sentence
> Your app uses the **MCP Client** to talk to an **MCP Server**, which provides **Tools** (for Claude), **Resources** (for your app), and **Prompts** (for your users).

---

*Course: [Anthropic SkillJar — Introduction to Model Context Protocol](https://anthropic.skilljar.com/introduction-to-model-context-protocol)*
