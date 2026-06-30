# Project Roadmap — Autonomous Build Plan

> Each phase maps to a GitHub Issue. Open the issue and add the `agent` label to trigger the autonomous build.
> Phases must be executed in order (each depends on the previous).

---

## Phase 1 — Foundation (Issue #1)

**Goal:** Establish the complete Python project structure with all dependencies and core shared code.

**Deliverables:**
- `pyproject.toml` — all dependencies, project metadata, CLI entrypoint (`app`)
- `uv.lock` — locked dependencies
- `docker-compose.yml` — Qdrant service
- `.env.example` — environment variables template
- `app/src/tfm_app/__init__.py`
- `app/src/tfm_app/core/config.py` — Pydantic Settings, YAML config loader
- `app/src/tfm_app/core/models.py` — `ChatResponse`, `ResponseMetadata`, `BenchmarkResult`
- `app/src/tfm_app/core/llm_client.py` — Anthropic client wrapper (async, with retry)
- `app/src/tfm_app/core/prompts.py` — versioned prompt templates
- `app/src/tfm_app/core/logging.py` — structured logging setup
- `app/src/tfm_app/core/utils.py` — timing utilities, cost calculator
- `app/src/tfm_app/datasets/loader.py` — load services.json, events.json, graph, benchmark questions
- `app/src/tfm_app/datasets/validator.py` — dataset consistency checks
- `app/src/tfm_app/cli/main.py` — Typer app skeleton (subcommands: ingest, chat, benchmark, reports)
- `app/src/tfm_app/api/main.py` — FastAPI app skeleton
- `tests/test_datasets.py` — validates all 52 services, 64 events, 120 questions load correctly

**Acceptance Criteria:**
- `uv sync` installs without errors
- `uv run app --help` shows all subcommands
- `uv run pytest tests/test_datasets.py` passes
- Dataset validation script prints: `Services: 52, Events: 64, Questions: 120, Bad refs: 0`

---

## Phase 2 — Unstructured Pipeline (Issue #2)

**Goal:** Build the RAG pipeline using LlamaIndex + Qdrant.

**Depends on:** Phase 1 complete

**Deliverables:**
- `app/src/tfm_app/approaches/unstructured/ingestion.py` — parse all docs/**/*.md and services/*/README.md + architecture.md into chunks
- `app/src/tfm_app/approaches/unstructured/indexing.py` — embed chunks, upsert to Qdrant collection `credityflow_docs`
- `app/src/tfm_app/approaches/unstructured/retrieval.py` — similarity search, top-k retrieval
- `app/src/tfm_app/approaches/unstructured/prompting.py` — RAG prompt assembly
- `app/src/tfm_app/approaches/unstructured/service.py` — `UnstructuredService.chat(question) -> ChatResponse`
- CLI: `app ingest unstructured` works end-to-end
- CLI: `app chat --approach unstructured` works interactively
- `tests/test_unstructured.py` — unit tests for ingestion + retrieval

**Acceptance Criteria:**
- `uv run app ingest unstructured` indexes all documents without errors
- `uv run app chat --approach unstructured` answers questions about CredityFlow
- Response matches `ChatResponse` contract
- Latency logged correctly

---

## Phase 3 — Structured Pipeline (Issue #3)

**Goal:** Build the knowledge graph query engine using NetworkX.

**Depends on:** Phase 1 complete

**Deliverables:**
- `app/src/tfm_app/approaches/structured/ingestion.py` — build NetworkX graph from structured/graph/nodes.json + edges.json + services.json + events.json
- `app/src/tfm_app/approaches/structured/graph.py` — `CredityFlowGraph` class with query methods (get_service, get_dependencies, get_events, find_path, get_team_services, etc.)
- `app/src/tfm_app/approaches/structured/query.py` — question → graph traversal → structured context
- `app/src/tfm_app/approaches/structured/prompting.py` — structured context → prompt assembly
- `app/src/tfm_app/approaches/structured/service.py` — `StructuredService.chat(question) -> ChatResponse`
- CLI: `app ingest structured` builds and validates graph
- CLI: `app chat --approach structured` works interactively
- `tests/test_structured.py` — unit tests for graph builder + query engine

**Acceptance Criteria:**
- `uv run app ingest structured` builds graph: 201 nodes, 669 edges
- `uv run app chat --approach structured` answers questions about CredityFlow
- Response matches `ChatResponse` contract
- Graph query methods cover all edge types

---

## Phase 4 — Benchmark Runner (Issue #4)

**Goal:** Run all 120 questions against both approaches and persist results.

**Depends on:** Phases 2 and 3 complete

**Deliverables:**
- `app/src/tfm_app/benchmark/runner.py` — iterate 120 questions, call both services, handle errors
- `app/src/tfm_app/benchmark/scoring.py` — Level 1 heuristic scoring (rapidfuzz similarity, entity presence, source verification) → `correct|partial|incorrect|error`
- `app/src/tfm_app/benchmark/judge.py` — Level 2 LLM-as-judge (optional, flag-gated)
- `app/src/tfm_app/benchmark/persistence.py` — save results to `outputs/runs/{timestamp}_{approach}/` as JSON
- `app/src/tfm_app/benchmark/metrics.py` — compute: success rate, avg latency, cost, similarity, by category/difficulty/approach
- CLI: `app benchmark run --approach all` executes full benchmark
- FastAPI: `POST /benchmark/run`, `GET /benchmark/runs/{id}`
- `tests/test_benchmark.py` — unit tests for scoring functions

**Acceptance Criteria:**
- `uv run app benchmark run --approach all` runs all 120 questions × 2 approaches
- Results saved to `outputs/runs/`
- Metrics computed: success rate by category, average latency, estimated cost
- `GET /benchmark/runs/{id}` returns persisted results

---

## Phase 5 — Reports (Issue #5)

**Goal:** Generate human-readable and machine-readable reports from benchmark runs.

**Depends on:** Phase 4 complete

**Deliverables:**
- `app/src/tfm_app/benchmark/reports.py` — Markdown + JSON + CSV report generation
- `templates/report_single.md.j2` — Jinja2 template for single-run report
- `templates/report_comparison.md.j2` — Jinja2 template for comparison report
- `outputs/reports/` — generated report files
- CLI: `app reports generate --latest` generates report for most recent run
- CLI: `app reports compare --run-a <id> --run-b <id>` generates comparison
- FastAPI: `POST /reports/generate`
- `tests/test_reports.py` — unit tests for report generation

**Acceptance Criteria:**
- Report includes: overall accuracy, latency comparison, cost comparison, per-category breakdown, per-difficulty breakdown, best-approach analysis
- Markdown report is readable and suitable for thesis inclusion
- CSV report has one row per question with all metrics
- Comparison report highlights differences between approaches

---

## Phase 6 — Refinement (Issue #6)

**Goal:** Add Streamlit UI, side-by-side comparison, and polish evaluation.

**Depends on:** Phases 1–5 complete

**Deliverables:**
- `app/src/tfm_app/ui/pages/chat.py` — Streamlit chat page (single approach)
- `app/src/tfm_app/ui/pages/compare.py` — Streamlit side-by-side comparison page
- `app/src/tfm_app/ui/pages/benchmark.py` — Streamlit benchmark dashboard
- `app/src/tfm_app/ui/pages/reports.py` — Streamlit reports viewer
- `app/src/tfm_app/ui/app.py` — Streamlit multi-page app entrypoint
- Config improvements: YAML config file with all tunable parameters
- Improve Level 1 scoring: tune rapidfuzz thresholds per category
- Add confidence scoring to both approaches
- Final README.md with full setup and usage instructions

**Acceptance Criteria:**
- `uv run streamlit run app/src/tfm_app/ui/app.py` starts UI
- Chat page works for both approaches
- Compare page shows answers side-by-side with scoring
- Benchmark dashboard shows run history and metrics
- README covers: prerequisites, setup, running each component, interpreting results

---

## Post-Completion Checklist

After all 6 phases:
- [ ] Full benchmark run: 120 questions × 2 approaches
- [ ] Comparison report generated
- [ ] Both approaches pass dataset validation
- [ ] All tests pass (`uv run pytest`)
- [ ] README.md complete
- [ ] `docs/agents/PROGRESS.md` shows all phases ✅

---

## Issue Creation Guide for Humans

To trigger an agent phase:
1. Go to GitHub Issues
2. Click "New Issue" → choose "Phase Implementation" template
3. Fill in the phase number and details
4. Submit the issue
5. Add the label `agent` → automation triggers immediately
6. Add `phase-N` label for tracking

To request a specific task (bug, improvement, new feature):
1. Choose "Agent Task" template
2. Describe the task clearly in the issue body
3. Include: what files to change, what the expected result is
4. Add label `agent` to trigger
