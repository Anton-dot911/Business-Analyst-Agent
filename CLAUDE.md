# CLAUDE.md — Insight (Business Analyst Agent)

## What this project is
Chat agent that answers business questions by routing to SQL (quantitative, demo company DB), RAG (internal policy docs), or Web search — or a combination — with charts, citations, and a transparent "How I answered" trace. Spec: `docs/TZ.md`. Plan: `docs/PLAN.md` — one task per session, in order.

## Stack
- Backend: Python 3.12, FastAPI, Pydantic v2, `anthropic` SDK, Supabase (Postgres + pgvector)
- Frontend: React 18 + Vite + TS(strict) + Tailwind + Recharts
- Streaming: SSE from FastAPI to frontend
- Reuse from DocFlow: `app/llm/client.py` wrapper pattern, Meter integration, repo layout

## Commands
- Backend dev: `cd backend && uv run fastapi dev app/main.py`
- Tests: `uv run pytest` | Lint: `uv run ruff check . && uv run mypy app`
- Seed demo data: `uv run python -m scripts.seed_demo`
- Index documents: `uv run python -m scripts.index_docs`
- Evals: `make eval-router | eval-sql | eval-rag | eval` (all)

## Hard rules
1. SQL safety is enforced by the database, not the prompt: executor connects as role `insight_readonly` (SELECT-only on schema `demo_company`), statement_timeout 10s, row limit 500, single statement only (reject on `;` outside literals). Tests must prove INSERT/UPDATE/DELETE and out-of-allowlist access fail.
2. Router, SQL generator: temperature 0, structured outputs via tool use + Pydantic validation (same retry rule as DocFlow: one retry with error).
3. RAG answers may contain ONLY claims supported by retrieved chunks, with inline `[n]` citations. If retrieval is empty/irrelevant → explicit "not found in documents" response. Never blend web knowledge into RAG answers.
4. Web-route facts must be marked external and carry URLs. Synthesizer keeps internal vs external facts visually separated.
5. Every user request creates a `trace` row before any LLM call; every step appends to `trace.steps`. The "How I answered" UI reads only from trace — no shadow logic.
6. Prompts in `backend/prompts/*.v<N>.md`, never inline. All LLM calls via metered client.
7. Ambiguous questions route to `clarify` — the agent asks, it does not guess. Treat clarify as success, not failure, in evals.
8. No write operations on demo data from the app, ever.

## Structure
```
backend/app/
  routes/chat.py        # POST /api/chat (SSE)
  services/router.py    # route classification
  services/sql_route.py # generate -> execute -> self-correct(1) -> result
  services/rag_route.py # retrieve -> answer with citations
  services/web_route.py # Claude web search tool
  services/synth.py     # merge route results, chart spec decision
  services/trace.py
  llm/  repos/  models/
scripts/seed_demo.py  scripts/index_docs.py
data/golden/{router,sql,rag}.jsonl
```

## Testing conventions
- Route services unit-tested with mocked LLM; SQL executor integration-tested against local Postgres with the readonly role.
- Injection test set `tests/test_sql_safety.py` is a release gate.
- Eval scripts follow DocFlow pattern: JSONL in, metrics table out, regression check vs previous run.
