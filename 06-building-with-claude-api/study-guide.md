# Building with the Claude API

*This guide was created after completing the Building with the Claude API course on Anthropic's learning platform. It summarizes and organizes the key concepts in one place, with AI assistance used to structure and format the content, to help learners grasp key concepts quickly.*

---

## Table of Contents

1. [Claude Model Overview](#1-claude-model-overview)
2. [API Request Lifecycle](#2-api-request-lifecycle)
3. [Making Your First Request](#3-making-your-first-request)
4. [Multi-Turn Conversations](#4-multi-turn-conversations)
5. [System Prompts](#5-system-prompts)
6. [Temperature](#6-temperature)
7. [Response Streaming](#7-response-streaming)
8. [Controlling Model Output](#8-controlling-model-output)
9. [Structured Data Generation](#9-structured-data-generation)
10. [Prompt Evaluation](#10-prompt-evaluation)
11. [Prompt Engineering Techniques](#11-prompt-engineering-techniques)
12. [Tool Use (Function Calling)](#12-tool-use-function-calling)
13. [Multi-Tool & Multi-Turn Tool Conversations](#13-multi-tool--multi-turn-tool-conversations)
14. [Advanced Tool Features](#14-advanced-tool-features)
15. [RAG — Retrieval Augmented Generation](#15-rag--retrieval-augmented-generation)
16. [Extended Thinking](#16-extended-thinking)
17. [Image & PDF Support](#17-image--pdf-support)
18. [Citations](#18-citations)
19. [Prompt Caching](#19-prompt-caching)
20. [Code Execution & Files API](#20-code-execution--files-api)
21. [Model Context Protocol (MCP)](#21-model-context-protocol-mcp)
22. [Claude Code](#22-claude-code)
23. [Computer Use](#23-computer-use)
24. [Agents & Workflows](#24-agents--workflows)

---

## 1. Claude Model Overview

### Three Model Families

| Model | Priority | Best For | Trade-offs |
|---|---|---|---|
| **Opus** | Highest intelligence | Complex, multi-step tasks, deep reasoning & planning | Higher cost, higher latency |
| **Sonnet** | Balanced | Most practical use cases, strong coding & precise editing | — |
| **Haiku** | Speed & cost | Real-time user interactions, high-volume processing | No reasoning capabilities |

### Selection Framework
- Intelligence priority → **Opus**
- Speed priority → **Haiku**
- Balanced requirements → **Sonnet**

### Key Points
- All models share core capabilities: text generation, coding, image analysis
- Main difference is optimization focus, not capability set
- **Common production pattern:** use multiple models within the same application — e.g., Haiku for dataset generation, Sonnet for the main task

---

## 2. API Request Lifecycle

### The 5-Step Flow

```
Client App → Your Server → Anthropic API → Model Processing → Response back
```

**Why you need a server:** Never call the Anthropic API directly from client code. Your API key would be exposed and anyone could extract it to make unauthorized requests.

### Step-by-Step Breakdown

1. **Client → Server:** User input is sent to your server
2. **Server → Anthropic API:** Request with API key, model name, messages list, max_tokens
3. **Model Processing (4 stages):**
   - **Tokenization** — input broken into tokens (words, word-parts, symbols, spaces)
   - **Embedding** — each token converted to a list of numbers representing all possible meanings
   - **Contextualization** — embeddings adjusted based on neighboring tokens to pin down precise meaning
   - **Generation** — output layer calculates probabilities for next token; model selects using probability + controlled randomness; repeats
4. **Stopping** — generation ends when `max_tokens` is reached OR a special end-of-sequence token is generated OR a stop sequence is hit
5. **API → Server → Client:** Response returned with generated text, usage counts, and `stop_reason`

### Key Vocabulary

| Term | Meaning |
|---|---|
| Token | Text chunk — word, part of word, symbol |
| Embedding | Numerical representation of word meaning |
| Contextualization | Meaning refinement using surrounding tokens |
| max_tokens | Generation length limit (a safety cap, not a target) |
| stop_reason | Why the model stopped generating |

---

## 3. Making Your First Request

### Setup

```python
# Install
%pip install anthropic python-dotenv

# .env file
ANTHROPIC_API_KEY="your-key-here"

# Python setup
from dotenv import load_dotenv
from anthropic import Anthropic

load_dotenv()
client = Anthropic()
model = "claude-sonnet-4-6"
```

> **Security:** Always store your key in `.env` and add `.env` to `.gitignore`.

### The `client.messages.create()` Call

```python
message = client.messages.create(
    model=model,
    max_tokens=1000,
    messages=[
        {"role": "user", "content": "What is quantum computing? One sentence."}
    ]
)
```

**Required parameters:**
- `model` — which Claude model to use
- `max_tokens` — safety limit (not a target length)
- `messages` — list of conversation exchanges

### Accessing the Response

```python
# Just the text
message.content[0].text

# Full response object has metadata, usage counts, stop_reason, etc.
```

---

## 4. Multi-Turn Conversations

### The Core Problem
**Claude stores nothing.** Every API request is completely independent with zero memory of previous exchanges.

### The Solution
You must:
1. Manually maintain a message list in your code
2. Send the **entire conversation history** with every follow-up request

### Message Structure
```python
# User message
{"role": "user", "content": "your text"}

# Assistant message
{"role": "assistant", "content": "claude's response"}
```

### Helper Functions Pattern

```python
def add_user_message(messages, text):
    messages.append({"role": "user", "content": text})

def add_assistant_message(messages, text):
    messages.append({"role": "assistant", "content": text})

def chat(messages):
    message = client.messages.create(
        model=model,
        max_tokens=1000,
        messages=messages,
    )
    return message.content[0].text

# Usage
messages = []
add_user_message(messages, "Define quantum computing in one sentence")
answer = chat(messages)
add_assistant_message(messages, answer)       # ← MUST save Claude's reply
add_user_message(messages, "Write another sentence")
final_answer = chat(messages)                  # ← Claude now has context
```

> Without appending Claude's response to history, follow-up questions have no context.

---

## 5. System Prompts

### What They Do
System prompts customize **how Claude responds**, not what it responds to. They assign a role or behavioral pattern.

### Implementation

```python
def chat(messages, system=None):
    params = {
        "model": model,
        "max_tokens": 1000,
        "messages": messages,
    }
    if system:
        params["system"] = system      # Only add if provided; API rejects system=None
    
    message = client.messages.create(**params)
    return message.content[0].text
```

### Example — Math Tutor

```python
system_prompt = """
You are a patient math tutor.
Do not directly answer a student's questions.
Guide them to a solution step by step.
"""

answer = chat(messages, system=system_prompt)
```

### Key Principles
- First line typically assigns the role
- Followed by specific behavioral instructions
- Same question → completely different treatment depending on the role assigned
- **Technical note:** Must conditionally include `system` — do not pass `None`

---

## 6. Temperature

### What It Controls
Temperature (0–1) controls the randomness of token selection.

| Temperature | Behavior | Use For |
|---|---|---|
| **0** | Deterministic — always picks highest-probability token | Data extraction, factual tasks requiring consistency |
| **Near 1** | Creative — lower-probability tokens become likely | Brainstorming, creative writing, jokes, marketing |

### How It Works
- Higher temperature → flattens the probability distribution → more varied outputs
- Lower temperature → sharpens the distribution → more predictable outputs
- Higher values **don't guarantee** different outputs, just increase the probability of variation

### Implementation
```python
client.messages.create(
    model=model,
    max_tokens=1000,
    messages=messages,
    temperature=0.9  # Add this parameter
)
```

---

## 7. Response Streaming

### The Problem
AI responses can take 10–30 seconds. Users sitting at a loading spinner have a poor experience.

### How Streaming Works
Instead of waiting for the full response, Claude immediately sends back chunks as they're generated.

**Stream Event Types:**

| Event | Meaning |
|---|---|
| `message_start` | Initial acknowledgment — no text yet |
| `content_block_start` | Text generation begins |
| `content_block_delta` | **Actual text chunks** — this is what you display |
| `content_block_stop` | Current block done |
| `message_stop` | Entire message complete |

### Implementation

**Basic (manual):**
```python
stream = client.messages.create(
    model=model, max_tokens=1000, messages=messages, stream=True
)
for event in stream:
    print(event)
```

**Simplified (recommended):**
```python
with client.messages.stream(model=model, max_tokens=1000, messages=messages) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)   # Real-time display
    
    final_message = stream.get_final_message()  # Save to DB
```

> `get_final_message()` assembles all chunks into a complete message object for storage or further processing.

---

## 8. Controlling Model Output

Two techniques for precise control over Claude's response direction and length.

### Technique 1 — Pre-filling Assistant Messages

Add a manual assistant message at the end of your conversation to steer the response.

```python
messages = []
add_user_message(messages, "Which is better, coffee or tea?")
add_assistant_message(messages, "Coffee is better because")   # ← Pre-fill

response = chat(messages)
# Claude continues from exactly "Coffee is better because..."
# You must stitch: "Coffee is better because" + response
```

**Key point:** Claude continues from the exact endpoint — not a complete sentence. You must concatenate the pre-fill with the generated text.

### Technique 2 — Stop Sequences

Force Claude to halt generation when a specific string appears.

```python
def chat(messages, stop_sequences=[]):
    params = {
        "model": model, "max_tokens": 1000, "messages": messages,
    }
    if stop_sequences:
        params["stop_sequences"] = stop_sequences
    return client.messages.create(**params).content[0].text

# Example
chat(messages, stop_sequences=["five"])
# Prompt "count 1 to 10" → Output: "one, two, three, four, "
# (stops before "five", which is NOT included in output)

# Refined version
chat(messages, stop_sequences=[", five"])
# Output: "one, two, three, four"  ← Clean
```

---

## 9. Structured Data Generation

### The Problem
When asking Claude to generate JSON or code, it naturally adds markdown fences, headers, and explanatory text. You often just want the raw data.

### The Solution Pattern

Combine **assistant message pre-filling** + **stop sequences**:

```python
messages = []
add_user_message(messages, "Generate a short EventBridge rule as JSON")
add_assistant_message(messages, "```json")         # ← Pre-fill opening delimiter

text = chat(messages, stop_sequences=["```"])      # ← Stop at closing delimiter

# Result: raw JSON with no fences or commentary
import json
clean_data = json.loads(text.strip())
```

**How it works:**
1. Pre-fill tells Claude it already started a code block
2. Claude generates only the structured content
3. Stop sequence halts generation before the closing fence
4. Result is clean, parseable data

> **Works for any structured type:** JSON, Python, regex, CSV, lists — any format with natural delimiters.

---

## 10. Prompt Evaluation

### Why It Matters
Engineers consistently under-test prompts. The three common paths after writing a prompt:

| Path | Risk |
|---|---|
| Test once → deploy | High risk of production failures |
| Test a few times, tweak for corner cases | Users find edge cases you didn't consider |
| **Run through an evaluation pipeline** | **Recommended — objective scoring** |

### The 6-Step Eval Workflow

```
1. Write initial prompt draft
2. Create evaluation dataset (test inputs)
3. Generate prompt variations (interpolate inputs)
4. Get LLM responses for each variation
5. Grade responses (1–10 scale) → average score
6. Modify prompt → repeat cycle → compare versions
```

### Generating a Test Dataset

```python
def generate_dataset():
    prompt = """Generate an evaluation dataset... [your criteria]
Example output:
```json
[{"task": "description"}, ...]
```"""
    messages = []
    add_user_message(messages, prompt)
    add_assistant_message(messages, "```json")          # Structured output trick
    text = chat(messages, stop_sequences=["```"])
    return json.loads(text)

dataset = generate_dataset()
with open('dataset.json', 'w') as f:
    json.dump(dataset, f, indent=2)
```

### Core Eval Pipeline

```python
def run_prompt(test_case):
    prompt = f"Please solve the following task:\n\n{test_case['task']}"
    messages = []
    add_user_message(messages, prompt)
    return chat(messages)

def run_test_case(test_case):
    output = run_prompt(test_case)
    score = grade_output(test_case, output)    # See grading section below
    return {"output": output, "test_case": test_case, "score": score}

def run_eval(dataset):
    results = [run_test_case(tc) for tc in dataset]
    avg = sum(r["score"] for r in results) / len(results)
    print(f"Average score: {avg}")
    return results
```

### Types of Graders

| Grader Type | How It Works | Best For |
|---|---|---|
| **Code grader** | Programmatic checks (JSON parsing, AST parsing, regex compile) | Syntax validation, format checking |
| **Model grader** | Another API call evaluates quality | Instruction-following, tone, accuracy |
| **Human grader** | Person reviews outputs | Highest accuracy but slow and expensive |

#### Code Grader Example

```python
import json, ast, re

def validate_json(text):
    try: json.loads(text.strip()); return 10
    except: return 0

def validate_python(text):
    try: ast.parse(text.strip()); return 10
    except SyntaxError: return 0

def validate_regex(text):
    try: re.compile(text.strip()); return 10
    except re.error: return 0

# Combined score
final_score = (model_score + syntax_score) / 2
```

#### Model Grader Example

```python
def grade_by_model(test_case, output):
    eval_prompt = f"""
    Evaluate this AI solution.
    Task: {test_case['task']}
    Solution: {output}
    
    Return JSON with: strengths (array), weaknesses (array), reasoning (string), score (1-10)
    """
    messages = []
    add_user_message(messages, eval_prompt)
    add_assistant_message(messages, "```json")
    text = chat(messages, stop_sequences=["```"])
    return json.loads(text)
```

> **Important:** Ask for strengths, weaknesses, AND reasoning alongside the score. Without this context, models default to middling scores (~6).

---

## 11. Prompt Engineering Techniques

### Starting Baseline
Initial naïve prompts score poorly (e.g., 2.32/10). Each technique adds measurable improvement.

---

### Technique 1 — Be Clear and Direct

**The first line is the most important part of your prompt.**

```
Structure: Action Verb + Clear Task + Output Specifications
```

| ❌ Vague | ✅ Clear & Direct |
|---|---|
| "What should this person eat?" | "Generate a one-day meal plan for an athlete that meets their dietary restrictions." |
| "Tell me about solar panels" | "Write three paragraphs about how solar panels work." |

**Result:** Score increase from 2.32 → 3.92 just from this change.

---

### Technique 2 — Be Specific (Add Guidelines)

**Two types of guidelines:**

**Type A — Output Attributes** (control characteristics of output):
```
- Include accurate daily calorie amount
- Show protein, fat, and carb amounts
- Specify when to eat each meal
- List all portion sizes in grams
```

**Type B — Process Steps** (control how Claude reasons):
```
1. Identify relevant factors for the athlete's goal
2. Consider any dietary restrictions carefully
3. Calculate approximate macros before writing the plan
4. Then write the meal plan
```

**When to use each:**
- Type A: **Almost always** — controls output quality and format
- Type B: For complex problems requiring broader perspective

**Result:** Score jumped from 3.92 → 7.86 with guidelines added.

---

### Technique 3 — Structure with XML Tags

Use XML tags to delineate different content sections, especially when interpolating external data.

```python
prompt = f"""
Generate a meal plan based on the athlete's information.

<athlete_information>
- Height: {height}
- Weight: {weight}
- Goal: {goal}
- Restrictions: {restrictions}
</athlete_information>
"""
```

**Why it works:** Claude clearly understands what is an instruction vs. what is external data.

**Naming matters:** `<sales_records>` is better than `<data>` — be descriptive.

---

### Technique 4 — Provide Examples (One-shot / Multi-shot)

One-shot = 1 example. Multi-shot = several examples.

```
<example>
  <sample_input>
    "Great game tonight!"
  </sample_input>
  <ideal_output>
    Positive
  </ideal_output>
  <explanation>
    Direct positive sentiment with no ambiguity.
  </explanation>
</example>

<example>
  <sample_input>
    "Oh yeah, I really needed a flight delay tonight! Excellent!"
  </sample_input>
  <ideal_output>
    Negative
  </ideal_output>
  <explanation>
    Sarcastic tone — the surface words are positive but the meaning is negative.
    Be especially careful with sarcasm.
  </explanation>
</example>
```

**Best practices:**
- Wrap examples clearly in XML tags
- Include the reasoning that makes the output ideal
- Use your highest-scoring eval results as example templates
- Place examples **after** main instructions

---

## 12. Tool Use (Function Calling)

### What Tools Solve
By default, Claude only knows its training data. Tools let Claude access real-time information, external APIs, and custom functionality.

### The Tool Use Flow

```
1. Send initial request + tool schema(s) to Claude
2. Claude evaluates → decides it needs external data
3. Claude responds with a tool_use block (name + arguments)
4. Your server executes the function
5. Send tool result back to Claude (full history required)
6. Claude generates final response using the result
```

### Building a Tool Function

```python
from datetime import datetime

def get_current_datetime(date_format="%Y-%m-%d %H:%M:%S"):
    if not date_format:
        raise ValueError("date_format cannot be empty")   # Claude sees this error!
    return datetime.now().strftime(date_format)
```

**Best practices for tool functions:**
- Use descriptive function and parameter names
- Validate inputs and raise errors with clear messages
- Claude can see error messages and will retry with corrections

### Building a Tool Schema

```python
from anthropic.types import ToolParam

get_current_datetime_schema = ToolParam({
    "name": "get_current_datetime",
    "description": """Returns the current date and time formatted according to the 
    specified format string. Use this when you need to know the current date or time. 
    Returns a formatted datetime string.""",
    "input_schema": {
        "type": "object",
        "properties": {
            "date_format": {
                "type": "string",
                "description": "Python strftime format string. Default: '%Y-%m-%d %H:%M:%S'",
                "default": "%Y-%m-%d %H:%M:%S"
            }
        },
        "required": []
    }
})
```

> **Shortcut:** Paste your function into Claude.ai and ask it to "write a valid JSON schema spec for tool calling for this function." Attach the Anthropic tool use docs page for best results.

### Making a Tool-Enabled Request

```python
response = client.messages.create(
    model=model,
    max_tokens=1000,
    messages=messages,
    tools=[get_current_datetime_schema],  # ← Include schema
)
```

### Understanding Multi-Block Responses

When Claude uses a tool, the response contains **multiple blocks**:

```python
# response.content now has multiple items:
# [0] TextBlock    — human-readable explanation
# [1] ToolUseBlock — {id, name, input (dict of args)}

# Append the ENTIRE content list (not just text!)
messages.append({"role": "assistant", "content": response.content})
```

### Sending Tool Results Back

```python
tool_result = {
    "type": "tool_result",
    "tool_use_id": response.content[1].id,   # Must match the ToolUseBlock ID
    "content": json.dumps(tool_output),       # Must be a string
    "is_error": False
}

messages.append({"role": "user", "content": [tool_result]})

# Make follow-up request — STILL include the tool schemas!
final_response = client.messages.create(
    model=model, max_tokens=1000, messages=messages,
    tools=[get_current_datetime_schema]
)
```

---

## 13. Multi-Tool & Multi-Turn Tool Conversations

### The Challenge
You can't predict how many tool calls a single user query will require. Some questions need 1 tool; others need 5 in sequence.

### The `stop_reason` Field

| Value | Meaning |
|---|---|
| `"tool_use"` | Claude wants to call a tool — keep looping |
| `"end_turn"` (or others) | Claude has a final answer — stop |

### The Conversation Loop

```python
def run_conversation(messages):
    while True:
        response = chat(messages, tools=[
            get_current_datetime_schema,
            add_duration_to_datetime_schema,
            set_reminder_schema
        ])
        add_assistant_message(messages, response)
        
        if response.stop_reason != "tool_use":
            break                                    # Done — no more tools needed
        
        tool_results = run_tools(response)
        add_user_message(messages, tool_results)    # Append ALL results as one user message
    
    return messages
```

### Processing Tool Requests

```python
def run_tools(message):
    tool_requests = [b for b in message.content if b.type == "tool_use"]
    tool_result_blocks = []
    
    for tool_request in tool_requests:
        try:
            output = run_tool(tool_request.name, tool_request.input)
            tool_result_blocks.append({
                "type": "tool_result",
                "tool_use_id": tool_request.id,
                "content": json.dumps(output),
                "is_error": False
            })
        except Exception as e:
            tool_result_blocks.append({
                "type": "tool_result",
                "tool_use_id": tool_request.id,
                "content": str(e),
                "is_error": True
            })
    
    return tool_result_blocks

def run_tool(tool_name, tool_input):
    if tool_name == "get_current_datetime":
        return get_current_datetime(**tool_input)
    elif tool_name == "add_duration_to_datetime":
        return add_duration_to_datetime(**tool_input)
    elif tool_name == "set_reminder":
        return set_reminder(**tool_input)
    # Add new tools here — scalable pattern
```

### Adding New Tools (3-Step Pattern)
1. Add the tool schema to the `tools` list in `run_conversation`
2. Add an `elif` case in `run_tool` to route to the function
3. Implement the actual function

---

## 14. Advanced Tool Features

### The Batch Tool (Parallel Execution)
Claude rarely sends multiple `tool_use` blocks spontaneously. The **batch tool** tricks Claude into parallel execution.

```python
# Schema accepts a list of invocations
batch_tool_schema = {
    "name": "batch",
    "description": "Run multiple tools in parallel",
    "input_schema": {
        "type": "object",
        "properties": {
            "invocations": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "tool_name": {"type": "string"},
                        "arguments": {"type": "string"}  # JSON-encoded args
                    }
                }
            }
        }
    }
}

def run_batch(invocations):
    batch_output = []
    for invocation in invocations:
        output = run_tool(invocation["tool_name"], json.loads(invocation["arguments"]))
        batch_output.append(output)
    return batch_output
```

### Tools for Structured Data Extraction
Alternative to pre-fill + stop sequences — more reliable but more setup.

```python
extraction_schema = {
    "name": "save_article",
    "description": "Save extracted article data",
    "input_schema": {
        "type": "object",
        "properties": {
            "title": {"type": "string"},
            "author": {"type": "string"},
            "summary": {"type": "string"}
        },
        "required": ["title", "author", "summary"]
    }
}

# Force Claude to always call this tool
response = client.messages.create(
    model=model,
    max_tokens=1000,
    messages=messages,
    tools=[extraction_schema],
    tool_choice={"type": "tool", "name": "save_article"}   # ← Force it
)

# Extract the structured data from the tool use block
structured_data = response.content[0].input
```

### Tool Streaming

Standard streaming + tools produces `input_json_delta` events:

```python
for chunk in stream:
    if chunk.type == "input_json":
        print(chunk.partial_json)   # Chunk of JSON args
        # chunk.snapshot = cumulative JSON so far
```

**Default behavior:** API buffers and validates full key-value pairs before sending → delayed bursts

**Fine-grained mode:** Set `fine_grained=True` → chunks sent immediately, but JSON may be invalid:
```python
# Fine-grained: faster but requires client-side error handling
try:
    parsed = json.loads(chunk.snapshot)
except json.JSONDecodeError:
    pass  # Handle gracefully
```

### Text Editor Tool (Built-in)
Claude has a built-in text editor tool schema — you only need to provide the implementation.

```python
# Schema stub (model version determines the type string)
text_edit_schema = {
    "type": "text_editor_20250124",  # claude-3-7-sonnet
    "name": "str_replace_editor"
}
# OR for claude-3-5-sonnet:
# "type": "text_editor_20241022"
```

Claude expands this stub into a full schema internally. You must implement the actual file operations (view, str_replace, create, etc.) yourself.

### Web Search Tool (Built-in)
No custom implementation needed — Claude handles everything.

```python
web_search_schema = {
    "type": "web_search_20250305",
    "name": "web_search",
    "max_uses": 5,
    # Optional: restrict to specific domains
    "allowed_domains": ["nih.gov"]
}

response = client.messages.create(
    model=model, max_tokens=1000, messages=messages,
    tools=[web_search_schema]
)
```

**Response blocks:**
- `TextBlock` — Claude's explanatory text
- `ServerToolUseBlock` — the search query used
- `WebSearchToolResultBlock` — found pages (title, URL)
- Citation blocks — specific text supporting Claude's statements

> **Note:** Your organization must enable the Web Search tool in the Anthropic console settings.

---

## 15. RAG — Retrieval Augmented Generation

### The Problem
Large documents (100–1000+ pages) can't fit in a single prompt. Even if they could, very long prompts reduce quality and increase cost.

### RAG vs. Direct Prompting

| Approach | Pros | Cons |
|---|---|---|
| **Direct (stuff everything in)** | Simple | Token limits, slower, expensive, less accurate |
| **RAG** | Focused, scalable, cheaper, faster | More complex, needs preprocessing |

### The Full RAG Flow (7 Steps)

```
Pre-processing (once):
1. Text Chunking      → split document into pieces
2. Generate Embeddings → convert chunks to number vectors
3. Normalize          → scale vectors to magnitude 1.0
4. Store in Vector DB → specialized numerical database

Per-query (real-time):
5. Embed user query   → same model as step 2
6. Similarity Search  → find closest matching embeddings
7. Prompt Assembly    → question + relevant chunks → LLM
```

### Text Chunking Strategies

#### Strategy 1 — Size-Based (most common in production)
```python
def chunk_by_char(text, chunk_size=150, chunk_overlap=20):
    chunks = []
    start_idx = 0
    while start_idx < len(text):
        end_idx = min(start_idx + chunk_size, len(text))
        chunks.append(text[start_idx:end_idx])
        start_idx = end_idx - chunk_overlap if end_idx < len(text) else len(text)
    return chunks
```
- **Pro:** Works with any document type, reliable fallback
- **Con:** Can cut words mid-sentence; overlap solves this but duplicates text

#### Strategy 2 — Structure-Based (best results for formatted docs)
```python
def chunk_by_section(text):
    return re.split(r"\n## ", text)  # Split on Markdown H2 headers
```
- **Pro:** Clean, meaningful sections
- **Con:** Requires guaranteed document formatting

#### Strategy 3 — Semantic-Based (most advanced)
- Groups sentences by semantic similarity using NLP
- Best quality, most complex to implement

#### Strategy 4 — Sentence-Based (good middle ground)
```python
def chunk_by_sentence(text, max_sentences=5, overlap=1):
    sentences = re.split(r"(?<=[.!?])\s+", text)
    chunks, start = [], 0
    while start < len(sentences):
        end = min(start + max_sentences, len(sentences))
        chunks.append(" ".join(sentences[start:end]))
        start += max_sentences - overlap
    return chunks
```

> **Rule:** No universal best strategy — depends on document type and use case.

### Text Embeddings

Embeddings are numerical representations of text meaning (lists of floats from -1 to +1).

```python
import voyageai

# Anthropic recommends Voyage AI for embeddings
client_voyage = voyageai.Client()

def generate_embedding(text, model="voyage-3-large", input_type="query"):
    result = client_voyage.embed([text], model=model, input_type=input_type)
    return result.embeddings[0]

# Also works with a list of strings for batch processing
embeddings = generate_embedding(chunks)
```

### Similarity — Cosine Distance

- **Cosine similarity** = cosine of the angle between two vectors, range −1 to 1
- **Cosine distance** = `1 - cosine_similarity` → **lower = more similar**

### Implementing the Full Pipeline

```python
# Step 1 & 2: Chunk and embed
chunks = chunk_by_section(text)
embeddings = generate_embedding(chunks)

# Step 3: Store in vector index
store = VectorIndex()
for embedding, chunk in zip(embeddings, chunks):
    store.add_vector(embedding, {"content": chunk})  # Store original text too!

# Step 4: Query time
user_embedding = generate_embedding("What did the software engineering dept do?")
results = store.search(user_embedding, k=2)  # Top 2 most relevant

# Step 5: Assemble prompt with retrieved context
for doc, distance in results:
    print(distance, doc["content"][:200])
```

### BM25 — Lexical Search

Semantic search can miss **exact term matches** (e.g., specific IDs like "INC-2023-Q4-011").

**BM25 algorithm:**
1. Tokenize query into terms
2. Count term frequency across all documents
3. Weight rare terms higher, common terms lower
4. Rank documents by weighted term frequency

### Hybrid Search Pipeline

Combine both for best accuracy:

```python
# Retriever wraps both indexes
class Retriever:
    def __init__(self, *indexes):
        self._indexes = list(indexes)
    
    def add_document(self, doc):
        for idx in self._indexes: idx.add_document(doc)
    
    def search(self, query, k=5):
        all_results = [idx.search(query, k) for idx in self._indexes]
        return reciprocal_rank_fusion(all_results)  # Merge by rank

retriever = Retriever(VectorIndex(), BM25Index())
```

**Reciprocal Rank Fusion (RRF):**
```
RRF_score(doc) = Σ [ 1 / (k + rank_i(doc)) ]
```
Documents ranked high in multiple indexes float to the top.

### Reranking Results

After hybrid retrieval, optionally run a reranking step:

```python
# Send candidate chunks + query to Claude for relevance ordering
rerank_prompt = f"""
User query: {query}

Rank these documents by relevance (most relevant first), return as JSON array of IDs:
{json.dumps(candidate_docs_with_ids)}
"""
# Use assistant pre-fill + stop sequence for structured output
```

**Trade-offs:** Better accuracy but adds latency from extra LLM call.

### Contextual Retrieval

Chunks lose document context when split. Fix this by prepending context:

```python
def add_context(chunk, source_document):
    prompt = f"""
    Here is the full document:
    <document>{source_document}</document>
    
    Here is the chunk to contextualize:
    <chunk>{chunk}</chunk>
    
    Write 1-2 sentences situating this chunk within the broader document.
    """
    messages = []
    add_user_message(messages, prompt)
    context = chat(messages)
    return context + "\n\n" + chunk   # Prepend context to chunk

# For very large docs: include only first 1-3 chunks (summary) + chunks immediately before target
```

---

## 16. Extended Thinking

### What It Does
Extended thinking gives Claude dedicated reasoning time before generating the final response. You can see the "scratchpad" reasoning process.

### When to Use It
- Only after standard prompt optimization has failed to achieve required accuracy
- Use prompt evals to determine if you actually need it
- It costs more (thinking tokens are billed) and increases latency

### Response Structure

```python
# response.content now has TWO blocks:
# [0] ThinkingBlock — {thinking: "...", signature: "..."}
# [1] TextBlock     — the final answer

thinking_text = response.content[0].thinking
final_answer = response.content[1].text
```

The **signature** is a cryptographic token that prevents tampering with thinking content.

### Implementation

```python
def chat(messages, thinking=False, thinking_budget=1024, max_tokens=2048):
    params = {
        "model": model,
        "max_tokens": max_tokens,     # MUST be > thinking_budget
        "messages": messages,
    }
    if thinking:
        params["thinking"] = {
            "type": "enabled",
            "budget": thinking_budget   # Minimum: 1024
        }
    return client.messages.create(**params)

# Usage
chat(messages, thinking=True, thinking_budget=2048, max_tokens=3000)
```

### Redacted Thinking Blocks
When thinking triggers internal safety systems, you receive an encrypted block:
```python
# {type: "redacted_thinking", data: "encrypted..."}
# Pass it back as-is in conversation history — Claude can still use it
```

> **Note:** Extended thinking is **incompatible** with message pre-filling and temperature.

---

## 17. Image & PDF Support

### Sending Images

```python
import base64

with open("image.png", "rb") as f:
    image_bytes = base64.standard_b64encode(f.read()).decode("utf-8")

messages = []
add_user_message(messages, [
    {
        "type": "image",
        "source": {
            "type": "base64",
            "media_type": "image/png",
            "data": image_bytes,
        }
    },
    {"type": "text", "text": "What do you see in this image?"}
])
```

**Limitations:**
- Max 100 images per request
- Max 5MB per image
- Single image: max 8000×8000px; multiple images: max 2000×2000px
- Token cost: `(width × height) / 750`
- Can also use `"type": "url"` with a public image URL

### Sending PDFs

```python
with open("document.pdf", "rb") as f:
    file_bytes = base64.standard_b64encode(f.read()).decode("utf-8")

add_user_message(messages, [
    {
        "type": "document",           # ← "document" not "image"
        "source": {
            "type": "base64",
            "media_type": "application/pdf",   # ← PDF media type
            "data": file_bytes,
        }
    },
    {"type": "text", "text": "Summarize this document in one sentence."}
])
```

Claude can extract: text, images, charts, tables, mixed content from PDFs.

### Prompting Tips for Images
Simple prompts often fail. Use:
- **Step-by-step analysis instructions**
- **One-shot/multi-shot examples** (alternating image + text pairs in message list)
- **Structured frameworks** with clear criteria

**Example — fire risk assessment:**
```
1. Locate the primary residence
2. Examine all trees for canopy overhang over the roof
3. Estimate percentage of roof covered (0-25%, 25-50%, 50-75%, 75%+)
4. Evaluate fire vulnerability factors
5. Assign risk rating 1-4 with one sentence per criterion
```

---

## 18. Citations

### Why Citations Matter
Citations let users verify where Claude found information, building trust and enabling source exploration.

### Implementation

```python
messages = []
add_user_message(messages, [
    {
        "type": "document",
        "source": {"type": "base64", "media_type": "application/pdf", "data": file_bytes},
        "title": "earth.pdf",               # ← Add title
        "citations": {"enabled": True}      # ← Enable citations
    },
    {"type": "text", "text": "How did Earth's atmosphere form?"}
])
```

### Citation Types

| Type | Used For | Location Info |
|---|---|---|
| `citation_page_location` | PDFs | document_index, title, start_page, end_page, cited_text |
| `citation_char_location` | Plain text | character position in text block |

### Plain Text Citations

```python
add_user_message(messages, [
    {
        "type": "document",
        "source": {
            "type": "text",
            "media_type": "text/plain",
            "data": article_text
        },
        "title": "earth_article",
        "citations": {"enabled": True}
    },
    {"type": "text", "text": "Your question here"}
])
```

---

## 19. Prompt Caching

### How It Works

**Without caching:** Every request → tokenize → embed → contextualize → generate → **discard all work** → repeat on next identical request

**With caching:** Initial request → process → **save to cache (1 hour TTL)** → follow-up requests with identical content → **retrieve cached work** → skip reprocessing

### Cache Rules

| Rule | Detail |
|---|---|
| Duration | 1 hour maximum |
| Activation | Manual — add cache breakpoints to specific blocks |
| Minimum tokens | 1024 tokens required to cache |
| Max breakpoints | 4 per request |
| Processing order | Tools → System prompt → Messages |
| Cache invalidation | Any change before breakpoint invalidates entire cache |

### Adding Cache Breakpoints

**Shorthand text (cannot cache):**
```python
{"role": "user", "content": "some text"}
```

**Longhand text (can cache):**
```python
{"role": "user", "content": [
    {
        "type": "text",
        "text": "some text",
        "cache_control": {"type": "ephemeral"}   # ← Cache breakpoint
    }
]}
```

### Caching Tool Schemas

```python
def chat(messages, tools=None, system=None):
    params = {"model": model, "max_tokens": 1000, "messages": messages}
    
    if tools:
        tools_clone = tools.copy()
        last_tool = tools_clone[-1].copy()
        last_tool["cache_control"] = {"type": "ephemeral"}   # Add to LAST tool
        tools_clone[-1] = last_tool
        params["tools"] = tools_clone
    
    if system:
        params["system"] = [{
            "type": "text",
            "text": system,
            "cache_control": {"type": "ephemeral"}
        }]
    
    return client.messages.create(**params)
```

### Reading Cache Usage

```python
usage = response.usage
# First request:  usage.cache_creation_input_tokens = 1772 (writes to cache)
# Follow-up:      usage.cache_read_input_tokens = 1772    (reads from cache)
```

---

## 20. Code Execution & Files API

### Files API

Upload files once, reference by ID in future requests.

```python
# Upload
with open("data.csv", "rb") as f:
    file_metadata = client.beta.files.upload(file=f)

file_id = file_metadata.id   # Reference this in future messages

# Download generated files
client.beta.files.download(file_id)
```

### Code Execution Tool

Claude executes Python code in isolated Docker containers — **no custom implementation needed**.

```python
code_execution_schema = {
    "type": "code_execution_20250522",
    "name": "code_execution"
}

# Docker containers have NO network access
# Use Files API to pass data in/out
```

### Combined Workflow

```python
file_metadata = upload("streaming.csv")

messages = []
add_user_message(messages, [
    {
        "type": "text",
        "text": "Run a detailed churn analysis. Include at least one detailed plot."
    },
    {
        "type": "container_upload",
        "file_id": file_metadata.id        # ← Reference uploaded file
    }
])

response = chat(messages, tools=[code_execution_schema])
```

**Response block types:**
- `TextBlock` — explanations
- `ServerToolUseBlock` — the Python code Claude wrote
- `CodeExecutionToolResultBlock` — execution output + generated file IDs

---

## 21. Model Context Protocol (MCP)

### What MCP Solves

Building integrations (e.g., GitHub, Jira, Slack) requires writing tool schemas and function implementations for every service. MCP shifts this burden to the MCP server maintainer.

```
Your App Server ←→ MCP Client ←→ MCP Server ←→ External Service
```

**MCP Server exposes:** Tools, Resources, Prompts

### The Communication Flow

1. User query → your server
2. Server asks MCP client for tool list
3. MCP client → `list_tools` request → MCP server
4. Server sends query + tools to Claude
5. Claude requests tool execution
6. Server → MCP client → `call_tool` request → MCP server
7. Results flow back: MCP server → client → server → Claude → user

### Defining Tools with MCP Python SDK

```python
from mcp import Server
from pydantic import Field

mcp = Server("my-server")

@mcp.tool(name="read_doc_contents", description="Read a document by ID")
def read_doc_contents(
    doc_id: str = Field(description="The document ID to read")
) -> str:
    if doc_id not in docs:
        raise ValueError(f"Document {doc_id} not found")
    return docs[doc_id]
```

The SDK **auto-generates the JSON schema** from function signatures and type hints — no manual schema writing.

### MCP Client Implementation

```python
class MCPClient:
    async def list_tools(self):
        result = await self.session.list_tools()
        return result.tools
    
    async def call_tool(self, tool_name: str, tool_input: dict):
        return await self.session.call_tool(tool_name, tool_input)
```

### MCP Resources

Resources expose data to clients proactively.

```python
# Direct resource (static URI)
@mcp.resource("docs://documents", mime_type="application/json")
def list_documents():
    return list(docs.keys())

# Templated resource (parameterized URI)
@mcp.resource("docs://documents/{doc_id}", mime_type="text/plain")
def get_document(doc_id: str):
    return docs.get(doc_id, "Not found")
```

**Resources vs. Tools:**
- Resources → provide data proactively (like `@mentions` in chat)
- Tools → perform actions reactively (when Claude decides to call them)

### MCP Client — Reading Resources

```python
async def read_resource(self, uri: str):
    result = await self.session.read_resource(AnyURL(uri))
    resource = result.contents[0]
    if resource.mime_type == "application/json":
        return json.loads(resource.text)
    return resource.text
```

### MCP Prompts

Pre-built, tested prompt templates that clients can call:

```python
@mcp.prompt(name="format_document", description="Format a document as markdown")
def format_document_prompt(document_id: str) -> list:
    return [UserMessage(
        f"Read document {document_id} using the read_doc_contents tool, "
        f"reformat to clean markdown, then save the changes."
    )]
```

**Client usage:**
```python
prompts = await client.list_prompts()
messages = await client.get_prompt("format_document", {"document_id": "doc123"})
# Pass messages directly to Claude
```

### MCP Inspector (Debug Tool)

```bash
mcp dev server.py    # Opens browser-based debugger
```

Connect → navigate to Tools section → select tool → input params → run → verify output

---

## 22. Claude Code

### What It Is
A terminal-based AI coding assistant that acts as a collaborative engineer — not just a code generator.

### Installation

```bash
npm install -g @anthropic-ai/claude-code
claude    # Login to Anthropic account
```

Requires Node.js. Full guide: `docs.anthropic.com`

### Built-in Capabilities
- Search, read, edit files
- Run terminal commands
- Fetch web content
- MCP client support (connect any MCP server)

### The `/init` Command

```bash
/init    # Scans codebase → creates CLAUDE.md with architecture, style, build commands
```

**CLAUDE.md types:**
- **Project** — shared with all engineers (checked into git)
- **Local** — personal notes (not committed)
- **User** — applies across all your projects

Add notes quickly: `# Always use descriptive variable names`

### Effective Prompting Workflows

**Method 1 — Three-Step:**
1. Ask Claude to read relevant files for context
2. Ask Claude to plan the solution (**no code yet**)
3. Ask Claude to implement the plan

**Method 2 — Test-Driven:**
1. Provide relevant context files
2. Ask Claude to suggest tests for the feature
3. Select tests → ask Claude to implement them
4. Ask Claude to write code until all tests pass

> Core principle: **More detailed instructions = significantly better results.** Treat Claude Code as a collaborative engineer.

### Connecting MCP Servers

```bash
claude mcp add [server-name] [startup-command]
# Example:
claude mcp add documents uv run main.py
```

Popular integrations: Sentry, Jira, Slack, Figma, Playwright, GitHub

### Parallelizing Claude Code (Git Worktrees)

```
Problem: Multiple Claude instances modifying the same files → conflicts
Solution: Git worktrees — each instance gets an isolated copy of the project
```

```bash
git worktree add ../feature-branch-name feature-branch-name
```

Each Claude instance works on its own branch, then merges back.

**Custom commands:** Create `.claude/commands/filename.md` with `$ARGUMENTS` placeholder for automation.

### Automated Debugging Pipeline

```
GitHub Action (daily) → fetch CloudWatch logs (last 24h)
→ Claude identifies & deduplicates errors
→ Claude analyzes each error → generates fixes
→ Creates pull request with proposed solutions
```

---

## 23. Computer Use

### What It Is
Claude can interact with computer interfaces visually — screenshots, clicks, typing, navigation.

### How It Works
Computer Use = **tool system** + **developer-provided computing environment** (Docker container).

```
User message → Claude sends tool_use request (mouse_move / left_click / screenshot / type)
→ Docker container executes programmatic input
→ Screenshot sent back to Claude
→ Claude decides next action
```

**Key:** Claude doesn't directly control computers. You provide a Docker environment; Claude coordinates through the tool system.

### Primary Use Cases
- Automated QA testing of web applications
- UI interaction testing across different scenarios
- Repetitive computer tasks
- Bug identification through systematic testing

Anthropic provides a reference Docker implementation — setup requires Docker + one command.

---

## 24. Agents & Workflows

### Decision Rule

```
Know exact steps? → Workflow
Don't know exact steps? → Agent
```

### Workflows

A predefined series of Claude calls with a known sequence of steps.

**Benefits:**
- Higher task completion rates
- Easier to test and evaluate
- More predictable

**Use when:** You can picture the exact flow ahead of time, or UX constrains users to specific tasks.

### Agents

Claude receives a goal + tools and dynamically plans how to achieve it.

**Benefits:**
- Flexible — handles varied, unpredictable tasks
- Can combine tools in unexpected ways
- Can ask users for clarification when needed

**Downsides:**
- Lower completion rates
- Harder to test (execution path is unpredictable)

---

### Workflow Patterns

#### Evaluator-Optimizer Pattern

```
Input → Producer (Claude) → Output → Evaluator (Claude)
                ↑                           |
                └──── Feedback (if failed) ──┘
```

Example: Image → 3D model → render → compare vs. original → fix if wrong → repeat

#### Parallelization Pattern

```
Input → [Subtask A] → |
        [Subtask B] → | Aggregator → Final Output
        [Subtask C] → |
```

Each subtask evaluates one independent aspect. Aggregator synthesizes results.

**Benefits:** Focus, modularity, scalability. Add new subtasks without rewriting existing ones.

#### Chaining Pattern

```
Input → Step 1 → Step 2 → Step 3 → ... → Output
```

Each step is a separate Claude call focusing on one subtask.

**Best for:** Complex multi-constraint prompts where Claude ignores some rules. First call generates content; second call revises and enforces constraints:

```python
# Step 1: Generate (may not follow all rules)
initial_content = chat(messages)

# Step 2: Fix specific violations
messages.append({"role": "user", "content": f"""
Revise this text:
1. Remove any sentences that identify the author as AI
2. Remove all emojis
3. Replace casual language with technical writer style

Text: {initial_content}
"""})
final_content = chat(messages)
```

#### Routing Pattern

```
Input → Router (Claude categorizes) → Pipeline A or B or C or D
```

```python
# Step 1: Categorize
category = claude_categorize(topic, categories=["Educational","Entertainment","Comedy"])

# Step 2: Route to specialized prompt
if category == "Educational":
    prompt = educational_prompt_template
elif category == "Entertainment":
    prompt = entertainment_prompt_template

# Step 3: Generate with specialized prompt
result = chat_with_prompt(prompt, topic)
```

---

### Tool Design for Agents

Provide **abstract, composable tools** rather than hyper-specialized ones.

| ❌ Specialized (avoid) | ✅ Abstract (prefer) |
|---|---|
| `refactor_code` | `bash` |
| `install_dependencies` | `read_file` |
| `run_tests` | `write_file` |
| `post_instagram` | `text_to_speech` |

Claude combines abstract tools creatively to handle tasks you never anticipated.

### Environment Inspection

After every action, Claude needs to observe the result. Always provide feedback mechanisms:

- **Computer use:** Screenshot after every click/keystroke
- **File editing:** Read file before modifying
- **Video generation:** Use Whisper for caption timing; use FFmpeg to extract frames for visual inspection
- **API calls:** Check response status and content

```
Action → Observe result → Adjust → Action → Observe → ...
```

### Recommendation

> **Always implement workflows first. Only use agents when flexibility is truly required.**
>
> Users want reliable products. Workflows have higher success rates. Agents are powerful but unpredictable. Build reliability first, then flexibility where needed.

---

## Quick Reference — Common Code Patterns

### API Setup
```python
from dotenv import load_dotenv
from anthropic import Anthropic
load_dotenv()
client = Anthropic()
model = "claude-sonnet-4-6"
```

### Basic Chat Function
```python
def chat(messages, system=None, temperature=1.0, stop_sequences=[], tools=None):
    params = {"model": model, "max_tokens": 1000, "messages": messages, "temperature": temperature}
    if system: params["system"] = system
    if stop_sequences: params["stop_sequences"] = stop_sequences
    if tools: params["tools"] = tools
    return client.messages.create(**params)
```

### Structured Output (Pre-fill + Stop Sequence)
```python
messages = []
add_user_message(messages, "Generate JSON for...")
add_assistant_message(messages, "```json")
text = chat(messages, stop_sequences=["```"])
data = json.loads(text.strip())
```

### Tool Conversation Loop
```python
def run_conversation(messages):
    while True:
        response = chat(messages, tools=all_schemas)
        add_assistant_message(messages, response)
        if response.stop_reason != "tool_use": break
        add_user_message(messages, run_tools(response))
    return messages
```

### Prompt Caching Setup
```python
# Cache last tool schema
tools_clone = tools.copy()
tools_clone[-1] = {**tools_clone[-1], "cache_control": {"type": "ephemeral"}}

# Cache system prompt
system_block = [{"type": "text", "text": system, "cache_control": {"type": "ephemeral"}}]
```
---

*Course: [Anthropic SkillJar — Building with the Claude API](https://anthropic.skilljar.com/claude-with-the-anthropic-api)*
