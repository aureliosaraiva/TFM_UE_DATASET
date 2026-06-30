# CredityFlow — Dataset sintético para comparação de RAG vs GraphRAG

Dataset sintético que simula uma fintech brasileira de crédito com garantia (**CredityFlow**), construído para o Trabajo Fin de Máster (TFM) de Aurelio Saraiva, no contexto da Universidad Europea de Madrid (UE).

O dataset suporta a comparação experimental entre duas abordagens de especialização de assistentes de chat técnicos sobre uma base de conhecimento empresarial:

- **Approach unstructured** — RAG (Retrieval-Augmented Generation) sobre documentação textual.
- **Approach structured** — Knowledge Graph + queries estruturadas.

> Este repositório contém **somente o dataset**. O código experimental (ingestão, retrieval, benchmark, relatórios) está no repositório irmão `~/projects/TFM` (não distribuído publicamente).

## Estrutura

```
TFM_EU_DATASET/
├── structured/                  # Camada estruturada (base do approach structured)
│   ├── services.json            # 52 microsserviços com ownership, eventos, dependências
│   ├── events.json              # 64 eventos de domínio (publishers/subscribers/payload)
│   └── graph/
│       ├── nodes.json           # 201 nós: serviços, times, DBs, eventos, domínios, produtos, parceiros
│       └── edges.json           # 669 arestas tipadas (OWNS, USES_DATABASE, PUBLISHES, ...)
│
├── services/                    # Documentação por serviço (base do approach unstructured)
│   └── <service-name>/          # 52 diretórios, um por serviço
│       ├── README.md            # Visão geral, ownership, responsabilidades, integrações
│       ├── architecture.md      # Arquitetura técnica detalhada
│       └── structured-spec.yaml # Espelho YAML do registro em services.json
│
├── docs/                        # Documentação cross-cutting do domínio
│   ├── company-overview.md      # Apresentação da empresa CredityFlow
│   ├── dataset-summary.md       # Sumário consolidado do dataset
│   ├── qa-report.md             # Auditoria de qualidade e consistência
│   ├── stack_guide.md           # Stack tecnológica e padrões arquiteturais
│   ├── events/catalog.md        # Catálogo de eventos
│   ├── teams/teams.md           # Times e responsabilidades
│   ├── teams/databases.md       # Bases de dados e ownership
│   └── questions/
│       └── benchmark.json       # 120 perguntas anotadas para o benchmark
│
└── _internal/                   # Artefatos de processo (não fazem parte do dataset publicado)
                                 # Mantidos para auditoria de geração; podem ser ignorados pela banca.
```

## Domínio modelado

CredityFlow é uma fintech sintética inspirada em empresas brasileiras de empréstimo com garantia (auto e imobiliário). O pipeline de concessão de crédito modela a cadeia:

> **lead → eligibility → proposal → documents → biometrics → fraud → credit → appraisal → contract → registry → disbursement → billing/collections**

Serviços comunicam-se via mensageria assíncrona (SNS/SQS) e REST síncrona. O dataset cobre 15 domínios, 15 times, 52 microsserviços, 64 eventos e 201 entidades no grafo (incluindo bancos de dados, parceiros externos e produtos).

## Estatísticas do benchmark

As **120 perguntas** em `docs/questions/benchmark.json` cobrem:

| Categoria | n | Knowledge type | n | Dificuldade | n |
|---|---|---|---|---|---|
| factual | 24 | architectural | 35 | easy | 36 |
| explanatory | 22 | factual | 26 | medium | 40 |
| impact | 21 | mixed | 25 | hard | 44 |
| relational | 20 | operational | 17 | | |
| discovery | 18 | relational | 17 | | |
| ambiguous | 15 | | | | |

Cada pergunta inclui: enunciado, resposta esperada, categoria, dificuldade, `required_knowledge_type`, `primary_target_services` (serviços de referência para grounding) e `sources` esperados.

## Schema de cada artefato

### `structured/services.json`

Lista de objetos com schema:

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

Grafo direcionado tipado. Convenção de IDs:

- `svc-<service-name>` — serviço
- `team-<team-id>` — time
- `db-<database-name>` — banco de dados
- `evt-<event-name>` — evento
- `domain-<domain-name>` — domínio
- `prod-<product-name>` — produto
- `partner-<partner-name>` — parceiro externo

Relações principais: `OWNS`, `USES_DATABASE`, `PUBLISHES`, `SUBSCRIBES_TO`, `DEPENDS_ON`, `BELONGS_TO_DOMAIN`, `INTEGRATES_WITH`.

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

## Regras de consistência interna

1. **`services.json` é autoritativo** para nomes, ownership, eventos publicados e dependências.
2. **Nomes de serviço são exatos** — e.g., `contract-generation-service`, não `contract-service`.
3. **Eventos têm equivalência bilateral** — publishers em `services.json` ↔ producers em `events.json`.
4. **IDs do grafo** seguem prefixos `svc-`, `team-`, `db-`, `evt-`, `domain-`, `prod-`, `partner-`.
5. **`primary_target_services` no benchmark** referencia somente serviços existentes em `services.json`.

Comando rápido de validação (Python ≥ 3.10):

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

Saída esperada: `Services: 52, Events: 64, Questions: 120, Bad refs: 0`.

## Reprodutibilidade

O experimento original do TFM avaliou cada pergunta sob **3 generators × 2 approaches × 3 repetições = 2.160 chamadas por approach**:

- Generators: `claude-sonnet-4-6`, `gpt-5.5`, `grok-build-0.1`
- Avaliação dupla: métrica heurística (rapidfuzz) + painel LLM-as-judge (3 juízes, 4 dimensões)
- Temperatura fixa em 0 para reprodutibilidade

Os resultados consolidados (matriz com IC 95% bootstrap e teste McNemar pareado) acompanham o documento principal do TFM.

## Como esse dataset foi gerado

O dataset é **sintético**, gerado a partir de uma especificação de domínio (fintech de crédito com garantia) usando LLMs como auxiliares de geração e revisão. Todas as entidades, nomes de serviço, times, eventos, bancos de dados e parceiros são fictícios — qualquer semelhança com sistemas reais é incidental.

A geração foi auditada para consistência cruzada via `qa-report.md` e validada pelo script de consistência acima.

## Licença e uso

Dataset desenvolvido como parte do Trabajo Fin de Máster de Aurelio Saraiva (Universidad Europea, 2026). Disponibilizado à banca avaliadora para fins de validação e reprodução experimental.

Citação sugerida:

> Saraiva, A. (2026). *CredityFlow: dataset sintético para comparação de RAG e GraphRAG em assistentes técnicos de chat empresarial*. Trabajo Fin de Máster, Universidad Europea de Madrid.

## Contato

Aurelio Saraiva — aurelio.saraiva@creditas.com.br
