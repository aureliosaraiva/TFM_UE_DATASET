# CredityFlow Synthetic Dataset — Final Summary

## Dataset Overview

| Metric | Value |
|--------|-------|
| **Total files** | 169 |
| **Total services** | 52 |
| **Total events** | 64 |
| **Total teams** | 15 (14 with services, 1 data-only) |
| **Total databases** | 44 unique |
| **Total benchmark questions** | 120 |
| **External partners** | 8 |
| **Products** | 6 |

## Services by Type

| Type | Count | Percentage |
|------|-------|------------|
| Backend | 42 | 80.8% |
| Integration | 6 | 11.5% |
| BFF | 2 | 3.8% |
| Worker | 2 | 3.8% |
| **Total** | **52** | **100%** |

## Services by Language

| Language/Framework | Count |
|-------------------|-------|
| Kotlin / Spring Boot | 42 |
| Python / FastAPI | 6 |
| Node.js / Express | 3 |
| Go / Gin | 1 |

## Events by Domain

| Domain | Events |
|--------|--------|
| Onboarding | 4 |
| Document Management | 5 |
| Biometrics | 5 |
| Fraud Prevention | 8 |
| Credit Analysis | 8 |
| Appraisal | 4 |
| Contract | 5 |
| Registry | 5 |
| Disbursement | 4 |
| Collections | 11 |
| Platform | 5 |
| **Total** | **64** |

## Teams

| Team | Services Owned |
|------|---------------|
| team-collections | 7 |
| team-credit | 5 |
| team-documents | 5 |
| team-fraud | 5 |
| team-appraisal | 4 |
| team-contracts | 4 |
| team-onboarding | 4 |
| team-registry | 4 |
| team-disbursement | 3 |
| team-identity | 3 |
| team-platform | 3 |
| team-backoffice | 2 |
| team-notifications | 2 |
| team-portal | 1 |
| team-data | 0 (data pipelines only) |

## Databases by Type

| Type | Count |
|------|-------|
| PostgreSQL | 38 |
| MongoDB+S3 | 3 |
| Redis | 7+ (caches) |
| Elasticsearch | 1 |
| S3 | 2 (standalone file storage) |

## Benchmark Questions

### By Category

| Category | Count | Percentage |
|----------|-------|------------|
| Factual | 24 | 20.0% |
| Explanatory | 22 | 18.3% |
| Impact | 21 | 17.5% |
| Relational | 20 | 16.7% |
| Discovery | 18 | 15.0% |
| Ambiguous | 15 | 12.5% |
| **Total** | **120** | **100%** |

### By Difficulty

| Difficulty | Count | Percentage |
|-----------|-------|------------|
| Hard | 44 | 36.7% |
| Medium | 40 | 33.3% |
| Easy | 36 | 30.0% |
| **Total** | **120** | **100%** |

### By Best Suited Approach

| Approach | Count | Percentage |
|----------|-------|------------|
| Unstructured (RAG) | 46 | 38.3% |
| Both | 46 | 38.3% |
| Structured (Knowledge Graph) | 28 | 23.3% |
| **Total** | **120** | **100%** |

## Dataset File Structure

```
/docs/
  company-overview.md          — Company description, products, domains, flows
  dataset-summary.md           — This file
  qa-report.md                 — QA validation report
  events/
    catalog.md                 — Complete event catalog (64 events)
  teams/
    teams.md                   — 15 teams with missions, KPIs, members
    databases.md               — 44 databases inventory
  questions/
    benchmark.json             — 120 questions consolidated
    benchmark_part1.json       — Q001-Q040
    benchmark_part2.json       — Q041-Q080
    benchmark_part3.json       — Q081-Q120

/structured/
  services.json                — 52 services catalog (full metadata)
  events.json                  — 64 events catalog (producers, consumers, payloads)
  graph/
    nodes.json                 — 201 knowledge graph nodes
    edges.json                 — 669 knowledge graph edges

/services/{service-name}/
  structured-spec.yaml         — Structured specification (YAML)
  README.md                    — Unstructured documentation
  architecture.md              — Architecture notes and ADRs
  (× 52 services = 156 files)
```

## Consistency Status

All consistency checks passed after fixes:
- Events ↔ Services: consistent
- Dependencies: all valid
- Ownership: no orphans
- Graph ↔ Services: aligned
- Benchmark: 120 questions, all answerable, all valid references
- Structured ↔ Unstructured docs: no contradictions
- Coverage: supports both RAG and Knowledge Graph approaches
