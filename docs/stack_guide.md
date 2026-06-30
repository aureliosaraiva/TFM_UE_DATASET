# Guia de stack do projeto — TFM

## 1. Objetivo deste guia

Este documento define a **stack tecnológica recomendada** para implementar o projeto experimental do TFM, cujo objetivo é comparar duas abordagens de especialização de um assistente técnico em formato de chat:

* **abordagem não estruturada**;
* **abordagem estruturada**.

A stack proposta foi escolhida para maximizar:

* clareza metodológica;
* velocidade de implementação;
* separação entre abordagens;
* capacidade de benchmark e geração de relatórios;
* baixo atrito para experimentação acadêmica.

---

## 2. Decisão principal de stack

A recomendação para este TFM é usar uma arquitetura com:

### Backend principal

**Python 3.12**

### API de aplicação

**FastAPI**

### Interface web

**Streamlit**

### CLI de automação e benchmark

**Typer**

### Processamento de dados

**Pydantic**, **PyYAML**, **orjson**, **pandas**

### Abordagem não estruturada

**LlamaIndex** + **Qdrant**

### Abordagem estruturada

**NetworkX** inicialmente, com opção de evolução para **Neo4j**

### Avaliação e relatórios

**pandas**, **Jinja2**, **Markdown/HTML/CSV**

### Testes

**pytest**

Essa composição é a mais equilibrada para um TFM, porque permite construir rápido, manter rigor técnico e separar bem os experimentos.

---

## 3. Justificativa da stack

## 3.1. Por que Python

Python é a melhor escolha para este contexto porque:

* possui ecossistema forte para LLM, RAG, embeddings e avaliação;
* simplifica parsing de JSON, YAML e Markdown;
* facilita benchmark, análise de dados e geração de relatórios;
* permite combinar API, CLI e experimentação em uma mesma base.

## 3.2. Por que FastAPI

FastAPI é adequada porque:

* é simples e rápida para expor endpoints de chat e benchmark;
* integra bem com Pydantic;
* facilita separar módulos por abordagem;
* oferece boa base para testes e documentação automática.

## 3.3. Por que Streamlit

Para o TFM, a UI não precisa ser uma SPA complexa. O foco é o experimento. Streamlit atende bem porque:

* acelera a construção da interface;
* facilita uma tela de chat e uma tela de comparação lado a lado;
* permite expor resultados de benchmark e tabelas sem esforço excessivo;
* reduz custo de desenvolvimento front-end.

## 3.4. Por que Typer

Typer oferece uma CLI excelente para:

* ingestão;
* indexação;
* execução do benchmark;
* geração de relatórios;
* automação local.

## 3.5. Por que LlamaIndex

LlamaIndex é uma boa escolha para a abordagem não estruturada porque:

* já resolve partes importantes do pipeline de RAG;
* facilita loaders, chunking, retrieval e citação de fontes;
* permite manter o experimento em nível metodológico, sem reimplementar tudo do zero.

## 3.6. Por que Qdrant

Qdrant é recomendado porque:

* é simples de subir localmente com Docker;
* funciona bem com retrieval vetorial e híbrido;
* tem integração prática com LlamaIndex;
* é suficiente para um dataset do tamanho do TFM.

## 3.7. Por que NetworkX primeiro e Neo4j opcionalmente

Para a abordagem estruturada, há duas opções:

### Opção recomendada inicial

**NetworkX**

Vantagens:

* zero dependência externa adicional;
* ótimo para carregar `nodes.json` e `edges.json`;
* suficiente para o tamanho atual do grafo;
* ideal para demonstrar consulta relacional e traversals.

### Opção de evolução

**Neo4j**

Vantagens:

* melhor para demonstrar consulta com Cypher;
* mais próximo de uma arquitetura enterprise de knowledge graph;
* mais forte para extensões futuras.

### Decisão recomendada

Começar com **NetworkX** e deixar um adaptador opcional para **Neo4j**.

Assim, o projeto fica mais simples sem perder valor acadêmico.

---

## 4. Stack detalhada por camada

## 4.1. Camada de aplicação

### Linguagem

* Python 3.12

### Gerenciamento de dependências

* **uv** ou **poetry**

### Recomendação

Usar **uv** se o foco for velocidade e simplicidade.
Usar **poetry** se quiser mais familiaridade acadêmica com lockfile e scripts.

### Decisão sugerida

* **uv** para instalação e execução
* `pyproject.toml` como fonte central de configuração do projeto

---

## 4.2. API backend

### Framework

* FastAPI

### Servidor ASGI

* Uvicorn

### Endpoints mínimos esperados

* `POST /chat`
* `POST /benchmark/run`
* `GET /benchmark/runs/{id}`
* `POST /reports/generate`
* `GET /health`

---

## 4.3. Interface de usuário

### Framework

* Streamlit

### Telas mínimas

1. **Chat**

   * seletor de abordagem
   * campo de pergunta
   * resposta
   * fontes
   * contexto recuperado

2. **Comparação lado a lado**

   * mesma pergunta
   * resposta unstructured
   * resposta structured

3. **Benchmark**

   * iniciar execução
   * acompanhar progresso
   * listar runs

4. **Relatórios**

   * visualizar métricas agregadas
   * comparar duas execuções

---

## 4.4. Modelagem e validação

### Bibliotecas

* Pydantic v2
* PyYAML
* orjson

### Uso esperado

* modelos de domínio;
* contratos de entrada/saída;
* parsing do benchmark;
* parsing de YAML dos serviços;
* persistência de resultados.

---

## 4.5. Abordagem não estruturada

### Bibliotecas principais

* LlamaIndex
* Qdrant client
* sentence-transformers ou embeddings do provedor escolhido

### Componentes

* document loader
* markdown parser
* chunker
* embedding generator
* vector index
* retriever
* reranker opcional
* answer synthesizer

### Fonte documental

* `docs/**/*.md`
* `services/*/README.md`
* `services/*/architecture.md`
* opcionalmente `structured-spec.yaml` textualizado por configuração

### Estratégia recomendada

* chunking semântico ou por tamanho fixo com overlap
* top-k entre 5 e 8
* temperatura 0 para benchmark

---

## 4.6. Abordagem estruturada

### Bibliotecas principais

* NetworkX
* pandas
* PyYAML

### Componentes

* loader de `services.json`
* loader de `events.json`
* loader de `nodes.json` e `edges.json`
* loader de `structured-spec.yaml`
* graph builder
* query engine
* evidence selector
* answer synthesizer

### Estratégia recomendada

* criar índice por tipo de nó;
* criar índice por `service_name`, `event_name`, `team_name`, `domain`;
* implementar consultas por traversal e filtros;
* transformar resultado em evidência textual controlada antes de chamar o LLM.

### Fonte principal

* `structured/services.json`
* `structured/events.json`
* `structured/graph/nodes.json`
* `structured/graph/edges.json`
* `services/*/structured-spec.yaml`

---

## 4.7. LLM e camada de geração

### Recomendação arquitetural

Manter uma abstração de provedor para o modelo gerador final.

### Interface sugerida

* `LLMClient`
* `generate_answer(prompt, context, config)`

### Provedores possíveis

* OpenAI
* Anthropic
* local model, se necessário

### Recomendação prática para o TFM

Escolher **um único modelo principal** para a comparação experimental e manter constante entre as duas abordagens.

### Configuração recomendada para benchmark

* temperatura: 0
* max tokens controlado
* prompts versionados em arquivo

---

## 4.8. Benchmark e avaliação

### Bibliotecas

* pandas
* numpy
* rapidfuzz
* scikit-learn opcionalmente

### Funções do módulo

* carregar benchmark de 120 perguntas;
* executar por abordagem;
* salvar resultados;
* calcular métricas;
* comparar runs.

### Avaliação mínima

* success rate
* latency
* estimated cost
* source coverage
* similarity com resposta esperada
* entity coverage
* classificação `correct`, `partial`, `incorrect`, `error`

---

## 4.9. Relatórios

### Bibliotecas

* pandas
* Jinja2
* Markdown
* CSV

### Artefatos recomendados

* `summary.json`
* `summary.md`
* `detailed_results.csv`
* `comparison.md`
* `comparison.csv`

### Relatórios mínimos

1. relatório por abordagem
2. relatório comparativo
3. relatório por categoria de pergunta
4. relatório por dificuldade

---

## 4.10. Testes

### Biblioteca

* pytest

### Tipos de teste

* unitário para loaders e parsers
* integração para pipelines
* end-to-end para benchmark reduzido

### Cobertura mínima desejada

* loaders do dataset
* indexação unstructured
* construção do grafo structured
* benchmark runner
* geração de relatórios

---

## 5. Estrutura recomendada do repositório

```text
tfm-app/
├── pyproject.toml
├── README.md
├── .env.example
├── docker-compose.yml
├── TFM/
├── src/
│   └── tfm_app/
│       ├── core/
│       │   ├── config/
│       │   ├── models/
│       │   ├── llm/
│       │   ├── prompts/
│       │   ├── logging/
│       │   └── utils/
│       ├── datasets/
│       │   ├── loaders/
│       │   ├── parsers/
│       │   └── validators/
│       ├── approaches/
│       │   ├── unstructured/
│       │   │   ├── ingestion/
│       │   │   ├── indexing/
│       │   │   ├── retrieval/
│       │   │   ├── service/
│       │   │   └── prompts/
│       │   └── structured/
│       │       ├── ingestion/
│       │       ├── graph/
│       │       ├── query/
│       │       ├── service/
│       │       └── prompts/
│       ├── benchmark/
│       │   ├── runner/
│       │   ├── scoring/
│       │   └── reports/
│       ├── api/
│       ├── ui/
│       └── cli/
├── tests/
├── outputs/
└── scripts/
```

---

## 6. Persistência e infraestrutura

## 6.1. Infra mínima

### Recomendação

* aplicação local
* Qdrant via Docker
* restante em processo local

### `docker-compose.yml` mínimo

* qdrant

### Sem necessidade inicial

* banco relacional para app
* fila
* cache distribuído
* observabilidade complexa

Para o TFM, isso seria excesso.

---

## 6.2. Persistência de runs

Os resultados do benchmark podem ser armazenados inicialmente em:

* JSON
* CSV
* Markdown

Local sugerido:

```text
outputs/
  runs/
  reports/
  exports/
```

Se necessário, pode-se adicionar SQLite depois, mas não é obrigatório.

---

## 7. Configuração recomendada

Exemplo:

```yaml
app:
  dataset_root: ./TFM
  output_root: ./outputs

llm:
  provider: anthropic
  model: claude-3-7-sonnet-latest
  temperature: 0
  max_tokens: 1200

unstructured:
  enabled: true
  index_backend: qdrant
  chunk_size: 800
  chunk_overlap: 120
  top_k: 6
  rerank: true

structured:
  enabled: true
  graph_backend: networkx
  top_k_entities: 12
  top_k_relations: 20

benchmark:
  input_file: ./TFM/docs/questions/benchmark.json
  save_detailed_context: true
  judge_mode: heuristic
```

---

## 8. Dependências sugeridas

Exemplo de dependências de alto nível:

```toml
fastapi
uvicorn
streamlit
typer
pydantic
pyyaml
orjson
pandas
numpy
jinja2
pytest
qdrant-client
llama-index
networkx
rapidfuzz
python-dotenv
```

Dependendo do provedor do LLM:

```toml
anthropic
```

ou

```toml
openai
```

---

## 9. Decisões explícitas do projeto

## 9.1. Stack oficial recomendada

### Obrigatório

* Python 3.12
* FastAPI
* Streamlit
* Typer
* Pydantic
* pandas
* pytest

### Abordagem não estruturada

* LlamaIndex
* Qdrant

### Abordagem estruturada

* NetworkX
* PyYAML

### Relatórios

* pandas
* Jinja2
* Markdown/CSV

---

## 9.2. O que evitar neste momento

Para não aumentar desnecessariamente a complexidade do TFM, evitar inicialmente:

* frontend React separado;
* múltiplos bancos de infraestrutura;
* Kubernetes;
* mensageria real no app experimental;
* microserviços para a própria aplicação do TFM;
* Neo4j como dependência obrigatória desde o início;
* avaliação excessivamente complexa antes do runner básico funcionar.

---

## 10. Roadmap técnico sugerido

## Etapa 1 — Fundacional

* inicializar projeto Python;
* configurar FastAPI, Streamlit e Typer;
* criar modelos Pydantic;
* criar loaders do dataset.

## Etapa 2 — Unstructured

* parser de documentos;
* indexação em Qdrant;
* chat funcional da abordagem não estruturada.

## Etapa 3 — Structured

* builder do grafo com NetworkX;
* query engine;
* chat funcional da abordagem estruturada.

## Etapa 4 — Benchmark

* runner das 120 perguntas;
* persistência de resultados.

## Etapa 5 — Relatórios

* relatórios por execução;
* comparação entre abordagens.

## Etapa 6 — Refinamento

* comparação lado a lado na UI;
* melhora de scoring;
* ajustes metodológicos.

---

## 11. Critérios de aceite da stack

A stack será considerada adequada se permitir:

1. ingestão simples do dataset existente;
2. implementação clara das duas abordagens;
3. execução local sem infraestrutura pesada;
4. benchmark reproduzível;
5. geração de relatórios comparativos;
6. código suficientemente claro para ser descrito no TFM.

---

## 12. Instrução final para Claude Code

Implemente o projeto usando esta stack como padrão oficial, salvo se surgir impedimento técnico forte.

Priorize:

1. simplicidade operacional;
2. clareza metodológica;
3. separação entre as abordagens;
4. benchmark e relatórios;
5. extensibilidade controlada.

Sempre que houver dúvida de implementação, escolha a alternativa que:

* reduza complexidade desnecessária;
* preserve a justiça experimental;
* facilite explicar o projeto no texto acadêmico do TFM.

