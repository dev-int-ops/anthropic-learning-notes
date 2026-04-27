# Model Context Protocol (MCP) — Advanced Topics

*This guide was created after completing the Model Context Protocol:Advanced Topics course on Anthropic's learning platform. It summarizes and organizes the key concepts in one place, with AI assistance used to structure and format the content, to help learners grasp key concepts quickly.*

---

## Table of Contents
1. [Sampling](#1-sampling)
2. [Logging & Progress Notifications](#2-logging--progress-notifications)
3. [Roots](#3-roots)
4. [MCP Message Types](#4-mcp-message-types)
5. [Transports — Stdio](#5-transports--stdio)
6. [Transports — Streamable HTTP](#6-transports--streamable-http)
7. [Stateless HTTP & JSON Response Flags](#7-stateless-http--json-response-flags)
8. [Quick-Reference Cheat Sheet](#8-quick-reference-cheat-sheet)

---

## 1. Sampling

### What Is Sampling?
Sampling lets an **MCP server request text generation from a language model (e.g., Claude) through the connected client**, rather than calling the model directly. The server delegates the AI call to the client.

### The Problem It Solves
Without sampling, an MCP server that needs to generate text (e.g., summarise Wikipedia articles) would have to:
- Hold its own API key
- Handle authentication independently
- Pay for token usage itself
- Implement all Claude integration code

This adds significant complexity and cost, especially for **public servers** where many users would consume your budget.

### How Sampling Works — Step by Step
```
1. Server finishes its work (e.g., fetches Wikipedia articles)
2. Server builds a prompt for text generation
3. Server sends a create_message (sampling) request to the client
4. Client calls Claude with the provided prompt
5. Client returns the generated text back to the server
6. Server uses that text in its final response
```

### Key Benefits
| Benefit | Why It Matters |
|---|---|
| **Reduces server complexity** | No need to integrate LLMs directly on the server |
| **Shifts cost to the client** | Client pays for token usage, not the server |
| **No API keys needed on server** | Server never needs Claude credentials |
| **Ideal for public servers** | Prevents unlimited AI costs from arbitrary users |

### Implementation

#### Server Side — request text generation
```python
@mcp.tool()
async def summarize(text_to_summarize: str, ctx: Context):
    prompt = f"""
    Please summarize the following text:
    {text_to_summarize}
    """

    result = await ctx.session.create_message(
        messages=[
            SamplingMessage(
                role="user",
                content=TextContent(
                    type="text",
                    text=prompt
                )
            )
        ],
        max_tokens=4000,
        system_prompt="You are a helpful research assistant",
    )

    if result.content.type == "text":
        return result.content.text
    else:
        raise ValueError("Sampling failed")
```

#### Client Side — handle the sampling request
```python
async def sampling_callback(
    context: RequestContext, params: CreateMessageRequestParams
):
    # Call Claude using the Anthropic SDK
    text = await chat(params.messages)

    return CreateMessageResult(
        role="assistant",
        model=model,
        content=TextContent(type="text", text=text),
    )

# Pass callback when initialising the session
async with ClientSession(
    read,
    write,
    sampling_callback=sampling_callback
) as session:
    await session.initialize()
```

### When to Use Sampling
- Building **publicly accessible MCP servers** (so you don't pay for every user's AI usage)
- When the server needs AI-generated text but you want to keep the server simple
- When the client already has the necessary Claude connection and credentials

---

## 2. Logging & Progress Notifications

### Why They Matter
When Claude calls a long-running tool (e.g., researching a topic), users see **nothing** until the operation finishes — they can't tell if the tool is working or stuck.

Logging and progress notifications provide **real-time feedback** during tool execution: progress bars, status messages, and live logs.

### How It Works
The **`Context`** argument (automatically injected into tool functions) provides methods to communicate back to the client mid-execution.

#### Key Methods
| Method | Purpose |
|---|---|
| `context.info("message")` | Send a log message to the client |
| `context.report_progress(current, total)` | Update progress indicator |

#### Server Side Example
```python
@mcp.tool(
    name="research",
    description="Research a given topic"
)
async def research(
    topic: str = Field(description="Topic to research"),
    *,
    context: Context
):
    await context.info("About to do research...")
    await context.report_progress(20, 100)
    sources = await do_research(topic)

    await context.info("Writing report...")
    await context.report_progress(70, 100)
    results = await generate_report(sources)

    return results
```

#### Client Side — Receiving Notifications
```python
# Handle log messages
async def logging_callback(params: LoggingMessageNotificationParams):
    print(params.data)

# Handle progress updates
async def print_progress_callback(
    progress: float, total: float | None, message: str | None
):
    if total is not None:
        percentage = (progress / total) * 100
        print(f"Progress: {progress}/{total} ({percentage:.1f}%)")
    else:
        print(f"Progress: {progress}")

# Wire them up
async with ClientSession(
    read,
    write,
    logging_callback=logging_callback      # ← set at session level
) as session:
    await session.initialize()

    await session.call_tool(
        name="add",
        arguments={"a": 1, "b": 3},
        progress_callback=print_progress_callback,   # ← set per tool call
    )
```

> **Note:** The logging callback is provided at **session creation**, while the progress callback is provided per **individual tool call**. This gives you flexibility to handle different tool calls differently.

### Presentation Options by App Type
| App Type | Recommended Approach |
|---|---|
| **CLI** | Print messages and progress directly to the terminal |
| **Web App** | Push updates via WebSockets, Server-Sent Events, or polling |
| **Desktop App** | Update progress bars and status labels in the UI |

### Important: Notifications Are Optional
Implementing these notifications is **purely a UX enhancement**. You can:
- Ignore them completely
- Show only certain types
- Present them in any format that suits your application

---

## 3. Roots

### What Are Roots?
Roots are a **permission and context system** that grants MCP servers access to specific files and directories on a user's local machine. They tell the server: *"You can access files within these directories."*

### The Problem Roots Solve
Without roots, when a user says *"convert biking.mp4 to MOV format"*, Claude would call the tool with just the filename — but has **no way to locate the file** in a complex file system.

The naive fix (requiring full paths from users) is terrible UX. Roots solve this elegantly.

### How Roots Change the Workflow
```
Without roots:
  User: "convert biking.mp4"          → Claude: ??? (can't find file)

With roots:
  1. User says "convert biking.mp4"
  2. Claude calls list_roots → sees accessible directories
  3. Claude calls read_dir on those directories → finds the file
  4. Claude calls the conversion tool with the full resolved path
```
This happens **automatically** — users can still speak naturally without providing full paths.

### Security Benefits
Roots also enforce **boundaries**:
- If only `~/Desktop` is granted as a root, the server **cannot** access `~/Documents` or `~/Downloads`
- Attempting to access a file outside the approved roots returns an error
- Claude can then inform the user that the file isn't accessible from the current configuration

### Implementation Pattern
The MCP SDK does **not** automatically enforce root restrictions — you must implement this yourself:

```python
# Typical helper pattern
def is_path_allowed(requested_path: str, ctx: Context) -> bool:
    approved_roots = ctx.get_roots()  # Get list of approved root paths
    for root in approved_roots:
        if requested_path.startswith(root):
            return True
    return False

# Use it before any file operation
@mcp.tool()
async def read_file(path: str, ctx: Context):
    if not is_path_allowed(path, ctx):
        raise PermissionError(f"Access denied: {path} is outside approved roots")
    # ... proceed with file operation
```

### Key Benefits Summary
| Benefit | Detail |
|---|---|
| **User-friendly** | Users don't need to type full file paths |
| **Focused search** | Claude only looks in approved directories — faster file discovery |
| **Security** | Prevents accidental access to sensitive files |
| **Flexible** | Roots can be provided via tools or injected directly into prompts |

---

## 4. MCP Message Types

### All Communication Is JSON
Every MCP interaction — tool calls, resource reads, notifications — uses JSON messages over a transport layer. Understanding message types is essential for working with different transports (especially HTTP).

### Two Categories of Messages

#### Category 1: Request–Result Pairs
These always come in pairs: you send a request and expect a result back.

| Request | Result |
|---|---|
| Call Tool Request | Call Tool Result |
| List Prompts Request | List Prompts Result |
| Read Resource Request | Read Resource Result |
| Initialize Request | Initialize Result |

#### Category 2: Notifications (One-Way)
These inform about events but require **no response**:
- Progress Notification
- Logging Message Notification
- Tool List Changed Notification
- Resource Updated Notification
- Initialized Notification
- Cancelled Notification

### Who Sends What?
MCP is a **bidirectional protocol** — both clients *and* servers can initiate communication.

| Sender | Message Types |
|---|---|
| **Client → Server** | Tool calls, resource reads, prompts, initialize |
| **Server → Client** | Sampling requests (create_message), list roots requests, progress & logging notifications |

> **This bidirectionality is critical** — it's why some transport methods (particularly HTTP) have limitations, since HTTP is designed for client→server requests, not server→client requests.

---

## 5. Transports — Stdio

### What Is the Stdio Transport?
The stdio (standard input/output) transport is the **most common transport used during development**. The client launches the MCP server as a subprocess and communicates via stdin/stdout.

```
Client  ──stdin──→  Server
Client  ←─stdout──  Server
```

### How It Works
- Client sends messages to the server via the **server's stdin**
- Server responds by writing to **stdout**
- Either party can send a message **at any time** (fully bidirectional)
- **Limitation:** Only works when client and server run on the **same machine**

### Testing Stdio Without a Client
You can test an MCP server directly from the terminal:
```bash
uv run server.py
# Then paste JSON messages directly — server reads from stdin and responds to stdout
```

### The MCP Connection Handshake (All Transports)
Every MCP connection starts with a **mandatory 3-message handshake**:

```
1. Client  →  Server:  Initialize Request
2. Server  →  Client:  Initialize Result (with capabilities)
3. Client  →  Server:  Initialized Notification  (no response expected)
```

Only after this handshake can tool calls, resource reads, etc. be made.

### Four Communication Patterns
With any transport, you handle four flows:

| Direction | Mechanism |
|---|---|
| Client → Server request | Client writes to stdin |
| Server → Client response | Server writes to stdout |
| Server → Client request | Server writes to stdout |
| Client → Server response | Client writes to stdin |

### Why Stdio Is the Baseline
Stdio represents the **ideal** case — seamless bidirectional communication. It's the mental model to understand before tackling the constraints of HTTP transports.

- **Best for:** Development, testing, local tools
- **Not suitable for:** Remote deployments, servers on different machines

---

## 6. Transports — Streamable HTTP

### What Is Streamable HTTP?
Streamable HTTP enables MCP clients to connect to **remotely hosted servers** over HTTP — so anyone can access your MCP server without being on the same machine.

### The Core HTTP Problem
HTTP was designed for **client → server** communication:
- Clients can easily make requests to servers (server has a known URL)
- Servers can easily respond to those requests
- Servers **cannot easily initiate requests** to clients (clients don't have known URLs)

This creates a problem for MCP features that require server-initiated messages (sampling, progress, logging).

### The Streamable HTTP Solution: Server-Sent Events (SSE)

#### Step 1: Initial Connection & Session ID
```
1. Client  →  POST /  :  Initialize Request
2. Server  →  Client  :  Initialize Result  +  mcp-session-id header  ← IMPORTANT
3. Client  →  POST /  :  Initialized Notification  (with session ID)
```

The `mcp-session-id` is crucial — it **uniquely identifies the client** and must be included in all future requests.

#### Step 2: SSE Connection for Server-to-Client Messages
After initialization, the client makes a **GET request** to establish a persistent SSE connection:
```
Client  →  GET /  :  Establish SSE connection
Server  ←──────────  Long-lived HTTP response (stream stays open)
Server can now push messages to client at any time through this stream
```

#### Step 3: Dual SSE Connections During Tool Calls
When the client makes a tool call, the system uses **two separate SSE connections**:

| Connection | Purpose | Lifetime |
|---|---|---|
| **Primary SSE** (GET) | Server-initiated requests & general notifications | Stays open indefinitely |
| **Tool-Specific SSE** (POST) | Per-tool progress, logs, and the final tool result | Closes automatically when result is sent |

#### Message Routing
| Message Type | Routed Through |
|---|---|
| Progress notifications | Primary SSE connection |
| Logging messages | Tool-specific SSE connection |
| Tool results | Tool-specific SSE connection |
| Sampling requests | Primary SSE connection |

### MCP Features That Break Without SSE
When SSE is unavailable (e.g., due to configuration flags), the following features fail:

| Feature | Why It Breaks |
|---|---|
| Server-initiated sampling (`create_message`) | Server can't reach client |
| List Roots requests | Server can't reach client |
| Progress notifications | No persistent channel |
| Logging notifications | No persistent channel |
| Initialized / Cancelled notifications | One-way server→client |

### Configuration Flags That Disable SSE
Two flags can break the SSE workaround:
- `stateless_http=True`
- `json_response=True`

*(Covered in detail in Section 7)*

---

## 7. Stateless HTTP & JSON Response Flags

### Why These Flags Exist: Horizontal Scaling

As your MCP server grows popular, a single instance can't handle all the traffic. The solution is **horizontal scaling** — running multiple server instances behind a load balancer.

```
             ┌─────────────┐
Clients ──→  │ Load Balancer│ ──→  Server Instance 1
             │             │ ──→  Server Instance 2
             └─────────────┘ ──→  Server Instance 3
```

**The problem:** An MCP client needs two connections:
1. A **GET SSE connection** for receiving server → client messages
2. **POST requests** for tool calls and responses

With a load balancer, these two connections might hit **different server instances**, which need to coordinate — creating complex cross-server state management.

### `stateless_http=True`

Setting this flag **eliminates the coordination problem** by removing server-side state entirely.

#### What It Disables
| Feature | Status |
|---|---|
| Session IDs | ❌ Not issued |
| Server → Client requests (sampling) | ❌ Unavailable |
| Progress notifications | ❌ Unavailable |
| Resource subscriptions / notifications | ❌ Unavailable |
| Client initialization handshake | No longer required (benefit) |

#### When to Use
- Need horizontal scaling with load balancers
- Don't need server-to-client communication
- Tools don't use AI model sampling
- Want to minimise connection overhead
- Building simple request/response tools only

### `json_response=True`

This flag is simpler: it **disables streaming for POST request responses**. Instead of an SSE stream with intermediate messages, you get only the **final result as plain JSON**.

#### Effect
| With Streaming (default) | With `json_response=True` |
|---|---|
| Intermediate progress messages | ❌ Gone |
| Log statements during execution | ❌ Gone |
| Final tool result | Still returned |
| Response format | SSE stream | Plain JSON |

#### When to Use
- Don't need streaming responses
- Integrating with systems expecting plain JSON
- Simpler client implementation

### Comparison Summary

| Flag | What It Sacrifices | Why You'd Use It |
|---|---|---|
| `stateless_http=True` | Session state, server→client messages, sampling, progress | Horizontal scaling / load balancers |
| `json_response=True` | Streaming, intermediate progress, log messages | Simpler HTTP responses, plain JSON consumers |

### Development vs Production Warning
> If you develop locally with **stdio transport** but deploy with **HTTP transport**, you must test with the same transport you'll use in production. The behaviour differences between stateful and stateless modes can be significant.

---

## 8. Quick-Reference Cheat Sheet

### Feature Availability by Configuration

| Feature | Stdio | Streamable HTTP (default) | HTTP + `stateless_http` | HTTP + `json_response` |
|---|:---:|:---:|:---:|:---:|
| Tool calls | ✅ | ✅ | ✅ | ✅ |
| Sampling (server→client AI) | ✅ | ✅ | ❌ | ✅ |
| Progress notifications | ✅ | ✅ | ❌ | ❌ |
| Logging notifications | ✅ | ✅ | ❌ | ❌ |
| Server-initiated requests | ✅ | ✅ | ❌ | ✅ |
| Horizontal scaling | ❌ | ❌ | ✅ | — |
| Remote hosting | ❌ | ✅ | ✅ | ✅ |

### MCP Handshake (All Transports)
```
Client → Initialize Request
Server → Initialize Result  (+session ID if HTTP)
Client → Initialized Notification
--- Only now can tool calls begin ---
```

### Key Concepts at a Glance

| Concept | One-Line Summary |
|---|---|
| **Sampling** | Server delegates AI text generation to the client |
| **Logging** | `context.info()` sends live log messages mid-execution |
| **Progress** | `context.report_progress(n, total)` sends live progress mid-execution |
| **Roots** | Permission system granting server access to specific local directories |
| **Stdio transport** | Client↔Server via stdin/stdout; same machine only; ideal for dev |
| **Streamable HTTP** | Remote hosting; uses SSE to enable server→client communication |
| **SSE** | Persistent HTTP GET connection; allows server to push messages to client |
| **Dual SSE** | Primary (always open) + Tool-specific (per-call) SSE connections |
| **`stateless_http`** | No session state; enables load balancers; disables sampling/progress |
| **`json_response`** | Returns plain JSON instead of SSE stream; disables intermediate updates |

---

*Course: [Anthropic SkillJar — Model Context Protocol: Advanced Topics](https://anthropic.skilljar.com/model-context-protocol-advanced-topics)*
