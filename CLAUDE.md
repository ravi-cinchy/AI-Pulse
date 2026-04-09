# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI Pulse is a **learning project** for building AI agents, memory systems, and composable skills. The domain is AI news aggregation — crawl, curate, and synthesize AI news — but the focus is on the architecture patterns: agent frameworks, multi-agent coordination, layered memory, and MCP-based skill composition.

## Tech Stack

- **Backend:** FastAPI (async Python), SQLite (Phase 1) -> PostgreSQL (later)
- **Frontend:** React + Vite
- **Vector DB:** ChromaDB
- **LLM:** Anthropic API — Haiku 4.5 for triage, Sonnet 4 for deep curation
- **Scheduler:** APScheduler
- **MCP:** Python `mcp` SDK
- **Scraping:** httpx + trafilatura / BeautifulSoup
- **Embeddings:** Anthropic Voyage or sentence-transformers
- **Logging:** structlog (structured JSON)
- **Email:** Resend (daily digest delivery)

## Architecture — Three Learning Pillars

1. **Agents** — Reusable `BaseAgent` framework with perceive/reason/act/reflect lifecycle, `@agent_tool` decorator for tool registration, typed message bus for agent-to-agent communication. Three concrete agents: Crawler, Curator, Synthesizer.
2. **Memory** — 4 layers, each implementing a `MemoryLayer` interface: Structured (SQLite), Long-term (ChromaDB vectors), Short-term (in-memory/Redis with TTL), Narrative (embedding clustering). Memory inspector UI for debugging.
3. **Skills & MCP** — Composable `BaseSkill` framework with `@skill` decorator and skill registry. Skills can chain other skills. All skills auto-exposed as MCP tools via the Python `mcp` SDK.

## Project Structure (Target)

```
ai-pulse/
├── backend/
│   ├── main.py              # FastAPI app + scheduler setup
│   ├── mcp_server.py         # MCP server entry point (Phase 2+)
│   ├── config.py             # Source registry, settings
│   ├── models.py             # SQLAlchemy + Pydantic models
│   ├── database.py           # DB connection + migrations
│   ├── normalization.py      # Content cleaning + canonical schema
│   ├── dedup.py              # URL normalization + content hash dedup
│   ├── crawlers/
│   │   ├── base.py           # Abstract crawler interface
│   │   ├── rss_crawler.py
│   │   └── html_crawler.py
│   ├── agents/               # BaseAgent framework, tool registry, message bus, concrete agents
│   ├── memory/               # MemoryLayer interface + 4 layer implementations
│   ├── skills/               # BaseSkill framework, skill registry, composable skills (Phase 4)
│   ├── routers/
│   │   ├── articles.py       # Article CRUD endpoints
│   │   ├── sources.py        # Source management endpoints
│   │   └── subscriptions.py  # Email subscription endpoints
│   ├── email/
│   │   ├── sender.py         # Resend integration
│   │   └── templates/        # HTML email templates
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── components/
│   │   └── api.js
│   └── package.json
└── docs/requirements.md      # Full design spec
```

## Common Commands

```bash
# Backend
cd backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --reload --port 8000

# Frontend
cd frontend
npm install
npm run dev

# Run backend tests
cd backend
pytest
pytest tests/test_crawlers.py -k "test_name"  # single test

# Run frontend tests
cd frontend
npm test
```

## Key API Endpoints

- `GET /articles` — list articles (with filters)
- `GET /articles/{id}` — article detail
- `POST /crawl` — manual crawl trigger
- `POST /subscribe` — email subscription
- `POST /unsubscribe` — remove subscription
- `GET /health/crawlers` — crawler health dashboard (Phase 2+)
- `GET /stats/costs` — LLM cost dashboard (Phase 2+)
- `GET /digest?date=YYYY-MM-DD` — daily digest (Phase 2+)
- `GET /search?q=...` — semantic search (Phase 3+)
- `POST /ask` — conversational Q&A via RAG (Phase 3+)

## Build Phases

The project is built incrementally. See `docs/requirements.md` for full details.

| Phase | Focus | Key Learning |
|-------|-------|--------------|
| 1 | Agent Framework + Crawling | BaseAgent, tool registry, perceive/reason/act/reflect lifecycle |
| 2 | Multi-Agent Pipeline + Memory | Message bus, agent tools, MemoryLayer interface, memory inspector |
| 3 | Vector Memory + Intelligence | 4-layer memory (structured/long-term/short-term/narrative), RAG, embeddings |
| 4 | Composable Skills + MCP | BaseSkill, skill chaining, skill registry, full MCP server |
| 5 | Scale Sources | New crawler plugins (ArXiv, Reddit, Twitter/X, HN, YouTube, GitHub) |

## Phase 1 Data Sources

- Anthropic Blog (anthropic.com/news)
- Google AI Blog (blog.google/technology/ai/)
- OpenAI Blog (openai.com/blog)
- Meta AI Blog (ai.meta.com/blog/)
- HuggingFace Blog (huggingface.co/blog)

## Design Decisions

- **Agent framework first:** Build `BaseAgent` with lifecycle hooks before implementing any concrete agent. Crawler Agent is the first test of the framework.
- **Message bus for agent coordination:** Agents communicate via typed messages (`ArticleCrawled`, `ArticleCurated`), not direct function calls. Keeps agents decoupled.
- **Memory as an abstraction:** Each memory layer implements a `MemoryLayer` interface (`store`, `recall`, `search`, `forget`). Concrete implementations are swappable.
- **Skills are composable:** Each skill can call other skills. The MCP server reads from the skill registry — add a skill once, it's automatically an MCP tool.
- **Crawler plugin architecture:** All crawlers inherit from an abstract base class in `crawlers/base.py`. New sources are added as plugins.
- **Config-driven sources:** Source list with URL, type (RSS/HTML), and schedule lives in `config.py`, not hardcoded in crawlers.
- **SQLite first:** Start with SQLite via aiosqlite for simplicity; schema should be migration-ready for PostgreSQL.
- **Crawler resilience:** 30s request timeout, 3 retries with exponential backoff (2s/8s/32s), 5-minute hard cap per crawl run, partial failure isolation across sources.
- **Content normalization:** RawArticle -> CleanArticle canonical schema via trafilatura. All downstream components consume the same clean format.
- **Deduplication:** URL normalization + SHA-256 content hash (Phase 1), embedding similarity gate in Phase 2.
- **LLM cost tiering:** Haiku for cheap triage, Sonnet only for articles scoring >= 5. Daily cost cap with queuing.
- **Early MCP dogfooding:** Minimal MCP server ships in Phase 2 (text search + digest), full skill suite in Phase 4.
- **RAG pipeline (Phase 3):** Embed question -> vector similarity search -> re-rank with LLM -> generate grounded answer with citations.
