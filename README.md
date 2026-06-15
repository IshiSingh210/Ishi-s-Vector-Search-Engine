# Vector Search Engine

An n8n workflow that routes user queries to the right AI tool automatically — general knowledge, live internet search, or AWS documentation RAG — with persistent memory per user session.

> Built under the guidance and mentorship of [Dr. Kanad Basu](https://www.marshall.usc.edu/personnel/kanad-basu)
---

## What It Does

When a query comes in, the agent decides:

- Is this a **general knowledge question** (math, code, explanations)? → answers directly using GPT-4o-mini
- Does this need **current, real-time information**? → searches the web via Traversaal Ares API
- Is this **AWS-specific** (services, APIs, pricing, architecture)? → retrieves from an AWS documentation knowledge base via Traversaal Pro RAG

This means you get grounded, accurate answers instead of outdated or hallucinated ones — without manually specifying which tool to use.

---

## Architecture

```
POST /chat-assistant
        │
        ▼
   [ Webhook ]
        │
        ▼
  [ AI Agent ] ─────────────────────────────────┐
  (GPT-4o-mini)                                  │
        │                                         │
        ├── general question ─────────────────────┤
        │                                         │
        ├── needs current info                    │
        │        └─► [ Internet Search Tool ]     │
        │              Traversaal Ares API         │
        │                                         │
        └── AWS-specific                          │
                 └─► [ AWS Document Search ]      │
                       Traversaal Pro RAG          │
                                                  │
  [ Memory Buffer ] ◄───────────────────────────-┘
  (per-user session key)
        │
        ▼
  [ Respond to Webhook ]
```

Memory is keyed by `username` from the request body, so each user gets their own conversation context across turns.

---

## Setup

### Prerequisites

- n8n (self-hosted or cloud)
- OpenAI API key
- Traversaal API access ([traversaal.ai](https://traversaal.ai))

### Step 1 — Import the workflow

Copy `workflow.json` and import it into n8n via **Workflows → Import from file**.

### Step 2 — Add OpenAI credentials

In n8n, go to **Credentials → New → OpenAI** and paste your API key. Then assign it to the `OpenAI Chat Model` node.

### Step 3 — Configure Traversaal Ares (Internet Search)

In the `Internet Search` node, set the `x-api-key` header to your Ares API key:

```
x-api-key: your_ares_key_here
```

Get your key at [api-ares.traversaal.ai](https://api-ares.traversaal.ai).

### Step 4 — Configure Traversaal Pro (AWS RAG)

In the `AWS Document Search` node, set the `Authorization` header:

```
Authorization: Bearer your_traversaal_pro_token
```

Upload your AWS documentation corpus at [pro.traversaal.ai](https://pro.traversaal.ai) and use the project Bearer token.

### Step 5 — Activate

Hit **Activate** in n8n. Your webhook is live at:

```
POST https://your-n8n-instance/webhook/chat-assistant
```

---

## API Usage

**Request body:**

```json
{
  "query": "What is the default timeout for AWS Lambda?",
  "username": "alice"
}
```

**Response:** Plain text answer from the agent, grounded in retrieved sources where applicable.

---

## Files in This Repo

```
├── workflow.json       # n8n workflow export (credentials redacted)
├── screenshot.png      # n8n canvas overview
└── README.md
```

### `.env.example`

```
OPENAI_API_KEY=
TRAVERSAAL_ARES_API_KEY=
TRAVERSAAL_PRO_BEARER_TOKEN=
```

> **Note:** The `workflow.json` contains no real credentials. All sensitive values have been replaced with placeholder strings. Add your own keys after importing.

---

## Customization

**Swap the LLM** — replace GPT-4o-mini with any model supported by n8n's LangChain nodes (GPT-4o, Claude, Mistral, etc.) by changing the `OpenAI Chat Model` node.

**Point RAG at a different knowledge base** — Traversaal Pro supports any document corpus. Replace the AWS docs with your own internal docs, product manuals, or legal documents.

**Adjust memory window** — the `Simple Memory` node uses a buffer window. Increase the window size in node settings for longer context retention.

**Change routing logic** — the agent's system prompt defines the tool selection rules. Edit it in the `AI Agent` node's system message to add new tools or change routing priority.

---

## Limitations

- Memory is in-process only; restarting n8n clears all sessions
- The agent uses GPT-4o-mini by default — complex reasoning may benefit from GPT-4o
- RAG quality depends on how well your document corpus is indexed in Traversaal Pro
- Internet search results are not always deterministic; the agent may phrase the same answer differently across runs
