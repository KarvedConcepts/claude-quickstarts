# Integration Guide: Claude Quickstarts into Karved_Concepts

This guide is designed to be taken into a separate Claude Code session on the Karved_Concepts repo. It contains everything needed to evaluate and integrate patterns from `claude-quickstarts` without needing access to the original repo.

---

## Table of Contents

1. [Agent Framework (Python)](#1-agent-framework---the-foundation)
2. [Customer Support with RAG (Next.js)](#2-customer-support-agent---rag--structured-output)
3. [Data Analyst with Visualization (Next.js)](#3-financial-data-analyst---tool-use--visualization)
4. [Autonomous Agent Security Model (Python)](#4-autonomous-coding-agent---security--session-persistence)
5. [Browser Automation (Python/Playwright)](#5-browser-use-demo---web-automation)
6. [Integration Decision Matrix](#integration-decision-matrix)

---

## 1. Agent Framework — The Foundation

**What:** A ~300-line Python agent loop that manages Claude API calls, tool execution, message history, and MCP server connections.

**When to use:** Any time you need Claude to take actions (call APIs, read files, search the web, execute code) in an automated loop.

### Files to Copy (7 files, ~600 lines total)

```
agents/
├── __init__.py
├── agent.py                    # Core agent loop (174 lines)
├── tools/
│   ├── __init__.py
│   ├── base.py                 # Tool base class (28 lines)
│   ├── think.py                # Reasoning tool (33 lines)
│   ├── file_tools.py           # File read/write (278 lines)
│   ├── code_execution.py       # Code execution server tool (15 lines)
│   └── web_search.py           # Web search server tool (38 lines)
└── utils/
    ├── history_util.py          # Message history + truncation (125 lines)
    ├── connections.py           # MCP server connections (151 lines)
    └── tool_util.py             # Tool execution helper
```

### Dependencies

```
pip install anthropic mcp
```

### Core Pattern — The Agent Loop

```python
# This is the heart of the pattern. Claude calls tools in a loop
# until it produces a final text response with no tool calls.

async def _agent_loop(self, user_input: str):
    await self.history.add_message("user", user_input, None)
    tool_dict = {tool.name: tool for tool in self.tools}

    while True:
        self.history.truncate()  # Prevent context overflow
        params = self._prepare_message_params()
        response = self.client.messages.create(**params)

        tool_calls = [b for b in response.content if b.type == "tool_use"]
        await self.history.add_message("assistant", response.content, response.usage)

        if tool_calls:
            tool_results = await execute_tools(tool_calls, tool_dict)
            await self.history.add_message("user", tool_results)
        else:
            return response  # Done — no more tool calls
```

### How to Create Custom Tools

```python
from agents.tools.base import Tool

class MyBusinessTool(Tool):
    def __init__(self):
        super().__init__(
            name="lookup_customer",
            description="Look up customer info by email or ID",
            input_schema={
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Email or customer ID"}
                },
                "required": ["query"],
            },
        )

    async def execute(self, query: str) -> str:
        # Your business logic here
        customer = await db.find_customer(query)
        return json.dumps(customer)
```

### What to Customize vs Use As-Is

| Component | Customize? | Notes |
|---|---|---|
| `agent.py` | Minimal | Change model, system prompt, temperature |
| `tools/base.py` | No | Use as-is, extend with subclasses |
| `tools/think.py` | No | Use as-is, improves reasoning quality |
| `utils/history_util.py` | No | Handles truncation + caching automatically |
| `utils/connections.py` | No | MCP support works out of the box |
| Custom tools | YES | Build your domain-specific tools |

---

## 2. Customer Support Agent — RAG + Structured Output

**What:** A Next.js app that retrieves context from a knowledge base (Amazon Bedrock), sends it to Claude with a structured output prompt, and gets back typed JSON with mood detection, categories, and redirect logic.

**When to use:** Customer-facing chatbots, internal helpdesks, any Q&A system grounded in your documents.

### Files to Copy (4 key files)

```
customer-support-agent/
├── app/api/chat/route.ts        # API route (302 lines) — THE KEY FILE
├── app/lib/utils.ts             # RAG retrieval (95 lines)
├── app/lib/customer_support_categories.json  # Category definitions
└── components/ChatArea.tsx      # Chat UI with model/KB switching
```

### Dependencies

```json
{
  "@anthropic-ai/sdk": "^0.27.1",
  "@aws-sdk/client-bedrock-agent-runtime": "^3.621.0",
  "zod": "latest"
}
```

### Core Pattern — Structured JSON Output with Zod Validation

```typescript
import { z } from "zod";

// Define exactly what Claude must return
const responseSchema = z.object({
  response: z.string(),
  thinking: z.string(),
  user_mood: z.enum(["positive", "neutral", "negative", "curious", "frustrated", "confused"]),
  suggested_questions: z.array(z.string()),
  debug: z.object({ context_used: z.boolean() }),
  matched_categories: z.array(z.string()).optional(),
  redirect_to_agent: z.object({
    should_redirect: z.boolean(),
    reason: z.string().optional(),
  }).optional(),
});

// Force Claude to output JSON by prefilling the assistant turn
anthropicMessages.push({ role: "assistant", content: "{" });

const response = await anthropic.messages.create({
  model, max_tokens: 1000, messages: anthropicMessages,
  system: systemPrompt, temperature: 0.3,
});

// Prepend the "{" we used as prefill
const textContent = "{" + response.content
  .filter((block) => block.type === "text")
  .map((block) => block.text)
  .join(" ");

const validated = responseSchema.parse(JSON.parse(textContent));
```

### Core Pattern — RAG Retrieval (Bedrock)

```typescript
import { BedrockAgentRuntimeClient, RetrieveCommand } from "@aws-sdk/client-bedrock-agent-runtime";

const client = new BedrockAgentRuntimeClient({
  region: "us-east-1",
  credentials: {
    accessKeyId: process.env.BAWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.BAWS_SECRET_ACCESS_KEY!,
  },
});

async function retrieveContext(query: string, knowledgeBaseId: string, n = 3) {
  const response = await client.send(new RetrieveCommand({
    knowledgeBaseId,
    retrievalQuery: { text: query },
    retrievalConfiguration: { vectorSearchConfiguration: { numberOfResults: n } },
  }));

  const context = response.retrievalResults
    .filter(r => r.content?.text)
    .map(r => r.content.text)
    .join("\n\n");

  return { context, isRagWorking: true };
}
```

### What to Customize

| Component | What to Change |
|---|---|
| System prompt | Replace "Anthropic" with your company name, adjust tone |
| Categories JSON | Define your own support categories |
| Knowledge Base ID | Point to your own Bedrock KB (or swap for Pinecone/Weaviate/etc) |
| Zod schema | Add/remove fields for your use case |
| Model list | Update to latest Claude models |
| RAG provider | Bedrock is optional — swap for any vector DB |

---

## 3. Financial Data Analyst — Tool Use + Visualization

**What:** Claude analyzes uploaded data (text, CSV, PDF, images) and calls a `generate_graph_data` tool to produce structured chart configurations that render as interactive Recharts visualizations.

**When to use:** Any data analysis, reporting, or dashboard feature. The pattern works for any domain — not just finance.

### Files to Copy (2 key files + chart components)

```
financial-data-analyst/
├── app/api/finance/route.ts     # API route with tool definition (465 lines) — KEY FILE
├── types/chart.ts               # Chart type definitions
└── components/                  # Chart rendering components (Recharts)
```

### Dependencies

```json
{
  "@anthropic-ai/sdk": "^0.29.0",
  "recharts": "^2.13.0",
  "pdfjs-dist": "^4.7.76"
}
```

### Core Pattern — Tool Use for Structured Data Output

```typescript
// Define what Claude can generate
const tools = [{
  name: "generate_graph_data",
  description: "Generate structured JSON data for creating charts and graphs.",
  input_schema: {
    type: "object",
    properties: {
      chartType: {
        type: "string",
        enum: ["bar", "multiBar", "line", "pie", "area", "stackedArea"],
      },
      config: {
        type: "object",
        properties: {
          title: { type: "string" },
          description: { type: "string" },
          trend: {
            type: "object",
            properties: {
              percentage: { type: "number" },
              direction: { type: "string", enum: ["up", "down"] },
            },
          },
          xAxisKey: { type: "string" },
        },
        required: ["title", "description"],
      },
      data: {
        type: "array",
        items: { type: "object", additionalProperties: true },
      },
      chartConfig: {
        type: "object",
        additionalProperties: {
          type: "object",
          properties: { label: { type: "string" } },
          required: ["label"],
        },
      },
    },
    required: ["chartType", "config", "data", "chartConfig"],
  },
}];

// Claude decides when to use the tool
const response = await anthropic.messages.create({
  model, max_tokens: 4096, temperature: 0.7,
  tools, tool_choice: { type: "auto" },
  messages: anthropicMessages,
  system: `You are a data visualization expert...`,
});

// Extract tool output
const toolUse = response.content.find(c => c.type === "tool_use");
const chartData = toolUse?.input; // Ready to pass to Recharts
```

### Core Pattern — Multi-Format File Upload to Claude

```typescript
// Text/CSV files → decode base64 and send as text
if (isText) {
  const textContent = decodeURIComponent(escape(atob(base64)));
  messages[last] = {
    role: "user",
    content: [
      { type: "text", text: `File contents of ${fileName}:\n\n${textContent}` },
      { type: "text", text: userMessage },
    ],
  };
}

// Images → send as base64 via Vision API
if (mediaType.startsWith("image/")) {
  messages[last] = {
    role: "user",
    content: [
      { type: "image", source: { type: "base64", media_type: mediaType, data: base64 } },
      { type: "text", text: userMessage },
    ],
  };
}
```

### What to Customize

| Component | What to Change |
|---|---|
| Tool schema | Define your own output structures (reports, tables, etc.) |
| Chart types | Add/replace with your visualization needs |
| System prompt | Swap "financial" for your domain |
| File handling | Add more formats as needed |

---

## 4. Autonomous Coding Agent — Security + Session Persistence

**What:** A defense-in-depth security model for autonomous agents, plus a two-agent architecture for multi-session work with git-based persistence.

**When to use:** Any time you're building agents that run autonomously (unattended), especially ones that execute code or commands.

### Files to Copy (4 key files)

```
autonomous-coding/
├── security.py                  # Bash allowlist + validation (360 lines) — KEY FILE
├── client.py                    # SDK client with security layers (123 lines)
├── agent.py                     # Multi-session agent loop (207 lines)
└── prompts/
    ├── initializer_prompt.md    # First-session planning prompt (107 lines)
    └── coding_prompt.md         # Continuation-session prompt (198 lines)
```

### Dependencies

```
pip install claude-code-sdk
```

### Core Pattern — Bash Command Allowlist

```python
ALLOWED_COMMANDS = {
    "ls", "cat", "head", "tail", "wc", "grep",  # Read-only
    "npm", "node",                                 # Dev tools
    "git",                                         # Version control
    "ps", "lsof", "sleep", "pkill",              # Process mgmt
}

async def bash_security_hook(input_data, tool_use_id=None, context=None):
    """Pre-tool-use hook that validates bash commands."""
    if input_data.get("tool_name") != "Bash":
        return {}

    command = input_data.get("tool_input", {}).get("command", "")
    commands = extract_commands(command)  # Handles pipes, &&, ||, ;

    for cmd in commands:
        if cmd not in ALLOWED_COMMANDS:
            return {"decision": "block", "reason": f"Command '{cmd}' not allowed"}

    return {}  # Allow
```

### Core Pattern — Defense-in-Depth Security Layers

```python
# Layer 1: OS-level sandbox
security_settings = {
    "sandbox": {"enabled": True, "autoAllowBashIfSandboxed": True},
    # Layer 2: Filesystem restrictions
    "permissions": {
        "defaultMode": "acceptEdits",
        "allow": [
            "Read(./**)", "Write(./**)", "Edit(./**)",  # Project dir only
            "Bash(*)",  # Allowed by sandbox + validated by hook
        ],
    },
}

# Layer 3: Command validation hook
client = ClaudeSDKClient(options=ClaudeCodeOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(matcher="Bash", hooks=[bash_security_hook]),
        ],
    },
))
```

### Core Pattern — Two-Agent Multi-Session Architecture

**Session 1 (Initializer):**
- Reads specification
- Creates feature/task list (source of truth)
- Sets up project structure
- Initializes git

**Session 2+ (Worker):**
- Fresh context window each session
- Reads progress file to orient itself
- Picks next task from the list
- Implements, tests, marks complete
- Commits progress before session ends

This pattern works for any long-running task — not just coding. Replace "features" with whatever work items your domain needs.

### What to Customize

| Component | What to Change |
|---|---|
| `ALLOWED_COMMANDS` | Add commands your agents need |
| Prompt templates | Rewrite for your domain |
| `feature_list.json` | Replace with your task/work-item schema |
| Security layers | Adjust permissions for your environment |

---

## 5. Browser Use Demo — Web Automation

**What:** A Playwright-based browser tool that gives Claude DOM-aware web automation — more reliable than coordinate-based clicking because it uses element references.

**When to use:** Web scraping, automated form filling, web testing, site monitoring, competitor analysis.

### Key Files (containerized — designed for Docker)

```
browser-use-demo/
├── browser_use_demo/
│   ├── tools/browser.py          # 23+ browser actions
│   ├── tools/coordinate_scaling.py # Resolution mapping
│   ├── loop.py                    # API sampling loop
│   └── browser_tool_utils/        # JS utilities for DOM extraction
├── Dockerfile
└── docker-compose.yml
```

### Dependencies

```
pip install playwright anthropic streamlit
playwright install chromium
```

### Key Concept — DOM-Based Element Targeting

Instead of clicking at pixel coordinates (brittle), elements get `ref` identifiers from the DOM:

```
# Claude sees structured DOM:
<button ref="42">Submit</button>
<input ref="43" type="email" placeholder="Enter email">

# Claude uses refs for reliable targeting:
{"action": "left_click", "ref": "42"}
{"action": "form_input", "ref": "43", "value": "user@example.com"}
```

### Available Actions (23 total)

**Web-specific (unique to browser tool):**
- `navigate` — Go to URL or use history (back/forward)
- `read_page` — Get DOM tree with element refs
- `get_page_text` — Extract all text from page
- `find` — Search and highlight text on page
- `form_input` — Set form element value directly by ref
- `scroll_to` — Scroll specific element into view
- `execute_js` — Run JavaScript in page context

**Shared with general interaction:**
- `left_click`, `right_click`, `double_click`, `hover` — via ref or coordinate
- `type`, `key` — keyboard input
- `screenshot`, `zoom` — visual capture

### What to Customize

| Component | What to Change |
|---|---|
| Allowed URLs | Add domain allowlist for safety |
| Docker setup | Adjust for your infrastructure |
| Actions | Add custom actions for your workflows |
| Resolution | Change viewport size if needed |

---

## Integration Decision Matrix

Use this to decide what to integrate based on your Karved_Concepts repo's needs:

### If your repo has a **Next.js/React frontend**:
- Integrate the **Financial Analyst** tool-use + chart pattern
- Integrate the **Customer Support** structured output + RAG pattern
- Both use the same stack (Next.js 14, shadcn/ui, Tailwind)

### If your repo has a **Python backend or scripts**:
- Integrate the **Agent Framework** as your automation backbone
- Add the **Security Model** if agents run unattended
- Use MCP connections if you need to integrate external tool servers

### If you need to **automate web interactions**:
- Integrate the **Browser Use Demo** patterns
- Run in Docker for safety isolation

### If you need **multi-session autonomous work**:
- Integrate the **Two-Agent Architecture** from autonomous-coding
- Task list + git persistence + fresh context per session

### If you need **knowledge-grounded Q&A**:
- Integrate the **RAG pattern** from customer-support-agent
- Swap Bedrock for your preferred vector DB if needed

---

## Quick Start Checklist

When you open this in your Karved_Concepts session:

1. **Assess your tech stack** — What languages/frameworks does Karved_Concepts use?
2. **Pick 1-2 patterns** from above that match your immediate needs
3. **Copy the key files** listed for each pattern
4. **Install dependencies** listed for each pattern
5. **Customize** the items marked in each "What to Customize" table
6. **Test incrementally** — get the basic pattern working before adding complexity

The patterns are designed to be composable. You can start with the agent loop, add tools, add RAG, add security — each layer is independent.
