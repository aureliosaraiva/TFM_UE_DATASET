# Agent Quick Reference

> Read this file at the start of every agent run.
> It contains the critical paths, conventions, and rules needed to work effectively.

---

## Repository Layout

```
TFM_UE_IA/
├── CLAUDE.md                        ← PRIMARY project guide — read first
├── docs/agents/
│   ├── PROGRESS.md                  ← Current implementation state
│   ├── DECISIONS.md                 ← Architecture decision log
│   ├── CONTEXT.md                   ← This file
│   └── ROADMAP.md                   ← Phase breakdown
├── docs/                            ← Unstructured knowledge (RAG input)
│   ├── company-overview.md
│   ├── dataset-summary.md
│   ├── events/catalog.md
│   ├── questions/benchmark.json     ← 120 benchmark questions
│   └── teams/
├── services/{service-name}/         ← 52 microservice docs (RAG input)
│   ├── README.md
│   ├── architecture.md
│   └── structured-spec.yaml
├── structured/                      ← Structured knowledge (graph input)
│   ├── services.json                ← AUTHORITATIVE source for service names
│   ├── events.json
│   └── graph/
│       ├── nodes.json               ← 201 nodes
│       └── edges.json               ← 669 edges
├── app/src/tfm_app/                 ← Python application (to be built)
│   ├── core/                        ← Shared: config, models, llm, prompts
│   ├── datasets/                    ← Dataset loaders and validators
│   ├── approaches/
│   │   ├── unstructured/            ← RAG pipeline
│   │   └── structured/             ← Graph pipeline
│   ├── benchmark/                   ← Runner and scoring
│   ├── api/                         ← FastAPI endpoints
│   ├── ui/                          ← Streamlit pages
│   └── cli/                         ← Typer commands
├── scripts/
│   ├── agent-entrypoint.sh         ← GitHub Actions entrypoint
│   └── update-agent-docs.sh        ← Stop hook for doc enforcement
├── .claude/
│   ├── settings.json               ← Permissions + Stop hook
│   └── commands/update-progress.md ← /update-progress slash command
└── .github/
    ├── workflows/claude-agent.yml  ← Automation trigger
    └── ISSUE_TEMPLATE/
```

---

## Key Commands

```bash
# Package management
uv sync                              # Install all dependencies
uv add <package>                     # Add a new dependency
uv run pytest                        # Run tests

# Application (once implemented)
uv run app ingest unstructured
uv run app ingest structured
uv run app chat --approach unstructured
uv run app chat --approach structured
uv run app chat --compare
uv run app benchmark run --approach all
uv run app reports generate --latest

# Dataset validation
python3 -c "
import json
services = json.load(open('structured/services.json'))
events = json.load(open('structured/events.json'))
questions = json.load(open('docs/questions/benchmark.json'))
names = {s['service_name'] for s in services}
bad = [(q['id'],svc) for q in questions for svc in q.get('primary_target_services',[]) if svc not in names]
print(f'Services: {len(services)}, Events: {len(events)}, Questions: {len(questions)}, Bad refs: {len(bad)}')
"
```

---

## Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| Agent branches | `agent/issue-{N}-{slug}` | `agent/issue-7-add-qdrant-ingestion` |
| Python modules | `snake_case` | `unstructured_service.py` |
| Python classes | `PascalCase` | `ChatResponse`, `GraphBuilder` |
| Config keys | `snake_case` | `chunk_size`, `top_k` |
| Service names | kebab-case (from services.json) | `lead-intake-service` |
| Event names | `PascalCase.v1` | `LeadCreated.v1` |
| Graph node IDs | `{type}-{name}` | `svc-auth-service`, `evt-lead-created` |

---

## Dataset Consistency Rules (CRITICAL)

1. **`structured/services.json` is authoritative** — service names, events, dependencies
2. **Exact service names** — always copy from services.json, never guess
3. **Bilateral event consistency** — publishers in services.json must match producers in events.json
4. **Graph ID format**: `svc-<name>`, `team-<id>`, `db-<name>`, `evt-<name>`, `domain-<name>`
5. **Benchmark refs** — `primary_target_services` must reference valid service names

---

## Standardized Response Contract

Both approaches MUST return this structure (Pydantic model in `core/models.py`):

```python
class ChatResponse(BaseModel):
    question: str
    approach: Literal["unstructured", "structured"]
    answer: str
    sources: list[str]
    retrieved_context: list[str]
    latency_ms: float
    metadata: ResponseMetadata

class ResponseMetadata(BaseModel):
    model: str
    tokens_input: int
    tokens_output: int
    estimated_cost: float
    confidence: float | None = None
```

---

## Infrastructure

- **Qdrant** (vector DB for RAG): runs via Docker on `localhost:6333`
  - Collection: `credityflow_docs`
  - Start with: `docker compose up -d qdrant`
- **LLM**: Claude via Anthropic API (`ANTHROPIC_API_KEY` env var)
- **Default model**: `claude-sonnet-4-6` (or from config)
- **Benchmark temperature**: 0 (deterministic)
- **Outputs**: `outputs/runs/` (benchmark results), `outputs/reports/` (generated reports)

---

## pyproject.toml Key Dependencies (Phase 1)

```toml
[project]
name = "tfm-app"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "streamlit>=1.40",
    "typer>=0.12",
    "pydantic>=2.9",
    "anthropic>=0.40",
    "llama-index>=0.11",
    "llama-index-vector-stores-qdrant>=0.3",
    "qdrant-client>=1.12",
    "networkx>=3.4",
    "pandas>=2.2",
    "jinja2>=3.1",
    "rapidfuzz>=3.10",
    "python-dotenv>=1.0",
    "pyyaml>=6.0",
    "rich>=13.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0", "pytest-asyncio>=0.24", "httpx>=0.27", "ruff>=0.8"]

[project.scripts]
app = "tfm_app.cli.main:app"
```

---

## How to Update Documentation (MANDATORY at end of every run)

1. Open `docs/agents/PROGRESS.md`
2. Update "In Progress" → "Completed Work" for your task
3. Update the Phase Summary table status
4. Update the File Map with new files created
5. Add a row to Agent Run History
6. If you made a non-obvious decision, add an ADR to `docs/agents/DECISIONS.md`
7. `git add docs/agents/` and include in your final commit

Or use the slash command: `/update-progress`

---

## Git Workflow for Agent Runs

```bash
# Branch is already created by GitHub Actions
# Work on it, then at the end:
git add -p                           # stage specific files
git commit -m "feat(phase-N): description

Resolves #<issue-number>"
# Push is handled by GitHub Actions after script exits
```

---

## CredityFlow Lending Pipeline (key context)

```
lead → eligibility → proposal → documents → biometrics →
fraud → credit → appraisal → contract → registry →
disbursement → billing/collections
```

Services communicate via **SNS/SQS** (async events) and **REST** (sync calls).
