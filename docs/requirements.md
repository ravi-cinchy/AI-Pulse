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
| **Crawler Agent** | Fetches content from sources on a schedule, extracts clean text from HTML/RSS, normalizes to canonical schema (title, author, published_at, plain-text body, URL, content hash) |
| **Curator Agent** | Reads raw content, scores relevance (1-10), extracts key claims, tags topics |
| **Synthesizer Agent** | Generates daily/weekly digests, answers questions by pulling from memory |

Each agent is an autonomous unit with a defined role, tools it can call, and a decision loop. This teaches you the core agent pattern: **perceive → reason → act → reflect**.

### 2. Memory (The Knowledge)

| Layer | Purpose | Tech |
|-------|---------|------|
| **Short-term** | Current session context, what you just asked about | In-memory / Redis |
| **Long-term** | Every article, summary, embedding, and topic tag | ChromaDB (vector store) |
| **Structured** | Article metadata, source configs, user preferences | SQLite → PostgreSQL |
| **Narrative tracking** | Links related articles into storylines that evolve over time | Embedding cosine similarity against storyline centroids; `storylines` + `storyline_articles` tables |

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
| **Logging** | structlog | Structured JSON logging for crawlers and agents |
| **Email** | Resend | Transactional email delivery for daily digests (free tier: 100/day) |

---

## Data Model

Tables are introduced incrementally across phases.

### Phase 1

| Table | Key Columns | Purpose |
|-------|-------------|---------|
| `sources` | id, name, url, source_type (RSS/HTML), crawl_interval_minutes, is_active, last_crawled_at, created_at | Config-driven source registry |
| `articles` | id, source_id (FK), url, url_normalized, title, author, content_clean, content_raw, content_hash (SHA-256), published_at, crawled_at | Deduplicated article storage |
| `crawl_logs` | id, source_id (FK), started_at, finished_at, status (success/partial/failed), articles_found, articles_new, error_message | Crawler health tracking |
| `subscribers` | id, email, is_active, subscribed_at, unsubscribed_at | Email digest subscribers |
| `digest_sends` | id, subscriber_id (FK), digest_date, sent_at, status | Tracks which digests were sent to whom |

### Phase 2

| Table | Key Columns | Purpose |
|-------|-------------|---------|
| `curation_results` | id, article_id (FK), relevance_score (1-10), is_major_announcement, model_used, prompt_tokens, completion_tokens, cost_usd, curated_at | Agent curation output + cost tracking |
| `article_claims` | id, curation_result_id (FK), claim_text, ordering | Key claims extracted per article |
| `topics` | id, name, slug, description | Canonical topic taxonomy |
| `article_topics` | article_id (FK), topic_id (FK), confidence | Many-to-many article-topic assignment |
| `digests` | id, digest_date, digest_type (daily/weekly), content_markdown, model_used, created_at | Generated AI digest storage |

### Phase 3

| Table | Key Columns | Purpose |
|-------|-------------|---------|
| `storylines` | id, title, summary, centroid_embedding_id, first_seen_at, last_updated_at, is_active, article_count | Narrative storyline cluster |
| `storyline_articles` | storyline_id (FK), article_id (FK), similarity_score, added_at | Articles assigned to storylines |
| `conversations` | id, started_at, last_message_at | Q&A session tracking |
| `conversation_messages` | id, conversation_id (FK), role (user/assistant), content, sources_cited (JSON), created_at | Chat history for session memory |

---

## Build Phases

### Phase 1 — The Foundation (Week 1–2)

**Goal:** A working feed that crawls 5 blogs, shows articles in a web UI, and sends daily digest emails to subscribers.

**Target sources:** Anthropic Blog, Google AI Blog, OpenAI Blog, Meta AI Blog, HuggingFace Blog (see Sources — Phase 1 above).

**Backend:**
- FastAPI app structure with proper project layout
- Source registry — config-driven list of sources with URL, type (RSS/HTML), schedule
- Crawler module — async fetcher using `httpx` + `trafilatura` for clean text extraction
- SQLite database — full schema defined in Data Model section above
- REST endpoints: `GET /articles`, `GET /articles/{id}`, `POST /crawl` (manual trigger)
- APScheduler — crawl every 30 minutes
- Structured logging with `structlog` — JSON output with correlation IDs, stdout + rotating file

**Crawler resilience:**
- **30-second per-request timeout**, configurable per source
- **3 retries with exponential backoff**: 2s / 8s / 32s delays for transient HTTP errors (429, 500, 502, 503)
- **5-minute hard cap per full crawl run** — if the overall crawl exceeds this, abort remaining sources and log partial results
- **Per-source rate limiting** via `asyncio.Semaphore` (default 1 req/s per domain)
- **Partial failure isolation** — one source failing doesn't block others; each source crawls independently
- **3 consecutive failures** for a source → mark `is_active = false`, surface in UI
- **No CAPTCHA handling** — if a source starts requiring CAPTCHAs, treat as dead and investigate

**Content normalization:**
- Every crawler produces a `RawArticle` → transformed to `CleanArticle` (canonical schema) before storage
- `CleanArticle` schema: title, author (nullable), published_at (fallback to crawl time), content_clean (plain text via trafilatura), content_raw (original HTML), url, url_normalized, content_hash, source_id
- trafilatura handles boilerplate removal; post-processing: strip excess whitespace, normalize Unicode, truncate to 50K chars
- Date parsing: `dateutil.parser` with fallback to crawl timestamp

**Deduplication:**
- URL normalization at crawl time — strip query params, fragments, trailing slashes, normalize to https. Store both raw `url` and `url_normalized`
- SHA-256 hash of `content_clean` with unique DB constraint — rejects exact duplicates at insert
- Title embedding similarity gate (>0.92 against last 7 days) added in Phase 2 when embeddings arrive

**Email subscriptions:**
- `POST /subscribe` — accepts email address, stores in `subscribers` table
- `POST /unsubscribe` — removes subscriber (or via unsubscribe link in email)
- `GET /subscribers` — admin-only list
- **Resend** integration for email delivery (free tier: 100 emails/day)
- **Daily digest job** — APScheduler task runs at a configured time (e.g., 8:00 AM), collects articles from the past 24 hours, renders an HTML email template, sends to all active subscribers via Resend API
- Email template: grouped by source, article title + snippet + link, date

**Frontend:**
- React app with Vite
- Article feed sorted by date with title, source badge, published date, and text snippet
- Article detail view — full extracted content
- Filter by source
- Basic text search
- Subscribe-to-digest form (email input + submit) with confirmation message

**Key learning:** async Python, web scraping, API design, email delivery, retry/resilience patterns.

```
ai-pulse/
├── backend/
│   ├── main.py              # FastAPI app + scheduler
│   ├── mcp_server.py         # MCP server entry point (ships Phase 2)
│   ├── config.py             # Source registry, settings
│   ├── models.py             # SQLAlchemy/Pydantic models
│   ├── database.py           # DB connection + migrations
│   ├── normalization.py      # Content cleaning + canonical schema
│   ├── dedup.py              # URL normalization + content hash dedup
│   ├── crawlers/
│   │   ├── base.py           # Abstract crawler interface
│   │   ├── rss_crawler.py    # RSS feed crawler
│   │   └── html_crawler.py   # HTML scraping crawler
│   ├── agents/
│   │   ├── base.py           # Abstract agent interface
│   │   ├── curator.py        # Curator agent (Phase 2)
│   │   └── synthesizer.py    # Synthesizer agent (Phase 2)
│   ├── routers/
│   │   ├── articles.py       # Article CRUD endpoints
│   │   ├── sources.py        # Source management endpoints
│   │   └── subscriptions.py  # Email subscription endpoints
│   ├── email/
│   │   ├── sender.py         # Resend integration + digest sender
│   │   └── templates/        # HTML email templates
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

**LLM cost management:**
- **Tiered processing pipeline:**
  1. **Haiku triage** — every new article gets a fast Haiku pass: binary relevance check (relevant to AI? yes/no) + preliminary score (1-10). Cost: ~$0.001/article.
  2. **Sonnet deep curation** — only articles scoring >= 5 in triage get full Sonnet processing: detailed claims extraction, nuanced topic tagging, announcement classification. Cost: ~$0.01/article.
  3. Expected filtering: ~40% of articles filtered at triage, reducing Sonnet calls by nearly half.
- Track token usage and cost per article in `curation_results` table (prompt_tokens, completion_tokens, cost_usd).
- Daily cost cap — configurable maximum daily LLM spend. When reached, queue remaining articles for next day.

**Observability:**
- **Crawler health endpoint** — `GET /health/crawlers` surfacing `crawl_logs` data: success rate per source, articles/crawl, error frequency, last successful crawl.
- **Agent decision logging** — every curator decision stored in `curation_results` with full traceability: input article ID, model used, score, topics, latency, token count.
- **Curation review UI** — admin page showing recent curation decisions side-by-side with the article. Allows manual override of scores/tags, feeding back into prompt tuning.
- **Cost dashboard** — `GET /stats/costs` with daily/weekly LLM spend broken down by model and agent type.

**Minimal MCP server (early preview):**
- Two read-only tools for dogfooding: `search_articles` (text search) + `get_digest` (latest digest)
- Ships as `mcp_server.py` using Python `mcp` SDK, connectable to Claude Desktop immediately
- Purpose: validate the MCP developer experience early, before investing in the full tool suite

**Frontend additions:**
- Importance score badges on articles
- Topic tag filters
- Daily digest page
- Crawler health + cost dashboards (admin)

**Key learning:** LLM tool calling, agent loops, structured output parsing, prompt engineering for curation, cost optimization.

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
- **Narrative tracking algorithm:**
  - **Storyline detection:** When a new article is embedded, compute cosine similarity against all active storyline centroids. If max similarity > 0.78, assign to that storyline. If no match, create a new storyline seeded by this article.
  - **Centroid update:** Storyline centroid = rolling weighted average of member article embeddings, with recent articles weighted higher (exponential decay, half-life = 14 days).
  - **Storyline merging:** Nightly batch job compares all active storyline centroids. If two centroids have similarity > 0.85, merge them (combine articles, recompute centroid, concatenate summaries for human review).
  - **Storyline lifecycle:** Storylines with no new articles for 30 days are marked inactive. They remain queryable but stop participating in centroid matching.
  - **Storyline metadata:** Each storyline has a title (LLM-generated from its top-3 articles), summary (regenerated weekly), article count, first/last activity dates.
  - **Thresholds (0.78 assignment, 0.85 merge) are configurable** and should be tuned empirically after accumulating ~500 articles.
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

**Goal:** Expand the minimal MCP server (shipped in Phase 2) into the full tool suite, with semantic search, narrative tools, and Q&A.

**New components:**
- **Full MCP server** — the Phase 2 server only exposed `search_articles` (text) and `get_digest`. Phase 4 upgrades `search_articles` to semantic search and adds the remaining tools, all powered by the ChromaDB + RAG infrastructure built in Phase 3:
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