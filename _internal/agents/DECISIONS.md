# Architecture Decisions Log

> Every significant technical decision made during autonomous agent runs is recorded here.
> Format: ADR (Architecture Decision Record) — lightweight version.
> Agents MUST add an entry when making a non-obvious architectural choice.

---

## ADR-001: Autonomous Workflow via GitHub Actions + Claude CLI

**Date:** 2026-03-30
**Context:** The project requires a fully autonomous build system where human interaction is limited to opening GitHub Issues.
**Decision:** Use GitHub Actions triggered by the `agent` label on issues. Each run installs Claude CLI, assembles a rich context prompt (CLAUDE.md + PROGRESS.md + DECISIONS.md + CONTEXT.md + issue body), and executes `claude --dangerously-skip-permissions -p "..."`. Results are committed to a feature branch.
**Consequences:**
- Requires `ANTHROPIC_API_KEY` as a GitHub Actions secret
- Each issue = one agent run = one feature branch
- Human must add the `agent` label to trigger execution (prevents accidental runs)
- Agent outputs are always reviewed before merging (branch → PR → merge)

---

## ADR-002: Documentation-First Agent Protocol

**Date:** 2026-03-30
**Context:** Autonomous agents lose context between runs. Each new agent starts fresh with no memory of prior work.
**Decision:** Agents MUST update `docs/agents/PROGRESS.md` and `docs/agents/DECISIONS.md` before finishing. A Stop hook in `.claude/settings.json` enforces this by running `scripts/update-agent-docs.sh` at session end.
**Consequences:**
- Documentation becomes the "memory" of the system
- Next agent always has full context of prior work
- Documentation debt is structurally impossible — it's enforced at the tooling level

---

## ADR-003: uv for Python Dependency Management

**Date:** 2026-03-30
**Context:** CLAUDE.md specifies Python 3.12 with uv. uv is significantly faster than pip and provides lock files.
**Decision:** Use uv as the sole package manager. `pyproject.toml` is the central config (PEP 517/518). No `requirements.txt` files.
**Consequences:**
- GitHub Actions must install uv (`pip install uv`)
- All dependencies managed via `pyproject.toml` + `uv.lock`
- Dev dependencies in `[project.optional-dependencies]` under `dev` group

---

## ADR-004: Single Response Contract for Both Approaches

**Date:** 2026-03-30
**Context:** Fair comparison requires identical interfaces. Both unstructured (RAG) and structured (graph) approaches must return the same JSON shape.
**Decision:** Define a shared Pydantic v2 model `ChatResponse` in `app/src/tfm_app/core/models.py`. Both approach services must return this model. The model includes: question, approach, answer, sources, retrieved_context, latency_ms, metadata (model, tokens, cost, confidence).
**Consequences:**
- Evaluation code is approach-agnostic
- UI and API are approach-agnostic
- Any breaking change to the contract requires updating both approaches simultaneously

---

## ADR-005: Qdrant via Docker for Unstructured Approach

**Date:** 2026-03-30
**Context:** The unstructured approach requires a vector database for RAG. Qdrant is specified in CLAUDE.md.
**Decision:** Run Qdrant locally via Docker (docker-compose.yml). Collection name: `credityflow_docs`. Chunk size: 512 tokens, overlap: 50. Top-k: 5.
**Consequences:**
- Docker must be running before unstructured ingestion or chat
- Qdrant data persists in a Docker volume (`qdrant_storage`)
- Config values (chunk_size, top_k) must be exposed in YAML config for reproducibility

---

## ADR-006: NetworkX In-Memory Graph for Structured Approach

**Date:** 2026-03-30
**Context:** The structured approach needs a knowledge graph. Neo4j is listed as optional evolution in CLAUDE.md.
**Decision:** Use NetworkX (in-memory) for Phase 3. Load from `structured/graph/nodes.json` + `structured/graph/edges.json` at startup. No persistence needed — graph is rebuilt on each run from source JSON files.
**Consequences:**
- Fast startup (no database connection)
- No Neo4j dependency for MVP
- Graph is always consistent with source JSON (no migration issues)
- If Neo4j is needed later, the graph builder interface must remain the same

---

## ADR-007: Shared Core Prompts Instead of Per-Approach Prompting Modules

**Date:** 2026-04-04
**Context:** ROADMAP Phase 2 and Phase 3 both listed a dedicated `prompting.py` module under each approach package. During implementation it was found that both approaches share the same prompt structure (system prompt + user message with question + context) and differ only in the system-prompt text and few-shot examples.
**Decision:** Use a single `core/prompts.py` module with versioned `PromptVersion` objects keyed by approach name (`"unstructured-v1"`, `"structured-v1"`). Each service imports `DEFAULT_PROMPTS` and `format_user_message` from `core/prompts.py` directly. No per-approach `prompting.py` files are created.
**Consequences:**
- Prompt versioning is centralised; changing a prompt version requires editing one file and bumping the version key.
- The ROADMAP deliverables `unstructured/prompting.py` and `structured/prompting.py` are intentionally absent; this ADR documents that this was a deliberate design decision, not an oversight.
- Any future approach (e.g., hybrid) reuses the same prompt infrastructure without needing a new module.

---

## ADR-008: Inline Report Templates and CSV as a First-Class Report Artefact

**Date:** 2026-04-04
**Context:** The ROADMAP listed `templates/report_single.md.j2` and `templates/report_comparison.md.j2` as deliverables, implying Jinja2 templating. The ROADMAP acceptance criterion also states "CSV report has one row per question with all metrics", but the initial Phase 5 implementation in `generator.py` only produced Markdown and JSON. The persistence module (`persistence.py`) already writes a `results.csv` per run, but its column set does not include `category`, `difficulty`, `required_knowledge_type`, or per-result cost — it is optimised for raw persistence, not for analysis.

**Decision (two parts):**

1. **No Jinja2 templates.** The report generator uses inline Python string construction. The `.j2` template files listed in the ROADMAP are not created. The inline approach is simpler, avoids an additional runtime dependency, is already tested at 100% coverage, and produces identical output. This is a deliberate deviation from the ROADMAP deliverable list, consistent with the precedent set by ADR-007.

2. **CSV report is generated by the reports module with a richer column set.** `generate_run_report` writes `report.csv` to `outputs/reports/<run_id>/` with columns: `run_id`, `approach`, `question_id`, `question`, `category`, `difficulty`, `required_knowledge_type`, `expected_answer`, `actual_answer`, `similarity_score`, `classification`, `latency_ms`, `tokens_input`, `tokens_output`, `estimated_cost`, `error`. `generate_comparative_report` writes `compare.csv` to `outputs/reports/compare-<a>-vs-<b>/` with a wide per-question-id joined format including deltas. No CSV is written when `run.results` is empty.

**Consequences:**
- `outputs/reports/<run_id>/` now contains three files: `report.md`, `report.json`, `report.csv`.
- `outputs/reports/compare-<a>-vs-<b>/` now contains three files: `compare.md`, `compare.json`, `compare.csv`.
- The ROADMAP deliverable `templates/` directory is intentionally absent.
- The persistence CSV (`outputs/runs/<run_id>/results.csv`) is unchanged and serves a different purpose (raw per-run artefact vs. analysis artefact).
- Phase 6 Streamlit UI can read `report.csv` directly for the benchmark dashboard without re-implementing join logic.

---

_Add new ADRs below this line as the project evolves._

---

## ADR-XXX: Tier 1 methodology — statistical rigour, semantic metric, judge validation, error analysis

**Date:** 2026-06-13
**Context:** The TFM compares two assistant approaches (unstructured RAG vs structured graph) on the same 120-question suite. Three methodological gaps weakened the comparison:
1. Single-point success rates without confidence intervals, no paired test → cannot say whether observed differences are statistically meaningful.
2. The only similarity metric (rapidfuzz `token_sort_ratio`) is lexical; it penalises rephrasing severely (average ≈ 0.38 on both clean runs), inflating the `incorrect` bucket for factual answers that *do* contain the right entities but in a different phrasing.
3. The LLM-as-judge was implemented in `benchmark/judge.py` but its scores were not aggregated nor stored — they were only logged. There was also no way to assess the judge itself as an instrument.

**Decision:**
- Add `benchmark/stats.py` with pure-Python/numpy implementations of Wilson confidence intervals, paired McNemar (exact binomial when discordants < 25, χ² with continuity correction otherwise), percentile bootstrap, Cohen's kappa, and percent agreement. No `scipy`/`statsmodels` dependency.
- Add a **complementary** semantic similarity using cosine over `BAAI/bge-small-en-v1.5` embeddings (the existing local LlamaIndex model — no new model, no API key). The embedder is dependency-injected through `compute_semantic_similarity(expected, actual, embedder)`; tests mock it. The lexical metric is **not** replaced; both are reported side-by-side.
- Aggregate `judge_eval` averages globally and per category in `BenchmarkRun.judge_summary`, `summary.json`, and Markdown reports. Add CLI commands `app benchmark sample-labels --run <id> --n 25 --seed 42` (generates a CSV with blank `human_*` columns) and `app benchmark judge-agreement --human-csv <path>` (Cohen's kappa + % agreement on the four dimensions). The judge dataclass is renamed neither in `JudgeEval` (model layer) nor in `JudgeScore` (compute layer); the model field stays `judge_eval` for backward compatibility with already-saved runs.
- Add `app reports semantic-recompute --run <id>` (writes `outputs/analysis/<id>/semantic.csv` + `semantic-summary.json`) and `app reports error-analysis --run <id>` (writes `error-analysis.md` highlighting low-lexical/high-semantic candidates and, for structured runs, splitting failures by empty-context vs real answer errors). Thresholds: lexical < 0.40, semantic ≥ 0.70.

**Consequences:**
- Comparisons can now report ICs and a p-value. On the 30-04 clean runs, McNemar shows no significant difference (strict p=0.84, lenient p=0.33) — confirming the two approaches are statistically indistinguishable on this suite.
- Average semantic similarity is ~0.84 on both runs vs ~0.38 lexical — strong evidence that the lexical-only score penalises good factual answers; this changes the narrative of the chapter.
- Judge persistence is now end-to-end (per-result `judge_eval` → aggregate `judge_summary` → summary.json/CSV/Markdown). Re-running the benchmark with `--judge` will surface dimensions in all artefacts; the judge↔human agreement command provides the instrument validation requested by methodology.
- Coverage of the new code is at 100% for stats/semantic/judge_validation/error_analysis/metrics modules; project total stays at ~98%.

---

## ADR-XXX: Switch the LLM transport to an OpenAI-compatible gateway

**Date:** 2026-06-15
**Context:** The original implementation talked to Anthropic directly via the `anthropic` SDK. Two pressures forced a change:
1. The Tier-2 design adds three generators (Anthropic, OpenAI, Google) and a judge panel from three more families. Keeping one SDK per provider blows up the test surface and locks the experiment to whatever each SDK happens to expose.
2. **Auto-evaluation bias**: when the only judge is the same model that produced the answer, the score inflates. The Tier-2 design mandates a cross-family panel — which requires a uniform wire format.

**Decision:**
- Replace `anthropic.AsyncAnthropic` with `openai.AsyncOpenAI` pointed at any OpenAI-compatible endpoint. Default: **OpenCode Zen** (`https://opencode.ai/zen/v1`).
- The public surface of `LLMClient` is preserved (`__init__`, `chat`, `LLMResponse`, `RetryConfig`) so downstream code in `approaches/*`, `benchmark/runner.py` and `benchmark/judge.py` does not change — only construction does.
- Cost handling moves from a hardcoded `_COST_PER_MILLION` table to `AppConfig.model_pricing` (USD per 1 M tokens). Missing entries yield `estimated_cost = None` and a warning; `ResponseMetadata.estimated_cost` becomes `float | None`. `BenchmarkRun.total_cost` keeps an additive float by treating `None` as 0.
- Authentication is decoupled from any single provider **and from on-disk configuration**: `AppConfig.llm_api_key_env` names the environment variable that holds the key (`OPENCODE_API_KEY` by default); `AppConfig.resolve_api_key()` reads **only** from that process env var. The API key is intentionally not a config field — it cannot be set via `.env`, YAML or constructor kwargs, so it never lands in `config_snapshot` nor in any persisted artefact.
- Multi-generator and judge-panel inputs live in `AppConfig.generator_models`, `judge_models`, and `model_family`. The family map powers the cross-family bias guard in later deliverables (D3) — `AppConfig.family_of(model)` returns the explicit tag or falls back to the model prefix.

**Consequences:**
- Tests no longer import `anthropic`; they mock `openai.AsyncOpenAI` instead. All retry/back-off behaviour and the `retry-after` honouring path remain unit-tested.
- A single set of credentials (`OPENCODE_API_KEY`) covers every Claude/GPT/Gemini/Grok variant exposed by Zen — embeddings still run locally (`BAAI/bge-small-en-v1.5`), so there is no proxy traffic for retrieval.
- The migration is the foundation for D2 (multi-generator benchmark matrix) and D3 (3-judge panel + cross-family aggregation). Neither piece could move without a uniform wire format first.
- `anthropic` is removed from `pyproject.toml`; `openai>=1.50` is added. Coverage stays ≥90 %.

---
