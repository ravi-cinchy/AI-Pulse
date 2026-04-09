# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI Pulse is an AI-powered news aggregation and conversation platform that crawls, curates, and synthesizes AI news. It provides a conversational interface and an MCP server for querying the knowledge base from Claude Desktop/Code.

## Tech Stack

- **Backend:** FastAPI (async Python), SQLite (Phase 1) -> PostgreSQL (later)
- **Frontend:** React + Vite
- **Vector DB:** ChromaDB
- **LLM:** Anthropic API — Claude Sonnet 4 for curation, Haiku 4.5 for lightweight tasks
- **Scheduler:** APScheduler
- **MCP:** Python `mcp` SDK
- **Scraping:** httpx + trafilatura / BeautifulSoup
- **Embeddings:** Anthropic Voyage or sentence-transformers

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
│   ├── config.py             # Source registry, settings
│   ├── models.py             # SQLAlchemy + Pydantic models
│   ├── database.py           # DB connection + migrations
│   ├── crawlers/
│   │   ├── base.py           # Abstract crawler interface
│   │   ├── rss_crawler.py
│   │   └── html_crawler.py
│   ├── routers/
│   │   ├── articles.py       # Article CRUD endpoints
│   │   └── sources.py        # Source management endpoints
│   ├── agents/               # Curator, Synthesizer agents (Phase 2+)
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── components/
│   │   └── api.js
│   └── package.json
└── project-idea.md           # Full design spec
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
- `GET /digest?date=YYYY-MM-DD` — daily digest (Phase 2+)
- `GET /search?q=...` — semantic search (Phase 3+)
- `POST /ask` — conversational Q&A via RAG (Phase 3+)

## Build Phases

The project is built incrementally. See `project-idea.md` for full details.

| Phase | Focus | Key Deliverable |
|-------|-------|-----------------|
| 1 | Foundation | Crawl 5 blogs, REST API, React feed UI |
| 2 | Agent Brain | Curator agent (Claude API scoring/tagging), daily digests |
| 3 | Memory | ChromaDB, semantic search, RAG Q&A, narrative tracking |
| 4 | MCP Server | Python MCP server exposing knowledge base as tools |
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
- **Agent loop pattern:** Agents process items sequentially — fetch unprocessed, run through LLM, store result. Keep this loop simple and observable.
- **RAG pipeline (Phase 3):** Embed question -> vector similarity search -> re-rank with LLM -> generate grounded answer with citations.
