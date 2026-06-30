
# Guia de implementação para Claude Code — TFM

## 1. Objetivo deste guia

Este documento define as instruções de implementação para construir a aplicação experimental do TFM com foco em **comparar duas estratégias de especialização de um assistente técnico em formato de chat**:

1. **Abordagem não estruturada**: o chat responde a partir de documentação textual recuperada do corpus.
2. **Abordagem estruturada**: o chat responde a partir de conhecimento estruturado, principalmente o grafo e os metadados do ecossistema.

A solução deve permitir:

* construir e executar as duas abordagens;
* deixar explícito onde começa e termina cada abordagem;
* reaproveitar componentes comuns quando fizer sentido;
* rodar toda a massa de teste de 120 perguntas;
* gerar relatórios comparativos reproduzíveis;
* servir como base metodológica e técnica do TFM.

---

## 2. Princípios arquiteturais do projeto

1. **Comparação justa**

   * O experimento deve alterar principalmente a **camada de conhecimento/retrieval**.
   * A interface de chat, o formato de resposta, os logs e a avaliação devem ser padronizados.
   * Sempre que possível, usar o mesmo modelo gerador final para ambas as abordagens.

2. **Separação clara entre abordagens**

   * Deve ficar evidente no código, na configuração e na execução qual fluxo pertence à abordagem estruturada e qual pertence à abordagem não estruturada.
   * Mesmo que exista uma única aplicação, ela deve ter fronteiras explícitas entre os módulos.

3. **Reprodutibilidade**

   * Toda execução de benchmark deve ser repetível.
   * Configurações, prompts, parâmetros de retrieval e geração devem estar versionados.

4. **Observabilidade e avaliação**

   * Toda pergunta executada deve gerar artefatos de avaliação, incluindo contexto recuperado, resposta final, tempo, custo estimado, fontes usadas e resultado esperado.

5. **Extensibilidade**

   * O projeto deve ser preparado para suportar no futuro uma terceira abordagem híbrida, mas sem misturar isso no experimento principal.

---

## 3. Escopo funcional

A aplicação deve oferecer os seguintes modos de operação:

### 3.1. Chat interativo

Modo para um usuário fazer perguntas manualmente e escolher qual abordagem deseja usar:

* `unstructured`
* `structured`

Opcionalmente, pode haver um modo futuro `hybrid`, mas ele não deve fazer parte da comparação principal do TFM.

### 3.2. Execução automatizada do benchmark

Modo para executar as 120 perguntas do dataset contra:

* abordagem não estruturada;
* abordagem estruturada.

Esse modo deve salvar resultados completos para posterior análise.

### 3.3. Geração de relatórios

Modo para consolidar os resultados e produzir relatórios comparativos.

---

## 4. Dataset disponível

A aplicação deve usar exatamente a seguinte estrutura de dataset como fonte primária de conhecimento:

```text
TFM/
├── docs/
│   ├── company-overview.md
│   ├── dataset-summary.md
│   ├── qa-report.md
│   ├── session/
│   │   └── generation-session-report.md
│   ├── events/
│   │   └── catalog.md
│   ├── teams/
│   │   ├── teams.md
│   │   └── databases.md
│   └── questions/
│       ├── benchmark.json
│       ├── benchmark_part1.json
│       ├── benchmark_part2.json
│       └── benchmark_part3.json
├── structured/
│   ├── services.json
│   ├── events.json
│   └── graph/
│       ├── nodes.json
│       └── edges.json
└── services/
    ├── lead-intake-service/
    │   ├── structured-spec.yaml
    │   ├── README.md
    │   └── architecture.md
    ├── eligibility-service/
    │   ├── ...
    ...
```

### 4.1. Uso esperado por abordagem

#### Abordagem não estruturada

Deve consumir principalmente:

* `docs/**/*.md`
* `services/*/README.md`
* `services/*/architecture.md`
* eventualmente `services/*/structured-spec.yaml` convertido em texto, se isso for explicitamente configurado

#### Abordagem estruturada

Deve consumir principalmente:

* `structured/services.json`
* `structured/events.json`
* `structured/graph/nodes.json`
* `structured/graph/edges.json`
* `services/*/structured-spec.yaml`

---

## 5. Requisito arquitetural central

A solução pode ser implementada como:

1. **duas aplicações separadas**, uma para cada abordagem; ou
2. **uma única aplicação com dois pipelines claramente separados**.

A recomendação para este TFM é:

## **uma aplicação única com arquitetura modular**, contendo dois pipelines independentes:

* `pipeline_unstructured`
* `pipeline_structured`

E módulos compartilhados para:

* interface de chat;
* carregamento do benchmark;
* execução batch;
* logging;
* avaliação;
* geração de relatórios.

Isso reduz duplicação, aumenta justiça experimental e facilita manutenção.

---

## 6. Estrutura sugerida do projeto

```text
app/
├── src/
│   ├── core/
│   │   ├── config/
│   │   ├── models/
│   │   ├── prompts/
│   │   ├── llm/
│   │   ├── logging/
│   │   ├── evaluation/
│   │   └── utils/
│   ├── datasets/
│   │   ├── loaders/
│   │   ├── parsers/
│   │   └── indexers/
│   ├── chat/
│   │   ├── api/
│   │   ├── application/
│   │   └── ui/
│   ├── approaches/
│   │   ├── unstructured/
│   │   │   ├── ingestion/
│   │   │   ├── indexing/
│   │   │   ├── retrieval/
│   │   │   ├── prompting/
│   │   │   └── service/
│   │   └── structured/
│   │       ├── ingestion/
│   │       ├── graph/
│   │       ├── query/
│   │       ├── prompting/
│   │       └── service/
│   ├── benchmark/
│   │   ├── runner/
│   │   ├── scoring/
│   │   └── reports/
│   └── cli/
│       ├── chat/
│       ├── benchmark/
│       └── reports/
├── data/
├── outputs/
│   ├── runs/
│   ├── reports/
│   └── exports/
└── README.md
```

---

## 7. Fronteiras entre as abordagens

## 7.1. O que é compartilhado

Os seguintes componentes devem ser compartilhados entre as duas abordagens:

* contrato da interface de chat;
* modelo de entrada da pergunta;
* modelo de saída da resposta;
* modelo de execução do benchmark;
* formato de persistência de resultados;
* sistema de logs e tracing;
* motor de avaliação;
* gerador de relatórios;
* configuração do modelo LLM final;
* template-base de resposta.

## 7.2. O que deve ser isolado

### Não estruturada

Deve ter isolamento em:

* ingestão documental;
* chunking;
* embeddings;
* indexação vetorial ou híbrida;
* retrieval documental;
* reranking, se existir;
* montagem de contexto textual.

### Estruturada

Deve ter isolamento em:

* ingestão de JSON/YAML estruturados;
* construção do índice relacional/grafo;
* motor de consulta estruturada;
* traversals e seleção de evidências;
* montagem de contexto factual/estrutural.

---

## 8. Contrato de resposta padronizado

Ambas as abordagens devem retornar a mesma estrutura lógica:

```json
{
  "question": "...",
  "approach": "unstructured | structured",
  "answer": "...",
  "sources": ["..."],
  "retrieved_context": ["..."],
  "latency_ms": 0,
  "metadata": {
    "model": "...",
    "tokens_input": 0,
    "tokens_output": 0,
    "estimated_cost": 0,
    "confidence": null
  }
}
```

Esse contrato deve ser o mesmo tanto no chat quanto no benchmark.

---

## 9. Requisitos da abordagem não estruturada

## 9.1. Objetivo

Responder perguntas a partir de documentação textual e artefatos semiestruturados convertidos em texto.

## 9.2. Entradas principais

* arquivos Markdown em `docs/`
* arquivos `README.md`
* arquivos `architecture.md`
* opcionalmente `structured-spec.yaml` textualizado por configuração

## 9.3. Pipeline esperado

1. carregar documentos;
2. normalizar metadados;
3. fragmentar em chunks;
4. gerar embeddings;
5. indexar;
6. recuperar top-k relevantes;
7. opcionalmente reranquear;
8. montar prompt final com contexto;
9. gerar resposta;
10. registrar fontes usadas.

## 9.4. Cuidados metodológicos

* preservar rastreabilidade da resposta para os documentos;
* evitar injetar dados do grafo ou estrutura nesta abordagem principal;
* manter separação clara para não contaminar a comparação.

---

## 10. Requisitos da abordagem estruturada

## 10.1. Objetivo

Responder perguntas a partir de conhecimento explicitamente estruturado sobre serviços, eventos, times, bancos, documentos e relações.

## 10.2. Entradas principais

* `structured/services.json`
* `structured/events.json`
* `structured/graph/nodes.json`
* `structured/graph/edges.json`
* `services/*/structured-spec.yaml`

## 10.3. Pipeline esperado

1. carregar dados estruturados;
2. validar integridade local;
3. construir representação navegável;
4. interpretar a pergunta;
5. selecionar estratégia de consulta;
6. executar consulta ou traversal;
7. reunir fatos e relações relevantes;
8. montar prompt final com evidências estruturadas;
9. gerar resposta;
10. registrar entidades e relações utilizadas.

## 10.4. Cuidados metodológicos

* priorizar fatos e relações explícitas;
* evitar depender de README ou `architecture.md` como fonte principal desta abordagem;
* garantir que a resposta reflita o grafo e os artefatos estruturados.

---

## 11. Requisito de interface

A aplicação deve permitir, no mínimo:

### 11.1. Chat por CLI

Comandos sugeridos:

```bash
app chat --approach unstructured
app chat --approach structured
```

### 11.2. Chat por UI web simples

A UI pode ser minimalista, mas deve permitir:

* selecionar abordagem;
* enviar pergunta;
* ver resposta;
* ver fontes;
* ver contexto recuperado.

### 11.3. Comparação manual

Seria desejável um modo de comparação lado a lado:

```bash
app chat --compare
```

Ou na UI:

* mesma pergunta;
* duas respostas exibidas lado a lado.

---

## 12. Requisito de benchmark

## 12.1. Fonte oficial do benchmark

Usar:

* `docs/questions/benchmark.json`

Os arquivos em partes podem ser usados apenas como suporte.

## 12.2. Execução obrigatória

Deve ser possível executar:

```bash
app benchmark run --approach unstructured
app benchmark run --approach structured
app benchmark run --approach all
```

## 12.3. Para cada pergunta, salvar

* pergunta original;
* categoria;
* dificuldade;
* conhecimento requerido;
* resposta esperada;
* abordagem usada;
* resposta produzida;
* fontes usadas;
* contexto recuperado;
* latência;
* custo estimado;
* status da execução;
* erros, quando houver.

## 12.4. Saída sugerida por execução

```text
outputs/runs/
  benchmark-unstructured-<timestamp>.json
  benchmark-structured-<timestamp>.json
  benchmark-all-<timestamp>.json
```

---

## 13. Avaliação e scoring

O sistema deve gerar avaliação comparável entre abordagens.

## 13.1. Métricas mínimas

* taxa de resposta gerada com sucesso;
* latência média;
* latência por categoria;
* custo estimado total;
* custo médio por pergunta;
* cobertura de fontes;
* aderência à resposta esperada;
* desempenho por categoria de pergunta;
* desempenho por dificuldade;
* desempenho por tipo de conhecimento.

## 13.2. Estratégia prática de scoring

Implementar dois níveis:

### Nível 1 — heurístico obrigatório

* comparação textual com resposta esperada;
* verificação de presença de entidades-chave;
* verificação de uso de fontes válidas;
* classificação básica: `correct`, `partial`, `incorrect`, `error`.

### Nível 2 — LLM-as-judge opcional

Se houver orçamento e controle:

* avaliar corretude;
* completude;
* groundedness;
* clareza.

Mas o modo heurístico deve existir independentemente.

---

## 14. Relatórios obrigatórios

A aplicação deve gerar relatórios reproduzíveis em formato legível.

## 14.1. Relatório por execução

Para cada run, gerar:

* resumo geral;
* métricas agregadas;
* tabela por categoria;
* tabela por dificuldade;
* exemplos de acertos;
* exemplos de falhas;
* top perguntas com maior latência;
* top perguntas com maior divergência da resposta esperada.

## 14.2. Relatório comparativo final

Gerar um relatório que compare diretamente:

* unstructured vs structured;
* acurácia/aderência;
* latência;
* custo;
* comportamento por tipo de pergunta;
* limitações observadas.

## 14.3. Formatos sugeridos

* Markdown para leitura humana;
* JSON para processamento;
* CSV opcional para tabelas consolidadas.

---

## 15. Comandos esperados

A aplicação deve expor comandos equivalentes a estes:

```bash
app ingest unstructured
app ingest structured
app chat --approach unstructured
app chat --approach structured
app benchmark run --approach unstructured
app benchmark run --approach structured
app benchmark run --approach all
app reports generate --latest
app reports compare --run-a <id> --run-b <id>
```

---

## 16. Configuração do projeto

Deve existir configuração explícita para:

* caminho do dataset;
* modelo LLM usado;
* parâmetros de retrieval;
* top-k;
* chunk size;
* temperatura;
* modo de avaliação;
* diretório de saída;
* ativação ou não de reranking.

Exemplo de arquivo de configuração:

```yaml
app:
  dataset_root: ./TFM
  output_dir: ./outputs
llm:
  provider: openai
  model: gpt-4.1-mini
  temperature: 0
unstructured:
  chunk_size: 800
  chunk_overlap: 120
  top_k: 6
  rerank: true
structured:
  top_k_entities: 12
  top_k_relations: 20
benchmark:
  input_file: ./TFM/docs/questions/benchmark.json
  judge_mode: heuristic
```

---

## 17. Não funcionais

## 17.1. Reprodutibilidade

* temperatura baixa ou zero no benchmark;
* versionamento de prompts;
* persistência de configuração por run.

## 17.2. Testabilidade

* testes unitários para loaders e parsers;
* testes de integração para pipelines;
* testes end-to-end para execução do benchmark.

## 17.3. Clareza acadêmica

* os nomes dos módulos devem refletir a abordagem;
* os artefatos de saída devem facilitar uso no TFM;
* o código deve deixar evidente como a comparação foi implementada.

---

## 18. Estratégia de implementação sugerida

## Fase 1

* criar loaders do dataset;
* criar modelos de domínio;
* criar contrato comum de resposta.

## Fase 2

* implementar pipeline unstructured;
* implementar indexação;
* validar chat básico.

## Fase 3

* implementar pipeline structured;
* validar consultas e chat básico.

## Fase 4

* implementar benchmark runner;
* persistência de resultados.

## Fase 5

* implementar scoring e relatórios.

## Fase 6

* refinar UX e comparação lado a lado.

---

## 19. Critérios de pronto

A aplicação será considerada pronta quando:

1. carregar corretamente o dataset fornecido;
2. responder perguntas em modo chat nas duas abordagens;
3. deixar explícito qual abordagem foi usada;
4. executar as 120 perguntas para ambas as abordagens;
5. salvar resultados completos por execução;
6. gerar relatórios comparativos;
7. manter separação metodológica clara entre estruturada e não estruturada.

---

## 20. Instrução final para Claude Code

Implemente este projeto como um artefato de pesquisa experimental, não como um protótipo genérico de chat.

Prioridades, nesta ordem:

1. correção metodológica;
2. separação clara entre abordagens;
3. reprodutibilidade do benchmark;
4. geração de relatórios úteis para o TFM;
5. qualidade de código e extensibilidade.

Sempre deixe explícito no código, na configuração e nos outputs:

* o que pertence à abordagem não estruturada;
* o que pertence à abordagem estruturada;
* o que é compartilhado;
* o que foi medido durante os experimentos.
