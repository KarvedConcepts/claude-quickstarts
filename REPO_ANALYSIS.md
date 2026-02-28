# Claude Quickstarts - Business Integration Analysis

## Overview

This repository contains 6 Anthropic reference projects demonstrating different Claude API capabilities. This document analyzes each project's patterns, business value, and import priority for workspace integration.

---

## Project Breakdown

### 1. Agents Framework (`agents/`)

**Type:** Python reference implementation (~300 lines core)

**What it demonstrates:**
- LLM agent loop: Claude calling tools in a cycle until reaching an answer
- Modular tool system (file read/write, code execution, web search, think tool)
- MCP (Model Context Protocol) server integration
- Message history compression and context window management

**Key files:**
- `agent.py` — Core agent loop with tool execution and history management
- `tools/` — Tool implementations (base, code execution, file I/O, web search, think)
- `utils/` — MCP connections and message history utilities

**Integration value:** HIGH — Foundational pattern for building any Claude-powered automation, internal tools, or assistants.

---

### 2. Customer Support Agent (`customer-support-agent/`)

**Type:** Next.js (TypeScript) full-stack application

**What it demonstrates:**
- RAG pipeline via Amazon Bedrock Knowledge Bases
- Structured JSON responses with Zod validation
- User mood detection (positive/neutral/negative/curious/frustrated/confused)
- Automatic human-agent redirect logic
- Customer inquiry categorization
- Multi-model switching at runtime
- Customizable UI with shadcn/ui + theming
- AWS Amplify deployment configuration

**Key files:**
- `app/api/chat/route.ts` — API route with RAG integration and structured output
- `app/lib/utils.ts` — Bedrock Knowledge Base retrieval
- `app/lib/customer_support_categories.json` — Category classification
- `styles/themes.js` — Theming configuration

**Integration value:** HIGH — Near-production-ready for customer-facing support. Swap knowledge base, adjust system prompt, and deploy.

---

### 3. Financial Data Analyst (`financial-data-analyst/`)

**Type:** Next.js (TypeScript) full-stack application with Edge Runtime

**What it demonstrates:**
- Claude Tool Use for generating structured chart data
- 6 chart types: line, bar, multi-bar, pie, area, stacked area
- Multi-format file upload: text, CSV, PDF (pdf.js), images
- Vision API for image analysis (base64)
- Interactive data visualization with Recharts

**Key files:**
- `app/api/finance/route.ts` — API route with tool definitions and chart processing
- `components/` — Chart rendering components
- `types/chart.ts` — Chart type definitions

**Integration value:** HIGH — The "analyze data + produce visualizations" pattern applies to any data-heavy domain (sales, ops, marketing, etc.).

---

### 4. Computer Use Demo (`computer-use-demo/`)

**Type:** Python + Docker (Streamlit UI, VNC/NoVNC)

**What it demonstrates:**
- Full desktop control via Claude's `computer_use` tool
- Mouse, keyboard, and screenshot interactions
- Docker-containerized virtual display environment
- Real-time observation via VNC

**Key files:**
- `computer_use_demo/loop.py` — Sampling loop for API calls
- `computer_use_demo/streamlit.py` — Chat interface
- `computer_use_demo/tools/` — Tool implementations

**Integration value:** MEDIUM — Useful for automating legacy desktop/GUI applications without APIs. Docker isolation pattern essential for security.

---

### 5. Browser Use Demo (`browser-use-demo/`)

**Type:** Python + Docker (Playwright, Streamlit UI)

**What it demonstrates:**
- DOM-aware browser automation (element refs instead of pixel coordinates)
- Full browser action set: navigate, read DOM, get text, search, fill forms, scroll, screenshot, execute JS
- Automatic coordinate scaling between Claude's resolution and viewport
- Docker-containerized Chromium with safety isolation

**Key files:**
- `browser_use_demo/tools/browser.py` — Main browser tool with all actions
- `browser_use_demo/loop.py` — API sampling loop
- `browser_use_demo/browser_tool_utils/` — JS utilities for DOM extraction

**Integration value:** MEDIUM-HIGH — Valuable for web scraping, automated form filling, web testing, or site monitoring.

---

### 6. Autonomous Coding Agent (`autonomous-coding/`)

**Type:** Python (Claude Code SDK)

**What it demonstrates:**
- Two-agent architecture: initializer plans features, coder implements across sessions
- Session persistence via git (progress in `feature_list.json`)
- Defense-in-depth security: OS sandbox + filesystem restrictions + bash command allowlist
- Claude Code SDK hooks (PreToolUse validation)
- MCP server integration (Puppeteer for browser automation)

**Key files:**
- `agent.py` — Agent session logic with multi-session continuity
- `client.py` — SDK client configuration with security layers
- `security.py` — Bash command allowlist with per-command validation
- `prompts/` — Initializer and coding session prompts

**Integration value:** MEDIUM — Security model and session-persistence patterns valuable for building any autonomous agent.

---

## Import Priority Matrix

### Tier 1 — Import First (Highest ROI)
| Project | Why |
|---|---|
| `agents/` | Foundation for all Claude-powered automation |
| `customer-support-agent/` | Ready-to-customize customer support with RAG |
| `financial-data-analyst/` | Tool use + visualization pattern for any data domain |

### Tier 2 — Import When Needed
| Project | Why |
|---|---|
| `browser-use-demo/` | Web automation (scraping, testing, form filling) |
| `autonomous-coding/` | Security model + session persistence for autonomous agents |
| `computer-use-demo/` | Desktop GUI automation for legacy applications |

---

## Claude API Feature Coverage

| Feature | Projects Using It |
|---|---|
| Tool Use | Financial analyst, agents, browser demo |
| Structured Output (JSON) | Customer support agent |
| RAG / Knowledge Retrieval | Customer support (Bedrock) |
| Vision / Image Analysis | Financial analyst, computer use |
| Computer Use | Computer use demo |
| Browser Automation | Browser use demo |
| MCP Servers | Agents, autonomous coding |
| Claude Code SDK | Autonomous coding |
| Streaming | Computer use demo |
| Multi-model Support | Customer support, financial analyst |

---

## Recommended Integration Approach

1. **Start with `agents/`** — Adapt the agent loop to your tech stack. Build domain-specific tools.
2. **Layer in RAG** — Use the customer support agent's Bedrock pattern (or swap in your own vector DB) to ground Claude in your company's data.
3. **Add tool use** — Follow the financial analyst's pattern to give Claude structured output capabilities (charts, reports, data extraction).
4. **Automate where needed** — Browser demo for web tasks, computer use for desktop tasks.
5. **Scale with autonomy** — Autonomous coding agent's security + persistence patterns for long-running, multi-session agents.
