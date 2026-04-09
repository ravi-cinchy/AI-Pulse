# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI Pulse is an AI-powered news aggregation and conversation platform that crawls, curates, and synthesizes AI news. It provides a conversational interface and an MCP server for querying the knowledge base from Claude Desktop/Code.

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

## Architecture

Three core subsystems:

1. **Agents** — Crawler (fetches content), Curator (scores/tags via Claude API), Synthesizer (generates digests, answers questions). Each follows a perceive-reason-act-reflect loop.
2. **Memory** — Short-term (in-memory/Redis), long-term (ChromaDB vectors), structured (SQLite/Postgres), narrative tracking (graph relationships + embeddings).
3. **MCP Server** — Exposes tools: `search_articles`, `get_digest`, `trending_topics`, `compare_announcements`, `get_narrative`, `ask_question`.

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
│   ├── agents/               # Curator, Synthesizer agents (Phase 2+)
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

| Phase | Focus | Key Deliverable |
|-------|-------|-----------------|
| 1 | Foundation | Crawl 5 blogs, feed UI, daily digest emails via Resend |
| 2 | Agent Brain | Curator agent (Haiku triage + Sonnet curation), minimal MCP server, observability |
| 3 | Memory | ChromaDB, semantic search, RAG Q&A, narrative tracking |
| 4 | Full MCP | Expand MCP server with semantic search, trending, narratives, Q&A tools |
| 5 | Scale Sources | ArXiv, Reddit, Twitter/X, HN, YouTube, GitHub |

## Phase 1 Data Sources

- Anthropic Blog (anthropic.com/news)
- Google AI Blog (blog.google/technology/ai/)
- OpenAI Blog (openai.com/blog)
- Meta AI Blog (ai.meta.com/blog/)
- HuggingFace Blog (huggingface.co/blog)

## Design Decisions

- **Crawler plugin architecture:** All crawlers inherit from an abstract base class in `crawlers/base.py`. New sources are added as plugins.
- **Config-driven sources:** Source list with URL, type (RSS/HTML), and schedule lives in `config.py`, not hardcoded in crawlers.
- **SQLite first:** Start with SQLite via aiosqlite for simplicity; schema should be migration-ready for PostgreSQL.
- **Crawler resilience:** 30s request timeout, 3 retries with exponential backoff (2s/8s/32s), 5-minute hard cap per crawl run, partial failure isolation across sources.
- **Content normalization:** RawArticle -> CleanArticle canonical schema via trafilatura. All downstream components consume the same clean format.
- **Deduplication:** URL normalization + SHA-256 content hash (Phase 1), embedding similarity gate in Phase 2.
- **LLM cost tiering:** Haiku for cheap triage, Sonnet only for articles scoring >= 5. Daily cost cap with queuing.
- **Agent loop pattern:** Agents process items sequentially — fetch unprocessed, run through LLM, store result. Keep this loop simple and observable.
- **Early MCP dogfooding:** Minimal MCP server ships in Phase 2 (text search + digest), full suite in Phase 4.
- **RAG pipeline (Phase 3):** Embed question -> vector similarity search -> re-rank with LLM -> generate grounded answer with citations.
