---
Documento: Design Brief (Fonte Única de Verdade Interna)
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0007 (herdada), ADR-0100..ADR-0109 (a propor por este módulo)
RFCs relacionados: RFC-0001 (baseline), RFC-0100 (a propor por este módulo)
Depende de: 001-Architecture, 003-RFC (RFC-0001), 005-Database, 011-Context, 017-Model-Router, 021-Security, 022-Policy, 023-Learning, 024-Observability, 025-Audit, 026-Cost-Optimizer, 040-Glossary
---

# AIOS — Módulo 010 · Memory — Design Brief

> **Este documento é a FONTE ÚNICA DE VERDADE INTERNA do módulo 010-Memory.**
> Os 26 documentos obrigatórios (ver `../_templates/MODULE_TEMPLATE.md`) DEVEM ser
> gerados a partir deste brief e **NÃO PODEM contradizê-lo**. Componentes, estados,
> campos, códigos de erro, subjects e faixas aqui declarados são canônicos. Toda
> divergência é defeito de documentação e bloqueia merge (`033-DeveloperGuide/DocumentationCI.md`).
>
> Reutiliza sem redefinir: URN `urn:aios:<tenant>:<tipo>:<id>` (RFC-0001 §5.1),
> envelope de evento CloudEvents (§5.2), subjects `aios.<tenant>.<dominio>.<entidade>.<acao>`
> (§5.3), envelope de erro RFC 7807 `AIOS-<DOMINIO>-<NNNN>` (§5.4), idempotência via
> `Idempotency-Key` (§5.5), correlação `traceparent`/`X-AIOS-Tenant` (§5.6).
> Palavras normativas conforme RFC 2119/8174 (**DEVE**, **NÃO DEVE**, **DEVERIA**, **PODE**).

---

## 0. Reservas deste módulo (para os próximos agentes)

| Recurso | Faixa reservada por 010-Memory |
|---------|--------------------------------|
| Domínio de subject NATS | `memory` (`aios.<tenant>.memory.<entidade>.<acao>`) |
| Domínio de código de erro | `AIOS-MEM-NNNN` (reservado `AIOS-MEM-0001`..`AIOS-MEM-0099`) |
| Faixa de ADR (010 × 10) | `ADR-0100` .. `ADR-0109` |
| RFC de módulo | `RFC-0100` (Memory Consolidation & Forgetting Contract) |
| Tipo de URN | `memory` (`urn:aios:<tenant>:memory:<ULID>`) já registrado em RFC-0001 §8 |
| Pacote gRPC | `aios.memory.v1` |
| Prefixo de métrica | `aios_memory_*` |
| Namespace de config | `memory.*` |

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Responsabilidades (o que o módulo DEVE fazer)

O `MemoryService` (010) é o **gerenciador de memória cognitiva hierárquica** do AIOS —
análogo à MMU + swap + filesystem de um SO clássico, porém para recursos cognitivos.
Ele DEVE:

1. **R1 — API uniforme de memória.** Expor `remember(item, layer)`, `recall(query, filtros) → ranked`,
   `consolidate()` e `forget(policy)` sobre uma superfície REST/gRPC estável e versionada,
   ocultando a heterogeneidade das 7 camadas físicas.
2. **R2 — Hierarquia de 7 camadas.** Manter e rotear itens entre Working, Short-Term,
   Long-Term, Semantic, Procedural, Episodic e Knowledge Graph, cada uma com backend
   físico e política próprios (ver §3, §4).
3. **R3 — Recuperação rankeada e seletiva.** Executar busca híbrida (lexical + vetorial
   ANN por `pgvector`/HNSW + travessia de grafo por Apache AGE), fundir e re-rankear
   resultados por relevância, recência, saliência e escopo (tenant/agente), colaborando
   com `011-Context` na recuperação seletiva.
4. **R4 — Consolidação dirigida.** Promover itens entre camadas (ex.: Short-Term → Long-Term →
   Semantic/Episodic → Knowledge Graph) sob direção do `023-Learning`, de forma **versionada**
   e **reversível** (proteção contra *catastrophic forgetting*).
5. **R5 — Esquecimento controlado.** Aplicar políticas de retenção/decaimento/expurgo por
   camada, respeitando cotas por tenant/agente, para conter crescimento descontrolado.
6. **R6 — Cotas e contabilização.** Medir uso (itens, bytes, vetores) por camada/tenant/agente
   e negar/represar escrita quando cota excedida, integrando com `026-Cost-Optimizer`.
7. **R7 — Embeddings.** Requisitar geração de embeddings ao `017-Model-Router` (nunca chamar
   LLM diretamente) e persisti-los em `pgvector` (índice HNSW).
8. **R8 — Blobs grandes.** Externalizar conteúdo acima do limite inline para `MinIO`,
   mantendo apenas referência + metadados + embedding no PostgreSQL.
9. **R9 — Eventos de domínio.** Emitir eventos `aios.<tenant>.memory.item.{stored,consolidated,forgotten,...}`
   via padrão Outbox (atomicidade DB + evento).
10. **R10 — Isolamento multi-tenant e privacidade.** Impor `tenant_id` em toda operação,
    Row-Level Security, base legal/retenção por item e o **direito ao esquecimento** (LGPD/GDPR)
    por expurgo rastreável.
11. **R11 — Observabilidade.** Emitir a métrica-chave **Memory Recall Rate** e telemetria OTel
    correlacionada em todo caminho.

### 1.2 Não-Responsabilidades (o que o módulo NÃO DEVE fazer)

| # | NÃO é responsabilidade | Dono correto |
|---|------------------------|--------------|
| N1 | Treinar/fine-tunar modelos ou gerar embeddings por conta própria (apenas requisita). | `017-Model-Router` |
| N2 | Decidir **quando/como** aprender uma política; apenas executa a consolidação solicitada. | `023-Learning` |
| N3 | Montar a janela de contexto final / compressão de prompt / cache semântico de inferência. | `011-Context` |
| N4 | Persistir o grafo de conhecimento *de domínio público* e GraphRAG de recuperação global. | `018/019-Knowledge` |
| N5 | Autorizar acesso (PDP) — apenas atua como PEP consultando o Policy Engine. | `022-Policy` |
| N6 | Trilha de auditoria imutável — apenas emite eventos que o Audit consome. | `025-Audit` |
| N7 | Ciclo de vida do agente (spawn/suspend/kill) — apenas reage a esses eventos. | `006/008` |
| N8 | Orquestração de workflow/saga de negócio. | `014-Workflow` |
| N9 | Cobrança/orçamento — apenas reporta consumo. | `026-Cost-Optimizer` |

> **Fronteira crítica:** o Working/Short-Term Memory de um agente pertence ao **plano de
> controle**; o Agent Runtime (Python, plano de dados) NÃO DEVE acessar PostgreSQL/Redis
> diretamente — DEVE passar pela API do `MemoryService` via gRPC/NATS (regra de camadas do
> `../001-Architecture/Architecture.md` §6).

---

## 2. Decomposição em Componentes Internos

### 2.1 Diagrama de componentes (ASCII, C4 nível 3)

```
┌──────────────────────────────── MEMORY SERVICE (010) · .NET 10 ────────────────────────────────┐
│                                                                                                  │
│   REST/gRPC (YARP)                                                                               │
│        │                                                                                          │
│   ┌────▼─────────────┐   consulta PDP   ┌──────────────────┐                                     │
│   │ MemoryApiFacade  │─────────────────▶│ MemoryPep        │──▶ Policy Engine (022)              │
│   │ remember/recall/ │                  │ (PEP + tenant    │                                     │
│   │ consolidate/forget│                 │  guard + idemp.) │──▶ IdempotencyStore (Redis)         │
│   └────┬─────────────┘                  └──────────────────┘                                     │
│        │ comando validado                                                                        │
│   ┌────▼───────────┐        ┌────────────────────┐        ┌───────────────────────┐             │
│   │ LayerRouter    │───────▶│ QuotaManager       │───────▶│ EmbeddingClient       │──▶ Model    │
│   │ (classifica +  │        │ (cotas/tenant/     │        │ (gera embedding via   │   Router    │
│   │  roteia p/camada)│      │  camada, backpress)│        │  017; cache dim)      │   (017)     │
│   └────┬───────────┘        └────────────────────┘        └───────────────────────┘             │
│        │                                                                                          │
│   ┌────▼──────────────────────── LAYER STORES (Strategy por camada) ──────────────────────────┐ │
│   │ WorkingMemoryStore   ShortTermMemoryStore   LongTermMemoryStore   SemanticMemoryStore      │ │
│   │   (Redis, TTL)         (Redis+PG, sessão)     (PG, durável)         (PG+pgvector/HNSW)      │ │
│   │ ProceduralMemoryStore  EpisodicMemoryStore   KnowledgeGraphAdapter                          │ │
│   │   (PG, skills)          (PG+pgvector, temporal)  (Apache AGE / openCypher)                  │ │
│   └────┬───────────────────────┬───────────────────────┬───────────────────┬───────────────────┘ │
│        │                       │                       │                   │                      │
│   ┌────▼─────────┐   ┌─────────▼────────┐   ┌──────────▼─────────┐  ┌──────▼──────────┐          │
│   │ RecallEngine │   │ ConsolidationEngine│ │ ForgettingEngine   │  │ BlobStoreAdapter│          │
│   │ (hybrid +    │   │ (promoção versio-  │ │ (decay/expiry/     │  │ (MinIO; conteúdo│          │
│   │  rerank +    │   │  nada, dirigida    │ │  purge, cotas,     │  │  grande + hash) │          │
│   │  RRF fusion) │   │  por 023)          │ │  RTBF)             │  └─────────────────┘          │
│   └──────────────┘   └─────────┬──────────┘ └────────┬───────────┘                               │
│                                │ snapshot/rollback    │ tombstone                                 │
│                      ┌─────────▼──────────┐  ┌────────▼──────────┐   ┌────────────────────────┐  │
│                      │ ConsolidationVersion│ │ RetentionScheduler│   │ OutboxPublisher        │  │
│                      │ Manager (rollback)  │ │ (jobs/cron)       │   │ (DB→NATS at-least-once)│  │
│                      └─────────────────────┘ └───────────────────┘   └───────────┬────────────┘  │
│                                                                                   │               │
│   ┌───────────────────────────────────────────────────────────────────────┐      │               │
│   │ MemoryTelemetry (OTel: métricas/traces/logs · Memory Recall Rate)      │      │               │
│   └───────────────────────────────────────────────────────────────────────┘      ▼               │
└──────────────────────────────────────────────────────────────────────────── NATS (JetStream) ───┘
        │ Redis            │ PostgreSQL (+pgvector +AGE)          │ MinIO
        ▼                  ▼                                       ▼
   estado quente      fonte da verdade (RLS por tenant)      blobs/checkpoints
```

### 2.2 Catálogo de componentes (PascalCase · responsabilidade · dependências)

| Componente | Responsabilidade | Depende de |
|------------|------------------|------------|
| **MemoryApiFacade** | Ponto único das 4 operações canônicas (`remember/recall/consolidate/forget`) + `getItem/getStats`; valida DTOs, aplica versionamento de API, traduz para comandos internos. | MemoryPep, LayerRouter, RecallEngine, ConsolidationEngine, ForgettingEngine |
| **MemoryPep** | Policy Enforcement Point: autoriza cada operação junto ao PDP (022), impõe `tenant` do contexto autenticado, valida `Idempotency-Key`, aplica *default deny*. | Policy Engine (022), IdempotencyStore, Security (021) |
| **IdempotencyStore** | Persiste resultado por `Idempotency-Key` por ≥ 24h (RFC-0001 §5.5); deduplica mutações. | Redis |
| **LayerRouter** | Classifica o item e roteia para a(s) camada(s) corretas; decide inline vs. blob; aciona QuotaManager e EmbeddingClient conforme a camada. | QuotaManager, EmbeddingClient, LayerStores, BlobStoreAdapter |
| **QuotaManager** | Contabiliza itens/bytes/vetores por (tenant, agente, camada); aplica cotas e *backpressure*; reporta consumo ao 026. | Redis (contadores quentes), PostgreSQL, Cost-Optimizer (026) |
| **EmbeddingClient** | Solicita embeddings ao Model Router (017) com *batching*, retry idempotente, cache de dimensão; nunca chama LLM diretamente. | Model Router (017) |
| **WorkingMemoryStore** | Memória de trabalho volátil do raciocínio corrente; Redis com TTL curto; escopo por agente/sessão. | Redis |
| **ShortTermMemoryStore** | Memória recente de sessão; Redis (hot) com projeção durável em PostgreSQL (CQRS). | Redis, PostgreSQL |
| **LongTermMemoryStore** | Memória persistente consolidada; PostgreSQL durável. | PostgreSQL |
| **SemanticMemoryStore** | Fatos/conceitos gerais; PostgreSQL + `pgvector` (índice HNSW) para busca ANN. | PostgreSQL+pgvector |
| **ProceduralMemoryStore** | Habilidades/rotinas "como fazer"; PostgreSQL (estrutura + versão de skill). | PostgreSQL |
| **EpisodicMemoryStore** | Eventos/experiências com contexto temporal; PostgreSQL + `pgvector`; particionado por tempo. | PostgreSQL+pgvector |
| **KnowledgeGraphAdapter** | Nós/arestas de memória consolidada do agente em Apache AGE (openCypher); travessia para recall. | PostgreSQL+Apache AGE |
| **BlobStoreAdapter** | Externaliza conteúdo > limite inline em MinIO; content-addressed (hash), deduplicação, TTL alinhado à retenção. | MinIO |
| **RecallEngine** | Recuperação híbrida (lexical + ANN + grafo), fusão RRF, re-rank por relevância/recência/saliência/escopo; produz `RecallResult` rankeado; calcula Memory Recall Rate. | LayerStores, KnowledgeGraphAdapter, EmbeddingClient, Context (011) |
| **ConsolidationEngine** | Promove/mescla/deduplica itens entre camadas sob direção do 023; grava snapshot versionado antes de mutar. | ConsolidationVersionManager, LayerStores, Learning (023), OutboxPublisher |
| **ConsolidationVersionManager** | Versiona cada consolidação; permite `rollback` a uma versão anterior (mitiga *catastrophic forgetting*). | PostgreSQL, MinIO (snapshots frios) |
| **ForgettingEngine** | Aplica decaimento (score), expiração (TTL), expurgo (RTBF) e poda por cota; cria tombstones. | LayerStores, QuotaManager, RetentionScheduler, OutboxPublisher |
| **RetentionScheduler** | Agenda jobs de decaimento/consolidação/expurgo (cron interno + reação a eventos); garante idempotência de jobs. | ForgettingEngine, ConsolidationEngine, NATS |
| **OutboxPublisher** | Publica eventos de domínio via padrão Outbox (linha DB + publicação NATS at-least-once). | PostgreSQL (outbox), NATS/JetStream |
| **MemoryTelemetry** | Instrumenta OTel (traces/metrics/logs Serilog→Seq); expõe `aios_memory_*` e Memory Recall Rate. | OpenTelemetry, Observability (024) |

---

## 3. Modelo de Dados / Entidades Canônico

Todos os identificadores externos DEVEM ser URN `urn:aios:<tenant>:<tipo>:<ULID>` (RFC-0001 §5.1).
Toda tabela DEVE ter `tenant_id` com Row-Level Security. `id` interno é ULID (128 bits, ordenável).
`content_ref` referencia blob MinIO quando `content` excede `memory.item.inline_max_bytes`.

### 3.1 `MemoryItem` (entidade central)

| Campo | Tipo | Chave/Constraint | Descrição |
|-------|------|------------------|-----------|
| `id` | `ULID` (char26) | PK | Identidade interna; URN = `urn:aios:<tenant>:memory:<id>`. |
| `tenant_id` | `text` (slug) | NOT NULL, RLS, index | Fronteira de isolamento. |
| `agent_id` | `ULID` | NULL, index | Agente dono (NULL = memória de tenant/compartilhada). |
| `session_id` | `ULID` | NULL, index | Sessão (Working/Short-Term). |
| `layer` | `enum memory_layer` | NOT NULL, index | `working\|short_term\|long_term\|semantic\|procedural\|episodic\|kg`. |
| `kind` | `enum memory_kind` | NOT NULL | `fact\|event\|skill\|observation\|reflection\|summary\|edge`. |
| `state` | `enum memory_state` | NOT NULL, index | Ver §4.1 (`ingested\|active\|...\|purged`). |
| `content` | `jsonb` | NULL | Conteúdo inline (quando ≤ limite). |
| `content_ref` | `text` (s3 uri) | NULL | Ponteiro MinIO (quando externalizado). |
| `content_hash` | `bytea` (sha256) | index | Deduplicação content-addressed. |
| `embedding` | `vector(D)` | HNSW (semantic/episodic) | Embedding pgvector; `D` = `memory.embedding.dim`. |
| `embedding_model` | `text` | NULL | Modelo/URN usado (rastreabilidade). |
| `salience` | `real` [0..1] | NOT NULL default 0.5 | Importância/saliência (dirige esquecimento e rank). |
| `decay_score` | `real` [0..1] | NOT NULL | Score de retenção decaído no tempo. |
| `access_count` | `bigint` | NOT NULL default 0 | Reforço por uso. |
| `last_access_at` | `timestamptz` | index | Recência de acesso. |
| `source_urn` | `text` (URN) | NULL | Origem (task/agent/tool) que gerou o item. |
| `consolidation_version` | `bigint` | NULL, FK→ConsolidationVersion | Versão de consolidação vigente. |
| `parent_ids` | `ULID[]` | NULL | Itens de origem consolidados (linhagem). |
| `legal_basis` | `enum legal_basis` | NOT NULL | Base legal LGPD/GDPR (`consent\|contract\|legitimate_interest\|legal_obligation`). |
| `retention_class` | `enum retention_class` | NOT NULL | `ephemeral\|standard\|extended\|legal_hold`. |
| `pii` | `boolean` | NOT NULL default false | Marca PII (redação/tokenização). |
| `tags` | `text[]` | GIN index | Rótulos de recuperação. |
| `metadata` | `jsonb` | — | Metadados livres versionados. |
| `expires_at` | `timestamptz` | NULL, index | TTL efetivo (esquecimento). |
| `created_at` | `timestamptz` | NOT NULL | Criação (UTC, RFC 3339). |
| `updated_at` | `timestamptz` | NOT NULL | Última mutação. |
| `tombstoned_at` | `timestamptz` | NULL | Marca lógica de esquecimento. |

### 3.2 Demais entidades

| Entidade | Campos-chave | Descrição |
|----------|--------------|-----------|
| **`MemoryLayerConfig`** | `tenant_id`, `layer` (PK), `ttl_default`, `max_items`, `max_bytes`, `decay_half_life`, `consolidation_threshold`, `hnsw_m`, `hnsw_ef` | Política física por camada/tenant (recarregável). |
| **`ConsolidationJob`** | `id` (ULID PK), `tenant_id`, `agent_id`, `trigger` (`learning\|scheduled\|manual\|threshold`), `from_layer`, `to_layer`, `state`, `version_id`, `started_at`, `finished_at`, `stats jsonb` | Execução de consolidação (ver §4.2). |
| **`ConsolidationVersion`** | `version_id` (bigint PK), `tenant_id`, `agent_id`, `snapshot_ref` (MinIO), `created_at`, `parent_version`, `active bool` | Snapshot versionado p/ rollback (anti *catastrophic forgetting*). |
| **`ForgettingPolicy`** | `id` (ULID PK), `tenant_id`, `scope` (`tenant\|agent\|layer`), `strategy` (`ttl\|decay\|lru\|quota\|rtbf`), `params jsonb`, `enabled bool` | Política declarativa de esquecimento. |
| **`MemoryQuota`** | `tenant_id`, `agent_id`, `layer` (PK composta), `limit_items`, `limit_bytes`, `used_items`, `used_bytes`, `updated_at` | Cotas e uso corrente. |
| **`KnowledgeEdge`** (AGE) | `src_id`, `dst_id`, `rel_type`, `tenant_id`, `weight`, `properties` | Aresta de grafo de memória do agente (openCypher). |
| **`OutboxEvent`** | `id` (ULID PK), `tenant_id`, `subject`, `payload jsonb`, `status` (`pending\|published`), `created_at` | Outbox transacional (DB→NATS). |
| **`IdempotencyRecord`** | `key` (PK), `tenant_id`, `operation`, `response_hash`, `result jsonb`, `expires_at` | Deduplicação de mutações (≥ 24h). |
| **`RecallQuery`/`RecallResult`** | transitório (não persistido) | Consulta e resultado rankeado da operação `recall`. |

### 3.3 Camadas físicas (mapa canônico)

| Camada | Backend | Persistência | TTL default | Escopo típico |
|--------|---------|--------------|-------------|---------------|
| Working | Redis | Efêmera (RAM) | 15 min | sessão/agente |
| Short-Term | Redis (hot) + PostgreSQL (proj.) | Volátil→durável | 24 h | sessão |
| Long-Term | PostgreSQL | Durável | 90 dias (config.) | agente/tenant |
| Semantic | PostgreSQL + pgvector (HNSW) | Durável | sem TTL (decay) | agente/tenant |
| Procedural | PostgreSQL | Durável | sem TTL | agente/tenant |
| Episodic | PostgreSQL + pgvector (part. tempo) | Durável | 365 dias (config.) | agente |
| Knowledge Graph | Apache AGE (openCypher) | Durável | sem TTL | agente/tenant |
| Blobs grandes | MinIO | Durável (S3) | alinhado à retenção | qualquer camada |

---

## 4. Máquinas de Estado Canônicas

### 4.1 Máquina de estado do `MemoryItem`

```
                remember()                     consolidate() (023)
    ┌──────────┐  ok    ┌────────┐  reforço/uso  ┌────────────────┐   promovido   ┌──────────────┐
──▶ │ INGESTED │───────▶│ ACTIVE │◀─────────────▶│ CONSOLIDATING  │──────────────▶│ CONSOLIDATED │
    └──────────┘        └───┬────┘               └───────┬────────┘               └──────┬───────┘
       │ falha embedding    │ decay/TTL                  │ falha/conflito                │ decay
       ▼                    ▼                            ▼ rollback                      ▼
    ┌────────┐          ┌──────────┐                 (volta ACTIVE)               ┌──────────────┐
    │ FAILED │          │ DECAYING │◀──────────────────────────────────────────── │  ARCHIVED    │
    └────────┘          └────┬─────┘   move p/ frio (MinIO)                        │ (cold/MinIO) │
                             │ score<θ  ou  expires_at  ou  RTBF  ou  cota         └──────┬───────┘
                             ▼                                                             │ reidrata
                       ┌───────────────┐   forget() confirma   ┌──────────┐               │
                       │ FORGET_PENDING│──────────────────────▶│FORGOTTEN │◀──────────────┘
                       └───────────────┘                       │(tombstone)│
                                                               └────┬──────┘
                                                    grace period ⌛ │ purge físico + RTBF
                                                                    ▼
                                                              ┌──────────┐
                                                              │  PURGED  │ (terminal)
                                                              └──────────┘
```

**Estados:** `INGESTED`, `ACTIVE`, `CONSOLIDATING`, `CONSOLIDATED`, `ARCHIVED`,
`DECAYING`, `FORGET_PENDING`, `FORGOTTEN`, `PURGED` (terminal), `FAILED` (terminal recuperável por retry).

| Transição | Gatilho | Guarda |
|-----------|---------|--------|
| `INGESTED→ACTIVE` | escrita persistida + embedding pronto (se aplicável) | cota disponível; PDP allow; embedding ok |
| `INGESTED→FAILED` | erro de persistência/embedding | após N retries idempotentes |
| `ACTIVE→CONSOLIDATING` | `consolidate()` (023) ou `consolidation_threshold` atingido | snapshot de versão gravado antes |
| `CONSOLIDATING→CONSOLIDATED` | promoção/merge concluídos | integridade validada |
| `CONSOLIDATING→ACTIVE` | falha/conflito de consolidação | rollback à versão anterior aplicado |
| `CONSOLIDATED→DECAYING` | tempo/uso reduzem `decay_score` | `decay_score < decay_threshold` |
| `ACTIVE→DECAYING` | job de decaimento | `decay_score < θ` ou `expires_at` vencido |
| `CONSOLIDATED→ARCHIVED` | frieza (baixo acesso) | move conteúdo p/ MinIO; mantém metadados |
| `ARCHIVED→ACTIVE` | recall exige reidratação | reidrata de MinIO/PG |
| `DECAYING→FORGET_PENDING` | política de esquecimento elegível | `strategy` casa (ttl/decay/lru/quota/rtbf) |
| `*→FORGET_PENDING` | `forget(policy)` ou RTBF (LGPD/GDPR) | não em `legal_hold` (exceto RTBF autorizado) |
| `FORGET_PENDING→FORGOTTEN` | confirmação | tombstone criado; evento `forgotten` emitido |
| `FORGOTTEN→PURGED` | fim do *grace period* | purge físico + remoção de blob/vetor |
| `FAILED→INGESTED` | retry | backoff exponencial |

**Invariantes:** (i) item em `legal_hold` NÃO DEVE ir a `PURGED` salvo RTBF autorizado;
(ii) toda transição para `CONSOLIDATING` DEVE ter `ConsolidationVersion` gravada antes;
(iii) `FORGOTTEN` mantém tombstone + linhagem para auditoria até `PURGED`.

### 4.2 Máquina de estado do `ConsolidationJob`

```
  PENDING ──admissão──▶ SNAPSHOTTING ──ok──▶ RUNNING ──ok──▶ VALIDATING ──ok──▶ COMMITTED (terminal)
     │                       │ falha            │ falha           │ inconsistência
     │ rejeitado             ▼                  ▼                 ▼
     ▼                    FAILED ◀───────── ROLLING_BACK ◀───────┘
  REJECTED (terminal)                          │ ok
                                               ▼
                                          ROLLED_BACK (terminal)
```

| Estado | Descrição / guarda |
|--------|--------------------|
| `PENDING` | Enfileirado; aguarda janela/admissão do RetentionScheduler. |
| `REJECTED` | Cota/política negam; terminal. |
| `SNAPSHOTTING` | Grava `ConsolidationVersion` (pré-imagem) — obrigatório antes de mutar. |
| `RUNNING` | Promove/mescla/deduplica itens entre `from_layer`→`to_layer`. |
| `VALIDATING` | Verifica integridade/linhagem/recall pós-consolidação. |
| `COMMITTED` | Versão marcada ativa; eventos `consolidated` emitidos; terminal. |
| `ROLLING_BACK` | Restaura versão anterior (falha ou regressão de Recall Rate). |
| `ROLLED_BACK` | Estado revertido; evento `consolidation.rolledback`; terminal. |
| `FAILED` | Erro irrecuperável após rollback seguro; terminal para inspeção. |

---

## 5. Superfície de API

Base REST: `/v1/memory` (versão por caminho + header `X-AIOS-Api-Version`, RFC-0001 §5.7).
gRPC: pacote `aios.memory.v1`, serviço `MemoryService`. Toda mutação DEVE exigir
`Idempotency-Key`, `X-AIOS-Tenant` e `traceparent`. Erros em RFC 7807 (RFC-0001 §5.4).

### 5.1 Recursos e endpoints REST / métodos gRPC

| Operação canônica | REST | gRPC | Semântica |
|-------------------|------|------|-----------|
| `remember(item, layer)` | `POST /v1/memory/items` | `Remember(RememberRequest)` | Cria/atualiza item; retorna URN + estado. Idempotente por `Idempotency-Key`. |
| `recall(query, filtros)` | `POST /v1/memory/recall` | `Recall(RecallRequest)` | Busca híbrida rankeada; retorna lista `RecallResult` + scores. |
| `consolidate()` | `POST /v1/memory/consolidate` | `Consolidate(ConsolidateRequest)` | Dispara `ConsolidationJob` (síncrono→job assíncrono; retorna `job_id`). |
| `forget(policy)` | `POST /v1/memory/forget` | `Forget(ForgetRequest)` | Aplica política de esquecimento/expurgo; retorna contagem afetada. |
| get item | `GET /v1/memory/items/{id}` | `GetItem(GetItemRequest)` | Recupera item por URN. |
| delete item (RTBF) | `DELETE /v1/memory/items/{id}` | `PurgeItem(PurgeRequest)` | Expurgo direcionado (direito ao esquecimento). |
| stats/cotas | `GET /v1/memory/stats` | `GetStats(StatsRequest)` | Uso por camada/tenant/agente + Memory Recall Rate. |
| listar camadas/config | `GET /v1/memory/layers` | `ListLayers(ListLayersRequest)` | Config vigente por camada. |
| status de job | `GET /v1/memory/jobs/{id}` | `GetJob(GetJobRequest)` | Estado do `ConsolidationJob`. |
| rollback de consolidação | `POST /v1/memory/consolidate/{jobId}/rollback` | `RollbackConsolidation(...)` | Reverte à versão anterior (anti *catastrophic forgetting*). |

Filtros de `recall` (DEVE): `agent_id`, `layer[]`, `kind[]`, `tags[]`, `k` (top-k), `min_score`,
`time_range`, `include_forgotten=false`, `mode` (`vector\|lexical\|graph\|hybrid`). Paginação por cursor.

### 5.2 Catálogo de códigos de erro (`AIOS-MEM-NNNN`)

| Código | HTTP | Retriable | Significado |
|--------|------|-----------|-------------|
| `AIOS-MEM-0001` | 400 | não | Requisição inválida (schema/DTO). |
| `AIOS-MEM-0002` | 404 | não | Item de memória inexistente. |
| `AIOS-MEM-0003` | 409 | não | Conflito de idempotência (mesma key, payload divergente). |
| `AIOS-MEM-0004` | 403 | não | Autorização negada pelo PDP (default deny). |
| `AIOS-MEM-0005` | 403 | não | Tenant divergente do contexto autenticado. |
| `AIOS-MEM-0010` | 429 | sim | Cota de camada excedida (itens/bytes/vetores). |
| `AIOS-MEM-0011` | 429 | sim | Backpressure ativo (fila de escrita saturada). |
| `AIOS-MEM-0020` | 422 | não | Camada inválida ou incompatível com o `kind`. |
| `AIOS-MEM-0021` | 422 | não | Dimensão de embedding incompatível com o índice. |
| `AIOS-MEM-0030` | 502 | sim | Falha ao obter embedding do Model Router (017). |
| `AIOS-MEM-0031` | 503 | sim | Backend de camada indisponível (Redis/PG/AGE/MinIO). |
| `AIOS-MEM-0040` | 409 | não | Consolidação em conflito (versão concorrente). |
| `AIOS-MEM-0041` | 500 | sim | Falha de consolidação (rollback aplicado). |
| `AIOS-MEM-0042` | 409 | não | Rollback impossível (versão inexistente/já ativa). |
| `AIOS-MEM-0050` | 423 | não | Item sob `legal_hold`; esquecimento bloqueado. |
| `AIOS-MEM-0051` | 202 | — | Expurgo (RTBF) aceito e agendado. |
| `AIOS-MEM-0060` | 500 | sim | Erro interno não classificado. |

> Reservado a este módulo: `AIOS-MEM-0001`..`AIOS-MEM-0099`. Registro central em `../004-API/Errors.md`.

---

## 6. Catálogo de Eventos NATS

Domínio: `memory`. Envelope CloudEvents (RFC-0001 §5.2). Subject: `aios.<tenant>.memory.<entidade>.<acao>`.
Entrega at-least-once; consumidores DEVEM deduplicar por `event.id`. Publicação via Outbox.

### 6.1 Eventos emitidos (produtor = 010-Memory)

| Subject | `type` | Quando | Consumidores principais |
|---------|--------|--------|-------------------------|
| `aios.<t>.memory.item.stored` | `aios.memory.item.stored` | Item passa a `ACTIVE`. | 023, 024, 025 |
| `aios.<t>.memory.item.consolidated` | `aios.memory.item.consolidated` | Item promovido/mesclado (`COMMITTED`). | 023, 019, 025 |
| `aios.<t>.memory.item.forgotten` | `aios.memory.item.forgotten` | Item → `FORGOTTEN` (tombstone). | 025, 026, 024 |
| `aios.<t>.memory.item.recalled` | `aios.memory.item.recalled` | Recuperação executada (amostrada). | 011, 024 |
| `aios.<t>.memory.item.decayed` | `aios.memory.item.decayed` | Item → `DECAYING`. | 024, 026 |
| `aios.<t>.memory.item.archived` | `aios.memory.item.archived` | Item → `ARCHIVED` (cold/MinIO). | 026, 024 |
| `aios.<t>.memory.item.purged` | `aios.memory.item.purged` | Purge físico concluído (RTBF). | 025 (auditoria RTBF) |
| `aios.<t>.memory.consolidation.started` | `aios.memory.consolidation.started` | `ConsolidationJob`→`RUNNING`. | 023, 024 |
| `aios.<t>.memory.consolidation.completed` | `aios.memory.consolidation.completed` | Job `COMMITTED`. | 023, 024 |
| `aios.<t>.memory.consolidation.failed` | `aios.memory.consolidation.failed` | Job `FAILED`. | 023, 024, 025 |
| `aios.<t>.memory.consolidation.rolledback` | `aios.memory.consolidation.rolledback` | Job `ROLLED_BACK`. | 023, 025 |
| `aios.<t>.memory.quota.exceeded` | `aios.memory.quota.exceeded` | Cota de camada estourada. | 026, 024 |

### 6.2 Eventos consumidos (010-Memory como consumidor)

| Subject (assinatura) | Produtor | Ação do MemoryService |
|----------------------|----------|-----------------------|
| `aios.<t>.learning.consolidation.requested` | 023-Learning | Dispara `ConsolidationJob` dirigido. |
| `aios.<t>.learning.policy.rolledback` | 023-Learning | Reverte consolidação associada (versão). |
| `aios.<t>.context.recall.requested` | 011-Context | Executa recall seletivo e responde. |
| `aios.<t>.agent.lifecycle.terminated` | 006/008 | Expira Working/Short-Term do agente. |
| `aios.<t>.agent.lifecycle.suspended` | 006/008 | Move estado quente → cold (MinIO/PG). |
| `aios.<t>.policy.decision.updated` | 022-Policy | Recarrega decisões de acesso a memória. |
| `aios.<t>.security.rtbf.requested` | 021/025 | Agenda expurgo (direito ao esquecimento). |

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (FR)

| ID | Requisito (MoSCoW) | Critério de aceite |
|----|--------------------|--------------------|
| FR-001 (Must) | `remember(item, layer)` persiste item na camada e retorna URN + estado. | Item consultável por `GET`; evento `stored` emitido; idempotente. |
| FR-002 (Must) | `recall(query, filtros)` retorna lista rankeada com scores por relevância. | Top-k ordenado; filtros aplicados; `mode` honrado. |
| FR-003 (Must) | Busca vetorial ANN via pgvector/HNSW nas camadas Semantic/Episodic. | Recall@10 ≥ 0,95 vs. exata em bench (NFR-002). |
| FR-004 (Must) | Roteamento automático para a camada correta por `kind`/hint. | Item aterrissa na camada prevista; erro `AIOS-MEM-0020` se incompatível. |
| FR-005 (Must) | `consolidate()` promove itens entre camadas de forma versionada. | `ConsolidationVersion` gravada antes; eventos de job emitidos. |
| FR-006 (Must) | `forget(policy)` aplica ttl/decay/lru/quota/rtbf. | Itens elegíveis → `FORGOTTEN`; contagem retornada; `legal_hold` respeitado. |
| FR-007 (Must) | Rollback de consolidação restaura versão anterior. | Estado idêntico à pré-imagem; evento `rolledback`. |
| FR-008 (Must) | Cotas por (tenant, agente, camada) com backpressure. | Escrita negada com `AIOS-MEM-0010/0011` ao exceder. |
| FR-009 (Must) | Externalização de blobs grandes para MinIO. | Conteúdo > limite vira `content_ref`; hash content-addressed. |
| FR-010 (Must) | Embeddings obtidos exclusivamente via Model Router (017). | Nenhuma chamada direta a LLM; `embedding_model` registrado. |
| FR-011 (Must) | Direito ao esquecimento (RTBF) por item/agente/tenant. | Purge rastreável; evento `purged`; auditoria (025). |
| FR-012 (Should) | Reidratação de itens `ARCHIVED` sob recall. | Recall retorna item frio reidratado transparentemente. |
| FR-013 (Should) | Grafo de memória do agente (Apache AGE) para recall por travessia. | `mode=graph` retorna itens conectados. |
| FR-014 (Could) | Deduplicação semântica na consolidação (merge por similaridade). | Duplicatas mescladas mantendo linhagem `parent_ids`. |

### 7.2 Requisitos Não-Funcionais (NFR) com metas (SLO/SLI)

| ID | Atributo | Meta (SLO) | SLI / verificação |
|----|----------|-----------|-------------------|
| NFR-001 | Latência `remember` | p99 ≤ 30 ms (inline, sem embedding); ≤ 120 ms (com embedding). | histograma `aios_memory_remember_duration_ms`. |
| NFR-002 | Qualidade de recuperação | **Memory Recall Rate ≥ 0,95**; Recall@10 ANN ≥ 0,95 vs. exata. | `aios_memory_recall_rate` (bench + amostragem produção). |
| NFR-003 | Latência `recall` | p99 ≤ 50 ms (Working/hot Redis); ≤ 150 ms (Semantic/Episodic ANN); ≤ 250 ms (grafo). | `aios_memory_recall_duration_ms` por `mode`. |
| NFR-004 | Throughput | ≥ 5.000 `remember`/s e ≥ 2.000 `recall`/s por shard. | teste de carga (`Benchmark.md`). |
| NFR-005 | Disponibilidade | ≥ 99,95% do serviço (control plane). | uptime SLI; erro 5xx budget. |
| NFR-006 | Durabilidade (camadas duráveis) | RPO ≤ 5 min (Long-Term/Semantic/Procedural/Episodic/KG). | replicação PG streaming; teste de DR. |
| NFR-007 | RTO | ≤ 15 min para o serviço de memória. | drill de failover. |
| NFR-008 | Working Memory | perda tolerada (efêmera); RPO não garantido; reconstruível. | política declarada; teste de kill. |
| NFR-009 | Crescimento controlado | uso por tenant NÃO DEVE exceder cota; poda mantém ≤ 100% da cota. | `aios_memory_usage_ratio` < 1,0. |
| NFR-010 | Isolamento | zero vazamento cross-tenant. | teste de RLS + fuzz de tenant. |
| NFR-011 | Consolidação sem regressão | Recall Rate pós-consolidação ≥ pré − 0,01, senão rollback automático. | comparação A/B por versão. |
| NFR-012 | Escala vetorial | ≥ 10^8 vetores por tenant com p99 ANN ≤ 150 ms. | bench pgvector/HNSW + sharding. |
| NFR-013 | Observabilidade | 100% das operações com trace OTel + `tenant_id`/`trace_id`. | auditoria de spans. |

---

## 8. Chaves de Configuração Principais

Namespace `memory.*`. Escopo: `global` (G), `por tenant` (T), `por agente` (A). Recarregável em runtime salvo indicado.

| Chave | Default | Faixa | Escopo | Recarregável |
|-------|---------|-------|--------|--------------|
| `memory.embedding.dim` | `1024` | 256–4096 | G | não (reindex) |
| `memory.embedding.model` | `text-embed-default` | URN de modelo | T | sim |
| `memory.item.inline_max_bytes` | `65536` | 1 KiB–1 MiB | T | sim |
| `memory.hnsw.m` | `16` | 8–64 | G/T | não (reindex) |
| `memory.hnsw.ef_construction` | `128` | 32–512 | G | não (reindex) |
| `memory.hnsw.ef_search` | `64` | 16–512 | T | sim |
| `memory.recall.default_k` | `10` | 1–200 | T/A | sim |
| `memory.recall.min_score` | `0.30` | 0.0–1.0 | T/A | sim |
| `memory.recall.fusion` | `rrf` | `rrf\|weighted` | T | sim |
| `memory.layer.working.ttl` | `900s` | 60s–3600s | T/A | sim |
| `memory.layer.short_term.ttl` | `86400s` | 1h–7d | T/A | sim |
| `memory.layer.long_term.ttl` | `90d` | 7d–730d/∞ | T | sim |
| `memory.layer.episodic.ttl` | `365d` | 30d–1825d/∞ | T | sim |
| `memory.decay.half_life` | `30d` | 1d–365d | T/A | sim |
| `memory.decay.threshold` | `0.15` | 0.0–1.0 | T/A | sim |
| `memory.quota.max_items` | `1000000` | ≥ 0 | T/A/layer | sim |
| `memory.quota.max_bytes` | `10GiB` | ≥ 0 | T/A/layer | sim |
| `memory.consolidation.threshold` | `500` | ≥ 1 | T/A | sim |
| `memory.consolidation.auto` | `true` | bool | T | sim |
| `memory.consolidation.version.retention` | `10` | 1–100 | T | sim |
| `memory.forget.grace_period` | `7d` | 0–90d | T | sim |
| `memory.rtbf.enabled` | `true` | bool | T | sim |
| `memory.backpressure.max_inflight` | `1000` | 10–100000 | G | sim |
| `memory.blob.bucket` | `aios-memory` | nome S3 | G | não |

---

## 9. Modos de Falha e Estratégia de Recuperação

| # | Modo de falha | Detecção | Recuperação | Idemp./retry | RTO/RPO |
|---|---------------|----------|-------------|--------------|---------|
| F1 | Redis (Working/hot) indisponível | health probe / timeout | *Fail-open* p/ camadas duráveis; Working reconstruível; degradação graciosa. | leitura retry; Working sem retry | RTO ≤ 1 min; RPO Working = tolerado (NFR-008) |
| F2 | PostgreSQL primário cai | replicação lag / erro | Failover p/ standby (027); reprocessa Outbox. | escrita idempotente por `Idempotency-Key` | RTO ≤ 15 min; RPO ≤ 5 min |
| F3 | Model Router (017) indisponível | erro `AIOS-MEM-0030` | Enfileira item sem embedding (`state=INGESTED`); gera embedding em backlog. | retry backoff exp.; batch | sem perda; embedding eventual |
| F4 | MinIO indisponível | erro S3 | Mantém conteúdo inline se possível; represa blob; retry. | put idempotente por hash | RPO ≤ 5 min |
| F5 | Apache AGE/grafo falha | erro consulta | Recall cai p/ `hybrid` sem grafo (degradação); `mode=graph` retorna `AIOS-MEM-0031`. | retry | sem perda |
| F6 | Consolidação corrompe/regressa Recall Rate | validação NFR-011 falha | Rollback automático à `ConsolidationVersion` anterior; evento `rolledback`. | job idempotente por `version_id` | estado restaurado |
| F7 | Estouro de cota | contador QuotaManager | Backpressure `AIOS-MEM-0011` / `0010`; dispara poda (ForgettingEngine). | 429 retriable | — |
| F8 | Evento não publicado (crash pós-commit) | Outbox `pending` | Outbox relay reenvia at-least-once. | dedup por `event.id` | RPO evento = 0 (Outbox) |
| F9 | Duplicidade de escrita | `Idempotency-Key` | Retorna resultado memoizado (IdempotencyStore). | dedup ≥ 24h | — |
| F10 | Poison message (consumo) | N falhas | Move p/ DLQ JetStream; alerta; inspeção manual. | — | — |

**Princípios:** toda mutação DEVE ser idempotente (RFC-0001 §5.5); retries com backoff exponencial + jitter;
consumidores DEVEM deduplicar por `event.id`; Outbox garante atomicidade DB+evento; camadas duráveis têm RPO ≤ 5 min.

---

## 10. Escalabilidade e Concorrência

| Dimensão | Estratégia |
|----------|-----------|
| **Sharding** | Determinístico `shard = hash(tenant_id, agent_id) mod N` (alinhado a `001-Architecture` §12). Vetores particionados por tenant; Episodic particionado por tempo (`RANGE` mensal). |
| **Índice vetorial** | HNSW por partição/tenant; `ef_search` ajustável por tenant; reindex online sem downtime. Caminho de migração p/ índice externo previsto (ADR-0005). |
| **Estado quente vs. frio** | Working/Short-Term em Redis; agentes suspensos têm memória "cold" em PG/MinIO, reidratada sob recall (agentes efêmeros → milhões). |
| **Concorrência de escrita** | Escrita append-preferencial; atualização de `MemoryItem` sob *optimistic concurrency* (`updated_at`/versão). Sem locks longos no caminho quente. |
| **Locks** | Locks distribuídos (Redis) apenas para consolidação por (tenant, agente) — serializa jobs concorrentes; jobs idempotentes por `version_id`. |
| **Backpressure** | `memory.backpressure.max_inflight` + JetStream; admissão de escrita represada ao saturar; sinaliza `AIOS-MEM-0011`. |
| **Leitura em escala** | CQRS: réplicas de leitura PG para recall; cache de resultados quentes em Redis; Short-Term serve de projeção. |
| **Consolidação em lote** | Jobs assíncronos fora do caminho quente, agendados pelo RetentionScheduler; particionados por shard. |
| **Limites-alvo** | ≥ 10^8 vetores/tenant (NFR-012); ≥ 5.000 remember/s por shard (NFR-004); crescimento sub-linear via poda + cotas. |

---

## 11. ADRs e RFCs a Propor

### 11.1 ADRs específicas do módulo (faixa reservada `ADR-0100`..`ADR-0109`)

| ADR | Título proposto |
|-----|-----------------|
| ADR-0100 | Modelo canônico de 7 camadas de memória e mapeamento a backends físicos (Redis/PG/pgvector/AGE/MinIO). |
| ADR-0101 | Estratégia de embeddings e índice vetorial (pgvector + HNSW) e parâmetros por tenant. |
| ADR-0102 | Consolidação versionada e rollback como defesa contra *catastrophic forgetting*. |
| ADR-0103 | Políticas de esquecimento controlado (TTL, decaimento, LRU, cota, RTBF) e sua precedência. |
| ADR-0104 | Fusão e re-ranqueamento de recall híbrido (lexical + vetorial + grafo; RRF vs. weighted). |
| ADR-0105 | Cotas de memória e backpressure por (tenant, agente, camada). |
| ADR-0106 | Externalização de blobs grandes (MinIO content-addressed) e limiar inline. |
| ADR-0107 | Fronteira de acesso Runtime↔Memory (sem acesso direto a datastore; gRPC/NATS). |
| ADR-0108 | Particionamento/sharding de memória vetorial e Episodic (tempo) rumo a milhões de agentes. |
| ADR-0109 | Modelo de retenção legal, base legal por item e direito ao esquecimento (LGPD/GDPR). |

### 11.2 RFCs relacionadas

| RFC | Relação |
|-----|---------|
| RFC-0001 (Accepted) | Baseline: URN, eventos, erros, idempotência, correlação — **reutilizada, não redefinida**. |
| RFC-0100 (a propor) | *Memory Consolidation & Forgetting Contract*: contrato de consolidação dirigida por 023, versionamento/rollback, e semântica de esquecimento inter-módulos. |

---

## 12. Decisões de Segurança

### 12.1 AuthN / AuthZ

- **AuthN:** tokens OAuth2/OIDC validados no Gateway (021); claims assinadas propagadas; mTLS entre serviços internos (RFC-0001 §6).
- **AuthZ:** `MemoryPep` (PEP) consulta o PDP (022) por operação/camada/recurso; *default deny*. Toda operação privilegiada DEVE emitir auditoria (025).
- **Isolamento de tenant:** `tenant` do recurso DEVE coincidir com o contexto autenticado; divergência → `AIOS-MEM-0005`. RLS obrigatória em toda tabela; namespace NATS por tenant (`aios.<tenant>.*`).

### 12.2 Threat Model STRIDE (resumido)

| Ameaça | Vetor no módulo | Mitigação |
|--------|-----------------|-----------|
| **S**poofing | Chamador forja tenant/agente. | OIDC + mTLS; tenant guard; claims assinadas. |
| **T**ampering | Alteração de item/embedding/versão. | Escrita idempotente; hash content-addressed; auditoria imutável (025); RLS. |
| **R**epudiation | Negar consolidação/esquecimento. | Eventos assinados + trilha imutável (025); linhagem `parent_ids`/`ConsolidationVersion`. |
| **I**nformation disclosure | Vazamento cross-tenant / PII em logs/eventos. | RLS; minimização/redação de PII (RFC-0001 §7); `pii` flag; erro sem `detail` sensível. |
| **D**enial of service | Flood de `remember`/`recall`; explosão de vetores. | Cotas + backpressure + rate-limit no Gateway; poda por esquecimento. |
| **E**levation of privilege | Acesso a memória de outro agente/tenant. | PEP/PDP default deny; capabilities por syscall cognitiva (006/021). |

### 12.3 LGPD / GDPR

- Todo `MemoryItem` DEVE carregar `legal_basis` e `retention_class` (RFC-0001 §7); PII marcada e redigida/tokenizada em eventos/logs.
- **Direito ao esquecimento (RTBF):** operação de expurgo rastreável (`DELETE`/`forget(rtbf)`), assíncrona, com evento `purged` e auditoria; `legal_hold` só é purgável sob RTBF autorizado.
- Minimização de dados: eventos/logs carregam o mínimo; conteúdo sensível fica em PG/MinIO sob RLS, não no envelope de evento.
- Retenção por classe (`ephemeral/standard/extended/legal_hold`) governa TTL/purge; expurgo remove vetor + blob + tombstone após grace period.

---

*Fim do Design Brief do Módulo 010-Memory. Este documento governa a geração dos 26 documentos obrigatórios listados em `README.md`.*
