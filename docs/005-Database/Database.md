---
Documento: Database
Módulo: 005-Database
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 005-Database
ADRs relacionados: ADR-0005, ADR-0050, ADR-0051, ADR-0052, ADR-0053, ADR-0054, ADR-0055, ADR-0057, ADR-0058
RFCs relacionados: RFC-0001, RFC-0050, RFC-0051
Depende de: _DESIGN_BRIEF.md §3, StateMachine.md, Security.md, Scalability.md
---

# 005-Database — Modelo Físico

Este documento é o **modelo físico consolidado** do AIOS: as convenções obrigatórias
para qualquer tabela do sistema e o DDL das entidades do próprio módulo (schema
`platform`). Os schemas de domínio pertencem aos módulos donos e são aqui apenas
**catalogados** — ver `./_DESIGN_BRIEF.md` §1.3 NR-01.

Plataforma: **PostgreSQL 16** com as extensões **`pgvector`** e **`age`** (Apache AGE),
conforme ADR-0005.

---

## 1. Mapa de Schemas

| Schema | Módulo dono | Conteúdo típico |
|--------|-------------|-----------------|
| `platform` | **005-Database** | Catálogo, migrações, políticas de partição/retenção, backups, expurgo. |
| `kernel` | `../006-Kernel/` | ACB, cotas, `syscall_log`, outbox. |
| `runtime` | `../007-Agent-Runtime/`, `../008-Agent-Lifecycle/` | Instâncias, gerações, checkpoints (ponteiros). |
| `sched` | `../009-Scheduler/` | Filas, reservas de slot, decisões de admissão. |
| `memory` | `../010-Memory/` | Itens de memória e embeddings (`vector`). |
| `context` | `../011-Context/` | Janelas de contexto, cache semântico. |
| `cognition` | `../012-Planning/`, `../013-Goals/`, `../014-Workflow/` | Planos, objetivos, execuções de workflow. |
| `tools` | `../015-Tool-Manager/`, `../016-Plugin-System/` | Registro de ferramentas, plugins, invocações. |
| `models` | `../017-Model-Router/` | Catálogo de modelos, roteamentos, custos por chamada. |
| `knowledge` | `../018-Knowledge/` | Documentos, *chunks*, embeddings. |
| `graph` | `../019-GraphRAG/` | Grafos AGE (vértices, arestas, propriedades). |
| `policy` | `../022-Policy/` | Bundles, bindings, decisões cacheadas. |
| `learning` | `../023-Learning/` | Amostras, avaliações, versões de política aprendida. |
| `audit` | `../025-Audit/` | Trilha imutável encadeada. |
| `cost` | `../026-Cost-Optimizer/` | Orçamentos, consumo agregado. |

Cada schema tem uma *role* própria (`<schema>_rw`) com privilégio mínimo. Nenhum
serviço de runtime é dono das tabelas nem possui `CREATE` — apenas a *role*
`platform_ddl`, usada pelo `MigrationEngine` durante uma migração autorizada.

---

## 2. Convenções Físicas Obrigatórias (normativas)

Toda tabela do AIOS **DEVE** obedecer às regras abaixo; o `DdlConventionValidator`
rejeita a migração que as viole (FR-002, `AIOS-MIGRATION-0002`).

| # | Convenção | Justificativa |
|---|-----------|---------------|
| CV-01 | Chave primária `urn text` no formato `urn:aios:<tenant>:<tipo>:<ULID>` (RFC-0001 §5.1), ou `id char(26)` (ULID puro) em tabelas de alto volume. | Identidade estável, ordenável por tempo, sem colisão entre tenants. |
| CV-02 | Tabela multi-tenant **DEVE** ter `tenant_id text NOT NULL` e política **RLS**. | Isolamento independente do código da aplicação (NFR-011). |
| CV-03 | Mutações concorrentes **DEVEM** usar `version bigint NOT NULL DEFAULT 0` (OCC). | Evita locks pessimistas no caminho quente (NFR-012). |
| CV-04 | Colunas temporais **DEVEM** ser `timestamptz` em UTC; `created_at` é obrigatória. | RFC 3339/UTC em todo o sistema; base de particionamento e retenção. |
| CV-05 | Toda FK **DEVE** ter índice na coluna referenciante. | Evita varredura em `DELETE`/`UPDATE` do lado referenciado. |
| CV-06 | Nomes em `snake_case`; tabelas no singular; sem abreviação obscura. | Legibilidade e previsibilidade de scripts. |
| CV-07 | Tabela multi-tenant **DEVE** declarar política de retenção (`retention_ref`). | Nenhum dado cresce para sempre por omissão. |
| CV-08 | Tabela classificada `pii` **DEVE** ter `legal_basis` na política de retenção. | LGPD art. 7º / GDPR art. 6º (`AIOS-DB-0009`). |
| CV-09 | Colunas `vector(N)` **DEVEM** estar registradas em `platform.vector_index` com `N` fixo. | Dimensão imutável; recall mensurável (NFR-006). |
| CV-10 | É **PROIBIDO** em migração: `DISABLE ROW LEVEL SECURITY`, `SECURITY DEFINER`, `GRANT`/`ALTER ROLE`, `SET search_path` global. | Vetores de escalação de privilégio (`./Security.md` §5). |
| CV-11 | DDL bloqueante estimado acima de `db.migration.max_block_ms` (200 ms) **NÃO DEVE** ser aplicado; reescreva como expand/contract. | NFR-010. |
| CV-12 | Índices criados/reconstruídos em produção **DEVEM** usar `CONCURRENTLY`. | Não bloquear escrita durante manutenção. |

---

## 3. DDL do Schema `platform`

### 3.1 Preparação

```sql
CREATE SCHEMA IF NOT EXISTS platform;
CREATE EXTENSION IF NOT EXISTS vector;   -- pgvector (010, 011, 018)
CREATE EXTENSION IF NOT EXISTS age;      -- Apache AGE (019)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Variável de sessão usada pelas políticas de RLS
-- (definida pelo ConnectionPoolGateway / pela aplicação ao abrir a sessão)
--   SET app.tenant_id = 'acme';
```

### 3.2 `platform.schema_registry`

```sql
CREATE TABLE platform.schema_registry (
    urn                text        PRIMARY KEY,
    schema_name        text        NOT NULL,
    table_name         text        NOT NULL,
    owner_module       text        NOT NULL,
    multi_tenant       boolean     NOT NULL,
    rls_enabled        boolean     NOT NULL DEFAULT false,
    data_class         text        NOT NULL
                       CHECK (data_class IN ('public','internal','confidential','pii')),
    partition_strategy text        NOT NULL DEFAULT 'none'
                       CHECK (partition_strategy IN ('none','range_time','hash_tenant')),
    retention_ref      uuid        REFERENCES platform.retention_policy(id),
    schema_version     text        NOT NULL,
    row_estimate       bigint      NOT NULL DEFAULT 0,
    bytes_estimate     bigint      NOT NULL DEFAULT 0,
    version            bigint      NOT NULL DEFAULT 0,
    created_at         timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_schema_table UNIQUE (schema_name, table_name),
    CONSTRAINT ck_multitenant_requires_rls
        CHECK (NOT multi_tenant OR (rls_enabled AND retention_ref IS NOT NULL))
);

CREATE INDEX ix_schema_registry_owner ON platform.schema_registry (owner_module);
CREATE INDEX ix_schema_registry_class ON platform.schema_registry (data_class);
```

> `ck_multitenant_requires_rls` codifica no próprio banco o invariante C-01 de
> `./ClassDiagrams.md`: uma tabela multi-tenant sem RLS e sem retenção **não pode
> sequer ser catalogada**.

### 3.3 `platform.migration`

```sql
CREATE TABLE platform.migration (
    urn                  text        PRIMARY KEY,
    version              text        NOT NULL UNIQUE,
    owner_module         text        NOT NULL,
    state                text        NOT NULL
                         CHECK (state IN ('Draft','Validated','DryRun','Applying',
                                          'Applied','Failed','RolledBack','Superseded')),
    phase                text        NOT NULL
                         CHECK (phase IN ('expand','migrate','contract')),
    up_sql_digest        text        NOT NULL,
    down_sql_digest      text,
    reversible           boolean     NOT NULL DEFAULT false,
    blocking_estimate_ms integer     NOT NULL DEFAULT 0,
    dry_run_at           timestamptz,
    applied_at           timestamptz,
    duration_ms          integer,
    applied_by           text        NOT NULL,
    failure_code         text,
    superseded_by        text        REFERENCES platform.migration(urn),
    occ_version          bigint      NOT NULL DEFAULT 0,
    created_at           timestamptz NOT NULL DEFAULT now(),
    updated_at           timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT ck_contract_not_reversible
        CHECK (NOT (phase = 'contract' AND reversible AND down_sql_digest IS NULL))
);

CREATE INDEX ix_migration_state ON platform.migration (state);
CREATE INDEX ix_migration_owner ON platform.migration (owner_module, applied_at DESC);
CREATE INDEX ix_migration_superseded ON platform.migration (superseded_by);
```

Aplicação sob lock global (invariante I1):

```sql
-- 005 é o namespace de lock reservado ao módulo Database
SELECT pg_advisory_lock(5, 1);      -- bloqueante, com lock_timeout configurado
-- … BEGIN; DDL; UPDATE platform.schema_registry; INSERT platform.outbox; COMMIT; …
SELECT pg_advisory_unlock(5, 1);
```

### 3.4 `platform.retention_policy`

```sql
CREATE TABLE platform.retention_policy (
    id           uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    name         text        NOT NULL UNIQUE,
    ttl          interval    NOT NULL,
    basis_column text        NOT NULL DEFAULT 'created_at',
    method       text        NOT NULL
                 CHECK (method IN ('drop_partition','delete_batch','tokenize')),
    legal_hold   boolean     NOT NULL DEFAULT false,
    legal_basis  text,
    batch_size   integer     NOT NULL DEFAULT 10000 CHECK (batch_size BETWEEN 100 AND 1000000),
    version      bigint      NOT NULL DEFAULT 0,
    created_at   timestamptz NOT NULL DEFAULT now(),
    updated_at   timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX ix_retention_hold ON platform.retention_policy (legal_hold) WHERE legal_hold;
```

### 3.5 `platform.partition_policy`

```sql
CREATE TABLE platform.partition_policy (
    id                 uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    schema_object_urn  text        NOT NULL UNIQUE
                       REFERENCES platform.schema_registry(urn) ON DELETE CASCADE,
    strategy           text        NOT NULL CHECK (strategy IN ('range_time','hash_tenant')),
    "interval"         interval,
    hash_modulus       integer     CHECK (hash_modulus > 0),
    precreate_ahead    integer     NOT NULL DEFAULT 7  CHECK (precreate_ahead BETWEEN 1 AND 90),
    detach_before_drop boolean     NOT NULL DEFAULT true,
    version            bigint      NOT NULL DEFAULT 0,
    created_at         timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT ck_strategy_params CHECK (
        (strategy = 'range_time'  AND "interval" IS NOT NULL AND hash_modulus IS NULL) OR
        (strategy = 'hash_tenant' AND hash_modulus IS NOT NULL AND "interval" IS NULL)
    )
);
```

### 3.6 `platform.erasure_request` (multi-tenant, com RLS)

```sql
CREATE TABLE platform.erasure_request (
    urn             text        PRIMARY KEY,
    tenant_id       text        NOT NULL,
    subject_urn     text        NOT NULL,
    scope           text        NOT NULL CHECK (scope IN ('subject','tenant')),
    status          text        NOT NULL
                    CHECK (status IN ('received','authorized','executing','completed','rejected')),
    tables_affected jsonb       NOT NULL DEFAULT '[]'::jsonb,
    rows_erased     bigint      NOT NULL DEFAULT 0,
    receipt_hash    text,
    idempotency_key text,
    requested_at    timestamptz NOT NULL DEFAULT now(),
    completed_at    timestamptz,
    CONSTRAINT uq_erasure_idem UNIQUE (tenant_id, idempotency_key)
);

CREATE INDEX ix_erasure_tenant_status ON platform.erasure_request (tenant_id, status);

ALTER TABLE platform.erasure_request ENABLE ROW LEVEL SECURITY;
ALTER TABLE platform.erasure_request FORCE ROW LEVEL SECURITY;

CREATE POLICY p_erasure_tenant_isolation ON platform.erasure_request
    USING       (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK  (tenant_id = current_setting('app.tenant_id', true));
```

### 3.7 `platform.backup_catalog`

```sql
CREATE TABLE platform.backup_catalog (
    urn         text        PRIMARY KEY,
    kind        text        NOT NULL CHECK (kind IN ('full','incremental','wal_segment')),
    object_uri  text        NOT NULL,
    lsn_start   pg_lsn      NOT NULL,
    lsn_end     pg_lsn,
    size_bytes  bigint      NOT NULL,
    checksum    text        NOT NULL,
    verified_at timestamptz,
    expires_at  timestamptz NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX ix_backup_kind_time ON platform.backup_catalog (kind, created_at DESC);
CREATE INDEX ix_backup_expires   ON platform.backup_catalog (expires_at);
```

### 3.8 `platform.vector_index`

```sql
CREATE TABLE platform.vector_index (
    id                uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    schema_object_urn text        NOT NULL REFERENCES platform.schema_registry(urn) ON DELETE CASCADE,
    column_name       text        NOT NULL,
    dimensions        integer     NOT NULL CHECK (dimensions BETWEEN 1 AND 16000),
    index_type        text        NOT NULL CHECK (index_type IN ('hnsw','ivfflat','none')),
    distance_op       text        NOT NULL CHECK (distance_op IN ('cosine','l2','inner_product')),
    params            jsonb       NOT NULL DEFAULT '{}'::jsonb,
    target_recall     numeric(4,3) NOT NULL DEFAULT 0.950,
    last_reindex_at   timestamptz,
    version           bigint      NOT NULL DEFAULT 0,
    CONSTRAINT uq_vector_column UNIQUE (schema_object_urn, column_name)
);
```

### 3.9 Contrato canônico da tabela `outbox`

Instanciada por cada módulo em seu próprio schema (ex.: `kernel.outbox`):

```sql
CREATE TABLE <schema>.outbox (
    id         char(26)    PRIMARY KEY,          -- ULID = event.id (RFC-0001 §5.2)
    tenant_id  text        NOT NULL,
    subject    text        NOT NULL,             -- RFC-0001 §5.3
    payload    jsonb       NOT NULL,             -- envelope CloudEvents completo
    published  boolean     NOT NULL DEFAULT false,
    created_at timestamptz NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Índice parcial: a varredura do relay é O(pendentes), não O(histórico)
CREATE INDEX ix_outbox_pending ON <schema>.outbox (created_at)
    WHERE published = false;

ALTER TABLE <schema>.outbox ENABLE ROW LEVEL SECURITY;
CREATE POLICY p_outbox_tenant ON <schema>.outbox
    USING (tenant_id = current_setting('app.tenant_id', true));
```

Retenção default: 7 dias, método `drop_partition`.

---

## 4. Padrão de RLS (aplicável a toda tabela multi-tenant)

```sql
ALTER TABLE <schema>.<tabela> ENABLE ROW LEVEL SECURITY;
ALTER TABLE <schema>.<tabela> FORCE  ROW LEVEL SECURITY;   -- vale até para o dono

CREATE POLICY p_<tabela>_tenant_isolation ON <schema>.<tabela>
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

- `FORCE` é obrigatório: sem ele, o dono da tabela ignora a política.
- `WITH CHECK` impede **escrever** linha de outro tenant, não apenas lê-la.
- A ausência de `app.tenant_id` na sessão resulta em **zero linhas** (falha fechada),
  nunca em acesso irrestrito.
- Consultas de plataforma que legitimamente cruzam tenants usam a *role*
  `platform_admin`, com `BYPASSRLS`, cujo uso é auditado em `../025-Audit/`.

---

## 5. Particionamento

### 5.1 RANGE por tempo (séries)

```sql
CREATE TABLE kernel.syscall_log (
    id         char(26)    NOT NULL,
    tenant_id  text        NOT NULL,
    created_at timestamptz NOT NULL,
    /* … demais colunas … */
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE kernel.syscall_log_2026_07_22 PARTITION OF kernel.syscall_log
    FOR VALUES FROM ('2026-07-22') TO ('2026-07-23');
```

O `PartitionManager` mantém `precreate_ahead` partições futuras (default 7) — a
ausência de partição futura significa **falha de escrita** ao virar o período, por
isso a criação é monitorada como um alerta P1 (UC-007 E2).

### 5.2 HASH por tenant (alta cardinalidade)

```sql
CREATE TABLE memory.item (
    urn       text NOT NULL,
    tenant_id text NOT NULL,
    /* … */
    PRIMARY KEY (urn, tenant_id)
) PARTITION BY HASH (tenant_id);

CREATE TABLE memory.item_p00 PARTITION OF memory.item
    FOR VALUES WITH (MODULUS 16, REMAINDER 0);
-- … p01 … p15
```

Alterar `MODULUS` exige migração de dados; por isso `hash_modulus` é tratado como
decisão de longo prazo (potência de 2, com folga de crescimento).

---

## 6. Índices Vetoriais (`pgvector`)

```sql
ALTER TABLE memory.item ADD COLUMN embedding vector(1536);

CREATE INDEX CONCURRENTLY ix_memory_item_embedding
    ON memory.item USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- Busca top-K (K=10) com esforço ajustável por sessão
SET hnsw.ef_search = 64;
SELECT urn, embedding <=> $1 AS distance
  FROM memory.item
 WHERE tenant_id = current_setting('app.tenant_id', true)
 ORDER BY embedding <=> $1
 LIMIT 10;
```

| Parâmetro | Default | Efeito |
|-----------|---------|--------|
| `m` = 16 | conectividade do grafo HNSW | maior ⇒ melhor recall, índice maior |
| `ef_construction` = 64 | qualidade de construção | maior ⇒ build mais lento, melhor recall |
| `hnsw.ef_search` = 64 | esforço de busca | maior ⇒ melhor recall, maior latência |

Meta: **p99 ≤ 50 ms** para top-10 com ≥ 10⁷ vetores (NFR-003) e **Recall@10 ≥ 0,95**
(NFR-006), medido contra busca exata amostrada. `dimensions` é **imutável**
(`AIOS-DB-0011`): trocar o modelo de embedding exige nova coluna e migração.

---

## 7. Grafo (Apache AGE)

```sql
LOAD 'age';
SET search_path = ag_catalog, "$user", public;

SELECT create_graph('graph_acme');           -- um grafo por tenant

SELECT * FROM cypher('graph_acme', $$
    MATCH (d:Document)-[:MENTIONS*1..3]->(e:Entity)
    WHERE d.urn = $urn
    RETURN e.name
$$) AS (name agtype);
```

- Isolamento: **um grafo por tenant** (`graph_<tenant>`), com acesso concedido apenas
  à *role* correspondente — equivalente funcional da RLS para vértices e arestas (FR-014).
- Profundidade de travessia limitada por `db.graph.max_traversal_depth` (default 6):
  travessia irrestrita é vetor de DoS (`./Security.md` §5).
- A **ontologia** (quais *labels* e relações existem) pertence a `../019-GraphRAG/`.

---

## 8. Migrações

| Aspecto | Regra |
|---------|-------|
| Nomenclatura | `YYYYMMDDHHMMSS_<slug>` (ordenável lexicograficamente). |
| Direção | **Forward-only**; rollback apenas para migrações declaradamente reversíveis. |
| Padrão | **Expand → Migrate → Contract** em migrações separadas, permitindo coexistência de versões de aplicação (RFC-0001 §5.7). |
| Atomicidade | Uma transação por migração (DDL transacional do PostgreSQL, invariante I2). |
| Exclusão mútua | `pg_advisory_lock(5,1)` global (invariante I1). |
| Ensaio | `dry-run` obrigatório em réplica quando `db.migration.require_dry_run = true`. |

Exemplo de expand/contract para renomear `memory.item.body` → `content`:

```sql
-- 20260722T090000_expand_memory_content  (phase = expand, reversible = true)
ALTER TABLE memory.item ADD COLUMN content text;
-- aplicação passa a escrever nas DUAS colunas e a ler de content com fallback

-- 20260723T090000_migrate_memory_content (phase = migrate)
UPDATE memory.item SET content = body WHERE content IS NULL;  -- em lotes

-- 20260801T090000_contract_memory_body   (phase = contract, reversible = false)
ALTER TABLE memory.item DROP COLUMN body;
```

Só a terceira migração é destrutiva, e ela ocorre **depois** de toda instância da
aplicação já ter sido atualizada.

---

## 9. Retenção e Expurgo

| Tabela (exemplo) | `data_class` | TTL | Método |
|------------------|--------------|-----|--------|
| `kernel.syscall_log` | `internal` | 90 dias | `drop_partition` |
| `<schema>.outbox` | `internal` | 7 dias | `drop_partition` |
| `memory.item` | `pii` (quando aplicável) | conforme base legal | `delete_batch` / `tokenize` |
| `audit.entry` | `confidential` | 7 anos | `drop_partition` (sob `legal_hold` quando aplicável) |
| `platform.migration` | `internal` | permanente | — (histórico imutável) |
| `platform.backup_catalog` | `internal` | `db.backup.retention_days` (35) | `delete_batch` |

`legal_hold` suspende **qualquer** expurgo, inclusive automático (FR-010): a obrigação
de preservar prevalece sobre a de apagar, e o conflito é registrado.

---

## 10. Backup, WAL e PITR

```
   primário ──WAL contínuo (archive_timeout = 60s)──▶ MinIO s3://aios-backups/wal/
        │
        ├── backup full a cada 24h ─────────────────▶ s3://aios-backups/full/
        └── incremental entre fulls ────────────────▶ s3://aios-backups/incr/

   janela de PITR = [lsn_start do full mais antigo retido .. último WAL arquivado]
```

- `synchronous_commit = on` com réplica síncrona no quórum → **perda zero** de
  transação commitada (NFR-008).
- `archive_timeout = 60 s` limita o RPO real; a meta contratual é **RPO ≤ 5 min**,
  com alvo operacional ≤ 1 min (NFR-009).
- Backup **não verificado** por *restore drill* não conta para o plano de recuperação
  (FR-012, invariante C-06).

---

## 11. Parâmetros do PostgreSQL (baseline)

| Parâmetro | Valor | Motivo |
|-----------|-------|--------|
| `synchronous_commit` | `on` | Durabilidade (NFR-008). |
| `wal_level` | `replica` | Streaming + archiving. |
| `archive_timeout` | `60s` | Limita o RPO. |
| `statement_timeout` | `30s` | Igual a `db.query.statement_timeout_ms`. |
| `idle_in_transaction_session_timeout` | `60s` | Impede transação ociosa segurando lock. |
| `lock_timeout` | `5s` | Igual a `db.migration.lock_timeout_ms`. |
| `default_transaction_isolation` | `read committed` | NFR-012. |
| `shared_preload_libraries` | `pg_stat_statements` | Insumo do `QueryGovernor`. |
| `autovacuum_vacuum_scale_factor` | `0.02` em tabelas quentes | Controle de *bloat* (NFR-016). |
| `max_connections` | 400 (atrás do PgBouncer) | Conexões lógicas multiplexadas. |

Os valores acima são idênticos aos de `./Configuration.md` — divergência é defeito.

---

## 12. Referências

- Brief: `./_DESIGN_BRIEF.md` §3
- Escala e particionamento: `./Scalability.md` · Segurança e RLS: `./Security.md`
- Migração (FSM): `./StateMachine.md` · API administrativa: `./API.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
