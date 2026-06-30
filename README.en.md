# CredityFlow — Synthetic Dataset for RAG vs GraphRAG Comparison

> **Versions:** [Español](README.md) · [English](README.en.md) · [Português](README.pt.md)

Synthetic dataset simulating a Brazilian asset-backed lending fintech (**CredityFlow**), built for the Master's Final Thesis (TFM) of Aurelio Saraiva, at the Universidad Europea de Madrid (UE).

The dataset supports experimental comparison between two approaches to specializing technical chat assistants over a corporate knowledge base:

- **Unstructured approach** — RAG (Retrieval-Augmented Generation) over textual documentation.
- **Structured approach** — Knowledge Graph + structured queries.

> This repository contains **only the dataset**. The experimental code (ingestion, retrieval, benchmark, reports) lives in the sibling repository `~/projects/TFM` (not publicly distributed).

## Structure

```
TFM_EU_DATASET/
├── structured/                  # Structured layer (base for the structured approach)
│   ├── services.json            # 52 microservices with ownership, events, dependencies
│   ├── events.json              # 64 domain events (publishers/subscribers/payload)
│   └── graph/
│       ├── nodes.json           # 201 nodes: services, teams, DBs, events, domains, products, partners
│       └── edges.json           # 669 typed edges (OWNS, USES_DATABASE, PUBLISHES, ...)
│
├── services/                    # Per-service documentation (base for the unstructured approach)
│   └── <service-name>/          # 52 directories, one per service
│       ├── README.md            # Overview, ownership, responsibilities, integrations
│       ├── architecture.md      # Detailed technical architecture
│       └── structured-spec.yaml # YAML mirror of the services.json registry entry
│
├── docs/                        # Cross-cutting domain documentation
│   ├── company-overview.md      # CredityFlow company presentation
│   ├── dataset-summary.md       # Consolidated dataset summary
│   ├── qa-report.md             # Quality and consistency audit
│   ├── stack_guide.md           # Technology stack and architectural patterns
│   ├── events/catalog.md        # Event catalog
│   ├── teams/teams.md           # Teams and responsibilities
│   ├── teams/databases.md       # Databases and ownership
│   └── questions/
│       └── benchmark.json       # 120 annotated questions for the benchmark
│
└── _internal/                   # Process artifacts (not part of the published dataset)
                                 # Kept for generation audit; may be ignored by the evaluation committee.
```

## Modelled Domain

CredityFlow is a synthetic fintech inspired by Brazilian asset-backed lending companies (auto and real estate). The credit origination pipeline models the chain:

> **lead → eligibility → proposal → documents → biometrics → fraud → credit → appraisal → contract → registry → disbursement → billing/collections**

Services communicate via asynchronous messaging (SNS/SQS) and synchronous REST. The dataset covers 15 domains, 15 teams, 52 microservices, 64 events, and 201 entities in the graph (including databases, external partners, and products).

## Benchmark Statistics

The **120 questions** in `docs/questions/benchmark.json` cover:

| Category | n | Knowledge type | n | Difficulty | n |
|---|---|---|---|---|---|
| factual | 24 | architectural | 35 | easy | 36 |
| explanatory | 22 | factual | 26 | medium | 40 |
| impact | 21 | mixed | 25 | hard | 44 |
| relational | 20 | operational | 17 | | |
| discovery | 18 | relational | 17 | | |
| ambiguous | 15 | | | | |

Each question includes: statement, expected answer, category, difficulty, `required_knowledge_type`, `primary_target_services` (reference services for grounding), and expected `sources`.

## Artifact Schemas

### `structured/services.json`

List of objects with schema:

```jsonc
{
  "service_name": "lead-intake-service",
  "domain": "Onboarding",
  "subdomain": "Lead Capture",
  "purpose": "...",
  "owner_team": "team-onboarding",
  "criticality": "high",
  "tech_stack": ["Java 17", "Spring Boot", "PostgreSQL"],
  "databases": ["lead_db"],
  "publishes_events": ["lead.created"],
  "subscribes_events": [],
  "depends_on": ["audit-service"],
  "external_integrations": []
}
```

### `structured/events.json`

```jsonc
{
  "event_name": "lead.created",
  "publisher": "lead-intake-service",
  "subscribers": ["eligibility-service", "audit-service"],
  "payload_schema": { ... },
  "domain": "Onboarding"
}
```

### `structured/graph/{nodes,edges}.json`

Typed directed graph. ID conventions:

- `svc-<service-name>` — service
- `team-<team-id>` — team
- `db-<database-name>` — database
- `evt-<event-name>` — event
- `domain-<domain-name>` — domain
- `prod-<product-name>` — product
- `partner-<partner-name>` — external partner

Main relationships: `OWNS`, `USES_DATABASE`, `PUBLISHES`, `SUBSCRIBES_TO`, `DEPENDS_ON`, `BELONGS_TO_DOMAIN`, `INTEGRATES_WITH`.

### `docs/questions/benchmark.json`

```jsonc
{
  "id": "Q001",
  "question": "Which team owns the lead-intake-service?",
  "expected_answer": "The lead-intake-service is owned by team-onboarding...",
  "category": "factual",
  "difficulty": "easy",
  "required_knowledge_type": "factual",
  "primary_target_services": ["lead-intake-service"],
  "sources": ["services/lead-intake-service", "structured/services.json"]
}
```

## Internal Consistency Rules

1. **`services.json` is authoritative** for names, ownership, published events, and dependencies.
2. **Service names are exact** — e.g., `contract-generation-service`, not `contract-service`.
3. **Events have bilateral equivalence** — publishers in `services.json` ↔ producers in `events.json`.
4. **Graph IDs** follow the prefixes `svc-`, `team-`, `db-`, `evt-`, `domain-`, `prod-`, `partner-`.
5. **`primary_target_services` in the benchmark** references only services that exist in `services.json`.

Quick validation command (Python ≥ 3.10):

```bash
python3 -c "
import json
services = json.load(open('structured/services.json'))
events   = json.load(open('structured/events.json'))
questions = json.load(open('docs/questions/benchmark.json'))
names = {s['service_name'] for s in services}
bad = [(q['id'], svc) for q in questions
       for svc in q.get('primary_target_services', [])
       if svc not in names]
print(f'Services: {len(services)}, Events: {len(events)}, Questions: {len(questions)}, Bad refs: {len(bad)}')"
```

Expected output: `Services: 52, Events: 64, Questions: 120, Bad refs: 0`.

## Reproducibility

The original TFM experiment evaluated each question under **3 generators × 2 approaches × 3 repetitions = 2,160 calls per approach**:

- Generators: `claude-sonnet-4-6`, `gpt-5.5`, `grok-build-0.1`
- Dual evaluation: heuristic metric (rapidfuzz) + LLM-as-judge panel (3 judges, 4 dimensions)
- Temperature fixed at 0 for reproducibility

Consolidated results (matrix with 95% bootstrap CI and paired McNemar test) accompany the main TFM document.

## How This Dataset Was Generated

The dataset is **synthetic**, generated from a domain specification (asset-backed lending fintech) using LLMs as generation and review aids. All entities, service names, teams, events, databases, and partners are fictional — any resemblance to real systems is coincidental.

Generation was audited for cross-consistency via `qa-report.md` and validated with the consistency script above.

## License and Use

Dataset developed as part of the Master's Final Thesis of Aurelio Saraiva (Universidad Europea, 2026). Made available to the evaluation committee for validation and experimental reproduction purposes.

Suggested citation:

> Saraiva, A. (2026). *CredityFlow: synthetic dataset for comparing RAG and GraphRAG in enterprise technical chat assistants*. Master's Final Thesis, Universidad Europea de Madrid.

## Contact

Aurelio Saraiva — aurelio.saraiva@creditas.com.br
