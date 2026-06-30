# Agent Progress Tracker

> This file is the **single source of truth** for implementation state.
> Every agent MUST update this file before finishing its task.

---

## Current Phase

**Phase 6 — Refinement** ✅ COMPLETED
**All phases complete.**

---

## Phase Summary

| Phase | Name | Status | Issue |
|-------|------|--------|-------|
| 0 | Autonomous Workflow Setup | ✅ Done | — |
| 1 | Foundation | ✅ Done | #1 |
| 2 | Unstructured Pipeline | ✅ Done | #2 |
| 3 | Structured Pipeline | ✅ Done | #3 |
| 4 | Benchmark Runner | ✅ Done | #4 |
| 5 | Reports | ✅ Done | #5 |
| 6 | Refinement | ✅ Done | #6 |

---

## Completed Work

### Phase 0 — Autonomous Workflow Setup
- Created autonomous agent workflow: `.github/workflows/claude-agent.yml`
- Created agent entrypoint script: `scripts/agent-entrypoint.sh`
- Created documentation update hook: `scripts/update-agent-docs.sh`
- Created Claude Code settings: `.claude/settings.json`
- Created custom slash command: `.claude/commands/update-progress.md`
- Created context documents: `docs/agents/` (PROGRESS, DECISIONS, CONTEXT, ROADMAP)
- Created issue templates: `.github/ISSUE_TEMPLATE/`
- Created GitHub labels: agent, phase-1 through phase-6, in-progress, completed, blocked
- Created 6 phase issues on GitHub

### Phase 1 — Foundation
- `pyproject.toml` — all dependencies, CLI entrypoint (`app`), uv configuration
- `uv.lock` — locked dependencies (generated via `uv sync`)
- `docker-compose.yml` — Qdrant service on port 6333
- `.env.example` — environment variables template
- `config.yaml` — default config (model, chunk_size, top_k, temperature, qdrant_url, dataset_path)
- `app/src/tfm_app/__init__.py` — package root
- `app/src/tfm_app/core/config.py` — Pydantic Settings v2, YAML config loader
- `app/src/tfm_app/core/models.py` — `ChatResponse`, `ResponseMetadata`, `BenchmarkQuestion`, `BenchmarkResult`, `BenchmarkRun`
- `app/src/tfm_app/core/llm_client.py` — Anthropic async client with retry logic (pre-existing)
- `app/src/tfm_app/core/prompts.py` — versioned prompt templates (unstructured-v1, structured-v1, judge-v1)
- `app/src/tfm_app/core/logging.py` — structured logging with Rich
- `app/src/tfm_app/core/utils.py` — TimingContext manager, cost calculator
- `app/src/tfm_app/datasets/loader.py` — loads services.json, events.json, nodes.json, edges.json, benchmark.json
- `app/src/tfm_app/datasets/validator.py` — dataset consistency checks
- `app/src/tfm_app/cli/main.py` — Typer app with ingest, chat, benchmark, reports subcommands
- `app/src/tfm_app/api/main.py` — FastAPI skeleton with POST /chat, POST /benchmark/run, GET /benchmark/runs/{id}, POST /reports/generate
- `tests/test_datasets.py` — 12 tests validating 52 services, 64 events, 201 nodes, 669 edges, 120 questions
- `tests/test_llm_client.py` — 9 tests for LLM client retry behaviour (pre-existing)

**Acceptance criteria met:**
- `uv sync` installs without errors ✅
- `uv run app --help` shows all subcommands ✅
- `uv run pytest tests/test_datasets.py -v` — 12/12 pass ✅
- Dataset validation: Services: 52, Events: 64, Questions: 120, Bad refs: 0 ✅
- `docker-compose.yml` ready for `docker compose up -d qdrant` ✅

### Phase 2 — Unstructured Pipeline
- `app/src/tfm_app/approaches/unstructured/__init__.py` — package marker
- `app/src/tfm_app/approaches/unstructured/ingestion.py` — parse all `docs/**/*.md` and `services/*/README.md` + `architecture.md` into LlamaIndex Document objects
- `app/src/tfm_app/approaches/unstructured/indexing.py` — embed chunks via LlamaIndex, upsert to Qdrant collection `credityflow_docs`
- `app/src/tfm_app/approaches/unstructured/retrieval.py` — similarity search, top-k retrieval returning typed `RetrievedChunk` objects
- `app/src/tfm_app/approaches/unstructured/service.py` — `UnstructuredService.chat(question) -> ChatResponse` with ingest/load/chat lifecycle
- CLI `app ingest unstructured` — end-to-end document parsing + Qdrant upsert
- CLI `app chat --approach unstructured` — interactive REPL
- `tests/test_unstructured.py` — unit tests with all I/O mocked (Qdrant, Anthropic)
- `tests/test_indexing.py` — unit tests for indexing layer

**Acceptance criteria met:**
- `UnstructuredService.chat()` returns `ChatResponse` contract ✅
- All I/O (Qdrant, Anthropic API) mocked in tests ✅
- Coverage ≥90% on all new files ✅

### Phase 3 — Structured Pipeline
- `app/src/tfm_app/approaches/structured/__init__.py` — package marker
- `app/src/tfm_app/approaches/structured/ingestion.py` — load `structured/services.json`, `events.json`, `graph/nodes.json`, `graph/edges.json` into `StructuredDataset`
- `app/src/tfm_app/approaches/structured/graph.py` — `build_graph(dataset) -> nx.DiGraph` with node enrichment from services.json and events.json; canonical node ID helpers
- `app/src/tfm_app/approaches/structured/query.py` — fuzzy entity extraction (rapidfuzz), 1-hop neighbourhood retrieval, keyword fallback, `EntityIndex`, `QueryResult`; `build_entity_index` + `query_graph`
- `app/src/tfm_app/approaches/structured/service.py` — `StructuredService.chat(question) -> ChatResponse` with ingest/load/chat lifecycle
- CLI `app ingest structured` — loads JSON files, builds in-memory NetworkX graph
- CLI `app chat --approach structured` — interactive REPL
- `tests/test_structured.py` — unit tests with all I/O mocked (Anthropic), in-memory graph fixture
- Shared prompts via `core/prompts.py` (see ADR-007 — no per-approach `prompting.py` module)

**Acceptance criteria met:**
- Graph builds from real dataset: 201 nodes, 669 edges ✅
- `StructuredService.chat()` returns `ChatResponse` contract ✅
- All I/O (Anthropic API) mocked in tests ✅
- Coverage ≥90% on all new files ✅

### Phase 4 — Benchmark Runner
- `app/src/tfm_app/benchmark/runner.py` — `run_benchmark(approach, config) -> BenchmarkRun`; iterates 120 questions, calls service, handles errors
- `app/src/tfm_app/benchmark/scoring.py` — Level 1 heuristic scoring (rapidfuzz similarity, entity presence, source verification) → `correct|partial|incorrect|error`
- `app/src/tfm_app/benchmark/persistence.py` — `save_run(run, outputs_dir)` serialises `BenchmarkRun` to `outputs/runs/{timestamp}_{approach}/result.json`
- CLI `app benchmark run --approach all` — runs full 120-question suite for one or both approaches
- FastAPI `POST /benchmark/run`, `GET /benchmark/runs/{id}` — wired in `api/main.py`
- `tests/test_benchmark.py` — unit tests for scoring, runner, persistence (all I/O mocked)

**Acceptance criteria met:**
- `BenchmarkRun` result persisted to `outputs/runs/` ✅
- Scoring produces `correct|partial|incorrect|error` classification ✅
- Coverage ≥90% on all new files ✅

### Phase 5 — Reports

- `app/src/tfm_app/reports/__init__.py` — package marker re-exporting `generate_run_report` and `generate_comparative_report`
- `app/src/tfm_app/reports/generator.py` — generates per-run and comparative reports in Markdown, JSON, and CSV formats
  - `generate_run_report(run, outputs_dir)` → writes `report.md`, `report.json`, `report.csv` to `outputs/reports/<run_id>/`
  - `generate_comparative_report(run_a, run_b, outputs_dir)` → writes `compare.md`, `compare.json`, `compare.csv` to `outputs/reports/compare-<a>-vs-<b>/`
  - CSV uses `_RUN_CSV_FIELDS` (16 columns, one row per question) and `_COMPARE_CSV_FIELDS` (22 columns, joined by question_id with deltas)
  - Markdown includes: summary table, category/difficulty/knowledge-type breakdowns, correct/incorrect examples, top latency, top divergence
  - JSON includes: all aggregated metrics plus per-group breakdowns
- CLI `app reports generate --latest` and `app reports compare --run-a <id> --run-b <id>` — fully wired in `cli/main.py`
- FastAPI `POST /reports/generate` — fully wired in `api/main.py`
- `tests/test_reports.py` — 64 tests covering all public functions, helpers, CLI, and API endpoints (including CSV export)

**Acceptance criteria met:**
- Report includes overall accuracy, latency comparison, cost comparison, per-category/difficulty breakdown ✅
- Markdown report is readable and suitable for thesis inclusion ✅
- CSV report has one row per question with all metrics (16 columns) ✅
- Comparison report highlights differences between approaches with delta columns ✅
- Coverage ≥90% on all new files (`generator.py` at 100%) ✅

**ADRs documented:** ADR-008 (inline templates instead of Jinja2; CSV design rationale)

---

## Phase 6 — Refinement

### Scoring improvements
- `app/src/tfm_app/benchmark/scoring.py` — added `CATEGORY_THRESHOLDS` dict with per-category
  (correct, partial) thresholds for all 6 benchmark categories: `factual`, `relational`,
  `explanatory`, `discovery`, `impact`, `ambiguous`
- Added `get_thresholds(category)` helper and updated `classify()` to accept custom thresholds
- Updated `score_response()` to accept optional `category` parameter (passed from benchmark runner)
- Updated `benchmark/runner.py` to pass `question.category` to `score_response`

### Confidence scoring
- `app/src/tfm_app/approaches/unstructured/service.py` — confidence = mean of LlamaIndex
  cosine-similarity chunk scores, clamped to [0, 1]; `None` when no chunks retrieved
- `app/src/tfm_app/approaches/structured/service.py` — confidence = mean of rapidfuzz graph-query
  scores (normalised from 0–100 to 0–1); `None` when no results

### Streamlit UI
- `app/src/tfm_app/ui/helpers.py` — pure, testable helper functions:
  `format_chat_response`, `confidence_label`, `compare_responses`, `get_run_options`,
  `summarise_run`, `filter_results_by_classification`, `results_to_table_rows`
- `app/src/tfm_app/ui/app.py` — Streamlit multi-page entrypoint (sidebar navigation)
- `app/src/tfm_app/ui/pages/chat.py` — single-approach interactive chat with session history,
  sources, context preview, and metadata (latency, confidence, cost)
- `app/src/tfm_app/ui/pages/compare.py` — side-by-side comparison with cross-similarity metric,
  latency delta, and per-approach confidence/sources
- `app/src/tfm_app/ui/pages/benchmark.py` — run selector, top-level metrics, per-category and
  per-difficulty breakdowns, filterable results table
- `app/src/tfm_app/ui/pages/reports.py` — single-run and comparative report viewer with
  on-demand report generation
- `app/src/tfm_app/ui/pages/__init__.py` — package marker

### Documentation
- `README.md` — full setup, usage, benchmark, evaluation, metrics, dataset and structure docs

### Tests (70 new tests, all passing)
- `tests/test_phase6_scoring.py` — `get_thresholds`, `classify` with custom thresholds,
  `score_response` with category parameter (27 tests)
- `tests/test_phase6_confidence.py` — unstructured and structured confidence scoring,
  boundary clamping, None when no results (9 tests)
- `tests/test_phase6_ui_helpers.py` — all 7 helper functions, boundary and edge cases (34 tests)

**Acceptance criteria met:**
- `uv run streamlit run app/src/tfm_app/ui/app.py` starts UI ✅
- Chat page works for both approaches ✅
- Compare page shows answers side-by-side with scoring ✅
- Benchmark dashboard shows run history and metrics ✅
- README covers: prerequisites, setup, running each component, interpreting results ✅
- Coverage ≥90% (97.62% across 440 tests) ✅
- ruff lint + format: zero errors ✅

## In Progress

_Nothing in progress — all 6 phases complete._

---

## Known Blockers

- None for local execution.
- For GitHub Actions: `ANTHROPIC_API_KEY` must be set as a repository secret.

---

## File Map (grows with each phase)

```
TFM_UE_IA/
├── CLAUDE.md                    ✅ (project guidance)
├── README.md                    ✅ Phase 6 (full setup + usage docs)
├── pyproject.toml               ✅ Phase 1
├── uv.lock                      ✅ Phase 1
├── config.yaml                  ✅ Phase 1
├── docker-compose.yml           ✅ Phase 1
├── .env.example                 ✅ Phase 1
├── docs/
│   ├── agents/                  ✅ (agent context docs)
│   │   ├── PROGRESS.md         ✅ this file
│   │   ├── DECISIONS.md        ✅
│   │   ├── CONTEXT.md          ✅
│   │   └── ROADMAP.md          ✅
│   ├── company-overview.md     ✅ (dataset)
│   ├── dataset-summary.md      ✅ (dataset)
│   ├── events/catalog.md       ✅ (dataset)
│   ├── questions/benchmark.json ✅ (dataset — 120 questions)
│   └── teams/                  ✅ (dataset)
├── services/                   ✅ (52 microservice docs)
├── structured/                 ✅ (services.json, events.json, graph/)
├── app/src/tfm_app/
│   ├── __init__.py              ✅ Phase 1
│   ├── core/
│   │   ├── __init__.py         ✅ Phase 1
│   │   ├── config.py           ✅ Phase 1
│   │   ├── models.py           ✅ Phase 1
│   │   ├── llm_client.py       ✅ Phase 1
│   │   ├── prompts.py          ✅ Phase 1 (shared, see ADR-007)
│   │   ├── logging.py          ✅ Phase 1
│   │   └── utils.py            ✅ Phase 1
│   ├── datasets/
│   │   ├── __init__.py         ✅ Phase 1
│   │   ├── loader.py           ✅ Phase 1
│   │   └── validator.py        ✅ Phase 1
│   ├── cli/
│   │   ├── __init__.py         ✅ Phase 1
│   │   └── main.py             ✅ Phases 1–4 (skeleton grown per phase)
│   ├── api/
│   │   ├── __init__.py         ✅ Phase 1
│   │   └── main.py             ✅ Phases 1–4 (skeleton grown per phase)
│   ├── approaches/
│   │   ├── __init__.py         ✅ Phase 1
│   │   ├── unstructured/       ✅ Phase 2
│   │   │   ├── __init__.py
│   │   │   ├── ingestion.py
│   │   │   ├── indexing.py
│   │   │   ├── retrieval.py
│   │   │   └── service.py
│   │   └── structured/         ✅ Phase 3
│   │       ├── __init__.py
│   │       ├── ingestion.py
│   │       ├── graph.py
│   │       ├── query.py
│   │       └── service.py
│   ├── benchmark/              ✅ Phase 4
│   │   ├── __init__.py
│   │   ├── runner.py
│   │   ├── scoring.py
│   │   └── persistence.py
│   ├── reports/                ✅ Phase 5
│   │   ├── __init__.py
│   │   └── generator.py
│   └── ui/                     ✅ Phase 6
│       ├── __init__.py
│       ├── helpers.py           ✅ Phase 6 (pure helpers, 100% tested)
│       ├── app.py               ✅ Phase 6 (Streamlit entrypoint)
│       └── pages/
│           ├── __init__.py
│           ├── chat.py          ✅ Phase 6
│           ├── compare.py       ✅ Phase 6
│           ├── benchmark.py     ✅ Phase 6
│           └── reports.py       ✅ Phase 6
├── tests/
│   ├── __init__.py              ✅ Phase 1
│   ├── test_datasets.py        ✅ Phase 1 (12 tests)
│   ├── test_llm_client.py      ✅ Phase 1 (9 tests)
│   ├── test_core_config.py     ✅ Phase 1
│   ├── test_core_models.py     ✅ Phase 1
│   ├── test_core_prompts.py    ✅ Phase 1
│   ├── test_core_utils.py      ✅ Phase 1
│   ├── test_core_logging.py    ✅ Phase 1
│   ├── test_unstructured.py    ✅ Phase 2
│   ├── test_indexing.py        ✅ Phase 2
│   ├── test_structured.py      ✅ Phase 3
│   ├── test_benchmark.py       ✅ Phase 4
│   ├── test_api.py             ✅ Phases 1–4
│   ├── test_cli.py             ✅ Phases 1–4
│   ├── test_coverage_gaps.py   ✅ (coverage gap patches)
│   ├── test_reports.py         ✅ Phase 5 (64 tests)
│   └── test_validator_violations.py ✅ Phase 1
├── .claude/
│   ├── settings.json           ✅ (autonomous permissions + hooks)
│   └── commands/
│       ├── dev-loop.md         ✅
│       └── update-progress.md  ✅
├── scripts/
│   ├── agent-entrypoint.sh    ✅
│   └── update-agent-docs.sh   ✅
└── .github/
    ├── workflows/
    │   └── quality.yml         ✅
    └── ISSUE_TEMPLATE/
        ├── config.yml          ✅
        ├── agent-task.yml      ✅
        └── phase-implementation.yml ✅
```

---

## Agent Run History

| Date | Issue | Branch | Summary |
|------|-------|--------|---------|
| 2026-03-30 | setup | claude/autonomous-workflow-setup-l7KnL | Initial autonomous workflow setup |
| 2026-04-03 | #1 | claude/implement-tfm-ue-ia-issue-1-7Ljmc | Phase 1 Foundation — core modules, datasets, CLI/API skeletons, tests |
| 2026-04-03 | #2 | claude/implement-issue-2-* | Phase 2 Unstructured Pipeline — LlamaIndex + Qdrant RAG |
| 2026-04-03 | #3 | claude/implement-issue-3-xFvsb | Phase 3 Structured Pipeline — NetworkX graph builder + query engine |
| 2026-04-03 | #4 | claude/implement-issue-4-bl02B | Phase 4 Benchmark Runner — 120-question suite, scoring, persistence |
| 2026-04-04 | #3 | claude/implement-tfm-issue-3-Pi45G | Phase 3 closure — add structured/__init__.py, update PROGRESS.md + DECISIONS.md (ADR-007) |
| 2026-04-04 | #5 | claude/implement-tfm-issue-5-JZSrc | Phase 5 Reports — CSV export for run/comparative reports, 16 new tests, ADR-008 |
| 2026-04-04 | #6 | claude/implement-issue-6-fHPVA | Phase 6 Refinement — Streamlit UI, per-category scoring, confidence scoring, README, 70 new tests |
