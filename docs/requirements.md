# AI Pulse — Your Personal AI Intelligence Hub

## The Problem

The AI space moves at breakneck speed. Anthropic ships MCP, Google drops Gemini updates, OpenAI announces agents, Meta open-sources a new model, a paper on ArXiv changes how everyone thinks about reasoning — and that's just one week. You can't monitor 20+ sources daily and still do your actual job.

## The Solution

An AI-powered platform that crawls, curates, and synthesizes AI news from across the internet — then lets you *have a conversation* with it. Not just a feed of links, but an intelligent system that understands context, tracks narratives over time, and tells you what actually matters.

## What Makes It Different From a Regular RSS Reader

- **Agent-powered curation** — an AI agent reads every article and decides if it's signal or noise, ranks importance, and tags themes
- **Memory across time** — the system tracks evolving stories ("MCP adoption" isn't one article, it's 40 articles over 3 months) and connects dots you'd miss
- **Conversational interface** — ask "what happened with open-source models this week?" or "compare Anthropic's and Google's agent approaches" and get synthesized answers grounded in actual sources
- **MCP server** — plug your knowledge base into Claude Code or Claude Desktop and query your AI news corpus from anywhere

---

## Architecture — Three Learning Pillars

### 1. Agents (The Brain)

| Agent | Role |
|-------|------|
| **Crawler Agent** | Fetches content from sources on a schedule, extracts clean text from HTML/RSS |
| **Curator Agent** | Reads raw content, scores relevance (1-10), extracts key claims, tags topics |
| **Synthesizer Agent** | Generates daily/weekly digests, answers questions by pulling from memory |

Each agent is an autonomous unit with a defined role, tools it can call, and a decision loop. This teaches you the core agent pattern: **perceive → reason → act → reflect**.

### 2. Memory (The Knowledge)

| Layer | Purpose | Tech |
|-------|---------|------|
| **Short-term** | Current session context, what you just asked about | In-memory / Redis |
| **Long-term** | Every article, summary, embedding, and topic tag | ChromaDB (vector store) |
| **Structured** | Article metadata, source configs, user preferences | SQLite → PostgreSQL |
| **Narrative tracking** | Links related articles into storylines that evolve over time | Graph relationships in DB + embeddings |

### 3. MCP Server (The Interface)

Build a custom MCP server from scratch that exposes your knowledge base as tools:

| Tool | Description |
|------|-------------|
| `search_articles` | Semantic search across your entire corpus |
| `get_digest` | Daily/weekly summary of top developments |
| `trending_topics` | What's hot in AI right now based on volume + velocity |
| `compare_announcements` | Side-by-side analysis of announcements from different companies |
| `get_narrative` | Track how a storyline has evolved over time |
| `ask_question` | Free-form Q&A grounded in your article corpus |

Connectable to Claude Desktop, Claude Code, or any MCP client.

---

## Sources — Phased Rollout

### Phase 1 — Core Blogs
- Anthropic Blog (https://www.anthropic.com/news)
- Google AI Blog (https://blog.google/technology/ai/)
- OpenAI Blog (https://openai.com/blog)
- Meta AI Blog (https://ai.meta.com/blog/)
- HuggingFace Blog (https://huggingface.co/blog)

### Phase 2 — Research & Extended Blogs
- ArXiv (cs.AI, cs.CL, cs.LG)
- Microsoft Research Blog
- DeepMind Blog
- NVIDIA AI Blog
- Mistral Blog

### Phase 3 — Community & Social
- Twitter/X (key accounts: @AnthropicAI, @GoogleAI, @OpenAI, etc.)
- Reddit (r/MachineLearning, r/LocalLLaMA, r/artificial)
- Hacker News (AI-tagged posts)

### Phase 4 — Multimedia & Code
- YouTube transcripts (AI channels — Yannic Kilcher, Two Minute Papers, etc.)
- Podcast transcripts
- GitHub trending repos (AI/ML category)

---

## Tech Stack

| Component | Technology | Why |
|-----------|-----------|-----|
| **Backend** | FastAPI (async) | You're comfortable with it, native async support for crawlers |
| **Frontend** | React + Vite | Lightweight, fast dev loop |
| **Vector DB** | ChromaDB | Local, zero-config, perfect for learning |
| **LLM** | Anthropic API (Claude Sonnet 4 for curation, Haiku 4.5 for lightweight tasks) | Best-in-class for structured reasoning |
| **Scheduler** | APScheduler | Simple, in-process, good enough for Phase 1 |
| **Database** | SQLite (Phase 1) → PostgreSQL (later) | Start simple, migrate when needed |
| **MCP** | Python MCP SDK (`mcp`) | Official SDK, well-documented |
| **Scraping** | httpx + BeautifulSoup / trafilatura | Async HTTP + clean text extraction |
| **Embeddings** | Anthropic Voyage / sentence-transformers | For semantic search in ChromaDB |

---

## Build Phases

### Phase 1 — The Foundation (Week 1–2)

**Goal:** A working feed that crawls 5 blogs and shows articles in a web UI.

**Backend:**
- FastAPI app structure with proper project layout
- Source registry — config-driven list of sources with URL, type (RSS/HTML), schedule
- Crawler module — async fetcher using `httpx` + `trafilatura` for clean text extraction
- SQLite database — articles table (id, source, title, url, content, published_at, created_at)
- REST endpoints: `GET /articles`, `GET /articles/{id}`, `POST /crawl` (manual trigger)
- APScheduler — crawl every 30 minutes

**Frontend:**
- React app with Vite
- Simple feed view — list of articles sorted by date
- Article detail view — full extracted content
- Filter by source
- Basic search (text match for now)

**Key learning:** async Python, web scraping, basic API design.

```
ai-pulse/
├── backend/
│   ├── main.py              # FastAPI app + scheduler
│   ├── config.py             # Source registry, settings
│   ├── models.py             # SQLAlchemy/Pydantic models
│   ├── database.py           # DB connection + migrations
│   ├── crawlers/
│   │   ├── base.py           # Abstract crawler interface
│   │   ├── rss_crawler.py    # RSS feed crawler
│   │   └── html_crawler.py   # HTML scraping crawler
│   ├── routers/
│   │   ├── articles.py       # Article CRUD endpoints
│   │   └── sources.py        # Source management endpoints
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── components/
│   │   │   ├── ArticleFeed.jsx
│   │   │   ├── ArticleCard.jsx
│   │   │   └── SourceFilter.jsx
│   │   └── api.js
│   └── package.json
└── README.md
```

---

### Phase 2 — The Agent Brain (Week 3–4)

**Goal:** Articles are now AI-curated, scored, tagged, and summarized.

**New components:**
- **Curator Agent** — for each new article:
  - Calls Claude API to score relevance (1–10)
  - Extracts 3–5 key claims / takeaways
  - Assigns topic tags (e.g., "agents", "open-source", "reasoning", "safety")
  - Detects if it's a major announcement vs. incremental update
- **Agent framework** — build a simple agent loop:
  ```
  while has_unprocessed_articles:
      article = get_next_unprocessed()
      result = curator_agent.process(article)
      store(result)
  ```
- **Daily Digest endpoint** — `GET /digest?date=2026-04-09`
  - Synthesizer agent takes today's top articles and writes a coherent summary
  - Grouped by theme, ranked by importance

**Frontend additions:**
- Importance score badges on articles
- Topic tag filters
- Daily digest page

**Key learning:** LLM tool calling, agent loops, structured output parsing, prompt engineering for curation.

---

### Phase 3 — Memory & Intelligence (Week 5–6)

**Goal:** Semantic search, narrative tracking, and conversational Q&A.

**New components:**
- **ChromaDB integration:**
  - Embed every article using Voyage / sentence-transformers
  - Store embeddings with metadata (source, date, topics, score)
- **Semantic search:** `GET /search?q=how is MCP adoption going`
  - Vector similarity search across entire corpus
  - Re-rank results with an LLM for relevance
- **Narrative tracking:**
  - Group related articles into "storylines" (e.g., "Claude MCP ecosystem")
  - Track how sentiment, claims, and key players evolve
  - Timeline view for each narrative
- **Conversational Q&A:** `POST /ask`
  - RAG pipeline: embed question → retrieve relevant articles → LLM generates grounded answer with citations
  - Session memory — remembers context within a conversation

**Frontend additions:**
- Semantic search bar (replaces text search)
- Narrative/storyline pages with timelines
- Chat interface for Q&A

**Key learning:** embeddings, vector databases, RAG, retrieval strategies, conversation memory management.

---

### Phase 4 — MCP Server (Week 7–8)

**Goal:** Your AI Pulse knowledge base is now accessible as an MCP server.

**New components:**
- **MCP server** using the Python `mcp` SDK:
  ```python
  @server.tool("search_articles")
  async def search_articles(query: str, limit: int = 5):
      """Search AI news articles semantically."""
      results = await vector_store.search(query, limit)
      return format_results(results)

  @server.tool("get_digest")
  async def get_digest(date: str = "today"):
      """Get AI news digest for a given date."""
      return await synthesizer.generate_digest(date)

  @server.tool("trending_topics")
  async def trending_topics(days: int = 7):
      """What's trending in AI over the past N days."""
      return await analytics.get_trending(days)
  ```
- **Claude Desktop integration** — add your MCP server to `claude_desktop_config.json`
- **Claude Code integration** — use your news corpus while coding

**Key learning:** MCP protocol internals, tool design, server lifecycle, client-server communication.

---

### Phase 5 — Scale Sources (Ongoing)

**Goal:** Expand beyond blogs to cover the full AI information landscape.

| Source | Approach |
|--------|----------|
| **ArXiv** | ArXiv API → filter by categories → abstract embedding + LLM summary |
| **Reddit** | Reddit API / PRAW → monitor subreddits → filter by upvotes + AI keywords |
| **Twitter/X** | Twitter API v2 → track key accounts + hashtags |
| **Hacker News** | HN Algolia API → filter AI/ML posts above score threshold |
| **YouTube** | YouTube Data API → get transcripts via `youtube-transcript-api` → summarize |
| **GitHub** | GitHub API → trending repos in AI/ML → extract README summaries |

Each source becomes a new crawler plugin inheriting from the base crawler interface built in Phase 1.

---

## Stretch Goals (Phase 6+)

- **Personalization** — learn your interests over time, boost articles about topics you engage with
- **Alerts** — push notifications when something big drops (model release, major paper, etc.)
- **Multi-user** — auth + per-user preferences and reading history
- **Export** — weekly newsletter generation, Slack integration
- **Fine-tuned classifier** — train a small model on your curation decisions to reduce LLM API costs
- **Knowledge graph** — Neo4j for tracking relationships between companies, models, papers, and people

---

## Getting Started

```bash
# Clone and setup
mkdir ai-pulse && cd ai-pulse
python -m venv .venv
source .venv/bin/activate

# Install core dependencies
pip install fastapi uvicorn httpx trafilatura beautifulsoup4 \
    sqlalchemy aiosqlite apscheduler anthropic chromadb mcp pydantic

# Create project structure
mkdir -p backend/{crawlers,routers,agents} frontend/src/components

# Start building Phase 1
touch backend/main.py backend/config.py backend/models.py backend/database.py
```

---

## Key Concepts You'll Learn

| Concept | Where You'll Learn It |
|---------|----------------------|
| Agent architecture (perceive → reason → act) | Phase 2 — Curator & Synthesizer agents |
| LLM tool calling | Phase 2 — structured extraction, Phase 4 — MCP tools |
| Short-term memory | Phase 3 — conversation context in Q&A |
| Long-term memory (vector store) | Phase 3 — ChromaDB embeddings + semantic search |
| RAG (Retrieval Augmented Generation) | Phase 3 — Q&A grounded in articles |
| MCP protocol | Phase 4 — building a server from scratch |
| Multi-agent coordination | Phase 2+ — crawler → curator → synthesizer pipeline |
| Prompt engineering | Throughout — curation prompts, digest generation, Q&A |
| Async Python patterns | Phase 1 — concurrent crawling with httpx |

---

*Start with Phase 1. Ship something that works. Then layer intelligence on top.*