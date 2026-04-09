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

Build a **reusable agent framework from scratch**, then implement three concrete agents on top of it.

**Agent Framework (you build this):**
- `BaseAgent` abstract class with lifecycle hooks: `perceive()` → `reason()` → `act()` → `reflect()`
- **Tool registry** — agents declare tools they can call (search, store, fetch, compare). Tools are Python functions with typed inputs/outputs, registered via decorator.
- **Message bus** — agents communicate via typed messages, not direct function calls. The Crawler Agent emits `ArticleCrawled` messages, the Curator Agent listens and emits `ArticleCurated`, the Synthesizer consumes curated articles.
- **Decision logging** — every agent decision (what it perceived, what it decided, what it did, what it learned) is logged for observability and debugging.

**Concrete Agents:**

| Agent | Role | Tools It Uses |
|-------|------|---------------|
| **Crawler Agent** | Fetches content, extracts text, normalizes to canonical schema | `fetch_url`, `extract_text`, `store_article`, `check_duplicate` |
| **Curator Agent** | Scores relevance, extracts claims, tags topics | `search_related`, `check_duplicate_topic`, `call_llm`, `store_curation` |
| **Synthesizer Agent** | Generates digests, answers questions | `search_articles`, `get_recent_curations`, `call_llm`, `store_digest` |

**Agent-to-agent pipeline:**
```
Crawler Agent → [ArticleCrawled] → Curator Agent → [ArticleCurated] → Synthesizer Agent
```

### 2. Memory (The Knowledge)

Each memory layer is a **distinct subsystem with its own API** — not just a database table, but an abstraction you interact with through a clean interface.

| Layer | Purpose | Tech | API Surface |
|-------|---------|------|-------------|
| **Short-term** | Current session context, conversation state | In-memory / Redis | `store(key, value, ttl)`, `recall(key)`, `forget(key)` |
| **Long-term** | Article embeddings, semantic search index | ChromaDB (vector store) | `embed(content)`, `search(query, k)`, `similar(embedding, threshold)` |
| **Structured** | Article metadata, source configs, relationships | SQLite → PostgreSQL | Standard CRUD via SQLAlchemy models |
| **Narrative** | Storyline clusters that evolve over time | Embeddings + clustering | `assign_to_storyline(article)`, `merge_storylines()`, `get_timeline(storyline_id)` |

**Memory Inspector UI** — a debug page that lets you see what each memory layer contains: browse stored embeddings, view storyline clusters, inspect session state, and query each layer directly. This makes the learning tangible — you can *see* what the agents remember.

### 3. Skills & MCP (The Interface)

Each capability is a **composable skill** — a self-contained unit that can be invoked standalone or chained with other skills. Skills are exposed as MCP tools so they're usable from Claude Desktop, Claude Code, or any MCP client.

| Skill | Description | Can Chain With |
|-------|-------------|----------------|
| `search_articles` | Semantic search across your entire corpus | `compare`, `summarize` |
| `get_digest` | Daily/weekly summary of top developments | `search_articles` |
| `trending_topics` | What's hot based on volume + velocity | `search_articles`, `get_narrative` |
| `compare` | Side-by-side analysis of announcements | `search_articles` |
| `get_narrative` | Track how a storyline has evolved | `summarize` |
| `ask_question` | Free-form Q&A grounded in articles | all skills |

**Skill chaining example:** "Compare Anthropic's and Google's agent approaches" →
1. `search_articles("Anthropic agents")` → top 5 results
2. `search_articles("Google agents")` → top 5 results
3. `compare(results_a, results_b)` → structured comparison

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

### Phase 1 — Agent Framework + Basic Crawling (Week 1–2)

**Goal:** Build the agent framework, implement the Crawler Agent as the first agent, crawl 5 blogs, show articles in a web UI, and send daily digest emails.

**Target sources:** Anthropic Blog, Google AI Blog, OpenAI Blog, Meta AI Blog, HuggingFace Blog (see Sources — Phase 1 above).

**Agent framework (build this first):**
- `BaseAgent` class with lifecycle: `perceive()` → `reason()` → `act()` → `reflect()`
- Tool registry — `@agent_tool` decorator to register callable tools with typed inputs/outputs
- Agent configuration — each agent declares its name, role description, and available tools
- Decision logging — every agent step is logged as structured JSON via `structlog`
- **Crawler Agent** — first concrete agent. Implements `perceive` (check sources for new content), `reason` (should I crawl this source now?), `act` (fetch + normalize + store), `reflect` (log results, update source status)

**Backend:**
- FastAPI app structure with proper project layout
- Source registry — config-driven list of sources with URL, type (RSS/HTML), schedule
- SQLite database — full schema defined in Data Model section above
- REST endpoints: `GET /articles`, `GET /articles/{id}`, `POST /crawl` (manual trigger)
- APScheduler — triggers Crawler Agent every 30 minutes
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

**Key learning:** Agent architecture (perceive/reason/act/reflect), tool registration pattern, async Python, web scraping, resilience patterns.

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
│   │   ├── base.py           # BaseAgent framework + lifecycle hooks
│   │   ├── tools.py          # @agent_tool decorator + tool registry
│   │   ├── messages.py       # Typed messages + message bus
│   │   ├── crawler.py        # Crawler Agent (Phase 1)
│   │   ├── curator.py        # Curator Agent (Phase 2)
│   │   └── synthesizer.py    # Synthesizer Agent (Phase 2)
│   ├── memory/
│   │   ├── base.py           # MemoryLayer abstract interface
│   │   ├── structured.py     # StructuredMemory (SQLite/Postgres)
│   │   ├── long_term.py      # LongTermMemory (ChromaDB) — Phase 3
│   │   ├── short_term.py     # ShortTermMemory (in-memory/Redis) — Phase 3
│   │   └── narrative.py      # NarrativeMemory (clustering) — Phase 3
│   ├── skills/
│   │   ├── base.py           # BaseSkill + @skill decorator — Phase 4
│   │   ├── registry.py       # Skill registry — Phase 4
│   │   ├── search.py         # search_articles skill
│   │   ├── compare.py        # compare skill
│   │   └── qa.py             # ask_question skill
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

### Phase 2 — Multi-Agent Pipeline + Memory Layers (Week 3–4)

**Goal:** Build the multi-agent pipeline (Crawler → Curator → Synthesizer), implement the memory subsystem with explicit APIs per layer, and add agent tools.

**Multi-agent pipeline:**
- **Message bus** — implement typed message passing between agents:
  - Crawler Agent emits `ArticleCrawled(article_id, source_id, url)`
  - Curator Agent listens for `ArticleCrawled`, emits `ArticleCurated(article_id, score, topics, claims)`
  - Synthesizer Agent listens for `ArticleCurated`, generates digests when triggered
- **Curator Agent** — second agent using the BaseAgent framework from Phase 1:
  - `perceive`: receive `ArticleCrawled` messages
  - `reason`: call LLM to score relevance, extract claims, assign topics
  - `act`: store curation results, emit `ArticleCurated`
  - `reflect`: log decision rationale, track accuracy over time
  - **Agent tools**: `search_related(query)` — find similar articles already in DB; `check_duplicate_topic(claims)` — detect if this covers the same ground as a recent article; `call_llm(prompt, model)` — invoke Claude API with cost tracking
- **Synthesizer Agent** — third agent:
  - `perceive`: collect today's curated articles
  - `reason`: group by theme, rank by importance
  - `act`: generate coherent digest via LLM, store result
  - `reflect`: log token usage and quality metrics
- **Daily Digest endpoint** — `GET /digest?date=2026-04-09`

**Memory subsystem:**
- **Structured memory** (SQLite) — `StructuredMemory` class wrapping SQLAlchemy with a clean API. This is the first memory layer, active since Phase 1 but now formalized.
- **Memory interface** — abstract `MemoryLayer` base class defining `store()`, `recall()`, `search()`, `forget()`. Each layer implements this interface.
- **Memory inspector UI** — debug page to browse each memory layer: view stored articles, see curation decisions, query the structured store directly. Makes learning tangible.

**LLM cost management:**
- **Tiered processing pipeline:**
  1. **Haiku triage** — every new article gets a fast Haiku pass: binary relevance + preliminary score (1-10). Cost: ~$0.001/article.
  2. **Sonnet deep curation** — only articles scoring >= 5 get full Sonnet processing: claims extraction, topic tagging, announcement classification. Cost: ~$0.01/article.
  3. Expected filtering: ~40% of articles filtered at triage, reducing Sonnet calls by nearly half.
- Track token usage and cost per article in `curation_results` table.
- Daily cost cap — configurable max daily LLM spend. When reached, queue remaining articles for next day.

**Observability:**
- **Crawler health endpoint** — `GET /health/crawlers` surfacing `crawl_logs` data.
- **Agent decision logging** — every agent decision stored with full traceability: input, model, output, latency, token count.
- **Curation review UI** — admin page showing curation decisions side-by-side with articles. Manual override feeds back into prompt tuning.
- **Cost dashboard** — `GET /stats/costs` with daily/weekly spend by model and agent.

**Minimal MCP server (early preview):**
- Two read-only tools for dogfooding: `search_articles` (text search) + `get_digest` (latest digest)
- Ships as `mcp_server.py` using Python `mcp` SDK, connectable to Claude Desktop immediately

**Frontend additions:**
- Importance score badges on articles
- Topic tag filters
- Daily digest page
- Memory inspector page (admin)
- Crawler health + cost dashboards (admin)

**Key learning:** Multi-agent coordination, message passing, agent tools, memory abstraction layers, LLM tool calling, prompt engineering, cost optimization.

---

### Phase 3 — Vector Memory + Intelligence (Week 5–6)

**Goal:** Activate the long-term and narrative memory layers. All 4 memory layers now operational.

**Long-term memory (ChromaDB):**
- Implement `LongTermMemory` class (extends `MemoryLayer` interface from Phase 2)
- `embed(content)` — generate embeddings using Voyage / sentence-transformers
- `search(query, k)` — vector similarity search across corpus
- `similar(embedding, threshold)` — find articles above similarity threshold
- Embed every article on ingest; store with metadata (source, date, topics, score)

**Short-term memory (session context):**
- Implement `ShortTermMemory` class (extends `MemoryLayer`)
- `store(key, value, ttl)` — store with time-to-live
- `recall(key)` — retrieve if not expired
- `forget(key)` — explicit removal
- Used by the conversational Q&A to track what the user just asked about

**Semantic search:** `GET /search?q=how is MCP adoption going`
  - Vector similarity search via `LongTermMemory.search()`
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

**Memory inspector upgrades:**
- Browse ChromaDB embeddings — see nearest neighbors for any article
- Visualize storyline clusters — which articles are grouped together and why
- View session memory — what the Q&A agent remembers about the current conversation
- All 4 memory layers now visible and queryable through the inspector

**Frontend additions:**
- Semantic search bar (replaces text search)
- Narrative/storyline pages with timelines
- Chat interface for Q&A

**Key learning:** Embeddings, vector databases, 4-layer memory architecture, RAG, retrieval strategies, session memory management.

---

### Phase 4 — Composable Skills + Full MCP (Week 7–8)

**Goal:** Refactor capabilities into composable, chainable skills. Expand the Phase 2 MCP server into the full tool suite.

**Skill framework (you build this):**
- `BaseSkill` class — each skill is a self-contained capability with typed input/output
- `@skill` decorator — register a function as a skill with name, description, input/output schemas
- **Skill chaining** — skills can call other skills. A `compare` skill calls `search_articles` twice then synthesizes. An `ask_question` skill orchestrates search → filter → generate.
- **Skill registry** — central registry that the MCP server reads from. Add a skill once, it's automatically available as an MCP tool.

**Concrete skills:**
```python
@skill("search_articles")
async def search_articles(query: str, limit: int = 5) -> list[Article]:
    """Semantic search across your entire corpus."""
    return await long_term_memory.search(query, limit)

@skill("compare")
async def compare(topic_a: str, topic_b: str) -> Comparison:
    """Side-by-side analysis of two topics."""
    results_a = await search_articles(topic_a)
    results_b = await search_articles(topic_b)
    return await synthesizer.compare(results_a, results_b)

@skill("ask_question")
async def ask_question(question: str, session_id: str) -> Answer:
    """Free-form Q&A — chains search, retrieval, and generation."""
    context = await short_term_memory.recall(session_id)
    articles = await search_articles(question)
    answer = await synthesizer.answer(question, articles, context)
    await short_term_memory.store(session_id, answer, ttl=3600)
    return answer
```

**Full MCP server:**
- Expand Phase 2 minimal server — every registered skill is automatically exposed as an MCP tool
- `search_articles` upgraded from text to semantic search
- Add `trending_topics`, `compare`, `get_narrative`, `ask_question`
- **Claude Desktop integration** — add to `claude_desktop_config.json`
- **Claude Code integration** — use your news corpus while coding

**Key learning:** Composable skill architecture, skill chaining, MCP protocol internals, tool design, server lifecycle.

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
mkdir -p backend/{crawlers,routers,agents,memory,skills,email/templates} frontend/src/components

# Start building Phase 1
touch backend/main.py backend/config.py backend/models.py backend/database.py
```

---

## Key Concepts You'll Learn

| Concept | Where You'll Learn It |
|---------|----------------------|
| **Agent framework design** | Phase 1 — BaseAgent, lifecycle hooks, tool registry |
| **Agent architecture (perceive → reason → act → reflect)** | Phase 1 — Crawler Agent as first concrete implementation |
| **Multi-agent coordination** | Phase 2 — message bus, Crawler → Curator → Synthesizer pipeline |
| **Agent tools** | Phase 2 — agents calling tools (search, LLM, store) with typed I/O |
| **Memory layer abstraction** | Phase 2 — MemoryLayer interface, StructuredMemory implementation |
| **Long-term memory (vector store)** | Phase 3 — ChromaDB, embeddings, LongTermMemory class |
| **Short-term memory (session)** | Phase 3 — ShortTermMemory with TTL, conversation context |
| **Narrative memory (graph/clustering)** | Phase 3 — storyline detection, centroid matching, merging |
| **RAG (Retrieval Augmented Generation)** | Phase 3 — Q&A grounded in articles via memory layers |
| **Composable skills** | Phase 4 — BaseSkill, skill chaining, skill registry |
| **MCP protocol** | Phase 2 (minimal) → Phase 4 (full) — skills exposed as MCP tools |
| **LLM tool calling** | Phase 2 — structured extraction, cost-tiered processing |
| **Prompt engineering** | Throughout — curation prompts, digest generation, Q&A |
| **Async Python patterns** | Phase 1 — concurrent crawling with httpx |

---

*Start with Phase 1. Build the agent framework first, then the crawler. Every phase adds a new learning pillar: agents → memory → skills.*