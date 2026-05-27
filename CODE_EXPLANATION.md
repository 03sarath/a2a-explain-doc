# Competitive Intelligence System — Complete Code Explanation

> This document explains every file in the project in plain language.
> It is written as a teaching script — you can read it top to bottom while walking students through the code.

---

## What Does This Project Do?

This system automatically generates a **Competitive Intelligence Report** about any company you name.

You type "Analyze OpenAI" and the system:
1. Searches the web for recent news about OpenAI
2. Analyzes public sentiment (what customers and analysts are saying)
3. Finds their pricing plans
4. Synthesizes everything into a structured executive report

It does all of this using **five AI agents** that talk to each other over the network — not a single monolithic AI, but a team of specialized agents working in sequence.

---

## Understanding JSON-RPC

Before diving into A2A, it helps to understand **JSON-RPC** — the communication protocol the agents use to talk to each other.

### What is RPC?

**RPC (Remote Procedure Call)** means calling a function on another machine as if it were a local function call. Instead of:
```python
result = market_scanner.scan("OpenAI")   # local call
```
you send a message over HTTP to a remote server that runs the function and sends back the result.

### What is JSON-RPC?

**JSON-RPC** is a standard that defines exactly how to format those messages using JSON. Every request looks like this:

```json
{
  "jsonrpc": "2.0",
  "method": "tasks/send",
  "id": "abc-123",
  "params": {
    "message": {
      "role": "user",
      "parts": [{ "text": "Analyze OpenAI" }]
    }
  }
}
```

And every response looks like this:
```json
{
  "jsonrpc": "2.0",
  "id": "abc-123",
  "result": {
    "status": { "state": "completed" },
    "artifacts": [{ "parts": [{ "text": "**MARKET SCAN**\n..." }] }]
  }
}
```

Key fields:
- `"jsonrpc": "2.0"` — always present, identifies the protocol version
- `"method"` — the name of the remote function to call (e.g. `tasks/send`, `tasks/get`)
- `"id"` — a unique ID so you can match responses to requests (important for async)
- `"params"` — the arguments to pass to the function
- `"result"` — the return value (present on success)
- `"error"` — error details (present on failure, instead of `result`)

### Why JSON-RPC instead of plain REST?

| REST | JSON-RPC |
|---|---|
| Multiple URLs (`/tasks`, `/tasks/{id}`, `/results`) | Single URL (`/`), method name in body |
| HTTP verbs carry meaning (GET, POST, DELETE) | Everything is POST, action is in `"method"` |
| Good for resource-based APIs (CRUD) | Good for action-based APIs (do this task) |

Agents are action-oriented ("run this task", "get task status") not resource-oriented, so JSON-RPC is a natural fit.

### How it works in this project

The host agent sends a JSON-RPC request to each specialist's `POST /` endpoint. The specialist processes the task (runs Gemini + Google Search) and returns a JSON-RPC response. The `a2a-sdk` library handles all the formatting — you never write raw JSON-RPC manually. The A2A protocol is built on top of JSON-RPC and adds the Agent Card discovery mechanism on top.

---

## The Core Concept: Agent-to-Agent (A2A) Communication

Traditional AI apps have one model doing everything. This project uses **A2A (Agent-to-Agent) protocol** — a Google open standard where:

- Each agent is a **separate microservice** running in the cloud
- Agents communicate via **HTTP + JSON-RPC**
- Each agent publishes an **Agent Card** (like a business card) at a well-known URL so other agents can discover what it does
- The host agent **orchestrates** the specialists without knowing their internal implementation

Think of it like a company: the manager (host) delegates work to specialists (market scanner, sentiment analyzer, etc.), collects their reports, and hands the final output to you.

---

## Project Structure

```
competitive-intel-a2a/
│
├── agents/
│   ├── host/                  ← The orchestrator (manager)
│   │   ├── host/
│   │   │   ├── agent.py       ← Defines the SequentialAgent
│   │   │   └── __init__.py
│   │   ├── main.py            ← Starts the FastAPI web server
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   │
│   ├── market_scanner/        ← Specialist: finds recent news
│   │   ├── agent.py
│   │   ├── main.py
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   │
│   ├── sentiment_analyzer/    ← Specialist: analyzes public opinion
│   │   ├── agent.py
│   │   ├── main.py
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   │
│   ├── pricing_intelligence/  ← Specialist: finds pricing plans
│   │   ├── agent.py
│   │   ├── main.py
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   │
│   └── report_generator/      ← Specialist: writes the final report
│       ├── agent.py
│       ├── main.py
│       ├── requirements.txt
│       └── Dockerfile
│
├── ui/
│   └── index.html             ← Browser-based front end
│
└── deploy.sh                  ← One-command deployment to Google Cloud
```

---

## How the Agents Work Together

```
User types: "Analyze OpenAI"
        │
        ▼
  ┌─────────────────────────────────┐
  │        HOST AGENT               │
  │   (competitive_intel_host)      │
  │   Runs agents 1→2→3→4 in order  │
  └────────────┬────────────────────┘
               │
       ┌───────┼───────────────────────┐
       │       │                       │
       ▼       ▼                       ▼
  [1] Market  [2] Sentiment  [3] Pricing  [4] Report
  Scanner     Analyzer       Intelligence  Generator
  (searches   (reads         (finds        (reads all
   news)       reviews)       pricing)      above, writes
                                            final report)
```

Each specialist's output is saved to the **shared session**. So when the Report Generator runs last, it can see everything the previous three agents wrote — just like a shared Google Doc that everyone edits in sequence.

---

## Architecture Block Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              USER BROWSER                                        │
│                           ui/index.html                                          │
│                                                                                  │
│   1. POST /apps/host/users/{uid}/sessions  →  create session                    │
│   2. POST /run_sse  →  stream results back via Server-Sent Events               │
└──────────────────────────────────┬──────────────────────────────────────────────┘
                                   │ HTTPS
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        HOST AGENT  (Cloud Run)                                   │
│                   agents/host/  —  FastAPI + Uvicorn                             │
│                                                                                  │
│   get_fast_api_app()  →  exposes /list-apps, /run_sse, /sessions endpoints      │
│   CORSMiddleware      →  allows browser requests from any origin                 │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────┐                   │
│   │              SequentialAgent (competitive_intel_host)    │                   │
│   │                                                          │                   │
│   │   RemoteA2aAgent        RemoteA2aAgent                  │                   │
│   │   (scan_market)    →    (analyze_sentiment)    →  ...   │                   │
│   │   fetches AgentCard     fetches AgentCard               │                   │
│   │   sends task via HTTP   sends task via HTTP             │                   │
│   └─────────────────────────────────────────────────────────┘                   │
│                                                                                  │
│   DatabaseSessionService  →  stores full conversation history                    │
└──────┬──────────┬──────────┬──────────┬──────────────────────────────────────────┘
       │ HTTP     │ HTTP     │ HTTP     │ HTTP          │
       │ JSON-RPC │ JSON-RPC │ JSON-RPC │ JSON-RPC      │ PostgreSQL
       ▼          ▼          ▼          ▼               ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  ┌─────────────────────────┐
│ MARKET   │ │SENTIMENT │ │ PRICING  │ │ REPORT   │  │   NEON PostgreSQL        │
│ SCANNER  │ │ANALYZER  │ │ INTEL    │ │GENERATOR │  │   (Cloud DB)             │
│          │ │          │ │          │ │          │  │                          │
│Cloud Run │ │Cloud Run │ │Cloud Run │ │Cloud Run │  │  host sessions           │
│          │ │          │ │          │ │          │  │  market_scanner sessions  │
│A2AStarlet│ │A2AStarlet│ │A2AStarlet│ │A2AStarlet│  │  sentiment sessions      │
│teApp     │ │teApp     │ │teApp     │ │teApp     │  │  pricing sessions        │
│          │ │          │ │          │ │          │  │  report sessions         │
│AgentCard │ │AgentCard │ │AgentCard │ │AgentCard │  └─────────────────────────┘
│at        │ │at        │ │at        │ │at        │           ▲
│/.well-   │ │/.well-   │ │/.well-   │ │/.well-   │           │
│known/    │ │known/    │ │known/    │ │known/    │    All 5 agents write
│agent.json│ │agent.json│ │agent.json│ │agent.json│    sessions here
└────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
     │             │             │             │
     ▼             ▼             ▼             │
┌─────────┐  ┌─────────┐  ┌─────────┐        │  (no search)
│ Gemini  │  │ Gemini  │  │ Gemini  │        ▼
│2.5 Flash│  │2.5 Flash│  │2.5 Flash│   ┌─────────┐
│         │  │         │  │         │   │ Gemini  │
│+        │  │+        │  │+        │   │2.5 Flash│
│ Google  │  │ Google  │  │ Google  │   │         │
│ Search  │  │ Search  │  │ Search  │   │ Reads   │
│ Tool    │  │ Tool    │  │ Tool    │   │ session │
│         │  │         │  │         │   │ history │
│ 3 web   │  │ 4 web   │  │ 3 web   │   │ only    │
│searches │  │searches │  │searches │   └─────────┘
└─────────┘  └─────────┘  └─────────┘


SECRET MANAGER (GCP)
┌──────────────────────────────────────┐
│  GOOGLE_API_KEY   → all 5 agents     │
│  SESSION_SERVICE_URI → all 5 agents  │
└──────────────────────────────────────┘

COMPLETE REQUEST FLOW
─────────────────────
① Browser creates session  →  Host saves to PostgreSQL
② Browser sends query via /run_sse
③ Host SequentialAgent starts:
   a. Calls Market Scanner  →  3 Google searches  →  MARKET SCAN output  →  saved to session
   b. Calls Sentiment Analyzer (with full session)  →  4 searches  →  SENTIMENT ANALYSIS  →  saved
   c. Calls Pricing Intelligence (with full session)  →  3 searches  →  PRICING INTEL  →  saved
   d. Calls Report Generator (with full session)  →  reads all 3 sections  →  writes final report
④ Host streams final report back to browser via SSE
⑤ Browser renders report as formatted Markdown
```

---

## FILE-BY-FILE CODE EXPLANATION

---

### 1. `agents/host/host/agent.py` — The Orchestrator

This is the brain of the system. It defines **which agents to call and in what order**.

```python
from google.adk.agents import SequentialAgent
from google.adk.agents.remote_a2a_agent import RemoteA2aAgent
from a2a.utils.constants import AGENT_CARD_WELL_KNOWN_PATH
```

**What these imports do:**
- `SequentialAgent` — an ADK agent type that runs sub-agents one after another
- `RemoteA2aAgent` — a proxy object that represents a remote agent running on another server
- `AGENT_CARD_WELL_KNOWN_PATH` — the standard URL path `/.well-known/agent.json` where every A2A agent publishes its Agent Card

```python
_REQUIRED = ["MARKET_SCANNER_URL", "SENTIMENT_ANALYZER_URL", "PRICING_INTEL_URL", "REPORT_GENERATOR_URL"]
_missing = [v for v in _REQUIRED if not os.environ.get(v)]
if _missing:
    sys.exit(f"Missing required environment variables: {', '.join(_missing)}")
```

**What this does:** At startup, it checks that all four specialist agent URLs are provided as environment variables. If any are missing, the process exits immediately with a clear error message instead of failing silently later. This is a "fail fast" pattern — better to crash at startup with a clear message than to fail mysteriously at runtime.

```python
remote_market_scanner = RemoteA2aAgent(
    name="scan_market",
    description="Scans the web for recent competitor news and strategic moves",
    agent_card=f"{os.environ['MARKET_SCANNER_URL']}{AGENT_CARD_WELL_KNOWN_PATH}",
)
```

**What this does:** Creates a **proxy object** for the remote Market Scanner agent. The host doesn't import the market scanner's code — it just knows the URL where the market scanner is running. When `scan_market` is called, ADK fetches the Agent Card from that URL to understand the agent's capabilities, then sends the request over HTTP.

This is the power of A2A: agents are **location-independent**. You could swap the market scanner for a completely different implementation deployed anywhere, and the host would not need any code changes.

The same pattern repeats for `remote_sentiment_analyzer`, `remote_pricing_intelligence`, and `remote_report_generator`.

```python
root_agent = SequentialAgent(
    name="competitive_intel_host",
    description="Orchestrates market scan, sentiment, pricing, and report generation",
    sub_agents=[
        remote_market_scanner,
        remote_sentiment_analyzer,
        remote_pricing_intelligence,
        remote_report_generator,
    ],
)
```

**What this does:** Creates the final orchestrator. `SequentialAgent` is an ADK built-in that runs its `sub_agents` list **strictly in order, one at a time**. Each agent's output is automatically appended to the shared session context, so each subsequent agent can read what the previous ones wrote. This is how the Report Generator "knows" what the Market Scanner found — it's all in the shared conversation history.

---

### 2. `agents/host/main.py` — The Host Web Server

```python
from fastapi.middleware.cors import CORSMiddleware
from google.adk.cli.fast_api import get_fast_api_app

app = get_fast_api_app(
    agents_dir=os.path.dirname(os.path.abspath(__file__)),
    session_service_uri=os.environ.get("SESSION_SERVICE_URI"),
    web=False,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**What `get_fast_api_app` does:** This is an ADK utility that automatically creates a full REST API server for your agent. It scans `agents_dir` for agent modules (finds the `host/` folder with `agent.py`), loads them, and registers HTTP endpoints including:
- `GET /list-apps` — lists available agents
- `POST /apps/{app}/users/{user}/sessions` — creates a session
- `POST /run_sse` — runs the agent and streams results back as Server-Sent Events

**What `session_service_uri` does:** If a PostgreSQL database URL is provided, sessions are stored in the database (persistent across server restarts). If not provided, ADK uses in-memory storage.

**What `web=False` does:** Disables the ADK browser-based dev UI. We use our own custom UI instead.

**What CORS middleware does:** Browsers block JavaScript from calling APIs on a different domain by default (security feature called Same-Origin Policy). Adding CORS middleware tells the browser "it's OK, allow requests from any origin." This is required because our HTML file and the Cloud Run API are on different domains.

---

### 3. `agents/market_scanner/agent.py` — Market Scanner Specialist

```python
from google.adk.agents import Agent
from google.adk.tools import google_search

root_agent = Agent(
    model="gemini-2.5-flash",
    name="market_scanner",
    instruction="""
You are a market intelligence specialist. Scan the web for recent developments
about the competitor mentioned in the conversation.

Steps:
1. Read the conversation to identify the competitor name.
2. Search for the competitor name + "news 2025", then + "product launch", then + "announcement".
3. Focus on the last 30-60 days only.
...
Return a structured section titled **MARKET SCAN** with: ...
""",
    tools=[google_search],
)
```

**What `Agent` does:** This is the core ADK class for a single LLM-powered agent. It takes:
- `model` — which Gemini model to use (`gemini-2.5-flash` is fast and capable)
- `name` — identifier used in logs and session history
- `instruction` — the system prompt that shapes the agent's behavior and output format
- `tools` — a list of tools the agent can use during its execution

**What `google_search` does:** This is a built-in ADK tool that lets the agent perform real Google web searches. When the Gemini model decides it needs to search for something, it calls this tool, gets back search results, and incorporates them into its response. The agent can call this tool multiple times — in this case, it searches three different queries about the competitor.

**Key design decision — structured output format:** The instruction tells the agent to always return a section titled `**MARKET SCAN**`. This is important because the Report Generator agent will later read the entire conversation history and look for these labeled sections to synthesize the final report.

---

### 4. `agents/sentiment_analyzer/agent.py` — Sentiment Analyst

```python
root_agent = Agent(
    model="gemini-2.5-flash",
    name="sentiment_analyzer",
    instruction="""
You are a brand sentiment analyst...
Search for: reviews, complaints, customer feedback, analyst opinion...
Return **SENTIMENT ANALYSIS** with:
- Overall Sentiment score (1-10)
- Top Positive Themes
- Top Negative Themes
- Analyst & Media Perception
""",
    tools=[google_search],
)
```

**Same structure as Market Scanner but different expertise.** This agent focuses on:
- How customers feel about the competitor (reviews, complaints)
- What industry analysts are saying
- Media coverage tone

It also has access to `google_search` and performs four different searches. It writes its output as `**SENTIMENT ANALYSIS**` so it can be identified in the session history.

**Important:** The instruction says "Read the conversation to identify the competitor name." This is how context flows — the host sends the user's original message ("Analyze OpenAI") as the starting context, and each agent reads it to know what to analyze.

---

### 5. `agents/pricing_intelligence/agent.py` — Pricing Analyst

```python
root_agent = Agent(
    model="gemini-2.5-flash",
    name="pricing_intelligence",
    instruction="""
You are a pricing intelligence analyst...
Search for: pricing 2025, plans cost, pricing change...
Return **PRICING INTELLIGENCE** with:
- Pricing Tiers (table: Tier | Price | Key Features)
- Recent Pricing Changes
- Free Trial / Freemium Availability
- Pricing Strategy Assessment
""",
    tools=[google_search],
)
```

**Same pattern, different specialty.** This agent finds:
- All pricing tiers and what's included in each
- Recent price increases or decreases
- Whether a free tier exists
- The overall pricing strategy (premium, freemium, value-based, etc.)

The instruction asks for a **table format** for pricing tiers, which makes the final report more readable.

---

### 6. `agents/report_generator/agent.py` — Report Writer

```python
root_agent = Agent(
    model="gemini-2.5-flash",
    name="report_generator",
    instruction="""
You are a senior business analyst. Using the MARKET SCAN, SENTIMENT ANALYSIS,
and PRICING INTELLIGENCE sections already gathered in this conversation,
write a final executive intelligence brief.

Do NOT perform any new searches. Synthesize only from what is already in the conversation above.
...
""",
    tools=[],   # ← No tools! It only reads and synthesizes.
)
```

**Key differences from the other agents:**

1. `tools=[]` — This agent has **no tools**. It cannot search the web. It can only read the conversation history and write a synthesis. This is intentional — we don't want it going off on its own searches at the end.

2. The instruction explicitly says "Do NOT perform any new searches." This is a safety instruction to prevent the agent from hallucinating or going beyond the data already collected.

3. The output format is a structured executive brief with: Executive Summary, Strategic Threats, Opportunities, Recommended Actions, and a Threat Level rating.

**Why this works:** By the time this agent runs, the session history contains the user's original query plus the full output from three previous agents. The Report Generator reads all of that as context and synthesizes it into the final report.

---

### 7. `agents/market_scanner/main.py` — How Specialists Become A2A Servers

All four specialist agents share the same `main.py` pattern. Here is the full explanation using the Market Scanner as the example.

```python
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService, DatabaseSessionService
from google.adk.artifacts import InMemoryArtifactService
from google.adk.memory.in_memory_memory_service import InMemoryMemoryService
from google.adk.a2a.executor.a2a_agent_executor import A2aAgentExecutor, A2aAgentExecutorConfig
from a2a.server.apps import A2AStarletteApplication
from a2a.server.request_handlers import DefaultRequestHandler
from a2a.server.tasks import InMemoryTaskStore
from a2a.types import AgentCard, AgentCapabilities, AgentSkill
```

**What all these imports are:**
- `Runner` — ADK's execution engine. It wires together the agent, session storage, memory, and artifacts into a runnable unit.
- `InMemorySessionService` / `DatabaseSessionService` — session storage. If `SESSION_SERVICE_URI` is set, sessions are stored in PostgreSQL (survives Cloud Run restarts). If not set, falls back to RAM.
- `InMemoryArtifactService` — stores file attachments in RAM. Our agents don't produce artifacts so persistence is not needed.
- `InMemoryMemoryService` — stores long-term cross-session memory in RAM. Our agents don't use memory across sessions so persistence is not needed.
- `A2aAgentExecutor` — bridges ADK's Runner with the A2A protocol. It takes an ADK agent and makes it speak A2A's communication standard.
- `A2AStarletteApplication` — the actual HTTP server that handles incoming A2A requests
- `AgentCard` — a data object that describes the agent (name, URL, skills, capabilities)

> **Note on `a2a-sdk` version:** We use `a2a-sdk==0.2.16` (not 0.3.x). `google-adk==1.9.0` requires this exact version range. `TransportProtocol` was added in 0.3.x and does NOT exist in 0.2.16 — importing it causes a startup crash. The Agent Card works fine without specifying a transport protocol.

```python
SERVICE_URL = os.environ.get("SERVICE_URL", "http://localhost:8080")

agent_card = AgentCard(
    name="Market Scanner Agent",
    url=SERVICE_URL,
    description="Scans the web for recent competitor news and strategic moves",
    version="1.0",
    capabilities=AgentCapabilities(streaming=True),
    default_input_modes=["text/plain"],
    default_output_modes=["text/plain"],
    skills=[
        AgentSkill(
            id="scan_market",
            name="Scan Market",
            description="Finds recent news and strategic moves for a competitor",
            tags=["market", "news", "competitive intelligence"],
            examples=["Scan market for Anthropic"],
        )
    ],
)
```

**What the Agent Card is:** This is the agent's "business card" — a structured description of what it can do. It gets published at `/.well-known/agent.json` on the agent's server. When the host agent creates a `RemoteA2aAgent`, it fetches this card to discover:
- What the agent is named
- What its URL is (so it knows where to send requests)
- What capabilities it has (e.g., streaming support)
- What skills/tasks it can perform

This is the **discovery mechanism** of the A2A protocol — agents find each other by reading Agent Cards rather than by hard-coded logic.

```python
_session_uri = os.environ.get("SESSION_SERVICE_URI")
_session_service = DatabaseSessionService(db_url=_session_uri) if _session_uri else InMemorySessionService()

runner = Runner(
    app_name=root_agent.name,
    agent=root_agent,
    artifact_service=InMemoryArtifactService(),
    session_service=_session_service,
    memory_service=InMemoryMemoryService(),
)
```

**What the session service selection does:** If `SESSION_SERVICE_URI` is set (Cloud Run with the Neon DB secret mounted), `DatabaseSessionService` stores session data in PostgreSQL so it survives Cloud Run cold starts and restarts. If not set (local development), it falls back to `InMemorySessionService`. This pattern makes the code work in both environments without any code changes.

**Why only sessions in PostgreSQL and not artifacts/memory?** Our agents don't produce file artifacts and don't need memory across separate user sessions — so `InMemoryArtifactService` and `InMemoryMemoryService` are sufficient.

**What Runner does:** Wraps the agent with all the services it needs to execute. Think of it as the "execution environment" for the agent.

```python
executor = A2aAgentExecutor(runner=runner, config=A2aAgentExecutorConfig())
handler = DefaultRequestHandler(agent_executor=executor, task_store=InMemoryTaskStore())
app = A2AStarletteApplication(agent_card=agent_card, http_handler=handler).build()
```

**What these three lines do — the A2A server stack:**
1. `A2aAgentExecutor` — translates incoming A2A protocol messages into ADK Runner calls and translates responses back into A2A format
2. `DefaultRequestHandler` — handles the HTTP request lifecycle: receives the request, passes it to the executor, tracks task status, returns the response
3. `A2AStarletteApplication` — the actual ASGI web application. It automatically creates two endpoints:
   - `POST /` — the JSON-RPC endpoint where other agents send tasks
   - `GET /.well-known/agent.json` — serves the Agent Card for discovery

```python
if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    uvicorn.run(app, host="0.0.0.0", port=port)
```

**What this does:** Starts the web server. `PORT` is read from the environment — Google Cloud Run automatically sets this to 8080. `host="0.0.0.0"` means "accept connections from any network interface" (required inside containers).

---

### 8. `Dockerfile` (same for all agents)

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```

**Line by line:**
- `FROM python:3.11-slim` — base image: a minimal Python 3.11 Linux container
- `WORKDIR /app` — sets the working directory inside the container
- `COPY requirements.txt .` — copies only the requirements file first (Docker layer caching: if requirements don't change, pip install is skipped on next build)
- `RUN pip install ...` — installs Python dependencies
- `COPY . .` — copies all other files (agent.py, main.py, etc.)
- `CMD ["python", "main.py"]` — the command that runs when the container starts

**Why copy requirements first?** Docker builds in layers. If you copy everything at once and then run pip install, any code change invalidates the pip install layer and forces a full re-install. By copying requirements.txt first, pip install only re-runs when requirements actually change — saving several minutes on each build.

---

### 9. `requirements.txt` files

**Host agent:**
```
google-adk[a2a]==1.9.0   ← ADK with A2A support enabled
google-genai              ← Google Generative AI SDK
a2a-sdk==0.2.16          ← A2A protocol library (must match ADK's expected version)
uvicorn                   ← ASGI web server
psycopg2-binary           ← PostgreSQL driver (for persistent sessions)
sqlalchemy                ← ORM used by ADK's DatabaseSessionService
```

**Specialist agents:**
```
google-adk[a2a]==1.9.0
google-genai
a2a-sdk==0.2.16
uvicorn
psycopg2-binary
sqlalchemy
```

Specialists now use the same PostgreSQL `SESSION_SERVICE_URI` as the host. This means each specialist's conversation history (tool calls + output) survives Cloud Run cold starts. `psycopg2-binary` and `sqlalchemy` are required for `DatabaseSessionService` to connect to Neon PostgreSQL. `InMemoryArtifactService` and `InMemoryMemoryService` remain in-memory because artifacts and cross-session memory are not used in this project.

---

## The Complete Request Flow (End to End)

Here is exactly what happens when you type "Analyze OpenAI" and click Submit:

```
1. Browser → POST /apps/host/users/ui_user/sessions
   ← Returns: { "id": "abc-123", ... }

2. Browser → POST /run_sse
   Body: { app_name: "host", user_id: "ui_user",
           session_id: "abc-123",
           new_message: { role: "user", parts: [{ text: "Analyze OpenAI" }] } }

3. Host agent starts. SequentialAgent calls sub-agent #1:
   Host → POST https://market-scanner.run.app/
   Body: { task with "Analyze OpenAI" as input }

4. Market Scanner agent receives the request.
   - Gemini reads the task, identifies "OpenAI" as the target
   - Calls google_search("OpenAI news 2025")
   - Calls google_search("OpenAI product launch")
   - Calls google_search("OpenAI announcement")
   - Writes a **MARKET SCAN** section with findings
   ← Returns the MARKET SCAN text to the host

5. Host saves Market Scanner's output to the session.
   SequentialAgent calls sub-agent #2:
   Host → POST https://sentiment-analyzer.run.app/
   Body: { task with full session history including MARKET SCAN }

6. Sentiment Analyzer receives the full conversation history.
   - Identifies "OpenAI" from context
   - Performs 4 sentiment searches
   - Writes **SENTIMENT ANALYSIS** section
   ← Returns to host

7. Host saves to session. Calls sub-agent #3 (Pricing Intelligence).
   Same pattern — 3 pricing searches → **PRICING INTELLIGENCE** section

8. Host saves to session. Calls sub-agent #4 (Report Generator).
   - Report Generator reads ALL previous sections in the session
   - Synthesizes into the final executive brief
   ← Returns the complete report

9. Host streams the final report back to the browser via SSE.
   The browser renders it as formatted Markdown.
```

---

### 10. `deploy.sh` — One-Command Deployment Script

This shell script deploys all 5 agents to Google Cloud Run in the correct order. Students only need to fill in their credentials at the top and run it once.

```bash
GCP_PROJECT="your-gcp-project-id"
GCP_REGION="us-central1"
GOOGLE_API_KEY="your-gemini-api-key"
DATABASE_URL="postgresql://USER:PASSWORD@HOST/DBNAME?sslmode=require"
```

**The only section students touch.** Everything below is automated.

---

**Step 1 — Enable GCP APIs:**
```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  secretmanager.googleapis.com \
  storage.googleapis.com
```

Before you can use a GCP service, you must enable it for your project. This enables:
- `run.googleapis.com` — Cloud Run (for deploying containers)
- `cloudbuild.googleapis.com` — Cloud Build (for building Docker images)
- `secretmanager.googleapis.com` — Secret Manager (for storing API keys)
- `storage.googleapis.com` — Cloud Storage (Cloud Build uploads your source code here before building)

The script also grants the compute service account `storage.objectAdmin` — required on fresh GCP projects so Cloud Build can upload the source bundle to the staging bucket.

---

**Step 2 — Store secrets in Secret Manager:**
```bash
echo "$GOOGLE_API_KEY" | gcloud secrets create GOOGLE_API_KEY --data-file=-
echo "$DATABASE_URL"   | gcloud secrets create SESSION_SERVICE_URI --data-file=-
```

**Why Secret Manager instead of environment variables?** If you put an API key directly in an environment variable via the CLI, it appears in your shell history and deployment logs. Secret Manager stores it encrypted, with access controlled by IAM. The `--data-file=-` flag means "read the value from stdin" (piped from `echo`) rather than from a file — avoids the key ever being written to disk.

The `2>/dev/null || ...versions add...` pattern handles the case where the secret already exists — it tries to create first, and if that fails (because it already exists), it adds a new version instead.

---

**Step 3 — Grant Cloud Run access to the secrets:**
```bash
PROJECT_NUMBER=$(gcloud projects describe "$GCP_PROJECT" --format="value(projectNumber)")
SA="serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com"

gcloud secrets add-iam-policy-binding GOOGLE_API_KEY --member="$SA" --role="roles/secretmanager.secretAccessor"
gcloud secrets add-iam-policy-binding SESSION_SERVICE_URI --member="$SA" --role="roles/secretmanager.secretAccessor"
```

Cloud Run containers run as a **service account** (a robot identity). By default, this service account cannot read secrets — you must explicitly grant it `secretmanager.secretAccessor` permission. The default compute service account is always named `PROJECT_NUMBER-compute@developer.gserviceaccount.com`.

---

**Step 4 — The `deploy_specialist` helper function:**
```bash
deploy_specialist() {
  local SERVICE=$1
  local AGENT_DIR=$2

  echo "  → Deploying $SERVICE..." >&2
  cd "$SCRIPT_DIR/$AGENT_DIR"

  gcloud run deploy "$SERVICE" \
    --source . \
    --region="$GCP_REGION" \
    --allow-unauthenticated \
    --update-secrets="GOOGLE_API_KEY=GOOGLE_API_KEY:latest,SESSION_SERVICE_URI=SESSION_SERVICE_URI:latest" \
    --set-env-vars="GOOGLE_GENAI_USE_VERTEXAI=FALSE" >&2

  local URL
  URL=$(gcloud run services describe "$SERVICE" --format="value(status.url)")

  gcloud run services update "$SERVICE" \
    --set-env-vars="GOOGLE_GENAI_USE_VERTEXAI=FALSE,SERVICE_URL=$URL" >&2

  echo "$URL"
}
```

This function is called once for each of the 4 specialist agents. Key points:

- `--source .` — Cloud Build automatically detects the Dockerfile and builds the container image. No manual `docker build` or `docker push` needed.
- `--allow-unauthenticated` — allows the host agent to call the specialists without OAuth tokens. In production you would use service-to-service authentication.
- `--update-secrets="GOOGLE_API_KEY=GOOGLE_API_KEY:latest,SESSION_SERVICE_URI=SESSION_SERVICE_URI:latest"` — mounts both secrets as environment variables. Specialists need `SESSION_SERVICE_URI` to use PostgreSQL for their own sessions.
- `GOOGLE_GENAI_USE_VERTEXAI=FALSE` — tells ADK to use the Gemini API directly (using the API key) rather than Vertex AI.
- The **two-step deploy** — first deploy, then update with `SERVICE_URL` — is necessary because the service URL is only known after the first deployment.
- **`>&2` on echo and gcloud commands** — this redirects all log/progress output to stderr. The function is called as `MARKET_SCANNER_URL=$(deploy_specialist ...)`, which captures stdout into the variable. Without `>&2`, the deployment logs would be captured too, producing a garbled URL like `→ Deploying market-scanner...\nhttps://...`. Only `echo "$URL"` stays on stdout so the variable gets the clean URL.

---

**Step 5 — Deploy the 4 specialists:**
```bash
MARKET_SCANNER_URL=$(deploy_specialist     "market-scanner"     "agents/market_scanner")
SENTIMENT_ANALYZER_URL=$(deploy_specialist "sentiment-analyzer" "agents/sentiment_analyzer")
PRICING_INTEL_URL=$(deploy_specialist      "pricing-intel"      "agents/pricing_intelligence")
REPORT_GENERATOR_URL=$(deploy_specialist   "report-generator"   "agents/report_generator")
```

Each specialist is deployed and its Cloud Run URL is captured in a variable. These URLs are then passed to the host agent so it knows where to find each specialist.

---

**Step 6 — Deploy the host agent:**
```bash
gcloud run deploy competitive-intel-host \
  --source . \
  --update-secrets="GOOGLE_API_KEY=GOOGLE_API_KEY:latest,SESSION_SERVICE_URI=SESSION_SERVICE_URI:latest" \
  --set-env-vars="GOOGLE_GENAI_USE_VERTEXAI=FALSE,\
    MARKET_SCANNER_URL=$MARKET_SCANNER_URL,\
    SENTIMENT_ANALYZER_URL=$SENTIMENT_ANALYZER_URL,\
    PRICING_INTEL_URL=$PRICING_INTEL_URL,\
    REPORT_GENERATOR_URL=$REPORT_GENERATOR_URL"
```

The host gets **both secrets** (API key + database URL) and **all four specialist URLs** as environment variables. This is how `host/agent.py` knows where to find each specialist at runtime — it reads `os.environ['MARKET_SCANNER_URL']` etc.

---

**Why deploy specialists before the host?** The host agent's `agent.py` reads the specialist URLs at startup and validates they exist. If the host were deployed first, the URLs would not yet exist and the startup check would fail.

---

## Running the UI Locally

After deploying all 5 agents, open the UI by serving the `ui/` folder with a local HTTP server.

**Why you can't just double-click `index.html`:** Browsers block `fetch()` calls from `file://` URLs (security restriction). You need a real HTTP server even for local files.

**Steps:**

1. Open a terminal and navigate to the project root:
   ```bash
   cd competitive-intel-a2a
   ```

2. Start a local HTTP server:
   ```bash
   python -m http.server 8000
   ```

3. Open your browser and go to:
   ```
   http://localhost:8000/ui/index.html
   ```

4. In the **Host Agent URL** field, paste your deployed Cloud Run host URL, for example:
   ```
   https://competitive-intel-host-xxxx-uc.a.run.app
   ```

5. Type a company name in the query box (e.g. `Analyze OpenAI`) and click **Run**.

The server keeps running until you press `Ctrl+C`. You do not need to restart it between queries.

> **Note:** The `python -m http.server` only serves the HTML/JS files locally. All actual AI work (agent calls, LLM, web searches) still happens on Cloud Run — your local machine just displays the results.

---

## Neon PostgreSQL — What Gets Stored

ADK's `DatabaseSessionService` automatically creates 4 tables on first startup:

| Table | Purpose | What's stored |
|---|---|---|
| **`sessions`** | One row per active session | `app_name` (which agent), `user_id`, `session_id`, `state` (jsonb), timestamps |
| **`events`** | Full conversation history | Every message, tool call, and agent response — the entire LLM exchange |
| **`app_states`** | App-level shared state | A JSON blob scoped to the agent name — currently `{}` empty but available for agents to persist app-wide data |
| **`user_states`** | User-level shared state | A JSON blob scoped to a user ID — persists user preferences or context across sessions |

After a successful run you will see 5 rows in `app_states` — one for each of the 5 agents (host, market_scanner, sentiment_analyzer, pricing_intelligence, report_generator).

**Why this matters:** On Cloud Run, containers can be killed and restarted at any time. Without PostgreSQL, every restart wipes all session history. With `DatabaseSessionService`, a restarted container reconnects to Neon and the conversation continues exactly where it left off.

**Known issue — stale connections:** If the system is idle for more than ~5 minutes, Neon closes the idle database connections server-side. SQLAlchemy's connection pool does not know this and tries to reuse the closed connection, causing:
```
psycopg2.OperationalError: SSL connection has been closed unexpectedly
```
This only affects the second run after an idle period. A retry immediately after resolves it. A production fix would add `pool_pre_ping=True` to the SQLAlchemy engine configuration.

---

## Key Technologies Summary

| Technology | Role |
|---|---|
| **Google ADK** | Framework for building AI agents with tools, sessions, and multi-agent orchestration |
| **A2A Protocol** | Open standard for agents to discover and communicate with each other over HTTP |
| **Gemini 2.5 Flash** | The LLM powering all five agents |
| **Google Search Tool** | Built-in ADK tool that gives agents real-time web search capability |
| **SequentialAgent** | ADK agent type that runs sub-agents in strict order, sharing context |
| **RemoteA2aAgent** | ADK proxy that represents a remote agent as if it were local |
| **Agent Card** | JSON descriptor published by each agent so others can discover its capabilities |
| **FastAPI + Uvicorn** | Web framework and server for the host agent's REST API |
| **Starlette + A2AStarletteApplication** | Web framework for the specialist agents' A2A server |
| **Server-Sent Events (SSE)** | HTTP streaming protocol that lets the server push updates to the browser in real time |
| **Cloud Run** | Google Cloud's serverless container platform — auto-scales, pay per request |
| **Secret Manager** | Google Cloud service for storing API keys securely (never in code or environment variables) |
| **Neon PostgreSQL** | Cloud-hosted Postgres database for storing sessions persistently |
