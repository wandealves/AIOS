---
Documento: Design Brief (Fonte Única de Verdade Interna)
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0115, ADR-0116, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001, RFC-0011 (proposta)
Depende de: 010-Memory, 017-Model-Router, 024-Observability, 022-Policy, 025-Audit, 020-Communication, 005-Database, 021-Security
---

# AIOS — Módulo 011 · Context — Design Brief

> **Natureza deste documento.** Este é o `_DESIGN_BRIEF.md`: a **fonte única de
> verdade interna** do módulo `011-Context`. Os 26 documentos obrigatórios
> (`Vision.md` … `RFC.md`) DEVEM ser derivados deste brief e **NÃO PODEM
> contradizê-lo**. Toda decisão de componente, estado, entidade, evento, código de
> erro, chave de configuração e ADR aqui fixada é normativa para o módulo. Onde
> este brief e um documento derivado divergirem, este brief prevalece até que uma
> ADR o altere explicitamente.
>
> Este documento reutiliza — sem redefinir — os contratos da
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) (URN
> `urn:aios:<tenant>:<tipo>:<id>`, envelope CloudEvents, subjects
> `aios.<tenant>.<dominio>.<entidade>.<acao>`, envelope de erro RFC 7807
> `AIOS-<DOMINIO>-<NNNN>`, `Idempotency-Key`, correlação `traceparent`/
> `X-AIOS-Tenant`) e o vocabulário do
> [Glossário](../040-Glossary/Glossary.md).

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Missão do módulo

O `011-Context` é o **gerenciador de janela de contexto** do AIOS. Sua função é
**montar o contexto ótimo por chamada de LLM dentro do orçamento de tokens do
modelo** (limites fornecidos pelo `017-Model-Router`), maximizando a preservação
de desempenho de tarefa (**Task Completion Rate**) enquanto minimiza tokens e
reinferências. É o análogo cognitivo do subsistema de **cache/paginação** de um SO
clássico: decide o que "mantém na página" (contexto ativo), o que "compacta" e o
que "evita recomputar" (cache semântico).

### 1.2 Responsabilidades (o módulo **DEVE**)

| # | Responsabilidade | Detalhe |
|---|------------------|---------|
| R-01 | **Assembly de contexto** | Montar um `ContextBundle` ótimo por chamada de LLM, respeitando o `token_budget` do modelo alvo. |
| R-02 | **Token budgeting** | Alocar o orçamento de tokens por tarefa/agente/seção (system, instruções, memória, histórico, ferramentas, saída reservada). |
| R-03 | **Compressão/sumarização hierárquica** | Reduzir tokens por sumarização map-reduce multinível, preservando semântica relevante. |
| R-04 | **Remoção de redundância** | Deduplicar e eliminar sobreposição entre fragmentos candidatos (near-duplicate detection). |
| R-05 | **Recuperação seletiva** | Solicitar ao `010-Memory` (recall) apenas os itens necessários e rankeá-los por relevância dentro do orçamento. |
| R-06 | **Cache semântico** | Manter cache de respostas/contextos indexado por similaridade (Redis + `pgvector`), com chave por embedding, para reduzir custo e latência. |
| R-07 | **Contagem/estimativa de tokens** | Contar tokens por tokenizer do modelo alvo (contrato via `017`). |
| R-08 | **Emissão de telemetria e eventos** | Emitir métricas (`Context Compression Ratio`, cache hit ratio) e eventos NATS `aios.<tenant>.context.window.*`. |
| R-09 | **Governança e isolamento** | Aplicar PEP/PDP (`022`), isolamento multi-tenant por `tenant_id` e trilha de auditoria (`025`). |
| R-10 | **Degradação graciosa** | Fornecer contexto viável (truncamento seguro) quando dependências (`010`/`017`) degradam. |

### 1.3 Não-responsabilidades (o módulo **NÃO DEVE**)

| # | Não-responsabilidade | Dono correto |
|---|----------------------|--------------|
| N-01 | Chamar diretamente provedores de LLM para **inferência final** da tarefa. | `017-Model-Router` |
| N-02 | Persistir memória de longo prazo, consolidar ou esquecer itens de memória. | `010-Memory` |
| N-03 | Decidir **qual modelo** executa a tarefa ou seus limites de janela. | `017-Model-Router` |
| N-04 | Executar ferramentas/tools ou hospedar o loop ReAct. | `015-Tool-Manager`, `007-Agent-Runtime` |
| N-05 | Manter o grafo de conhecimento / GraphRAG. | `018/019-Knowledge` |
| N-06 | Definir políticas RBAC/ABAC (apenas as **consome** como PEP). | `022-Policy` |
| N-07 | Escalonar agentes/tarefas ou aplicar cotas de CPU/RAM. | `009-Scheduler`, `006-Kernel` |
| N-08 | Ser a fonte da verdade de auditoria (apenas **emite** eventos auditáveis). | `025-Audit` |

> **Fronteira de compressão vs. sumarização de memória.** O `010-Memory` decide o
> que *é lembrado* e por quanto tempo; o `011-Context` decide o que *entra na
> janela agora* e sob qual forma comprimida. A sumarização feita aqui é **efêmera
> e por chamada**; sumários reutilizáveis de longo prazo pertencem ao `010`. O
> `011` PODE materializar `SummaryNode` como cache de curta duração, nunca como
> memória canônica.

---

## 2. Decomposição em Componentes Internos

### 2.1 Tabela de componentes

| Componente (PascalCase) | Responsabilidade | Depende de (interno/externo) |
|-------------------------|------------------|------------------------------|
| `ContextApiGateway` | Superfície REST/gRPC do módulo; validação, authN/Z (PEP), idempotência, versionamento. | `ContextAssembler`, `SemanticCacheManager`, `022-Policy`, `021-Security` |
| `ContextAssembler` | Orquestrador do pipeline de montagem; coordena budget→retrieve→rank→dedup→compress→emit. | Todos os componentes abaixo |
| `TokenBudgeter` | Calcula e aloca o orçamento por seção a partir dos limites do modelo e do `BudgetProfile`. | `ModelRouterClient`, `TokenCounter` |
| `TokenCounter` | Conta/estima tokens por tokenizer do modelo alvo. | `ModelRouterClient` (registry de tokenizers) |
| `SelectiveRetriever` | Solicita recall seletivo ao `010-Memory` e coleta candidatos de contexto. | `MemoryClient` (`010`) |
| `RelevanceRanker` | Rankeia/scoreia fragmentos candidatos (similaridade + recência + prioridade). | `EmbeddingClient`, `SelectiveRetriever` |
| `RedundancyEliminator` | Detecta e remove near-duplicates e sobreposição semântica entre fragmentos. | `EmbeddingClient` |
| `HierarchicalCompressor` | Compressão/sumarização map-reduce multinível; extrai `SummaryNode`. | `SummarizationClient` (via `017`), `TokenCounter` |
| `SemanticCacheManager` | Lookup/store/invalidação do cache semântico (Redis L1 + `pgvector` L2), chave por similaridade. | `EmbeddingClient`, Redis, PostgreSQL |
| `EmbeddingClient` | Gera embeddings de prompts/fragmentos para chave de cache e ranking. | `017-Model-Router` (modelo de embedding) |
| `EvictionManager` | Política de eviction/expiração do cache (TTL + LRU semântico + event-driven). | `SemanticCacheManager`, Redis |
| `BundleStore` | Persistência de `ContextBundle`/`ContextFragment`; offload de blobs grandes ao MinIO. | PostgreSQL, MinIO |
| `ContextPolicyGuard` | PEP local: consulta o PDP (`022`) para operações privilegiadas e limites de política. | `022-Policy` |
| `EventPublisher` | Publica eventos `aios.<tenant>.context.*` via outbox transacional. | NATS/JetStream, `BundleStore` |
| `ModelRouterClient` | Cliente gRPC para limites de modelo, tokenizer e endpoints de embedding/sumarização (`017`). | `017-Model-Router` |
| `MemoryClient` | Cliente gRPC/NATS para recall seletivo no `010-Memory`. | `010-Memory` |
| `TelemetryEmitter` | Métricas OTel/Prometheus, traces e logs estruturados (Serilog→Seq). | `024-Observability` |

### 2.2 Diagrama de componentes (C4 Nível 3, ASCII)

```
┌──────────────────────────── CONTEXT SERVICE (011) ─────────────────────────────────┐
│                                                                                      │
│   REST/gRPC (X-AIOS-Tenant, Idempotency-Key, traceparent)                            │
│        │                                                                             │
│  ┌─────▼──────────────┐        ┌──────────────────────┐                              │
│  │ ContextApiGateway  │───PEP─▶│ ContextPolicyGuard    │──▶ Policy Engine (022)       │
│  │ (validação/authZ)  │        └──────────────────────┘                              │
│  └─────┬──────────────┘                                                              │
│        │ assemble()                                                                  │
│  ┌─────▼───────────────────────────────────────────────────────────────────────┐   │
│  │                          ContextAssembler (orquestrador)                       │   │
│  │                                                                                │   │
│  │  1.┌──────────────┐  2.┌────────────────────┐  ──cache lookup──┐               │   │
│  │    │ TokenBudgeter│    │ SemanticCacheManager│◀────────────────┘               │   │
│  │    └──────┬───────┘    └──────┬──────────────┘   HIT? ─────────► serve + emit   │   │
│  │           │ limites           │ L1 Redis / L2 pgvector                          │   │
│  │  ┌────────▼───────┐   3.┌─────▼──────────┐  4.┌──────────────────┐              │   │
│  │  │ TokenCounter   │     │SelectiveRetrievr│──▶│ RelevanceRanker  │              │   │
│  │  └────────────────┘     └─────┬──────────┘   └────────┬─────────┘              │   │
│  │           ▲                   │ recall              5.│                          │   │
│  │           │             ┌─────▼──────────┐   ┌────────▼──────────┐             │   │
│  │           │             │ RedundancyElim. │──▶│HierarchicalCompr. │             │   │
│  │           │             └────────────────┘   └────────┬──────────┘             │   │
│  │  ┌────────┴────────┐         ▲   ▲                 6.  │ assembled              │   │
│  │  │ EmbeddingClient │─────────┘   │            ┌────────▼──────────┐             │   │
│  │  └────────┬────────┘             │            │ BundleStore       │──▶ MinIO    │   │
│  │           │                      │            └────────┬──────────┘  (blobs)    │   │
│  └───────────┼──────────────────────┼─────────────────────┼─────────────────────┘   │
│              │                      │                      │ outbox                   │
│  ┌───────────▼────────┐  ┌──────────▼────────┐   ┌─────────▼─────────┐               │
│  │ ModelRouterClient  │  │ MemoryClient      │   │ EventPublisher    │──▶ NATS/JS     │
│  │ (limites/tokenizer/│  │ (recall seletivo) │   │ (context.window.*)│  aios.<t>.ctx │
│  │  embedding/summ.)  │  └──────────┬────────┘   └───────────────────┘               │
│  └───────────┬────────┘             │                                                 │
│              │                 ┌─────▼──────┐   ┌──────────────────┐                   │
│              ▼                 │ EvictionMgr│   │ TelemetryEmitter │──▶ OTel/Prom/Seq  │
│      Model Router (017)        └────────────┘   └──────────────────┘                   │
│                                                                                      │
│  Persistência: PostgreSQL(+pgvector) · Redis(L1 cache/locks) · MinIO(blobs)          │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

**Padrão arquitetural do fluxo:** `Gateway → PEP → Orquestrador → (cache lookup) →
retrieve → rank → dedup → compress → store → outbox/eventos`, alinhado ao padrão
canônico da Arquitetura Global (§5 de `../001-Architecture/Architecture.md`).

---

## 3. Modelo de Dados / Entidades Canônico

> Todos os recursos seguem a identidade URN da RFC-0001 §5.1
> (`urn:aios:<tenant>:<tipo>:<id>`, `<id>` = ULID). Este módulo **propõe** ao
> registro de tipos (`004-API/Resources.md`) os novos `<tipo>`: **`context`**
> (bundle), **`ctxfrag`** (fragmento) e **`ctxcache`** (entrada de cache). Todas as
> tabelas têm `tenant_id` obrigatório e **Row-Level Security** por `tenant_id`.

### 3.1 `ContextBundle` — contexto montado por chamada

URN: `urn:aios:<tenant>:context:<ulid>`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `bundle_id` | `ULID` | PK | Identidade do bundle. |
| `tenant_id` | `text` | NN, RLS, IDX | Fronteira de isolamento. |
| `agent_urn` | `text` | NN, IDX | URN do agente solicitante. |
| `task_urn` | `text` | NULL, IDX | URN da tarefa, quando aplicável. |
| `model_id` | `text` | NN | Modelo alvo (do `017`). |
| `model_max_tokens` | `int` | NN | Limite de janela informado por `017`. |
| `token_budget` | `int` | NN | Orçamento de entrada alocado. |
| `reserved_output_tokens` | `int` | NN | Reserva para geração. |
| `tokens_in` | `int` | NN | Tokens finais montados. |
| `tokens_raw` | `int` | NN | Tokens antes da compressão. |
| `compression_ratio` | `numeric(5,4)` | NN | `1 - tokens_in/tokens_raw`. |
| `status` | `text` | NN, CHECK enum | Estado (ver §4.1). |
| `cache_outcome` | `text` | NULL, CHECK | `HIT`/`MISS`/`BYPASS`. |
| `idempotency_key` | `text` | UQ(tenant_id,key) | Dedup de requisições. |
| `budget_profile_id` | `ULID` | FK→BudgetProfile | Perfil aplicado. |
| `trace_id` | `text` | NN | Correlação OTel. |
| `created_at` | `timestamptz` | NN, IDX | Criação. |
| `expires_at` | `timestamptz` | NN | TTL do bundle (efêmero). |
| `dataschema` | `text` | NN | Versão do schema. |

### 3.2 `ContextFragment` — pedaço de contexto candidato/incluído

URN: `urn:aios:<tenant>:ctxfrag:<ulid>`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `fragment_id` | `ULID` | PK | Identidade. |
| `tenant_id` | `text` | NN, RLS | Isolamento. |
| `bundle_id` | `ULID` | FK→ContextBundle, IDX | Bundle dono. |
| `source_kind` | `text` | NN, CHECK | `system`\|`instruction`\|`memory`\|`knowledge`\|`history`\|`tool_result`. |
| `source_urn` | `text` | NULL | URN de origem (ex.: `memory` item). |
| `role` | `text` | NN | `system`\|`user`\|`assistant`\|`tool`. |
| `ordinal` | `int` | NN | Ordem de montagem. |
| `tokens_raw` | `int` | NN | Tokens originais. |
| `tokens_final` | `int` | NN | Tokens após compressão. |
| `relevance_score` | `numeric(6,5)` | NN | Score do `RelevanceRanker`. |
| `compression_method` | `text` | NN, CHECK | `none`\|`extractive`\|`abstractive`\|`truncate`\|`dedup`. |
| `included` | `bool` | NN | Entrou no bundle final? |
| `embedding` | `vector(1536)` | pgvector, IVFFLAT | Embedding do fragmento. |
| `content_inline` | `text` | NULL | Conteúdo (se pequeno). |
| `content_ref` | `text` | NULL | Ponteiro MinIO (se grande). |
| `created_at` | `timestamptz` | NN | Criação. |

### 3.3 `SemanticCacheEntry` — entrada de cache semântico

URN: `urn:aios:<tenant>:ctxcache:<ulid>`

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `cache_id` | `ULID` | PK | Identidade. |
| `tenant_id` | `text` | NN, RLS, IDX | Isolamento (namespace de cache). |
| `cache_scope` | `text` | NN, CHECK | `tenant`\|`agent`\|`task_type`. |
| `scope_ref` | `text` | NULL | URN do escopo (agente/tipo). |
| `model_id` | `text` | NN, IDX | Modelo que produziu a resposta. |
| `prompt_hash` | `text` | UQ(tenant,model,hash) | Hash exato do prompt normalizado (fast path). |
| `prompt_embedding` | `vector(1536)` | pgvector, HNSW | Chave de similaridade (slow path). |
| `similarity_threshold` | `numeric(4,3)` | NN | Limiar mínimo de aceitação. |
| `response_ref` | `text` | NN | Ponteiro MinIO/inline do payload cacheado. |
| `tokens_saved` | `int` | NN | Tokens evitados por hit. |
| `cost_saved_usd` | `numeric(12,6)` | NN | Custo estimado evitado. |
| `hit_count` | `bigint` | NN, default 0 | Contador de hits. |
| `ttl_seconds` | `int` | NN | Expiração base. |
| `created_at` | `timestamptz` | NN | Criação. |
| `last_hit_at` | `timestamptz` | NULL, IDX | Recência (LRU semântico). |
| `invalidated_at` | `timestamptz` | NULL | Marca de invalidação. |

### 3.4 `BudgetProfile` — perfil de orçamento de tokens

URN: `urn:aios:<tenant>:policy:<ulid>` (subtipo lógico `context-budget`)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `profile_id` | `ULID` | PK | Identidade. |
| `tenant_id` | `text` | NN, RLS | Isolamento. |
| `scope` | `text` | NN, CHECK | `tenant`\|`agent`\|`task_type`. |
| `scope_ref` | `text` | NULL | URN/rótulo do escopo. |
| `reserved_output_ratio` | `numeric(4,3)` | NN, default 0.25 | Fração reservada p/ saída. |
| `alloc_system` | `numeric(4,3)` | NN | Peso da seção system. |
| `alloc_instruction` | `numeric(4,3)` | NN | Peso de instruções. |
| `alloc_memory` | `numeric(4,3)` | NN | Peso de memória recuperada. |
| `alloc_history` | `numeric(4,3)` | NN | Peso do histórico. |
| `alloc_tools` | `numeric(4,3)` | NN | Peso de resultados de ferramentas. |
| `min_compression_ratio` | `numeric(4,3)` | NN, default 0.30 | Meta mínima de compressão. |
| `updated_at` | `timestamptz` | NN | Última alteração. |

> **Invariante de alocação:** `alloc_system + alloc_instruction + alloc_memory +
> alloc_history + alloc_tools = 1.0` (validado em escrita, erro `AIOS-CTX-0008`).

### 3.5 `SummaryNode` — nó de sumário hierárquico (cache de curta duração)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `node_id` | `ULID` | PK | Identidade. |
| `tenant_id` | `text` | NN, RLS | Isolamento. |
| `source_urn` | `text` | NN, IDX | Origem sumarizada. |
| `level` | `int` | NN | Nível na hierarquia (0=folha). |
| `parent_id` | `ULID` | FK self, NULL | Nó pai. |
| `tokens` | `int` | NN | Tamanho do sumário. |
| `content_ref` | `text` | NN | Conteúdo (inline/MinIO). |
| `embedding` | `vector(1536)` | pgvector | Para reuso por similaridade. |
| `expires_at` | `timestamptz` | NN | TTL de reuso. |

### 3.6 Relacionamentos (ASCII)

```
BudgetProfile 1───* ContextBundle 1───* ContextFragment
                          │                    │
                          │                    └──(embedding)──► pgvector
                          └──(cache_outcome)──► SemanticCacheEntry *───1 (model_id)
SummaryNode 1───* SummaryNode  (árvore de sumarização hierárquica, self-FK)
```

---

## 4. Máquinas de Estado Canônicas

### 4.1 `ContextBundle` — pipeline de montagem

Estados: `RECEIVED`, `POLICY_CHECK`, `CACHE_LOOKUP`, `BUDGET_ALLOCATED`,
`RETRIEVING`, `RANKING`, `COMPRESSING`, `ASSEMBLED`, `SERVED`, `REJECTED`,
`DEGRADED`, `FAILED`, `EXPIRED`. Terminais: `SERVED`, `REJECTED`, `FAILED`,
`EXPIRED`.

```
            ┌──────────┐  authZ deny (AIOS-CTX-0011)     ┌──────────┐
 request ─▶ │ RECEIVED │──────────────────────────────▶ │ REJECTED │◀─┐
            └────┬─────┘                                 └──────────┘  │
                 │ validate ok                                        │ budget infeasible
            ┌────▼────────┐  PDP deny                                 │ (AIOS-CTX-0001)
            │ POLICY_CHECK│───────────────────────────────────────────┤
            └────┬────────┘                                           │
                 │ allow                                              │
            ┌────▼────────┐  HIT ─────────► emit cache_hit ──────┐    │
            │ CACHE_LOOKUP│                                      │    │
            └────┬────────┘  MISS/BYPASS                         │    │
                 │ emit cache_miss                               │    │
            ┌────▼──────────────┐  017 down                      │    │
            │ BUDGET_ALLOCATED  │──┐ (fallback default limits)   │    │
            └────┬──────────────┘  │                             │    │
                 │ ok              ▼                             │    │
            ┌────▼──────┐   ┌────────────┐  010 timeout          │    │
            │ RETRIEVING│──▶│  DEGRADED  │◀──(AIOS-CTX-0005)     │    │
            └────┬──────┘   └─────┬──────┘                       │    │
                 │ candidates     │ partial context             │    │
            ┌────▼────┐           │                             │    │
            │ RANKING │           │                             │    │
            └────┬────┘           │                             │    │
            ┌────▼──────────┐     │ compression fail            │    │
            │  COMPRESSING  │─────┤ (AIOS-CTX-0004 → truncate)  │    │
            └────┬──────────┘     │                             │    │
                 │ within budget  │                             │    │
            ┌────▼──────┐◀────────┘                             │    │
            │ ASSEMBLED │──────────────────────────────────────┘    │
            └────┬──────┘                                            │
                 │ deliver + persist + outbox                        │
            ┌────▼───┐   TTL                     unrecoverable error │
            │ SERVED │──────────► EXPIRED        ──────────► FAILED ◀─┘
            └────────┘
```

**Guardas principais:**

| Transição | Guarda (condição) |
|-----------|-------------------|
| `RECEIVED→POLICY_CHECK` | Payload válido; `tenant` do token == `X-AIOS-Tenant`. |
| `POLICY_CHECK→CACHE_LOOKUP` | PDP `allow` (`022`). |
| `CACHE_LOOKUP→SERVED` | `similarity ≥ similarity_threshold` E entrada não invalidada. |
| `CACHE_LOOKUP→BUDGET_ALLOCATED` | `MISS` OU `bypass_cache=true`. |
| `BUDGET_ALLOCATED→RETRIEVING` | `token_budget > 0` E alocação viável. |
| `*→REJECTED` | Orçamento infeasível (`AIOS-CTX-0001`) OU authZ deny (`AIOS-CTX-0011`). |
| `RETRIEVING/COMPRESSING→DEGRADED` | Dependência `010`/`017` indisponível dentro do deadline. |
| `COMPRESSING→ASSEMBLED` | `tokens_in ≤ token_budget`. |
| `ASSEMBLED→SERVED` | Persistência + publicação outbox confirmadas. |
| `SERVED→EXPIRED` | `now > expires_at`. |

### 4.2 `SemanticCacheEntry` — ciclo de vida

Estados: `PENDING`, `INDEXED`, `SERVING`, `STALE`, `INVALIDATED`, `EVICTED`.

```
 store() ─▶ PENDING ──embedding+persist──▶ INDEXED ──hit──▶ SERVING
                                              │                │
                             TTL/decay ───────┼────────────────┘
                                              ▼
                                            STALE ──event/ttl──▶ EVICTED
                                              ▲
                     memory.item.consolidated │  (invalidação event-driven)
                              ────────────────┴──────────────▶ INVALIDATED ─▶ EVICTED
```

**Guardas:** `INDEXED→SERVING` requer `similarity ≥ threshold`;
`INDEXED/SERVING→INVALIDATED` disparado por evento de mudança da origem (`010`/
`019`) que sobrepõe o item; `STALE/INVALIDATED→EVICTED` pelo `EvictionManager`
(TTL expirado, pressão de memória por LRU semântico, ou expurgo LGPD).

---

## 5. Superfície de API (REST e gRPC)

> Todas as rotas seguem RFC-0001: versionadas por caminho (`/v1`), exigem
> `X-AIOS-Tenant`, `traceparent`; mutações exigem `Idempotency-Key`. Erros no
> envelope RFC 7807 com `code = AIOS-CTX-<NNNN>`.

### 5.1 REST (principais)

| Método | Rota | Descrição | Idempotente |
|--------|------|-----------|-------------|
| `POST` | `/v1/context/assemble` | Monta um `ContextBundle` ótimo p/ o modelo alvo. | Sim (via key) |
| `GET` | `/v1/context/bundles/{bundle_id}` | Recupera bundle montado. | Sim |
| `POST` | `/v1/context/compress` | Comprime/sumariza um conjunto de fragmentos avulsos. | Sim (via key) |
| `POST` | `/v1/context/cache/lookup` | Consulta cache semântico por prompt/embedding. | Sim |
| `POST` | `/v1/context/cache/entries` | Armazena resposta no cache semântico. | Sim (via key) |
| `DELETE` | `/v1/context/cache/entries/{cache_id}` | Invalida entrada específica. | Sim |
| `POST` | `/v1/context/cache/invalidate` | Invalida por `scope`/`source_urn` (event-driven manual). | Sim (via key) |
| `POST` | `/v1/context/tokens/estimate` | Estima tokens de um payload p/ um `model_id`. | Sim |
| `GET` | `/v1/context/budgets/{scope}` | Lê o `BudgetProfile` efetivo. | Sim |
| `PUT` | `/v1/context/budgets/{scope}` | Cria/atualiza `BudgetProfile`. | Sim (via key) |

### 5.2 gRPC (`aios.context.v1`)

```
service ContextService {
  rpc Assemble(AssembleRequest) returns (AssembleResponse);       // POST /assemble
  rpc GetBundle(GetBundleRequest) returns (ContextBundle);
  rpc Compress(CompressRequest) returns (CompressResponse);
  rpc CacheLookup(CacheLookupRequest) returns (CacheLookupResponse);
  rpc CacheStore(CacheStoreRequest) returns (CacheStoreResponse);
  rpc InvalidateCache(InvalidateCacheRequest) returns (InvalidateCacheResponse);
  rpc EstimateTokens(EstimateTokensRequest) returns (EstimateTokensResponse);
  rpc GetBudget(GetBudgetRequest) returns (BudgetProfile);
  rpc UpsertBudget(UpsertBudgetRequest) returns (BudgetProfile);
}
```

### 5.3 Catálogo de códigos de erro `AIOS-CTX-<NNNN>`

| Código | HTTP/gRPC | Título | Retriable | Semântica |
|--------|-----------|--------|-----------|-----------|
| `AIOS-CTX-0001` | 422 / FAILED_PRECONDITION | BudgetInfeasible | não | Contexto mínimo não cabe no orçamento do modelo. |
| `AIOS-CTX-0002` | 503 / UNAVAILABLE | ModelLimitUnavailable | sim | `017` não retornou limites do modelo. |
| `AIOS-CTX-0003` | 503 / UNAVAILABLE | TokenizerUnavailable | sim | Tokenizer do modelo indisponível. |
| `AIOS-CTX-0004` | 500 / INTERNAL | CompressionFailed | sim | Falha na sumarização; fallback truncate aplicado. |
| `AIOS-CTX-0005` | 504 / DEADLINE_EXCEEDED | MemoryRecallTimeout | sim | `010` excedeu o deadline de recall. |
| `AIOS-CTX-0006` | 503 / UNAVAILABLE | CacheBackendUnavailable | sim | Redis/pgvector indisponível; bypass de cache. |
| `AIOS-CTX-0007` | 404 / NOT_FOUND | BundleNotFound | não | `bundle_id` inexistente/expirado. |
| `AIOS-CTX-0008` | 422 / INVALID_ARGUMENT | InvalidBudgetProfile | não | Pesos de alocação não somam 1.0 ou faixa inválida. |
| `AIOS-CTX-0009` | 503 / UNAVAILABLE | EmbeddingServiceUnavailable | sim | Modelo de embedding indisponível (`017`). |
| `AIOS-CTX-0010` | 502 / UNAVAILABLE | SummarizationModelError | sim | Erro no modelo de sumarização. |
| `AIOS-CTX-0011` | 403 / PERMISSION_DENIED | TenantMismatch | não | `tenant` do token ≠ contexto (isolamento). |
| `AIOS-CTX-0012` | 409 / ABORTED | IdempotencyConflict | não | `Idempotency-Key` reusada com payload divergente. |
| `AIOS-CTX-0013` | 413 / RESOURCE_EXHAUSTED | PayloadTooLarge | não | Payload de entrada excede limite configurado. |
| `AIOS-CTX-0014` | 400 / INVALID_ARGUMENT | ValidationError | não | Requisição malformada. |
| `AIOS-CTX-0015` | 429 / RESOURCE_EXHAUSTED | RateLimited | sim | Limite de taxa por tenant excedido. |

---

## 6. Catálogo de Eventos NATS

> Domínio de subject: **`context`** (RFC-0001 §5.3). Envelope CloudEvents (§5.2).
> At-least-once + idempotência por `event.id`. Produtor: `EventPublisher` (outbox).

### 6.1 Emitidos

| Subject | `type` | Quando | Payload-chave |
|---------|--------|--------|---------------|
| `aios.<tenant>.context.window.assembled` | `aios.context.window.assembled` | Bundle montado (`ASSEMBLED→SERVED`). | `bundleId, modelId, tokensRaw, tokensIn, compressionRatio, budget` |
| `aios.<tenant>.context.window.compressed` | `aios.context.window.compressed` | Compressão aplicada a fragmentos. | `bundleId, method, tokensBefore, tokensAfter, ratio` |
| `aios.<tenant>.context.window.cache_hit` | `aios.context.window.cache_hit` | Lookup resolvido por cache. | `cacheId, similarity, tokensSaved, costSavedUsd` |
| `aios.<tenant>.context.window.cache_miss` | `aios.context.window.cache_miss` | Lookup sem hit válido. | `modelId, reason` |
| `aios.<tenant>.context.window.rejected` | `aios.context.window.rejected` | Orçamento infeasível/authZ deny. | `bundleId, code, reason` |
| `aios.<tenant>.context.cache.evicted` | `aios.context.cache.evicted` | Entrada removida (TTL/LRU/expurgo). | `cacheId, cause` |
| `aios.<tenant>.context.cache.invalidated` | `aios.context.cache.invalidated` | Entrada invalidada por mudança de origem. | `cacheId, sourceUrn` |
| `aios.<tenant>.context.budget.updated` | `aios.context.budget.updated` | `BudgetProfile` alterado. | `profileId, scope` |

### 6.2 Consumidos

| Subject (origem) | Produtor | Ação no `011` |
|------------------|----------|---------------|
| `aios.<tenant>.memory.item.consolidated` | `010-Memory` | Invalidar/atualizar cache e `SummaryNode` que dependem do item. |
| `aios.<tenant>.memory.item.deleted` | `010-Memory` | Invalidar cache + expurgo (LGPD). |
| `aios.<tenant>.model.registry.updated` | `017-Model-Router` | Atualizar limites/tokenizers; invalidar budget cacheado. |
| `aios.<tenant>.knowledge.node.updated` | `018/019` | Invalidar cache derivado de conhecimento. |
| `aios.<tenant>.agent.lifecycle.suspended` | `006/008` | Evict de contexto/cache por-agente (opcional, escopo agente). |
| `aios.<tenant>.agent.lifecycle.terminated` | `006/008` | Expurgo de bundles/fragments efêmeros do agente. |
| `aios.<tenant>.policy.decision.updated` | `022-Policy` | Recarregar limites de política aplicáveis. |

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito | Prioridade | Critério de aceite |
|----|-----------|------------|--------------------|
| FR-001 | Montar `ContextBundle` respeitando `token_budget` do modelo alvo. | MUST | `tokens_in ≤ token_budget` em 100% dos bundles `SERVED`. |
| FR-002 | Alocar orçamento por seção via `BudgetProfile`. | MUST | Alocação respeita pesos ±2%; invariante soma=1.0 validada. |
| FR-003 | Comprimir por sumarização hierárquica map-reduce. | MUST | Reduz tokens preservando itens de alta relevância (score ≥ p90). |
| FR-004 | Remover redundância entre fragmentos. | MUST | Near-duplicates (cos ≥ 0.95) colapsados; sem perda de item único. |
| FR-005 | Recuperar seletivamente do `010-Memory`. | MUST | Recall limitado ao top-K por relevância dentro do orçamento. |
| FR-006 | Cache semântico com chave por similaridade (Redis+pgvector). | MUST | Hit quando `similarity ≥ threshold`; miss caso contrário. |
| FR-007 | Contar/estimar tokens por tokenizer do modelo. | MUST | Erro de estimativa ≤ 2% vs. contagem exata do provedor. |
| FR-008 | Invalidar cache por evento de mudança de origem. | MUST | Invalidação ≤ 5 s após `memory.item.consolidated`. |
| FR-009 | Emitir eventos `context.window.*` e métricas. | MUST | 100% dos assemblies emitem `assembled`/`cache_*`. |
| FR-010 | Degradação graciosa quando `010`/`017` indisponíveis. | MUST | Retorna bundle truncado seguro; estado `DEGRADED`. |
| FR-011 | Idempotência de `assemble`/`cache store`/`budget upsert`. | MUST | Repetição com mesma key retorna mesmo resultado (≥24h). |
| FR-012 | Expurgo LGPD de bundles/cache por `source_urn`/tenant. | SHOULD | Expurgo rastreável emite `cache.evicted` + auditoria. |
| FR-013 | Offload de fragmentos grandes ao MinIO. | SHOULD | Fragmento > `max_inline_bytes` armazenado por `content_ref`. |
| FR-014 | Reuso de `SummaryNode` por similaridade. | COULD | Sumário reutilizado quando `similarity ≥ reuse_threshold` e não expirado. |

### 7.2 Requisitos Não-Funcionais (NFR) — metas numéricas (SLO/SLI)

| ID | Atributo | SLI | SLO (meta) | Verificação |
|----|----------|-----|------------|-------------|
| NFR-001 | Desempenho (cache hit) | latência `CacheLookup` p99 | ≤ 20 ms | Benchmark + Prometheus histogram. |
| NFR-002 | Desempenho (assembly miss) | latência `Assemble` p99 (sem sumarização LLM) | ≤ 150 ms | Load test. |
| NFR-003 | Desempenho (com sumarização) | latência `Assemble` p99 (com 1 chamada summ.) | ≤ 1200 ms | Load test. |
| NFR-004 | Eficácia de compressão | **Context Compression Ratio** médio | ≥ 0.50 | Métrica `aios_context_compression_ratio`. |
| NFR-005 | Preservação de qualidade | Δ Task Completion Rate vs. sem compressão | ≥ -2% (abs.) | Benchmark `035`. |
| NFR-006 | Eficiência de cache | cache hit ratio steady-state | ≥ 0.35 | `hits/(hits+misses)`. |
| NFR-007 | Economia de custo | custo evitado / custo bruto de inferência | ≥ 0.25 | `cost_saved_usd`. |
| NFR-008 | Disponibilidade | uptime do serviço (mensal) | ≥ 99,95% | Probes + SLO burn-rate. |
| NFR-009 | Escalabilidade | throughput de `assemble` por réplica | ≥ 500 req/s | Load test. |
| NFR-010 | Precisão de tokens | erro de estimativa vs. exato | ≤ 2% | Suite de contagem. |
| NFR-011 | Precisão do cache | taxa de falso-hit (resposta inadequada servida) | ≤ 0.5% | Amostragem/eval offline. |
| NFR-012 | Recuperabilidade | RTO / RPO | RTO ≤ 5 min / RPO ≤ 60 s | Chaos/DR drill. |
| NFR-013 | Observabilidade | cobertura de traces em caminhos críticos | 100% | Auditoria de spans OTel. |
| NFR-014 | Isolamento | vazamento de cache entre tenants | 0 | Testes de RLS + fuzz de `tenant`. |

---

## 8. Chaves de Configuração Principais

| Chave | Default | Faixa | Escopo | Recarregável |
|-------|---------|-------|--------|--------------|
| `context.budget.reserved_output_ratio` | `0.25` | `0.05–0.60` | tenant/agent | sim |
| `context.budget.min_compression_ratio` | `0.30` | `0.0–0.90` | tenant | sim |
| `context.cache.enabled` | `true` | bool | tenant/agent | sim |
| `context.cache.similarity_threshold` | `0.92` | `0.80–0.99` | tenant/task_type | sim |
| `context.cache.reuse_threshold` | `0.95` | `0.85–0.99` | tenant | sim |
| `context.cache.default_ttl_seconds` | `3600` | `60–604800` | tenant | sim |
| `context.cache.max_entries_per_tenant` | `1_000_000` | `10^3–10^8` | tenant | sim |
| `context.cache.l1_redis_ttl_seconds` | `300` | `30–3600` | global | sim |
| `context.cache.vector_index` | `hnsw` | `hnsw`\|`ivfflat` | global | não |
| `context.compression.method` | `hierarchical` | `hierarchical`\|`extractive`\|`truncate` | tenant/task_type | sim |
| `context.compression.max_levels` | `3` | `1–5` | tenant | sim |
| `context.compression.summarization_model_hint` | `cheap-small` | rótulo `017` | tenant | sim |
| `context.retrieval.top_k` | `32` | `1–256` | tenant/task_type | sim |
| `context.retrieval.memory_deadline_ms` | `120` | `20–1000` | tenant | sim |
| `context.dedup.cosine_threshold` | `0.95` | `0.80–0.999` | tenant | sim |
| `context.embedding.model_hint` | `default-embed` | rótulo `017` | tenant | sim |
| `context.embedding.dimensions` | `1536` | `256–4096` | global | não |
| `context.limits.max_inline_bytes` | `16384` | `1024–262144` | global | sim |
| `context.limits.max_payload_bytes` | `4_194_304` | `65536–33554432` | tenant | sim |
| `context.ratelimit.assemble_rps_per_tenant` | `1000` | `1–100000` | tenant | sim |
| `context.degraded.allow_truncate_fallback` | `true` | bool | tenant | sim |
| `context.shard.count` | `64` | `1–4096` | global | não |
| `context.telemetry.cache_miss_sample_rate` | `1.0` | `0.0–1.0` | tenant/task_type | sim |

---

## 9. Modos de Falha e Estratégia de Recuperação

| ID | Modo de falha | Detecção | Estratégia | Idempotência/Retry | RTO/RPO |
|----|---------------|----------|------------|--------------------|---------|
| FM-01 | `017` indisponível (limites/tokenizer) | timeout/circuit breaker | Usar limites cacheados; `BUDGET_ALLOCATED` c/ default; erro `AIOS-CTX-0002` se sem cache. | Retry idempotente c/ backoff exp. (100ms→2s, 5x) | RTO ≤ 30 s |
| FM-02 | `010-Memory` recall timeout | deadline `memory_deadline_ms` | Estado `DEGRADED`: montar com contexto parcial (system+history). | Retry parcial idempotente. | RPO 0 (efêmero) |
| FM-03 | Redis/pgvector cache down | health probe/erro | Bypass de cache (`cache_outcome=BYPASS`); segue miss path; `AIOS-CTX-0006`. | Sem retry no caminho quente. | RTO ≤ 60 s |
| FM-04 | Falha de sumarização | erro do modelo | Fallback `truncate` determinístico; `AIOS-CTX-0004`; ainda respeita orçamento. | Idempotente por bundle. | RPO 0 |
| FM-05 | Embedding indisponível | erro `017` | Fallback para `prompt_hash` (exact-match) no cache; ranking por recência. | Retry idempotente. | RTO ≤ 60 s |
| FM-06 | Orçamento infeasível | `tokens(system+instr) > budget` | `REJECTED` + `AIOS-CTX-0001`; sugere modelo maior ao `017`. | Determinístico. | — |
| FM-07 | Falha ao publicar evento | erro NATS | **Outbox**: persistir evento na transação; publisher re-tenta do outbox. | Dedup por `event.id`. | RPO ≤ 60 s |
| FM-08 | Cache poisoning / falso-hit | eval offline + `similarity` audit | Elevar `similarity_threshold`; invalidar cluster afetado; quarentena. | Invalidação idempotente. | RTO ≤ 5 min |
| FM-09 | Idempotency conflict | key repetida, payload ≠ | `AIOS-CTX-0012`; não reexecuta. | Registro de idempotência 24h. | — |
| FM-10 | Perda de nó do serviço | probe/orchestrator | Stateless: réplicas assumem; estado quente em Redis, verdade em PG. | N/A. | RTO ≤ 5 min / RPO ≤ 60 s |
| FM-11 | Payload de entrada excede `context.limits.max_payload_bytes` | validação no `ContextApiGateway` | Rejeita com `AIOS-CTX-0013` (413). | Não retriable. | — |
| FM-12 | `BudgetProfile` com pesos que não somam 1.0 | validação de escrita (invariante §3.4) | Rejeita com `AIOS-CTX-0008` (422). | Não retriable. | — |
| FM-13 | `bundle_id` inexistente/expirado em `GetBundle` | consulta a `BundleStore` | Retorna `AIOS-CTX-0007` (404). | Não retriable. | — |
| FM-14 | Estouro de rate-limit por tenant | contador do `ContextApiGateway` | Rejeita com `AIOS-CTX-0015` (429) + `Retry-After`. | Retry pelo cliente após backoff. | — |

**Princípios:** toda mutação é idempotente por `Idempotency-Key`/`event.id`;
publicação de eventos usa **outbox transacional**; caminho quente evita I/O
bloqueante e degrada em vez de falhar (`context.degraded.allow_truncate_fallback`).

---

## 10. Escalabilidade e Concorrência

| Dimensão | Estratégia |
|----------|-----------|
| **Statelessness** | `ContextService` é stateless (verdade em PostgreSQL, estado quente em Redis); escala horizontal por réplica sem afinidade. |
| **Sharding** | Cache e bundles particionados por `shard = hash(tenant_id, agent_id) mod context.shard.count`; RLS por `tenant_id`. |
| **Cache L1/L2** | L1 Redis (sub-ms, TTL curto, por shard) + L2 `pgvector` (durável, HNSW). Hit em L1 evita busca vetorial. |
| **Índice vetorial** | HNSW para busca ANN de baixa latência; IVFFLAT como alternativa em datasets menores (config). |
| **Locks** | Locks distribuídos por Redis (`SET NX PX`) por `(tenant, cache_key)` apenas na escrita de cache (evita thundering herd em cache-fill); leitura livre de lock. |
| **Concorrência** | Pipeline assíncrono; `RelevanceRanker`/`EmbeddingClient` batelam embeddings; sumarização paralela por nível (map) e sequencial na redução. |
| **Backpressure** | Rate-limit por tenant (`assemble_rps_per_tenant`); admission control por deadline; JetStream para eventos com limites de consumidor. |
| **Cold-path** | Bundles são efêmeros (`expires_at`); `SummaryNode`/cache expiram por TTL + LRU semântico; agentes suspensos têm cache evicted por evento. |
| **Rumo a milhões de agentes** | Cache dominante em L2 particionado; single-flight de cache-fill; sumários reutilizáveis reduzem chamadas de LLM; back-of-envelope: 10⁶ agentes × 35% hit ⇒ redução ~35% de chamadas de embedding/inferência de contexto. |

```
 1 → 10^3 agentes   : réplicas poucas; 1 shard; L1 Redis único.
 10^3 → 10^5        : shards por tenant; Redis cluster; pgvector com HNSW + partições por tenant.
 10^5 → 10^6+       : cache L2 particionado por (tenant, hash(agent)); single-flight global;
                      reuso agressivo de SummaryNode; sumarização roteada a modelos pequenos (017/026).
```

---

## 11. ADRs e RFCs a Propor

> **Faixa reservada de ADR deste módulo:** `ADR-0110`–`ADR-0119` (regra `011×10`).
> Nenhum outro módulo DEVE usar esta faixa.

| ADR | Título proposto |
|-----|-----------------|
| ADR-0110 | Compressão hierárquica (map-reduce summarization) vs. truncamento como padrão. |
| ADR-0111 | Cache semântico em dois níveis (Redis L1 + pgvector L2) com chave por similaridade. |
| ADR-0112 | Limiar de similaridade e política precision/recall do cache semântico. |
| ADR-0113 | Algoritmo de token budgeting e alocação por seção (system/instrução/memória/histórico/tools). |
| ADR-0114 | Registro de tokenizers multi-provedor: estimativa vs. contagem exata via `017`. |
| ADR-0115 | Modelo de sumarização (roteado a modelo barato via `017`) e verificação de fidelidade. |
| ADR-0116 | Política de invalidação e eviction (TTL + LRU semântico + event-driven). |
| ADR-0117 | Offload de fragmentos grandes em MinIO vs. armazenamento inline em PostgreSQL. |
| ADR-0118 | Degradação graciosa e truncamento seguro quando `010`/`017` indisponíveis. |
| ADR-0119 | Isolamento de cache por tenant e defesa contra cache poisoning/falso-hit. |

**RFCs relacionadas:**
- **RFC-0001** (Accepted) — baseline reutilizada integralmente (URN, eventos, erros, idempotência, correlação).
- **RFC-0011** (proposta) — *Context Assembly & Semantic Cache Contract*: formaliza o contrato de `assemble`, a semântica de chave de cache por similaridade, a definição normativa de **Context Compression Ratio** e os novos `<tipo>` de URN (`context`, `ctxfrag`, `ctxcache`) para registro em `004-API/Resources.md`.

---

## 12. Decisões de Segurança

### 12.1 AuthN/AuthZ

- **AuthN:** validado no `API Gateway` (OAuth2/OIDC), claims propagadas; interno via **mTLS** (RFC-0001 §6).
- **AuthZ (PEP/PDP):** `ContextPolicyGuard` é o PEP local; toda operação privilegiada (assemble para outro agente, upsert de budget, invalidação em massa) consulta o PDP do `022-Policy` — *default deny*.
- **Isolamento de tenant:** `tenant` do token DEVE igualar `X-AIOS-Tenant`; divergência ⇒ `AIOS-CTX-0011`. RLS por `tenant_id` em todas as tabelas; namespace de cache e subjects NATS por tenant (`aios.<tenant>.context.*`).

### 12.2 Threat Model (STRIDE resumido)

| Ameaça | Vetor no `011` | Mitigação |
|--------|----------------|-----------|
| **S**poofing | Requisição com `tenant` forjado. | mTLS + validação de claims no Gateway; check `tenant==X-AIOS-Tenant`. |
| **T**ampering | Alteração de bundle/cache em trânsito. | mTLS interno; hashes de payload; imutabilidade de eventos. |
| **R**epudiation | Negar assembly/invalidacão feita. | Eventos auditáveis (`025`) com `trace_id`, `actor`, `event.id`. |
| **I**nformation disclosure | Vazamento de contexto entre tenants via cache. | RLS + namespace por tenant + `similarity` estrito; teste NFR-014 (falso-hit ≤ 0.5%). |
| **D**enial of service | Flood de `assemble`/cache-fill. | Rate-limit por tenant, single-flight, backpressure, bulkhead/circuit breaker. |
| **E**levation of privilege | Uso de budget/invalidação sem permissão. | PEP/PDP default-deny; capabilities via `021/022`. |

### 12.3 LGPD/GDPR

- **Minimização:** payloads de eventos e logs carregam apenas identificadores/URN e métricas; conteúdo sensível vai por `content_ref` (MinIO) com acesso controlado, nunca em `detail` de erro (RFC-0001 §6/§7).
- **Base legal e retenção:** fragmentos derivados de memória herdam metadados de base legal do `010`; bundles e `SummaryNode` são efêmeros (`expires_at`) por padrão.
- **Direito ao esquecimento:** expurgo rastreável por `source_urn`/tenant (FR-012) invalida/evicta cache e emite `cache.evicted` + auditoria; propagado por `memory.item.deleted`.
- **PII no cache:** entradas de cache com PII DEVEM respeitar TTL reduzido e ser expurgáveis; embeddings de PII tratados como dado pessoal (não reversível, mas isolado por tenant).

---

*Fim do Design Brief do Módulo 011-Context. Este documento governa a geração dos 26
documentos obrigatórios listados em [README.md](./README.md).*
