# CredityFlow — Dataset sintético para comparación de RAG vs GraphRAG

> **Versiones / Versions:** [Español](README.md) · [English](README.en.md) · [Português](README.pt.md)

Dataset sintético que simula una fintech brasileña de crédito con garantía (**CredityFlow**), construido para el Trabajo Fin de Máster (TFM) de Aurelio Saraiva, en el contexto de la Universidad Europea de Madrid (UE).

El dataset soporta la comparación experimental entre dos enfoques de especialización de asistentes de chat técnicos sobre una base de conocimiento empresarial:

- **Enfoque no estructurado** — RAG (Retrieval-Augmented Generation) sobre documentación textual.
- **Enfoque estructurado** — Knowledge Graph + consultas estructuradas.

> Este repositorio contiene **únicamente el dataset**. El código experimental (ingestión, retrieval, benchmark, informes) está en el repositorio hermano `~/projects/TFM` (no distribuido públicamente).

## Estructura

```
TFM_EU_DATASET/
├── structured/                  # Capa estructurada (base del enfoque estructurado)
│   ├── services.json            # 52 microservicios con ownership, eventos, dependencias
│   ├── events.json              # 64 eventos de dominio (publishers/subscribers/payload)
│   └── graph/
│       ├── nodes.json           # 201 nodos: servicios, equipos, BDs, eventos, dominios, productos, socios
│       └── edges.json           # 669 aristas tipadas (OWNS, USES_DATABASE, PUBLISHES, ...)
│
├── services/                    # Documentación por servicio (base del enfoque no estructurado)
│   └── <service-name>/          # 52 directorios, uno por servicio
│       ├── README.md            # Visión general, ownership, responsabilidades, integraciones
│       ├── architecture.md      # Arquitectura técnica detallada
│       └── structured-spec.yaml # Espejo YAML del registro en services.json
│
├── docs/                        # Documentación transversal del dominio
│   ├── company-overview.md      # Presentación de la empresa CredityFlow
│   ├── dataset-summary.md       # Resumen consolidado del dataset
│   ├── qa-report.md             # Auditoría de calidad y consistencia
│   ├── stack_guide.md           # Stack tecnológico y patrones arquitectónicos
│   ├── events/catalog.md        # Catálogo de eventos
│   ├── teams/teams.md           # Equipos y responsabilidades
│   ├── teams/databases.md       # Bases de datos y ownership
│   └── questions/
│       └── benchmark.json       # 120 preguntas anotadas para el benchmark
│
└── _internal/                   # Artefactos de proceso (no forman parte del dataset publicado)
                                 # Mantenidos para auditoría de generación; pueden ser ignorados por el tribunal.
```

## Dominio modelado

CredityFlow es una fintech sintética inspirada en empresas brasileñas de préstamo con garantía (auto e inmobiliario). El pipeline de concesión de crédito modela la cadena:

> **lead → eligibility → proposal → documents → biometrics → fraud → credit → appraisal → contract → registry → disbursement → billing/collections**

Los servicios se comunican mediante mensajería asíncrona (SNS/SQS) y REST síncrona. El dataset cubre 15 dominios, 15 equipos, 52 microservicios, 64 eventos y 201 entidades en el grafo (incluyendo bases de datos, socios externos y productos).

## Estadísticas del benchmark

Las **120 preguntas** en `docs/questions/benchmark.json` cubren:

| Categoría | n | Tipo de conocimiento | n | Dificultad | n |
|---|---|---|---|---|---|
| factual | 24 | architectural | 35 | easy | 36 |
| explanatory | 22 | factual | 26 | medium | 40 |
| impact | 21 | mixed | 25 | hard | 44 |
| relational | 20 | operational | 17 | | |
| discovery | 18 | relational | 17 | | |
| ambiguous | 15 | | | | |

Cada pregunta incluye: enunciado, respuesta esperada, categoría, dificultad, `required_knowledge_type`, `primary_target_services` (servicios de referencia para grounding) y `sources` esperados.

## Esquema de cada artefacto

### `structured/services.json`

Lista de objetos con esquema:

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

Grafo dirigido tipado. Convención de IDs:

- `svc-<service-name>` — servicio
- `team-<team-id>` — equipo
- `db-<database-name>` — base de datos
- `evt-<event-name>` — evento
- `domain-<domain-name>` — dominio
- `prod-<product-name>` — producto
- `partner-<partner-name>` — socio externo

Relaciones principales: `OWNS`, `USES_DATABASE`, `PUBLISHES`, `SUBSCRIBES_TO`, `DEPENDS_ON`, `BELONGS_TO_DOMAIN`, `INTEGRATES_WITH`.

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

## Reglas de consistencia interna

1. **`services.json` es autoritativo** para nombres, ownership, eventos publicados y dependencias.
2. **Los nombres de servicio son exactos** — e.g., `contract-generation-service`, no `contract-service`.
3. **Los eventos tienen equivalencia bilateral** — publishers en `services.json` ↔ producers en `events.json`.
4. **Los IDs del grafo** siguen los prefijos `svc-`, `team-`, `db-`, `evt-`, `domain-`, `prod-`, `partner-`.
5. **`primary_target_services` en el benchmark** referencia únicamente servicios existentes en `services.json`.

Comando rápido de validación (Python ≥ 3.10):

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

Salida esperada: `Services: 52, Events: 64, Questions: 120, Bad refs: 0`.

## Reproducibilidad

El experimento original del TFM evaluó cada pregunta bajo **3 generators × 2 enfoques × 3 repeticiones = 2.160 llamadas por enfoque**:

- Generators: `claude-sonnet-4-6`, `gpt-5.5`, `grok-build-0.1`
- Evaluación doble: métrica heurística (rapidfuzz) + panel LLM-as-judge (3 jueces, 4 dimensiones)
- Temperatura fija en 0 para reproducibilidad

Los resultados consolidados (matriz con IC 95% bootstrap y test McNemar pareado) acompañan el documento principal del TFM.

## Cómo fue generado este dataset

El dataset es **sintético**, generado a partir de una especificación de dominio (fintech de crédito con garantía) usando LLMs como auxiliares de generación y revisión. Todas las entidades, nombres de servicio, equipos, eventos, bases de datos y socios son ficticios — cualquier semejanza con sistemas reales es incidental.

La generación fue auditada para consistencia cruzada mediante `qa-report.md` y validada con el script de consistencia anterior.

## Licencia y uso

Dataset desarrollado como parte del Trabajo Fin de Máster de Aurelio Saraiva (Universidad Europea, 2026). Puesto a disposición del tribunal evaluador para fines de validación y reproducción experimental.

Cita sugerida:

> Saraiva, A. (2026). *CredityFlow: dataset sintético para comparación de RAG y GraphRAG en asistentes técnicos de chat empresarial*. Trabajo Fin de Máster, Universidad Europea de Madrid.

## Contacto

Aurelio Saraiva — aurelio.saraiva@creditas.com.br
