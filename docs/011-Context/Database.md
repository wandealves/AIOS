---
Documento: Database
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0111, ADR-0112, ADR-0116, ADR-0117, ADR-0119; ADR-0005 (herdada), ADR-0006 (herdada)
RFCs relacionados: RFC-0001 (baseline), RFC-0011 (proposta)
Depende de: 003-RFC (RFC-0001), 005-Database, 010-Memory, 017-Model-Router, 021-Security, 025-Audit
---

# 011-Context — Database

> **Escopo.** Modelo físico de persistência do `ContextService`: DDL
> PostgreSQL (fonte da verdade de `ContextBundle`/`ContextFragment`/
> `SemanticCacheEntry`/`BudgetProfile`/`SummaryNode`, com `pgvector` para busca
> por similaridade) e estruturas de estado quente em Redis (L1 do cache
> semântico, locks, rate limiting). Convenções de URN, ULID e RLS por tenant
> **DEVEM** seguir `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.1 e o
> padrão geral de `../005-Database/Architecture.md`. Consistência absoluta com
> `./_DESIGN_BRIEF.md` §3.

---

## 1. Justificativa de tecnologia: `pgvector` sim, Apache AGE não

O `011-Context` **usa `pgvector`** intensivamente — é a base do `RelevanceRanker`
(similaridade fragmento↔query), do `RedundancyEliminator` (detecção de
near-duplicates) e do L2 do `SemanticCacheManager` (busca por similaridade de
prompt). O módulo **NÃO utiliza Apache AGE**: não há grafo de conhecimento
neste domínio — o `GraphRAG` e o `Knowledge Graph` são responsabilidade de
`018/019-Knowledge` (brief §1.3-N05); o `011` apenas **consome** fragmentos já
extraídos do grafo como candidatos de contexto (`source_kind=knowledge`), sem
persistir nós/arestas.

| Armazenamento | Papel | Por quê |
|-----------------|-------|---------|
| PostgreSQL + `pgvector` | Fonte da verdade de `ContextBundle`/`ContextFragment`/`SemanticCacheEntry` (L2 durável)/`BudgetProfile`/`SummaryNode`; busca ANN por similaridade. | Durabilidade, RLS por tenant, consistência transacional (outbox), índices HNSW/IVFFLAT nativos. |
| Redis | Estado **quente**: L1 do cache semântico (sub-ms), locks distribuídos de cache-fill, contadores de rate-limit/backpressure. | Latência sub-ms exigida por NFR-001 (`CacheLookup` p99 ≤ 20 ms). |
| MinIO | Offload de fragmentos/blobs grandes (`content_ref`/`response_ref`) acima de `context.limits.max_inline_bytes`. | Evita inflar linhas PostgreSQL com payloads grandes; leitura sob demanda. |

---

## 2. Esquema PostgreSQL — DDL

### 2.1 Schema e extensões

```sql
CREATE SCHEMA IF NOT EXISTS context;

-- Busca por similaridade (ANN) — base de RelevanceRanker, RedundancyEliminator
-- e SemanticCacheManager (L2).
CREATE EXTENSION IF NOT EXISTS vector;

-- Apenas para geração de UUID auxiliar em índices internos; IDs de negócio
-- são ULID gerados na aplicação (RFC-0001 §5.1), armazenados como CHAR(26).
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

> **Nota sobre ULID:** ULIDs (26 caracteres, Base32 Crockford) são
> armazenados como `CHAR(26)` (ordenáveis lexicograficamente por tempo de
> criação). `tenant_id` é `TEXT` (slug `[a-z0-9-]+`). Dimensão de embedding
> padrão: `1536` (`context.embedding.dimensions`, ver
> [Configuration.md](./Configuration.md)) — **não recarregável** sem reindex.

### 2.2 Tabela `context.context_bundle` — contexto montado por chamada

Deriva da §3.1 do brief. Tabela de **alto volume, efêmera** (TTL curto via
`expires_at`); particionada por tempo (§3).

```sql
CREATE TABLE context.context_bundle (
    bundle_id                CHAR(26)        NOT NULL,
    tenant_id                TEXT            NOT NULL,
    agent_urn                TEXT            NOT NULL,
    task_urn                 TEXT,
    model_id                 TEXT            NOT NULL,
    model_max_tokens         INTEGER         NOT NULL CHECK (model_max_tokens > 0),
    token_budget             INTEGER         NOT NULL CHECK (token_budget > 0),
    reserved_output_tokens   INTEGER         NOT NULL CHECK (reserved_output_tokens >= 0),
    tokens_in                INTEGER         NOT NULL CHECK (tokens_in >= 0),
    tokens_raw               INTEGER         NOT NULL CHECK (tokens_raw >= 0),
    compression_ratio        NUMERIC(5,4)    NOT NULL CHECK (compression_ratio BETWEEN 0 AND 1),
    status                   TEXT            NOT NULL
        CHECK (status IN ('RECEIVED','POLICY_CHECK','CACHE_LOOKUP','BUDGET_ALLOCATED',
                           'RETRIEVING','RANKING','COMPRESSING','ASSEMBLED','SERVED',
                           'REJECTED','DEGRADED','FAILED','EXPIRED')),
    cache_outcome            TEXT            CHECK (cache_outcome IN ('HIT','MISS','BYPASS')),
    idempotency_key          TEXT,
    budget_profile_id        CHAR(26)        NOT NULL,
    trace_id                 TEXT            NOT NULL,
    created_at               TIMESTAMPTZ     NOT NULL DEFAULT now(),
    expires_at                TIMESTAMPTZ     NOT NULL,
    dataschema                TEXT            NOT NULL,

    CONSTRAINT pk_context_bundle PRIMARY KEY (tenant_id, bundle_id, created_at),
    CONSTRAINT uq_context_bundle_idem UNIQUE (tenant_id, idempotency_key),
    CONSTRAINT chk_bundle_agent_urn CHECK (agent_urn ~ '^urn:aios:[a-z0-9-]+:agent:[0-9A-HJKMNP-TV-Z]{26}$'),
    CONSTRAINT chk_bundle_expires_after_created CHECK (expires_at > created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE context.context_bundle_2026_07
    PARTITION OF context.context_bundle
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

CREATE INDEX idx_ctx_bundle_agent ON context.context_bundle (tenant_id, agent_urn, created_at DESC);
CREATE INDEX idx_ctx_bundle_task ON context.context_bundle (tenant_id, task_urn) WHERE task_urn IS NOT NULL;
CREATE INDEX idx_ctx_bundle_status ON context.context_bundle (tenant_id, status, created_at DESC);
CREATE INDEX idx_ctx_bundle_expires ON context.context_bundle (expires_at);

ALTER TABLE context.context_bundle ENABLE ROW LEVEL SECURITY;
ALTER TABLE context.context_bundle FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_context_bundle ON context.context_bundle
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

**Invariantes:** `(tenant_id, bundle_id, created_at)` é a PK composta
(particionamento exige a coluna de partição na PK); `idempotency_key` tem
unicidade por tenant (RFC-0001 §5.5), permitindo `NULL` para leituras que não
originam de mutação; `expires_at` DEVE ser posterior a `created_at` (bundle é
efêmero por construção, brief §1.2-N/A2).

### 2.3 Tabela `context.context_fragment` — fragmento candidato/incluído

Deriva da §3.2 do brief. `embedding vector(1536)` com índice **IVFFLAT**
(alternativa configurável a HNSW, ver `context.cache.vector_index` em
[Configuration.md](./Configuration.md); aqui o índice é fixo em IVFFLAT por
ser adequado ao volume por-bundle, tipicamente pequeno).

```sql
CREATE TABLE context.context_fragment (
    fragment_id          CHAR(26)        NOT NULL,
    tenant_id            TEXT            NOT NULL,
    bundle_id            CHAR(26)        NOT NULL,
    bundle_created_at    TIMESTAMPTZ     NOT NULL,   -- espelha a coluna de partição do bundle p/ FK
    source_kind          TEXT            NOT NULL
        CHECK (source_kind IN ('system','instruction','memory','knowledge','history','tool_result')),
    source_urn           TEXT,
    role                 TEXT            NOT NULL
        CHECK (role IN ('system','user','assistant','tool')),
    ordinal              INTEGER         NOT NULL CHECK (ordinal >= 0),
    tokens_raw           INTEGER         NOT NULL CHECK (tokens_raw >= 0),
    tokens_final         INTEGER         NOT NULL CHECK (tokens_final >= 0 AND tokens_final <= tokens_raw),
    relevance_score       NUMERIC(6,5)    NOT NULL CHECK (relevance_score BETWEEN 0 AND 1),
    compression_method    TEXT            NOT NULL
        CHECK (compression_method IN ('none','extractive','abstractive','truncate','dedup')),
    included             BOOLEAN         NOT NULL,
    embedding            vector(1536),
    content_inline        TEXT,
    content_ref           TEXT,
    created_at            TIMESTAMPTZ     NOT NULL DEFAULT now(),

    CONSTRAINT pk_context_fragment PRIMARY KEY (tenant_id, fragment_id),
    CONSTRAINT fk_context_fragment_bundle
        FOREIGN KEY (tenant_id, bundle_id, bundle_created_at)
        REFERENCES context.context_bundle (tenant_id, bundle_id, created_at)
        ON DELETE CASCADE,
    CONSTRAINT chk_fragment_content_present
        CHECK (content_inline IS NOT NULL OR content_ref IS NOT NULL)
);

CREATE INDEX idx_ctx_fragment_bundle ON context.context_fragment (tenant_id, bundle_id, ordinal);
CREATE INDEX idx_ctx_fragment_source ON context.context_fragment (tenant_id, source_kind, source_urn)
    WHERE source_urn IS NOT NULL;
CREATE INDEX idx_ctx_fragment_embedding_ivfflat
    ON context.context_fragment USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

ALTER TABLE context.context_fragment ENABLE ROW LEVEL SECURITY;
ALTER TABLE context.context_fragment FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_context_fragment ON context.context_fragment
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

**Invariantes:** `tokens_final ≤ tokens_raw` (compressão nunca aumenta
tokens); todo fragmento DEVE ter `content_inline` **ou** `content_ref`
(offload MinIO acima de `context.limits.max_inline_bytes`, `ADR-0117`); a FK
composta inclui `bundle_created_at` para respeitar o particionamento por
`RANGE (created_at)` do bundle pai.

### 2.4 Tabela `context.semantic_cache_entry` — entrada de cache semântico

Deriva da §3.3 do brief. `prompt_embedding vector(1536)` com índice **HNSW**
(`context.cache.vector_index=hnsw`, default). Tabela **não** particionada por
tempo — entradas são de vida longa (TTL de horas/dias), ao contrário de
`context_bundle`/`context_fragment`; gestão de crescimento é feita pelo
`EvictionManager` (TTL + LRU semântico), não por poda de partição.

```sql
CREATE TABLE context.semantic_cache_entry (
    cache_id                CHAR(26)        NOT NULL,
    tenant_id                TEXT            NOT NULL,
    cache_scope              TEXT            NOT NULL
        CHECK (cache_scope IN ('tenant','agent','task_type')),
    scope_ref                TEXT,
    model_id                 TEXT            NOT NULL,
    prompt_hash               TEXT            NOT NULL,
    prompt_embedding          vector(1536)    NOT NULL,
    similarity_threshold      NUMERIC(4,3)    NOT NULL CHECK (similarity_threshold BETWEEN 0 AND 1),
    response_ref              TEXT            NOT NULL,
    tokens_saved              INTEGER         NOT NULL CHECK (tokens_saved >= 0),
    cost_saved_usd            NUMERIC(12,6)   NOT NULL CHECK (cost_saved_usd >= 0),
    hit_count                 BIGINT          NOT NULL DEFAULT 0 CHECK (hit_count >= 0),
    ttl_seconds               INTEGER         NOT NULL CHECK (ttl_seconds > 0),
    created_at                TIMESTAMPTZ     NOT NULL DEFAULT now(),
    last_hit_at               TIMESTAMPTZ,
    invalidated_at            TIMESTAMPTZ,

    CONSTRAINT pk_semantic_cache_entry PRIMARY KEY (tenant_id, cache_id),
    CONSTRAINT uq_semantic_cache_prompt_hash UNIQUE (tenant_id, model_id, prompt_hash)
);

CREATE INDEX idx_cache_entry_model ON context.semantic_cache_entry (tenant_id, model_id);
CREATE INDEX idx_cache_entry_last_hit ON context.semantic_cache_entry (tenant_id, last_hit_at DESC NULLS LAST);
CREATE INDEX idx_cache_entry_scope ON context.semantic_cache_entry (tenant_id, cache_scope, scope_ref)
    WHERE scope_ref IS NOT NULL;
-- Índice parcial: apenas entradas ativas (não invalidadas) entram na busca ANN quente.
CREATE INDEX idx_cache_entry_embedding_hnsw
    ON context.semantic_cache_entry USING hnsw (prompt_embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 128)
    WHERE invalidated_at IS NULL;

ALTER TABLE context.semantic_cache_entry ENABLE ROW LEVEL SECURITY;
ALTER TABLE context.semantic_cache_entry FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_semantic_cache_entry ON context.semantic_cache_entry
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

**Invariantes (defesa contra cache poisoning, `ADR-0119`):** `prompt_hash`
único por `(tenant_id, model_id)` garante fast-path exato sem colisão
cross-tenant; o índice HNSW é **filtrado** (`WHERE invalidated_at IS NULL`)
para que entradas invalidadas nunca sejam candidatas a hit, mesmo antes da
remoção física pelo `EvictionManager`; `similarity_threshold` é persistido
**por entrada** (não apenas configuração global) para permitir ajuste fino
pós-incidente (FM-08) sem reprocessar entradas existentes.

### 2.5 Tabela `context.budget_profile` — perfil de orçamento de tokens

Deriva da §3.4 do brief. Tabela de baixo volume (configuração), sem
particionamento.

```sql
CREATE TABLE context.budget_profile (
    profile_id                CHAR(26)        NOT NULL,
    tenant_id                  TEXT            NOT NULL,
    scope                      TEXT            NOT NULL
        CHECK (scope IN ('tenant','agent','task_type')),
    scope_ref                  TEXT,
    reserved_output_ratio      NUMERIC(4,3)    NOT NULL DEFAULT 0.25
        CHECK (reserved_output_ratio BETWEEN 0.05 AND 0.60),
    alloc_system               NUMERIC(4,3)    NOT NULL CHECK (alloc_system BETWEEN 0 AND 1),
    alloc_instruction          NUMERIC(4,3)    NOT NULL CHECK (alloc_instruction BETWEEN 0 AND 1),
    alloc_memory               NUMERIC(4,3)    NOT NULL CHECK (alloc_memory BETWEEN 0 AND 1),
    alloc_history              NUMERIC(4,3)    NOT NULL CHECK (alloc_history BETWEEN 0 AND 1),
    alloc_tools                NUMERIC(4,3)    NOT NULL CHECK (alloc_tools BETWEEN 0 AND 1),
    min_compression_ratio      NUMERIC(4,3)    NOT NULL DEFAULT 0.30
        CHECK (min_compression_ratio BETWEEN 0 AND 0.90),
    updated_at                 TIMESTAMPTZ     NOT NULL DEFAULT now(),

    CONSTRAINT pk_budget_profile PRIMARY KEY (tenant_id, profile_id),
    CONSTRAINT uq_budget_profile_scope UNIQUE (tenant_id, scope, scope_ref),
    -- Invariante normativa do brief §3.4: soma das alocações = 1.0 (±2%, AIOS-CTX-0008)
    CONSTRAINT chk_budget_alloc_sum_1_0
        CHECK (abs((alloc_system + alloc_instruction + alloc_memory + alloc_history + alloc_tools) - 1.0) <= 0.02)
);

CREATE INDEX idx_budget_profile_scope ON context.budget_profile (tenant_id, scope, scope_ref);

ALTER TABLE context.budget_profile ENABLE ROW LEVEL SECURITY;
ALTER TABLE context.budget_profile FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_budget_profile ON context.budget_profile
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

**Invariante `AIOS-CTX-0008`:** a `CHECK` constraint reforça no banco a
invariante já validada na camada de aplicação (`ContextApiGateway`) antes do
`INSERT`/`UPDATE` — dupla defesa (validação de borda + constraint física).
Qualquer tentativa de violar a soma (mesmo por escrita direta) é rejeitada
pelo PostgreSQL.

### 2.6 Tabela `context.summary_node` — nó de sumário hierárquico

Deriva da §3.5 do brief. Árvore de sumarização por auto-FK (`parent_id`);
cache de **curta duração** — nunca memória canônica (fronteira brief §1.3).

```sql
CREATE TABLE context.summary_node (
    node_id       CHAR(26)        NOT NULL,
    tenant_id      TEXT            NOT NULL,
    source_urn     TEXT            NOT NULL,
    level          INTEGER         NOT NULL CHECK (level >= 0),
    parent_id      CHAR(26),
    tokens         INTEGER         NOT NULL CHECK (tokens >= 0),
    content_ref    TEXT            NOT NULL,
    embedding      vector(1536),
    expires_at     TIMESTAMPTZ     NOT NULL,

    CONSTRAINT pk_summary_node PRIMARY KEY (tenant_id, node_id),
    CONSTRAINT fk_summary_node_parent
        FOREIGN KEY (tenant_id, parent_id)
        REFERENCES context.summary_node (tenant_id, node_id)
        ON DELETE CASCADE
);

CREATE INDEX idx_summary_node_source ON context.summary_node (tenant_id, source_urn, level);
CREATE INDEX idx_summary_node_parent ON context.summary_node (tenant_id, parent_id) WHERE parent_id IS NOT NULL;
CREATE INDEX idx_summary_node_expires ON context.summary_node (expires_at);
CREATE INDEX idx_summary_node_embedding_ivfflat
    ON context.summary_node USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 50);

ALTER TABLE context.summary_node ENABLE ROW LEVEL SECURITY;
ALTER TABLE context.summary_node FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_summary_node ON context.summary_node
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

**Invariantes:** `level = 0` identifica folhas (fragmento original
sumarizado); níveis crescentes representam reduções sucessivas do
map-reduce (`HierarchicalCompressor`, `context.compression.max_levels`); a
remoção em cascata (`ON DELETE CASCADE`) reflete que um nó pai sem filhos
associados perde sentido de reuso; reuso por similaridade (FR-014) consulta
`embedding` com `context.cache.reuse_threshold`.

### 2.7 Tabela `context.idempotency_record` — persistência durável de idempotência

Espelha o padrão de `009-Scheduler`/`010-Memory`: Redis é o caminho quente,
PostgreSQL é o *fallback*/registro de longo prazo (RFC-0001 §5.5, ≥ 24h).

```sql
CREATE TABLE context.idempotency_record (
    tenant_id          TEXT            NOT NULL,
    idempotency_key     TEXT            NOT NULL,
    operation           TEXT            NOT NULL
        CHECK (operation IN ('assemble','compress','cache_store','cache_invalidate','budget_upsert')),
    response_snapshot   JSONB           NOT NULL,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT now(),
    expires_at          TIMESTAMPTZ     NOT NULL,

    CONSTRAINT pk_context_idempotency PRIMARY KEY (tenant_id, idempotency_key)
);

CREATE INDEX idx_ctx_idempotency_expires ON context.idempotency_record (expires_at);

ALTER TABLE context.idempotency_record ENABLE ROW LEVEL SECURITY;
ALTER TABLE context.idempotency_record FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_ctx_idempotency ON context.idempotency_record
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

### 2.8 Diagrama relacional (ASCII)

```
┌────────────────────────────┐  1        N ┌──────────────────────────────┐
│ budget_profile              │────────────▶│ context_bundle                │
│ PK (tenant_id,profile_id)   │  (budget_profile_id, aplicado)              │
│ scope, alloc_* (soma=1.0)   │             │ PK (tenant_id,bundle_id,created_at)│
└────────────────────────────┘             │ agent_urn, model_id, status    │
                                            │ cache_outcome, tokens_*         │
                                            └───────────────┬────────────────┘
                                                             │ 1        N
                                                             ▼
                                            ┌──────────────────────────────┐
                                            │ context_fragment              │
                                            │ PK (tenant_id,fragment_id)    │
                                            │ FK → context_bundle           │
                                            │ embedding vector(1536) IVFFLAT│
                                            └──────────────────────────────┘

┌──────────────────────────────┐            ┌──────────────────────────────┐
│ semantic_cache_entry          │            │ summary_node                  │
│ PK (tenant_id,cache_id)       │            │ PK (tenant_id,node_id)        │
│ UQ (tenant,model,prompt_hash) │            │ FK self (parent_id) — árvore  │
│ prompt_embedding HNSW         │            │ embedding vector(1536) IVFFLAT│
└──────────────────────────────┘            └──────────────────────────────┘

┌──────────────────────────────┐
│ idempotency_record            │
│ PK (tenant_id,idempotency_key)│
└──────────────────────────────┘
```

---

## 3. Particionamento

| Tabela | Estratégia | Granularidade | Justificativa |
|--------|-----------|-----------------|------------------|
| `context_bundle` | `RANGE (created_at)` | Mensal | Alto volume de escrita (NFR-009: ≥ 500 req/s por réplica de `assemble`); bundles são efêmeros (`expires_at`), poda de partição é O(1) (`DETACH PARTITION`). |
| `context_fragment` | Herdado via FK composta ao bundle particionado | Mensal (acompanha o bundle) | Mesmo racional; volume ainda maior (N fragmentos por bundle). |
| `semantic_cache_entry` | **Não particionada por tempo** — entradas de vida longa geridas por `EvictionManager` (TTL/LRU) | — | Particionar por tempo destruiria a localidade de busca ANN (HNSW); crescimento é limitado por `context.cache.max_entries_per_tenant`. Caminho de escala futuro: partição por `HASH(tenant_id)` quando o volume por tenant justificar (ver [Scalability.md](./Scalability.md)). |
| `budget_profile` | Não particionada | — | Volume trivial (configuração). |
| `summary_node` | Não particionada por tempo; poda por `expires_at` + `ON DELETE CASCADE` a partir da raiz expirada | — | Volume moderado; TTL curto evita crescimento descontrolado sem necessidade de partição física. |
| `idempotency_record` | Não particionada (TTL curto, expurgo frequente por `expires_at`) | — | Volume limitado pelo TTL (≥ 24h); índice em `expires_at` é suficiente. |

Partições futuras (mês seguinte) de `context_bundle` DEVEM ser criadas com
antecedência mínima de 7 dias por job agendado (`pg_partman` ou equivalente,
operado por `027-Cluster`/`005-Database`); ausência de partição futura é
alarme de severidade alta (ver [Monitoring.md](./Monitoring.md)).

---

## 4. Políticas de retenção e expurgo

### 4.1 Retenção padrão

| Tabela | Retenção "quente" | Retenção total | Destino pós-retenção |
|--------|----------------------|-------------------|------------------------|
| `context_bundle` / `context_fragment` | `expires_at` (minutos a poucas horas, efêmero por design) | 1 partição mensal corrente + 1 anterior (auditoria de curto prazo) | `DROP PARTITION`; sem arquivamento frio (conteúdo é derivado/reproduzível, não é fonte de verdade — brief §1.3). |
| `semantic_cache_entry` | `ttl_seconds` por entrada (default `context.cache.default_ttl_seconds`) | Até eviction (TTL/LRU/invalidação event-driven) | `DELETE` físico via `EvictionManager`; sem arquivamento frio. |
| `summary_node` | `expires_at` | Até eviction | `DELETE` em cascata a partir da raiz expirada. |
| `budget_profile` | Indefinida (configuração ativa) | — | Histórico de mudança auditado via evento `context.budget.updated` + `025-Audit`, não via tabela de histórico local. |
| `idempotency_record` | — | `expires_at` (≥ 24h) | `DELETE` direto por job periódico. |

### 4.2 Job de expurgo de bundles expirados (exemplo)

```sql
-- Executado periodicamente (ex.: a cada 5 min) por worker do ContextService.
DELETE FROM context.context_bundle
WHERE expires_at < now()
LIMIT 10000; -- em lotes; ON DELETE CASCADE remove os fragmentos associados.
```

### 4.3 Job de expurgo de cache semântico (TTL + LRU)

```sql
-- TTL expirado
DELETE FROM context.semantic_cache_entry
WHERE created_at + make_interval(secs => ttl_seconds) < now()
LIMIT 10000;

-- LRU semântico sob pressão de cota (context.cache.max_entries_per_tenant)
DELETE FROM context.semantic_cache_entry
WHERE tenant_id = $1
  AND cache_id IN (
    SELECT cache_id FROM context.semantic_cache_entry
    WHERE tenant_id = $1
    ORDER BY COALESCE(last_hit_at, created_at) ASC
    LIMIT $2  -- quantidade a evictar para voltar abaixo da cota
  );
```

### 4.4 Direito ao esquecimento (LGPD/GDPR) e invalidação por origem

Disparado por `memory.item.deleted`/`memory.item.consolidated` (ver
[Events.md](./Events.md) §4) ou por chamada manual a
`/v1/context/cache/invalidate`:

```sql
-- Invalidação lógica (marca invalidated_at; remove do índice ANN via WHERE parcial)
UPDATE context.semantic_cache_entry
SET invalidated_at = now()
WHERE tenant_id = $1 AND response_ref LIKE ('%' || $2 || '%'); -- $2 = source_urn

-- Expurgo físico de bundles/fragmentos remanescentes de um agente terminado (RTBF/lifecycle)
DELETE FROM context.context_fragment WHERE tenant_id = $1 AND fragment_id IN (
  SELECT fragment_id FROM context.context_fragment f
  JOIN context.context_bundle b USING (tenant_id, bundle_id)
  WHERE b.tenant_id = $1 AND b.agent_urn = $3
);
DELETE FROM context.context_bundle WHERE tenant_id = $1 AND agent_urn = $3;
```

`content_inline`/`content_ref`/`response_ref` NÃO DEVEM conter PII bruta sem
correspondência de base legal — o dado de origem (memória/conhecimento)
permanece sob a governança de `010-Memory`/`018-019-Knowledge`; este módulo
apenas referencia e expurga por propagação de evento (brief §12.3).

---

## 5. Redis — Estruturas de Estado Quente

> Fora do escopo de DDL relacional, mas **contratual** ao módulo — o
> comportamento do caminho quente (`CacheLookup` p99 ≤ 20 ms, NFR-001)
> depende inteiramente destas estruturas. Prefixos configuráveis, ver
> [Configuration.md](./Configuration.md).

### 5.1 L1 do cache semântico — `ctx:cache:{tenant}:{model_id}:{prompt_hash}` (STRING)

| Campo | Tipo | Notas |
|-------|------|-------|
| Chave | `ctx:cache:{tenant}:{model_id}:{prompt_hash}` | Fast-path de match exato (hash normalizado do prompt). |
| Valor | `STRING` (JSON serializado) | Espelha `semantic_cache_entry` (response_ref, tokens_saved, cost_saved_usd). |
| TTL | `context.cache.l1_redis_ttl_seconds` (default 300s) | Curto — L1 é acelerador, L2 (`pgvector`) é a fonte durável. |

```
SET ctx:cache:acme:gpt-x-mini:9f2b0c... "{\"cacheId\":\"01J9...\",\"responseRef\":\"minio://...\"}" EX 300
```

### 5.2 Locks de cache-fill — `ctx:lock:{tenant}:{cache_key}` (single-flight)

| Campo | Tipo | Notas |
|-------|------|-------|
| Chave | `ctx:lock:{tenant}:{cache_key}` | `SET NX PX` — impede *thundering herd* quando N requisições concorrentes miss no mesmo prompt. |
| TTL | `context.cache.lock_ttl_ms` | Curto (ordem de segundos); expira automaticamente se o cache-fill travar. |

```
SET ctx:lock:acme:9f2b0c... "worker-7" NX PX 5000
```

Apenas a escrita de cache usa lock (brief §10); leitura (`CacheLookup`) é
livre de lock — múltiplas leituras concorrentes não competem.

### 5.3 Rate limiting e backpressure — `ctx:ratelimit:*`

| Chave | Tipo | Notas |
|-------|------|-------|
| `ctx:ratelimit:{tenant}:assemble` | contador com janela deslizante (script Lua) | Compara contra `context.ratelimit.assemble_rps_per_tenant`; excedente → `AIOS-CTX-0015`. |

### 5.4 Snapshot de limites de modelo (fallback de degradação)

| Chave | Tipo | Notas |
|-------|------|-------|
| `ctx:modellimits:{model_id}` | `HASH` + TTL | Cache dos últimos limites conhecidos de `017-Model-Router` (`model_max_tokens`, tokenizer); usado em `BUDGET_ALLOCATED` quando `017` está indisponível (FM-01). |

---

## 6. Migrações

- Ferramenta: migrações versionadas (`Flyway`/EF Core Migrations, detalhado
  em [Deployment.md](./Deployment.md)), nomeadas `V<NNNN>__<descricao>.sql`,
  aplicadas de forma **aditiva e retrocompatível** (nunca `DROP COLUMN`/
  `ALTER TYPE` destrutivo sem janela de coexistência ≥ 2 versões, espelhando
  RFC-0001 §5.7).
- Toda migração DEVE ser idempotente (`IF NOT EXISTS`/`IF EXISTS`) e
  reversível (script de rollback correspondente).
- Alterar `context.embedding.dimensions` (não recarregável) exige migração
  de **reindex online** (recriar índices `ivfflat`/`hnsw` com a nova
  dimensão) coordenada com janela de manutenção — DEVE ser tratada como
  mudança de infraestrutura, nunca hot-reload.
- Criação de partições futuras de `context_bundle` é migração recorrente
  automatizada (§3), não manual.
- Mudança em `CHECK` constraints (ex.: novo `source_kind` ou
  `compression_method`) exige RFC/ADR de módulo (consistência com
  `_DESIGN_BRIEF.md` §11) antes de aplicar.

---

## 7. Referências

- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.1, §5.7.
- Padrão de banco do AIOS: `../005-Database/Architecture.md`.
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §3.
- Superfície de API que lê/escreve estas tabelas: `./API.md`.
- Eventos publicados a partir do outbox: `./Events.md`.
- Chaves de configuração citadas (`context.embedding.dimensions`,
  `context.cache.vector_index`, `context.cache.l1_redis_ttl_seconds`,
  `context.cache.max_entries_per_tenant`, `context.ratelimit.*`):
  `./Configuration.md`.
- Modos de falha de Redis/PostgreSQL: `./FailureRecovery.md`.
- Modelo de sharding/particionamento de carga: `./Scalability.md`.
