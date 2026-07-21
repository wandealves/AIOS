---
Documento: Database
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0100, ADR-0101, ADR-0102, ADR-0106, ADR-0108, ADR-0109
RFCs relacionados: RFC-0001 (baseline), RFC-0100 (a propor)
Depende de: 003-RFC (RFC-0001), 005-Database, 021-Security, 025-Audit, 026-Cost-Optimizer, 027-Cluster, 040-Glossary
---

# 010-Memory — Database

> **Escopo.** Modelo físico de persistência do `MemoryService`: DDL
> PostgreSQL (fonte da verdade das camadas duráveis, com `pgvector` para
> busca vetorial ANN e Apache AGE para o grafo de conhecimento do agente),
> estruturas de estado quente em Redis e o esquema de blobs em MinIO.
> Convenções de URN, ULID e RLS por tenant **DEVEM** seguir
> [RFC-0001 §5.1](../003-RFC/RFC-0001-Architecture-Baseline.md) e o padrão
> geral de [`005-Database/Architecture.md`](../005-Database/Architecture.md).
> Consistência absoluta com `./_DESIGN_BRIEF.md` §3. Palavras normativas
> conforme RFC 2119/8174.

---

## 1. Justificativa de tecnologia

| Armazenamento | Papel | Por quê |
|----------------|-------|---------|
| **PostgreSQL** (relacional + `jsonb`) | Fonte da verdade de `MemoryItem` para as camadas duráveis (Long-Term, Semantic, Procedural, Episodic), proveniência de consolidação, cotas e outbox. | Transações ACID, RLS nativa por tenant, `jsonb` para `metadata`/`content` inline, particionamento declarativo. |
| **pgvector** (extensão do PostgreSQL) | Índice **HNSW** para busca vetorial ANN nas camadas Semantic e Episodic. | Coexiste no mesmo motor transacional que os metadados do item — evita um segundo *datastore* vetorial e a inconsistência eventual entre metadado e vetor (R3, ADR-0101). |
| **Apache AGE** (extensão do PostgreSQL, openCypher) | Nós/arestas do grafo de memória consolidada do agente (Knowledge Graph). | Mesma justificativa de coesão transacional; openCypher permite travessia de grafo sem sair do PostgreSQL, evitando um terceiro motor (Neo4j) só para este módulo (ADR-0100). O GraphRAG de domínio público/global é responsabilidade de 018/019, fora deste escopo (N4 do brief). |
| **Redis** | Estado **quente**: Working Memory (TTL curto), projeção quente de Short-Term, contadores de cota, locks de consolidação, `IdempotencyStore`. | Latência sub-ms exigida por NFR-001 (p99 ≤ 30 ms) e NFR-003 (p99 ≤ 50 ms para Working). |
| **MinIO** (S3-compatible) | Blobs grandes externalizados (`content_ref`), *snapshots* frios de `ConsolidationVersion`, conteúdo de itens `ARCHIVED`. | Custo/byte muito menor que PostgreSQL para conteúdo acima de `memory.item.inline_max_bytes`; *content-addressed* por `content_hash` habilita deduplicação (ADR-0106). |

> **Fronteira:** o Agent Runtime (Python, plano de dados) **NÃO DEVE** conectar
> diretamente a nenhum destes armazenamentos — todo acesso passa pela API do
> `MemoryService` (gRPC/NATS), conforme `_DESIGN_BRIEF.md` §1.2 e
> `../001-Architecture/Architecture.md` §6.

---

## 2. Esquema PostgreSQL — DDL

### 2.1 Schema e extensões

```sql
CREATE SCHEMA IF NOT EXISTS memory;

-- Extensões requeridas por este módulo (registro central: 005-Database).
CREATE EXTENSION IF NOT EXISTS vector;      -- pgvector: tipo `vector`, índice HNSW
CREATE EXTENSION IF NOT EXISTS age;         -- Apache AGE: grafo openCypher
CREATE EXTENSION IF NOT EXISTS pgcrypto;    -- utilidades de hash (sha256 via digest())

LOAD 'age';
SET search_path = ag_catalog, memory, "$user", public;
```

> **Nota sobre ULID:** IDs de negócio são ULID (RFC-0001 §5.1), armazenados
> como `CHAR(26)` (Base32 Crockford, ordenável por tempo de criação).
> `tenant_id` é `TEXT` (slug `[a-z0-9-]+`).

> **Nota sobre dimensão do vetor:** `memory.embedding.dim` (default `1024`,
> ver [Configuration.md](./Configuration.md)) fixa a dimensão da coluna
> `vector(D)`. A chave **não é recarregável** — mudar `D` exige migração com
> reindexação completa (`ALTER TABLE ... ALTER COLUMN embedding TYPE
> vector(D)` seguido de `REINDEX`), nunca uma alteração silenciosa em runtime
> (ADR-0101).

### 2.2 Tabela `memory.item` — entidade central (partição por camada)

A tabela reproduz fielmente a §3.1 do brief. É particionada por **LIST
(`layer`)** — cada camada durável tem sua própria partição física, permitindo
índices e políticas de manutenção específicos por camada (ex.: HNSW apenas
em Semantic/Episodic; particionamento temporal adicional apenas em
Episodic). Working e Short-Term **não residem aqui como fonte primária**
(vivem em Redis, §5) — a partição `item_short_term` existe apenas como
**projeção durável CQRS** da Short-Term Memory (brief §2.2, componente
`ShortTermMemoryStore`).

```sql
CREATE TABLE memory.item (
    id                      CHAR(26)      NOT NULL,
    tenant_id               TEXT          NOT NULL,
    agent_id                CHAR(26),
    session_id              CHAR(26),
    layer                   TEXT          NOT NULL
        CHECK (layer IN ('short_term','long_term','semantic','procedural','episodic')),
    kind                    TEXT          NOT NULL
        CHECK (kind IN ('fact','event','skill','observation','reflection','summary','edge')),
    state                   TEXT          NOT NULL DEFAULT 'INGESTED'
        CHECK (state IN ('INGESTED','ACTIVE','CONSOLIDATING','CONSOLIDATED',
                          'ARCHIVED','DECAYING','FORGET_PENDING','FORGOTTEN',
                          'PURGED','FAILED')),
    content                 JSONB,
    content_ref             TEXT,
    content_hash            BYTEA,
    embedding               vector(1024),                 -- D = memory.embedding.dim
    embedding_model         TEXT,
    salience                REAL          NOT NULL DEFAULT 0.5
        CHECK (salience BETWEEN 0 AND 1),
    decay_score             REAL          NOT NULL DEFAULT 1.0
        CHECK (decay_score BETWEEN 0 AND 1),
    access_count            BIGINT        NOT NULL DEFAULT 0 CHECK (access_count >= 0),
    last_access_at          TIMESTAMPTZ,
    source_urn              TEXT,
    consolidation_version   BIGINT,
    parent_ids              CHAR(26)[],
    legal_basis             TEXT          NOT NULL
        CHECK (legal_basis IN ('consent','contract','legitimate_interest','legal_obligation')),
    retention_class         TEXT          NOT NULL
        CHECK (retention_class IN ('ephemeral','standard','extended','legal_hold')),
    pii                     BOOLEAN       NOT NULL DEFAULT false,
    tags                    TEXT[],
    metadata                JSONB         NOT NULL DEFAULT '{}'::jsonb,
    expires_at              TIMESTAMPTZ,
    created_at              TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ   NOT NULL DEFAULT now(),
    tombstoned_at           TIMESTAMPTZ,

    CONSTRAINT pk_memory_item PRIMARY KEY (tenant_id, layer, id),
    CONSTRAINT chk_item_content_xor
        CHECK (content IS NOT NULL OR content_ref IS NOT NULL),
    CONSTRAINT chk_item_legal_hold_no_expiry
        CHECK (NOT (retention_class = 'legal_hold' AND expires_at IS NOT NULL))
) PARTITION BY LIST (layer);

CREATE TABLE memory.item_short_term  PARTITION OF memory.item FOR VALUES IN ('short_term');
CREATE TABLE memory.item_long_term   PARTITION OF memory.item FOR VALUES IN ('long_term');
CREATE TABLE memory.item_semantic    PARTITION OF memory.item FOR VALUES IN ('semantic');
CREATE TABLE memory.item_procedural  PARTITION OF memory.item FOR VALUES IN ('procedural');

-- Episodic é, adicionalmente, particionada por tempo (brief §3.3 e §10):
CREATE TABLE memory.item_episodic (LIKE memory.item INCLUDING ALL)
    PARTITION BY RANGE (created_at);
ALTER TABLE memory.item ATTACH PARTITION memory.item_episodic FOR VALUES IN ('episodic');

CREATE TABLE memory.item_episodic_2026_07
    PARTITION OF memory.item_episodic
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
```

**Índices comuns a todas as partições:**

```sql
-- Recall lexical/filtros — aplicado a cada partição (herdado via template);
-- reproduzido aqui para a partição semantic como exemplo canônico.
CREATE INDEX idx_item_semantic_tenant_agent
    ON memory.item_semantic (tenant_id, agent_id, state);
CREATE INDEX idx_item_semantic_last_access
    ON memory.item_semantic (tenant_id, last_access_at DESC);
CREATE INDEX idx_item_semantic_expires
    ON memory.item_semantic (expires_at) WHERE expires_at IS NOT NULL;
CREATE INDEX idx_item_semantic_tags_gin
    ON memory.item_semantic USING GIN (tags);
CREATE INDEX idx_item_semantic_content_hash
    ON memory.item_semantic (content_hash);
CREATE INDEX idx_item_semantic_state
    ON memory.item_semantic (tenant_id, state) WHERE state NOT IN ('PURGED');

-- Índice HNSW (ANN) — apenas nas partições com embedding relevante
-- (Semantic e Episodic). Parâmetros: memory.hnsw.m / memory.hnsw.ef_construction.
CREATE INDEX idx_item_semantic_embedding_hnsw
    ON memory.item_semantic
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 128);

CREATE INDEX idx_item_episodic_embedding_hnsw
    ON memory.item_episodic_2026_07
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 128);
CREATE INDEX idx_item_episodic_tenant_agent_time
    ON memory.item_episodic_2026_07 (tenant_id, agent_id, created_at DESC);

-- (Índices análogos DEVEM ser replicados nas demais partições —
--  short_term, long_term, procedural — omitindo o HNSW quando não há
--  embedding relevante, ex.: procedural tipicamente sem busca vetorial.)
```

**RLS (idêntica em todas as partições — aplicada uma vez na tabela-mãe e
herdada):**

```sql
ALTER TABLE memory.item ENABLE ROW LEVEL SECURITY;
ALTER TABLE memory.item FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_item ON memory.item
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

> Partições PostgreSQL não herdam automaticamente `ENABLE ROW LEVEL
> SECURITY` de forma implícita em todas as versões — a política DEVE ser
> replicada (ou herdada via `ONLY`/propagação de políticas do particionamento
> declarativo, conforme a versão do PostgreSQL em uso, ver
> `../005-Database/Architecture.md`). Migrações (§6) validam a presença da
> política em toda partição nova.

**Invariantes:** (i) toda linha carrega `tenant_id` com RLS ativo; (ii)
`content` xor `content_ref` — nunca ambos nulos (`chk_item_content_xor`);
(iii) item em `legal_hold` **não tem** `expires_at` (esquecimento por TTL não
se aplica — apenas RTBF autorizado, `chk_item_legal_hold_no_expiry`); (iv) a
dimensão de `embedding` DEVE casar com `memory.embedding.dim` vigente — item
com dimensão incompatível é rejeitado na escrita com `AIOS-MEM-0021` (ver
[API.md](./API.md) §5), nunca persistido; (v) mutação de `state` segue
estritamente a máquina de estados do brief §4.1 — a aplicação (não uma
constraint SQL) impõe as transições válidas, pois o grafo de transições
depende de contexto (cota, PDP, versão de consolidação).

### 2.3 Tabela `memory.layer_config`

```sql
CREATE TABLE memory.layer_config (
    tenant_id                  TEXT        NOT NULL,
    layer                      TEXT        NOT NULL
        CHECK (layer IN ('working','short_term','long_term','semantic',
                          'procedural','episodic','kg')),
    ttl_default                INTERVAL,
    max_items                  BIGINT      CHECK (max_items >= 0),
    max_bytes                  BIGINT      CHECK (max_bytes >= 0),
    decay_half_life            INTERVAL,
    consolidation_threshold    INTEGER     CHECK (consolidation_threshold >= 1),
    hnsw_m                     SMALLINT    CHECK (hnsw_m BETWEEN 8 AND 64),
    hnsw_ef_search             SMALLINT    CHECK (hnsw_ef_search BETWEEN 16 AND 512),
    updated_at                 TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT pk_layer_config PRIMARY KEY (tenant_id, layer)
);

ALTER TABLE memory.layer_config ENABLE ROW LEVEL SECURITY;
ALTER TABLE memory.layer_config FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_layer_config ON memory.layer_config
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

Espelha os *overrides* por tenant do namespace `memory.layer.*`/`memory.hnsw.*`
de [Configuration.md](./Configuration.md); ausência de linha usa o default
global.

### 2.4 Tabela `memory.consolidation_job`

```sql
CREATE TABLE memory.consolidation_job (
    id              CHAR(26)      NOT NULL,
    tenant_id       TEXT          NOT NULL,
    agent_id        CHAR(26),
    trigger         TEXT          NOT NULL
        CHECK (trigger IN ('learning','scheduled','manual','threshold')),
    from_layer      TEXT          NOT NULL,
    to_layer        TEXT          NOT NULL,
    state           TEXT          NOT NULL DEFAULT 'PENDING'
        CHECK (state IN ('PENDING','REJECTED','SNAPSHOTTING','RUNNING',
                          'VALIDATING','COMMITTED','ROLLING_BACK',
                          'ROLLED_BACK','FAILED')),
    version_id      BIGINT,
    started_at      TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ,
    stats           JSONB         NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),

    CONSTRAINT pk_consolidation_job PRIMARY KEY (tenant_id, id)
);

CREATE INDEX idx_consolidation_job_state
    ON memory.consolidation_job (tenant_id, state, created_at DESC);
CREATE INDEX idx_consolidation_job_agent
    ON memory.consolidation_job (tenant_id, agent_id, created_at DESC);

ALTER TABLE memory.consolidation_job ENABLE ROW LEVEL SECURITY;
ALTER TABLE memory.consolidation_job FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_consolidation_job ON memory.consolidation_job
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

Reflete a máquina de estados do brief §4.2. Uma `(tenant_id, agent_id)` NÃO
DEVE ter mais de um job em estado ativo (`SNAPSHOTTING`/`RUNNING`/
`VALIDATING`/`ROLLING_BACK`) simultaneamente — serializado por *lock*
distribuído em Redis (§5.4), não por constraint SQL.

### 2.5 Tabela `memory.consolidation_version`

```sql
CREATE TABLE memory.consolidation_version (
    version_id      BIGINT        GENERATED ALWAYS AS IDENTITY,
    tenant_id       TEXT          NOT NULL,
    agent_id        CHAR(26),
    snapshot_ref    TEXT          NOT NULL,     -- URI MinIO da pré-imagem
    parent_version  BIGINT,
    active          BOOLEAN       NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),

    CONSTRAINT pk_consolidation_version PRIMARY KEY (tenant_id, version_id),
    CONSTRAINT fk_consolidation_version_parent
        FOREIGN KEY (tenant_id, parent_version)
        REFERENCES memory.consolidation_version (tenant_id, version_id)
);

-- Apenas uma versão ativa por (tenant, agente) — imposto por índice único parcial.
CREATE UNIQUE INDEX uq_consolidation_version_active
    ON memory.consolidation_version (tenant_id, agent_id)
    WHERE active;

ALTER TABLE memory.consolidation_version ENABLE ROW LEVEL SECURITY;
ALTER TABLE memory.consolidation_version FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_consolidation_version ON memory.consolidation_version
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

**Uso:** toda transição para `CONSOLIDATING`/`SNAPSHOTTING` DEVE gravar uma
linha aqui **antes** de mutar qualquer `memory.item` (invariante do brief
§4.1(ii)). `rollback` desativa a versão corrente (`active=false`) e reativa
`parent_version` (`active=true`) na mesma transação. Retenção de versões
antigas governada por `memory.consolidation.version.retention` (default 10,
ver [Configuration.md](./Configuration.md)).

### 2.6 Tabela `memory.forgetting_policy`

```sql
CREATE TABLE memory.forgetting_policy (
    id          CHAR(26)      NOT NULL,
    tenant_id   TEXT          NOT NULL,
    scope       TEXT          NOT NULL
        CHECK (scope IN ('tenant','agent','layer')),
    strategy    TEXT          NOT NULL
        CHECK (strategy IN ('ttl','decay','lru','quota','rtbf')),
    params      JSONB         NOT NULL DEFAULT '{}'::jsonb,
    enabled     BOOLEAN       NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),

    CONSTRAINT pk_forgetting_policy PRIMARY KEY (tenant_id, id)
);

CREATE INDEX idx_forgetting_policy_enabled
    ON memory.forgetting_policy (tenant_id, scope, enabled);

ALTER TABLE memory.forgetting_policy ENABLE ROW LEVEL SECURITY;
ALTER TABLE memory.forgetting_policy FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_forgetting_policy ON memory.forgetting_policy
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

Lida pelo `RetentionScheduler`/`ForgettingEngine` a cada ciclo de avaliação;
precedência entre estratégias concorrentes é resolvida pela aplicação
(RTBF sempre vence sobre TTL/decay/LRU/quota, brief §12.3).

### 2.7 Tabela `memory.quota`

```sql
CREATE TABLE memory.quota (
    tenant_id    TEXT          NOT NULL,
    agent_id     CHAR(26)      NOT NULL DEFAULT '',   -- '' = cota agregada do tenant
    layer        TEXT          NOT NULL,
    limit_items  BIGINT        NOT NULL CHECK (limit_items >= 0),
    limit_bytes  BIGINT        NOT NULL CHECK (limit_bytes >= 0),
    used_items   BIGINT        NOT NULL DEFAULT 0 CHECK (used_items >= 0),
    used_bytes   BIGINT        NOT NULL DEFAULT 0 CHECK (used_bytes >= 0),
    updated_at   TIMESTAMPTZ   NOT NULL DEFAULT now(),

    CONSTRAINT pk_memory_quota PRIMARY KEY (tenant_id, agent_id, layer)
);

ALTER TABLE memory.quota ENABLE ROW LEVEL SECURITY;
ALTER TABLE memory.quota FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_quota ON memory.quota
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

**Uso:** réplica fria/durável de `sched:quota:*`-like counters mantidos
quentes em Redis (§5.2) pelo `QuotaManager`; PostgreSQL é o *fallback* de
reconciliação periódica e a fonte consultada por
`GET /v1/memory/stats` (ver [API.md](./API.md)).

### 2.8 Grafo de conhecimento — Apache AGE

```sql
SELECT create_graph('memory_graph');
```

> **Isolamento multi-tenant no grafo:** Apache AGE não possui RLS nativa por
> vértice/aresta. O `KnowledgeGraphAdapter` **DEVE** incluir `tenant_id` como
> propriedade obrigatória em todo vértice/aresta e **DEVE** filtrar toda
> consulta Cypher por `tenant_id` na cláusula `WHERE` — nunca confiar apenas
> na convenção de nomenclatura do grafo. Alternativa de um grafo
> `memory_graph_<tenant>` por tenant foi avaliada e **descartada** por
> multiplicar objetos de catálogo sem ganho de isolamento real (a aplicação
> já precisa validar `tenant_id` em toda leitura) — ver riscos em §7.

```cypher
-- Criação de vértice de memória consolidada (via KnowledgeGraphAdapter)
SELECT * FROM cypher('memory_graph', $$
  CREATE (n:MemoryNode {
    tenant_id: 'acme',
    item_urn: 'urn:aios:acme:memory:01J9Z8QCITEMULID0000000001',
    agent_id: '01J9Z8Q6H7K2M4N6P8R0S2T4V6',
    kind: 'fact',
    label: 'preferencia_faturamento'
  })
  RETURN n
$$) AS (n agtype);

-- Criação de aresta (KnowledgeEdge, brief §3.2)
SELECT * FROM cypher('memory_graph', $$
  MATCH (a:MemoryNode {tenant_id: 'acme', item_urn: 'urn:aios:acme:memory:0001'}),
        (b:MemoryNode {tenant_id: 'acme', item_urn: 'urn:aios:acme:memory:0002'})
  CREATE (a)-[r:RELATES_TO {tenant_id: 'acme', weight: 0.82}]->(b)
  RETURN r
$$) AS (r agtype);

-- Travessia para recall (mode=graph, API.md §6)
SELECT * FROM cypher('memory_graph', $$
  MATCH (n:MemoryNode {tenant_id: 'acme', agent_id: '01J9Z8Q6H7K2M4N6P8R0S2T4V6'})
        -[r:RELATES_TO*1..2]-(m:MemoryNode {tenant_id: 'acme'})
  RETURN m, r
  LIMIT 20
$$) AS (m agtype, r agtype);
```

`KnowledgeEdge` (brief §3.2: `src_id, dst_id, rel_type, tenant_id, weight,
properties`) mapeia diretamente para arestas openCypher acima —
`rel_type`→ tipo da aresta, `weight`/`properties` → propriedades. Índices
sobre propriedades (`tenant_id`, `item_urn`) DEVEM ser criados via
`CREATE PROPERTY INDEX` (sintaxe AGE) para sustentar NFR-003 (recall por
grafo, p99 ≤ 250 ms).

### 2.9 Tabela `memory.outbox_event`

```sql
CREATE TABLE memory.outbox_event (
    id          CHAR(26)      NOT NULL,
    tenant_id   TEXT          NOT NULL,
    subject     TEXT          NOT NULL,
    payload     JSONB         NOT NULL,
    status      TEXT          NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','published')),
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ,

    CONSTRAINT pk_outbox_event PRIMARY KEY (tenant_id, id)
);

CREATE INDEX idx_outbox_event_pending
    ON memory.outbox_event (created_at)
    WHERE status = 'pending';

ALTER TABLE memory.outbox_event ENABLE ROW LEVEL SECURITY;
ALTER TABLE memory.outbox_event FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_outbox_event ON memory.outbox_event
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

Toda mutação de `memory.item`/`memory.consolidation_job` que produz um
evento de domínio (ver [Events.md](./Events.md)) grava a linha correspondente
**na mesma transação**; o `OutboxPublisher` faz *polling* de `status=pending`
e publica no NATS/JetStream, marcando `published` (padrão *Outbox*, RPO de
evento = 0).

### 2.10 Tabela `memory.idempotency_record`

```sql
CREATE TABLE memory.idempotency_record (
    tenant_id          TEXT          NOT NULL,
    idempotency_key    TEXT          NOT NULL,
    operation          TEXT          NOT NULL,
    response_hash      BYTEA         NOT NULL,
    result             JSONB         NOT NULL,
    created_at         TIMESTAMPTZ   NOT NULL DEFAULT now(),
    expires_at         TIMESTAMPTZ   NOT NULL,

    CONSTRAINT pk_idempotency_record PRIMARY KEY (tenant_id, idempotency_key)
);

CREATE INDEX idx_idempotency_record_expires
    ON memory.idempotency_record (expires_at);

ALTER TABLE memory.idempotency_record ENABLE ROW LEVEL SECURITY;
ALTER TABLE memory.idempotency_record FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_idempotency_record ON memory.idempotency_record
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

Camada durável do `IdempotencyStore` (Redis é o caminho quente, ≥ 24h de
retenção mínima por RFC-0001 §5.5); PostgreSQL é *fallback* quando a chave
expira em Redis antes do TTL de negócio.

### 2.11 Diagrama relacional (ASCII)

```
┌──────────────────────────┐   FK (tenant,version_id)   ┌──────────────────────────┐
│ memory.consolidation_version│◀──────────────────────────│ memory.item              │
│ PK (tenant_id,version_id)│                            │ PK (tenant_id,layer,id)   │
│ snapshot_ref (MinIO)     │                            │ consolidation_version ────┼──┐
│ parent_version, active   │                            │ embedding vector(D)       │  │
└─────────────┬────────────┘                            │ content / content_ref    │  │
              │ 1                                        └──────────────┬────────────┘  │
              │ N (jobs referenciam a versão que criam)                 │ N              │
┌─────────────▼────────────┐                            ┌───────────────▼────────────┐  │
│ memory.consolidation_job │                            │ memory.layer_config        │  │
│ PK (tenant_id,id)        │                            │ PK (tenant_id,layer)        │  │
│ state (FSM §4.2 brief)   │                            │ ttl_default, hnsw_m/ef      │  │
│ from_layer,to_layer      │                            └────────────────────────────┘  │
└───────────────────────────┘                                                            │
                                                                                          │
┌──────────────────────────┐   ┌──────────────────────────┐   ┌──────────────────────┐  │
│ memory.forgetting_policy │   │ memory.quota              │   │ memory.outbox_event  │◀─┘
│ PK (tenant_id,id)        │   │ PK (tenant,agent,layer)   │   │ PK (tenant_id,id)     │
│ scope,strategy,params    │   │ limit_*/used_*            │   │ subject,payload,status│
└──────────────────────────┘   └──────────────────────────┘   └──────────────────────┘

┌──────────────────────────┐        Apache AGE (grafo separado, sem FK relacional)
│ memory.idempotency_record│        ┌────────────────────────────────────────────┐
│ PK (tenant,idem_key)     │        │ memory_graph: (MemoryNode)-[RELATES_TO]->  │
│ result, expires_at       │        │ tenant_id como propriedade obrigatória     │
└──────────────────────────┘        └────────────────────────────────────────────┘
```

---

## 3. Particionamento

| Tabela | Estratégia | Granularidade | Justificativa |
|--------|-----------|----------------|----------------|
| `memory.item` | `LIST (layer)` | 5 partições lógicas (short_term, long_term, semantic, procedural, episodic) | Isola manutenção (reindex HNSW, `VACUUM`) por camada; consultas de `recall` já filtram por `layer[]`, aproveitando *partition pruning*. |
| `memory.item_episodic` | `RANGE (created_at)`, sub-partição de `episodic` | Mensal | Episodic é a camada de maior volume/rotatividade (eventos); poda de partições antigas é `DETACH PARTITION` O(1) (brief §3.3, retenção 365d). |
| `memory.consolidation_job` / `_version` | Sem particionamento (volume moderado) | — | Volume ordens de magnitude menor que `item`; índice em `(tenant_id, state, created_at)` é suficiente. |
| `memory.outbox_event` | Sem particionamento; expurgo por job após `published` | — | Vida curta (drenada pelo `OutboxPublisher` em milissegundos a segundos). |
| `memory.idempotency_record` | Sem particionamento; expurgo por `expires_at` | — | Volume limitado pelo TTL (≥ 24h). |

**Sharding lógico (nível de aplicação, brief §10):** `shard =
hash(tenant_id, agent_id) mod N` determina o *shard* de banco/schema para
tenants de grande volume (vetores particionados por tenant); a distribuição
física entre *shards* (schemas/bancos distintos coordenados por
`027-Cluster`) é detalhada em [Scalability.md](./Scalability.md) — este
documento descreve o esquema lógico **por shard**.

Partições futuras (mês seguinte de `item_episodic`) DEVEM ser criadas com
antecedência mínima de 7 dias por job agendado (`pg_partman` ou equivalente);
ausência de partição futura é alarme de severidade alta (ver
[Monitoring.md](./Monitoring.md)).

---

## 4. Políticas de retenção e expurgo

### 4.1 Retenção por camada (alinhada à brief §3.3 e §8)

| Camada / tabela | TTL/retenção padrão | Mecanismo |
|------------------|----------------------|-----------|
| Working (Redis) | `memory.layer.working.ttl` (900s) | Expiração nativa Redis (`EXPIRE`); sem persistência em PostgreSQL. |
| Short-Term (`item_short_term`) | `memory.layer.short_term.ttl` (86400s) | `ForgettingEngine` marca `DECAYING→FORGET_PENDING` por job; Redis quente expira independentemente. |
| Long-Term (`item_long_term`) | `memory.layer.long_term.ttl` (90d, configurável/∞) | Job de decaimento (`decay_score`) + TTL explícito. |
| Semantic (`item_semantic`) | Sem TTL fixo; decaimento (`memory.decay.half_life`=30d) | Score de retenção decaído; sem expurgo por tempo absoluto. |
| Procedural (`item_procedural`) | Sem TTL | Skills não decaem por tempo; apenas por política explícita/RTBF. |
| Episodic (`item_episodic`) | `memory.layer.episodic.ttl` (365d) | Partição mensal expurgada (`DETACH`+`DROP`) após retenção + exportação fria (auditoria 025). |
| Knowledge Graph (AGE) | Sem TTL | Poda por política explícita (grau de conectividade/saliência) via `ForgettingEngine`. |
| Blobs (MinIO) | Alinhado à `retention_class` do item referenciador | Removido junto ao `PURGED` do item (§4.3). |
| `outbox_event` | Minutos (drenagem) | `DELETE` após `published_at` + margem de segurança (24h) para auditoria de curto prazo. |
| `idempotency_record` | `expires_at` (≥ 24h) | Job periódico de expurgo (§4.2). |

### 4.2 Job de expurgo — exemplos

```sql
-- Idempotency: executado periodicamente (ex.: a cada 15 min).
DELETE FROM memory.idempotency_record
WHERE expires_at < now()
LIMIT 10000;

-- Outbox: remove eventos já publicados há mais de 24h (retenção de auditoria curta).
DELETE FROM memory.outbox_event
WHERE status = 'published' AND published_at < now() - INTERVAL '24 hours'
LIMIT 10000;

-- Poda de itens elegíveis por decaimento (ForgettingEngine, brief §4.1):
UPDATE memory.item_semantic
SET state = 'FORGET_PENDING', updated_at = now()
WHERE tenant_id = $1
  AND state = 'DECAYING'
  AND decay_score < $2               -- memory.decay.threshold
  AND retention_class <> 'legal_hold';
```

### 4.3 Direito ao esquecimento (LGPD/GDPR, RTBF)

Expurgo físico é **assíncrono** e coordenado com auditoria (025):

```sql
BEGIN;

-- 1. Marca tombstone (idempotente; item já FORGOTTEN é no-op)
UPDATE memory.item_semantic
SET state = 'FORGOTTEN', tombstoned_at = now(), updated_at = now()
WHERE tenant_id = $1 AND agent_id = $2 AND state <> 'PURGED';

-- 2. (Após grace period, job separado) purge físico:
--    remove vetor (coluna embedding), referência de blob e a linha,
--    preservando apenas metadados mínimos de auditoria em 025-Audit
--    (fora deste schema — ver 025-Audit/Database.md).
UPDATE memory.item_semantic
SET state = 'PURGED', embedding = NULL, content = NULL, content_ref = NULL,
    updated_at = now()
WHERE tenant_id = $1 AND agent_id = $2 AND state = 'FORGOTTEN'
  AND tombstoned_at < now() - INTERVAL '7 days';  -- memory.forget.grace_period

COMMIT;
```

`legal_hold` **bloqueia** a etapa 1 salvo quando a operação é RTBF
explicitamente autorizado (`AIOS-MEM-0050` caso contrário, ver
[API.md](./API.md) §5.2). Blob correspondente em MinIO é removido por job
separado do `BlobStoreAdapter`, correlacionado por `content_hash`.

---

## 5. Redis — Estruturas de Estado Quente

> Fora do escopo de DDL relacional, mas **contratual** ao módulo — Working
> Memory e o caminho quente de cotas/locks dependem inteiramente destas
> estruturas. Prefixo de chave: `mem`.

### 5.1 Working Memory — `mem:working:{tenant}:{agent}:{session}` (HASH/STRING + TTL)

| Campo | Tipo | Notas |
|-------|------|-------|
| Chave | `mem:working:{tenant}:{agent}:{session}:{item_id}` | TTL = `memory.layer.working.ttl` (default 900s). |
| Valor | `STRING` (JSON serializado do item) ou `HASH` (campos individuais) | Sem persistência em PostgreSQL — perda tolerada (NFR-008). |
| Índice auxiliar | `mem:working:index:{tenant}:{agent}:{session}` (`SET` de `item_id`) | Suporta enumeração/`recall` restrito à sessão sem `SCAN`. |

```
SET mem:working:acme:01J9..agent:01J9..sess:01J9..item "{...json...}" EX 900
SADD mem:working:index:acme:01J9..agent:01J9..sess 01J9..item
```

### 5.2 Cotas — `mem:quota:*` (contadores atômicos)

| Chave | Tipo | Notas |
|-------|------|-------|
| `mem:quota:{tenant}:{agent}:{layer}:items` | `INT` (atômico, script Lua) | Espelha `used_items` de `memory.quota`; reconciliado periodicamente com PostgreSQL. |
| `mem:quota:{tenant}:{agent}:{layer}:bytes` | `INT` | Idem para bytes. |
| `mem:inflight` | `INT` (global) | Escritas em voo; comparado a `memory.backpressure.max_inflight`. |

Incremento/decremento via script Lua (CAS atômico) para impedir *overcommit*
antes da confirmação em PostgreSQL — mesma técnica de reserva/confirmação
descrita em `009-Scheduler/Database.md` §5.2, adaptada a `remember`.

### 5.3 Locks de consolidação — `mem:lock:consolidation:{tenant}:{agent}`

| Chave | Tipo | Notas |
|-------|------|-------|
| `mem:lock:consolidation:{tenant}:{agent}` | `STRING` + TTL (lease) | Serializa `ConsolidationJob`s concorrentes para o mesmo agente (brief §10); TTL de segurança alinhado ao tempo máximo esperado de um job. |

### 5.4 Idempotência quente — `mem:idem:*`

| Chave | Tipo | Notas |
|-------|------|-------|
| `mem:idem:{tenant}:{idempotency_key}` | `STRING` (JSON) + TTL | Espelha `memory.idempotency_record`; TTL ≥ 24h (RFC-0001 §5.5). *Miss* cai para consulta em PostgreSQL antes de tratar como chave nova. |

---

## 6. MinIO — Esquema de Blobs

| Aspecto | Regra |
|---------|-------|
| Bucket | `memory.blob.bucket` (default `aios-memory`). |
| Chave de objeto | `{tenant_id}/{yyyy}/{mm}/{content_hash}.bin` — *content-addressed*; dois itens com conteúdo idêntico compartilham objeto (deduplicação). |
| Metadados do objeto | `x-amz-meta-tenant-id`, `x-amz-meta-item-urn`, `x-amz-meta-retention-class` (espelham colunas de `memory.item` para auditoria independente do PostgreSQL). |
| Snapshots de consolidação | `{tenant_id}/consolidation/{version_id}.snapshot.json.gz` — pré-imagem referenciada por `consolidation_version.snapshot_ref`. |
| Ciclo de vida | Regra de expiração do bucket alinhada à `retention_class` mais permissiva presente (aplicação garante que a remoção do objeto só ocorre quando **nenhum** item ativo referencia o `content_hash`, via contagem de referência). |

---

## 7. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-----------------|--------------------------|
| Apache AGE sem RLS nativa pode vazar dados cross-tenant se a aplicação esquecer o filtro `tenant_id`. | `KnowledgeGraphAdapter` centraliza toda consulta Cypher com `tenant_id` obrigatório; teste de fuzz de tenant cobre o grafo (NFR-010). Alternativa (`create_graph` por tenant) descartada por explosão de catálogo em milhares de tenants. |
| `vector(D)` fixo por coluna torna `memory.embedding.dim` caro de mudar. | Documentado como não recarregável; procedimento de migração com reindex online descrito em [Configuration.md](./Configuration.md) §5. Alternativa (dimensão variável via `jsonb`) descartada — perderia o índice HNSW nativo do pgvector. |
| Particionamento por `LIST(layer)` + `RANGE(created_at)` em Episodic aumenta complexidade de migração. | Automação via `pg_partman`; testes de presença de partição futura no CI de deployment. |
| Reconciliação Redis↔PostgreSQL de cotas pode divergir sob falha. | Job de reconciliação periódica (`QuotaManager`); PostgreSQL é sempre a fonte de verdade final; Redis é otimista. |
| Deduplicação de blob por `content_hash` mantém objeto órfão se a contagem de referência falhar. | Job de *garbage collection* de blobs sem referência ativa, auditado antes de `DELETE` físico. |

---

## 8. Migrações

- Ferramenta: migrações versionadas (EF Core Migrations/Flyway, a definir em
  [Deployment.md](./Deployment.md)), nomeadas `V<NNNN>__<descricao>.sql`,
  aplicadas de forma **aditiva e retrocompatível** (nunca `DROP COLUMN`/
  `ALTER TYPE` destrutivo sem janela de coexistência ≥ 2 versões, espelhando
  RFC-0001 §5.7).
- Toda migração DEVE ser idempotente (`IF NOT EXISTS`/`IF EXISTS`) e
  reversível (script de rollback correspondente).
- Mudança em `memory.embedding.dim`/`memory.hnsw.*` (não recarregáveis) É
  uma migração formal: `ALTER COLUMN ... TYPE vector(D)` seguido de
  `REINDEX CONCURRENTLY` — executada fora de horário de pico, com
  monitoramento de `aios_memory_ann_search_duration_ms` durante o processo.
- Criação de partições futuras de `item_episodic` é migração recorrente
  automatizada (§3), não manual.
- Alteração de `CHECK` constraints (ex.: novo `kind` ou `layer`) exige
  RFC/ADR de módulo (consistência com `_DESIGN_BRIEF.md` §11) antes de
  aplicar.
- Criação/alteração de grafo AGE (`memory_graph`) segue o mesmo pipeline de
  migração, com scripts Cypher versionados junto ao SQL.

---

## 9. Referências

- Contratos centrais: [RFC-0001 §5.1, §5.7](../003-RFC/RFC-0001-Architecture-Baseline.md).
- Padrão de banco do AIOS: [`005-Database/Architecture.md`](../005-Database/Architecture.md).
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §3.
- Superfície de API que lê/escreve estas tabelas: [API.md](./API.md).
- Eventos publicados a partir do outbox: [Events.md](./Events.md).
- Chaves de configuração citadas (`memory.embedding.dim`, `memory.hnsw.*`,
  `memory.layer.*.ttl`, `memory.decay.*`, `memory.forget.grace_period`,
  `memory.blob.bucket`): [Configuration.md](./Configuration.md).
- Modos de falha de Redis/PostgreSQL/AGE/MinIO: [FailureRecovery.md](./FailureRecovery.md).
- Modelo de sharding/particionamento de carga: [Scalability.md](./Scalability.md).
