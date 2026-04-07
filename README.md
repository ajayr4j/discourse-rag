# discourse-rag

Turn any public Discourse forum into a RAG-powered customer support pipeline.

Fetch community knowledge, embed it, and query it semantically — with source links back to the original threads. Works with any Discourse instance. No API key required for public forums.

---

## What it does

```
Discourse forum  →  SQLite  →  Vector store  →  Structured answer + source links
```

A user submits a support ticket. The pipeline retrieves the most relevant threads from your forum's history, passes them to an LLM, and returns a structured answer with direct links to the source discussions. If confidence is low, it escalates to a human agent.

Built with [Outlines](https://github.com/dottxt-ai/outlines) for guaranteed structured outputs — no fragile JSON parsing.

---

## Motivation

Discourse forums accumulate years of solved problems, edge case discussions, and community wisdom. Most support teams never tap this. This tool makes that knowledge queryable in real time.

**Example use case:** Zerodha's [TradingQnA](https://tradingqna.com) has 498k posts across F&O, algo trading, taxation, and platform issues. discourse-rag turns that into an instant-answer layer for support tickets — before a human agent ever needs to look at it.

---

## Stack

| Layer | Tool |
|---|---|
| Discourse API | `requests` (no auth needed for public forums) |
| Storage | `sqlite3` (built-in) |
| Embeddings | Azure OpenAI `text-embedding-3-small` or any OpenAI-compatible endpoint |
| Vector store | ChromaDB (default) or FAISS |
| LLM | Azure OpenAI GPT-4o via [Outlines](https://github.com/dottxt-ai/outlines) |
| Structured output | Pydantic + Outlines |

---

## Project structure

```
discourse-rag/
├── README.md
├── .env.example
├── requirements.txt
│
├── ingestion/
│   ├── fetcher.py        # Discourse API → SQLite
│   ├── chunker.py        # SQLite posts → passages
│   └── embedder.py       # passages → vector store
│
├── pipeline/
│   ├── retriever.py      # query → top-k passages
│   ├── generator.py      # passages + query → structured answer
│   └── schemas.py        # Pydantic output models
│
├── config/
│   └── settings.py       # forum URL, category IDs, thresholds
│
└── examples/
    ├── zerodha_fo.py     # TradingQnA F&O use case
    └── generic.py        # minimal working example for any forum
```

---

## Quickstart

### 1. Clone and install

```bash
git clone https://github.com/your-username/discourse-rag
cd discourse-rag
pip install -r requirements.txt
```

### 2. Configure your forum

Copy `.env.example` to `.env` and fill in your details:

```bash
cp .env.example .env
```

```env
# Your Discourse forum
DISCOURSE_BASE_URL=https://your-forum.com

# Category to fetch (get IDs from /categories.json)
DISCOURSE_CATEGORY_ID=19
DISCOURSE_CATEGORY_SLUG=your-category

# Azure OpenAI
AZURE_OPENAI_KEY=your-key
AZURE_OPENAI_ENDPOINT=https://your-resource.cognitiveservices.azure.com/
AZURE_OPENAI_DEPLOYMENT=your-deployment
AZURE_OPENAI_EMBED_DEPLOYMENT=your-embed-deployment
AZURE_OPENAI_API_VERSION=2024-12-01-preview

# Pipeline settings
CONFIDENCE_THRESHOLD=0.7
TOP_K_RESULTS=5
```

### 3. Fetch and index your forum

```bash
# Phase 1: fetch all topics and posts into SQLite (~hours depending on forum size)
python ingestion/fetcher.py

# Phase 2: chunk posts into passages and embed them
python ingestion/chunker.py
python ingestion/embedder.py
```

### 4. Query it

```python
from pipeline.generator import answer_ticket

response = answer_ticket("What happens to my short straddle position on expiry day?")

print(response.answer)
print(response.confidence)       # "high" / "medium" / "low"
print(response.escalate_to_human)

for source in response.sources:
    print(f"  [{source.relevance}] {source.topic_title}")
    print(f"           {source.url}")
```

---

## Output schema

Every query returns a structured `TicketResponse` — guaranteed valid by Outlines, no parsing required:

```python
class TicketResponse(BaseModel):
    answer: str
    sources: List[Source]           # up to 3, with direct forum links
    confidence: Literal["high", "medium", "low"]
    category: Literal[
        "order_execution", "margin_pledge", "expiry_settlement",
        "platform_issue", "taxation", "account_access", "other"
    ]
    escalate_to_human: bool
```

---

## How the pipeline works

```
1. Ticket text arrives
      ↓
2. Embed the query (same model used during indexing)
      ↓
3. Retrieve top-k semantically similar passages from vector store
      ↓
4. Pass retrieved passages + query to GPT-4o via Outlines
      ↓
5. Get back TicketResponse with answer, sources, confidence
      ↓
6a. confidence=high  →  return auto-answer with source links
6b. confidence=low   →  escalate to human agent (with RAG answer as draft)
```

The human-in-the-loop path means agents review rather than write from scratch. Response time stays fast even on escalated tickets.

---

## Fetcher details

The fetcher is resumable — if interrupted, it picks up exactly where it left off. Progress is tracked in the SQLite `fetch_state` table.

```
Phase 1 (metadata)  ~120 pages  →  ~3 minutes
Phase 2 (posts)     ~3600 requests per category  →  2-4 hours
```

Expected DB size for a category with ~26k posts: 150–300 MB.

Rate limiting is set to 1.5s between requests by default. Adjust `SLEEP_SEC` in `fetcher.py` — don't go below 1s.

---

## Discovering your forum's categories

Any public Discourse forum exposes its structure at `/categories.json` and stats at `/site/statistics.json`:

```python
import requests

base = "https://your-forum.com"

# All categories with post counts
cats = requests.get(f"{base}/categories.json").json()["category_list"]["categories"]
for c in cats:
    print(f"[{c['id']}] {c['name']}  topics:{c['topic_count']}  posts:{c['post_count']}")

# Forum-wide stats
stats = requests.get(f"{base}/site/statistics.json").json()
print(f"Total posts: {stats['posts_count']}")
```

---

## Use cases

- **Broker / fintech support** — index your community forum, auto-answer tickets before agent involvement
- **Open source projects** — turn your Discourse into a searchable knowledge base for contributor questions
- **SaaS helpdesks** — augment Zendesk / Freshdesk with community-sourced answers
- **Internal wikis** — if your team uses Discourse for internal documentation, make it queryable

---

## Roadmap

- [ ] FAISS backend option (for offline / no-internet environments)
- [ ] Webhook mode (real-time ingestion of new posts as they're published)
- [ ] Confidence calibration with human feedback loop
- [ ] Multi-category indexing
- [ ] OpenAI direct support (non-Azure)
- [ ] Simple FastAPI wrapper for HTTP ticket submission

---

## Contributing

PRs welcome. If you use this for a specific Discourse forum, consider adding an example under `examples/` — the more concrete use cases documented, the more useful this gets for everyone.

---

## License

MIT
