# PLAN.md — Insight Implementation Plan

One task = one session. Contracts are verbatim.

---

## Contracts

### Models

```python
from enum import Enum
from pydantic import BaseModel, Field

class Route(str, Enum):
    sql = "sql"
    rag = "rag"
    web = "web"
    clarify = "clarify"

class RouterDecision(BaseModel):
    routes: list[Route] = Field(min_length=1, max_length=3)
    reasoning: str
    clarify_question: str | None = None   # required iff routes == [clarify]

class SqlAttempt(BaseModel):
    sql: str
    ok: bool
    error: str | None
    row_count: int | None

class SqlRouteResult(BaseModel):
    attempts: list[SqlAttempt]            # 1 or 2 (self-correction)
    columns: list[str]
    rows: list[list]                      # ≤500
    summary: str

class ChartSpec(BaseModel):
    kind: str                             # "line" | "bar" | "none"
    x_key: str | None
    y_keys: list[str] = []
    title: str | None

class RagChunk(BaseModel):
    doc_title: str
    section: str | None
    text: str
    score: float

class RagRouteResult(BaseModel):
    found: bool
    answer_md: str                        # with [n] citations; empty if not found
    chunks: list[RagChunk]

class WebRouteResult(BaseModel):
    answer_md: str
    sources: list[str]                    # URLs

class FinalAnswer(BaseModel):
    answer_md: str
    chart: ChartSpec
    trace_id: str
```

### Trace DDL

```sql
create table traces (
  id uuid primary key default gen_random_uuid(),
  session_id uuid not null,
  question text not null,
  routes text[] not null default '{}',
  steps jsonb not null default '[]',   -- [{name, started_at, latency_ms, cost_usd, detail}]
  total_cost numeric(10,5), total_latency_ms int,
  created_at timestamptz not null default now()
);
```

### Demo company schema (`demo_company`)
Tables: `products(id, sku, name, category, unit_cost, unit_price)`, `customers(id, name, segment, city)`, `orders(id, customer_id, order_date, channel)`, `order_items(order_id, product_id, qty, unit_price)`. Seed: ~10k order rows across 24 months, 6 categories, seasonal pattern. Column comments in Ukrainian+English (they feed the SQL prompt).

### API
```
POST /api/chat        {session_id, message} -> SSE stream:
  events: token | route_decision | step_done | chart | final(FinalAnswer)
GET  /api/traces/{id} full trace for "How I answered"
GET  /api/suggestions 8 example questions
```

---

## Tasks

**T1. Scaffold + demo data.** Project via scaffolder; `demo_company` schema, seeder (deterministic seed), readonly role + grants; `scripts/seed_demo.py` idempotent.
DoD: seed twice → same row counts; readonly role proven SELECT-only by tests.

**T2. SQL executor + safety.** `sql_route.execute(sql)` with role, timeout, row limit, single-statement guard. Injection test suite.
DoD: `test_sql_safety.py` green; malicious set (DROP, UPDATE, multi-statement, pg_sleep) all rejected.

**T3. SQL generation + self-correction.** DDL+comments+5 few-shot pairs in `prompts/sql_gen.v1.md`; on execution error, one corrective round with the error message.
DoD: llm smoke: 5 sample questions → ≥4 executable first try.

**T4. Chat backbone.** SSE endpoint, session history persistence, streaming plumbing frontend↔backend, trace creation per message.
DoD: echo-mode chat streams end-to-end; trace row written.

**T5. Router.** `prompts/router.v1.md` (3 few-shots per route incl. mixed + clarify); RouterDecision structured output; clarify path returns question to user and pauses.
DoD: unit tests with mocked LLM for all branches; llm smoke on 8 canonical questions.

**T6. Charts.** Synthesizer decides ChartSpec from SqlRouteResult shape (heuristics first, LLM only if ambiguous); Recharts renderer for line/bar.
DoD: component tests: time series → line, categorical → bar, single scalar → none.

**T7. RAG indexing.** 15 demo policy docs (markdown, UA business flavor); section-based chunking (~500 tokens, heading metadata); embedding pipeline into pgvector; `scripts/index_docs.py`.
DoD: reindex idempotent; retrieval smoke returns expected doc for 5 probe queries. Embedding model choice documented in `docs/decisions.md` (compare 2 options on the retrieval probes).

**T8. RAG answering.** Hybrid retrieval (vector + keyword ilike boost); `prompts/rag_answer.v1.md` (citations format, refusal rule); found=false path.
DoD: llm smoke: 5 answerable + 2 unanswerable behave correctly.

**T9. Web route + Synthesizer.** Claude web search tool call, source extraction; synthesizer merges multi-route results, marks external facts, produces FinalAnswer.
DoD: mixed question (scenario 3 from TZ) produces separated internal/external answer with URLs.

**T10. "How I answered" UI.** Expandable panel per answer: routes, SQL text, chunks, search queries, per-step latency/cost from trace.
DoD: matches trace exactly for the three TZ scenarios.

**T11. Evals.** Three datasets via Goldsmith (router 60, sql 30 result-compare, rag 25); `make eval` aggregate + regression gate. Route-miss severity metric (dangerous misroutes counted separately).
DoD: metrics table by dataset; router accuracy ≥92% or documented iteration log toward it.

**T12. Demo & release.** Suggestion chips, demo session without auth, rate limit; README(EN) with metrics, architecture, decisions (router design, sql safety, embeddings choice); deploy; video.
DoD: ТЗ §9 checklist green.

---

## Session prompt template
> Read CLAUDE.md and docs/PLAN.md. Implement task T<N> only. Contracts are verbatim — ask before deviating. Finish with tests green and a short decision summary.
