---
Documento: Database.md
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0086, ADR-0089
RFCs relacionados: RFC-0001 (baseline), RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore Protocol, a propor)
Depende de: 001-Architecture, 003-RFC/RFC-0001, 005-Database, 006-Kernel, 021-Security, 025-Audit
---

# 008-Agent-Lifecycle — Database

> **Escopo.** Modelo físico PostgreSQL 16 do Lifecycle Manager: DDL completo,
> índices, particionamento, Row-Level Security (RLS) por tenant, política de
> retenção e estratégia de migração. Alinhado a `_DESIGN_BRIEF.md` §3 (Modelo
> de Dados) e §10 (Escalabilidade). O módulo 008 **NÃO usa** `pgvector`/Apache
> AGE diretamente (não armazena embeddings nem grafos — isso é
> responsabilidade de `010-Memory`/`019-GraphRAG`); usa PostgreSQL relacional
> puro + Redis para estado quente (ACB projetado, leases) + MinIO para blobs
> de checkpoint.

---

## 1. Visão geral do schema

| Tabela | Papel | Fonte da verdade para |
|--------|-------|------------------------|
| `lifecycle.agent_control_block` | Projeção de estado atual do ACB (parte de ciclo de vida). | Estado canônico (`state`, `generation`, `desired_state`, ponteiros) — §3.1 do brief. |
| `lifecycle.transition` | Log event-sourced, append-only, de todas as transições da FSM. | Histórico completo e auditável do ciclo de vida — §3.2 do brief. |
| `lifecycle.checkpoint` | Metadados/índice de checkpoints (blob real em MinIO). | Rastreabilidade de snapshots consistentes/crash — §3.3 do brief. |
| `lifecycle.hibernation_record` | Registro de agentes hibernados e métricas de wake. | Estado de cold agent e histórico de materialização — §3.4 do brief. |
| `lifecycle.migration_job` | Saga de migração entre shards/nós. | Fases, status e checkpoint de transferência — §3.5 do brief. |
| `lifecycle.outbox` | Mensagens pendentes de publicação no NATS/JetStream. | Padrão Outbox transacional (ADR-0086). |
| `lifecycle.idempotency_result` | Resultados persistidos por `Idempotency-Key`. | Deduplicação de mutações (RFC-0001 §5.5). |
| `lifecycle.schema_migrations` | Controle de versão de migrações aplicadas. | Auditoria de evolução do schema. |

Todas as tabelas de negócio **DEVEM** ter `tenant_id` e Row-Level Security
ativa (RFC-0001 §5.1, §6). O schema PostgreSQL é `lifecycle` (dedicado,
isolado de outros módulos no mesmo cluster lógico, conforme
`../005-Database/`). O componente `Lease` (§3.6 do brief) **não** é
persistido em PostgreSQL — vive exclusivamente em Redis (ver §8).

---

## 2. DDL completo

### 2.1 Extensões e funções auxiliares

```sql
CREATE SCHEMA IF NOT EXISTS lifecycle;

-- ULID como texto de 26 caracteres (Crockford base32); validação por CHECK.
-- Geração de ULID ocorre na aplicação (.NET 10 / NUlid) — não no banco.
CREATE OR REPLACE FUNCTION lifecycle.is_valid_ulid(v text) RETURNS boolean
LANGUAGE sql IMMUTABLE AS $$
  SELECT v ~ '^[0-9A-HJKMNP-TV-Z]{26}$';
$$;

CREATE OR REPLACE FUNCTION lifecycle.is_valid_urn(v text, tipo text) RETURNS boolean
LANGUAGE sql IMMUTABLE AS $$
  SELECT v ~ ('^urn:aios:[a-z0-9-]+:' || tipo || ':[0-9A-HJKMNP-TV-Z]{26}$');
$$;
```

### 2.2 Tabela `lifecycle.agent_control_block`

```sql
CREATE TABLE lifecycle.agent_control_block (
    agent_id            char(26)    NOT NULL CHECK (lifecycle.is_valid_ulid(agent_id)),
    tenant_id           text        NOT NULL,
    state               text        NOT NULL CHECK (state IN (
                            'Created','Ready','Running','Suspended',
                            'Hibernated','Migrating','Terminated','Failed')),
    generation          bigint      NOT NULL DEFAULT 1 CHECK (generation >= 1),
    desired_state       text        NOT NULL CHECK (desired_state IN (
                            'Created','Ready','Running','Suspended',
                            'Hibernated','Migrating','Terminated','Failed')),
    priority_class      text        NOT NULL DEFAULT 'interactive'
                            CHECK (priority_class IN ('system','interactive','batch','best_effort')),
    placement_shard     int         NOT NULL,
    placement_node      text        NULL,
    runtime_instance_id char(26)    NULL CHECK (runtime_instance_id IS NULL OR lifecycle.is_valid_ulid(runtime_instance_id)),
    working_memory_ref  text        NULL,
    context_ref         text        NULL,
    policy_ref          text        NOT NULL,
    quota_ref           text        NOT NULL,
    last_checkpoint_id  char(26)    NULL,
    cold_since          timestamptz NULL,
    wake_count          int         NOT NULL DEFAULT 0 CHECK (wake_count >= 0),
    fencing_token       bigint      NOT NULL DEFAULT 0,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    terminated_at       timestamptz NULL,
    failure_code        text        NULL,
    labels              jsonb       NOT NULL DEFAULT '{}'::jsonb,

    CONSTRAINT pk_acb PRIMARY KEY (tenant_id, agent_id),
    CONSTRAINT fk_acb_last_checkpoint FOREIGN KEY (last_checkpoint_id)
        REFERENCES lifecycle.checkpoint (checkpoint_id) ON DELETE SET NULL,

    -- (ii) estado terminal ⇒ desired_state terminal e runtime_instance_id nulo
    CONSTRAINT ck_acb_terminal_consistency CHECK (
        (state IN ('Terminated','Failed') AND desired_state IN ('Terminated','Failed')
             AND runtime_instance_id IS NULL AND terminated_at IS NOT NULL)
        OR (state NOT IN ('Terminated','Failed'))
    ),
    -- (iii) Hibernated ⇒ last_checkpoint_id não nulo (INV4 do brief)
    CONSTRAINT ck_acb_hibernated_has_checkpoint CHECK (
        (state = 'Hibernated' AND last_checkpoint_id IS NOT NULL)
        OR (state <> 'Hibernated')
    ),
    -- runtime_instance_id só é não-nulo em estados quentes
    CONSTRAINT ck_acb_runtime_scope CHECK (
        (runtime_instance_id IS NOT NULL AND state IN ('Running','Suspended','Migrating'))
        OR runtime_instance_id IS NULL
    ),
    -- cold_since só é não-nulo em Hibernated
    CONSTRAINT ck_acb_cold_since_scope CHECK (
        (cold_since IS NOT NULL AND state = 'Hibernated')
        OR (cold_since IS NULL AND state <> 'Hibernated')
    )
) PARTITION BY HASH (tenant_id);

-- 32 partições hash por tenant_id — volume alvo ≥ 10⁶ ACBs por cluster (NFR-004).
CREATE TABLE lifecycle.acb_p00 PARTITION OF lifecycle.agent_control_block FOR VALUES WITH (MODULUS 32, REMAINDER 0);
CREATE TABLE lifecycle.acb_p01 PARTITION OF lifecycle.agent_control_block FOR VALUES WITH (MODULUS 32, REMAINDER 1);
-- ... p02..p31 seguem o mesmo padrão (omitidos por brevidade; ver migração 0001).

CREATE INDEX ix_acb_tenant_state    ON lifecycle.agent_control_block (tenant_id, state);
CREATE INDEX ix_acb_shard           ON lifecycle.agent_control_block (placement_shard);
CREATE INDEX ix_acb_hibernation_idle ON lifecycle.agent_control_block (tenant_id, updated_at)
    WHERE state = 'Suspended';   -- varredura de candidatos à hibernação (HibernationController)
CREATE INDEX ix_acb_cold             ON lifecycle.agent_control_block (tenant_id, cold_since)
    WHERE state = 'Hibernated';  -- varredura/estatística de cold agents (NFR-004)
CREATE INDEX ix_acb_desired_mismatch ON lifecycle.agent_control_block (tenant_id)
    WHERE state IS DISTINCT FROM desired_state;  -- fila de trabalho do ReconciliationController
CREATE INDEX ix_acb_labels_gin       ON lifecycle.agent_control_block USING gin (labels jsonb_path_ops);

ALTER TABLE lifecycle.agent_control_block ENABLE ROW LEVEL SECURITY;
ALTER TABLE lifecycle.agent_control_block FORCE ROW LEVEL SECURITY;

CREATE POLICY acb_tenant_isolation ON lifecycle.agent_control_block
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

> **Nota de projeto:** `ix_acb_desired_mismatch` é o índice que sustenta o
> **padrão controller/reconcile** (R9 do brief): o `ReconciliationController`
> varre periodicamente (`reconcile.interval_ms`) apenas as linhas onde
> `state ≠ desired_state`, mantendo o custo da varredura proporcional ao
> número de agentes em deriva, não ao total de agentes (crítico para escalar
> a 10⁶ ACBs).

### 2.3 Tabela `lifecycle.transition`

```sql
CREATE TABLE lifecycle.transition (
    transition_id   char(26)    NOT NULL CHECK (lifecycle.is_valid_ulid(transition_id)),
    agent_id        char(26)    NOT NULL,
    tenant_id       text        NOT NULL,
    seq             bigint      NOT NULL,
    from_state      text        NULL,
    to_state        text        NOT NULL,
    trigger         text        NOT NULL CHECK (trigger IN (
                        'spawn','ready','start','suspend','resume','hibernate',
                        'wake','migrate','terminate','fail','reconcile')),
    actor           text        NOT NULL,
    guard_result    jsonb       NOT NULL DEFAULT '{}'::jsonb,
    reason          text        NULL,
    generation      bigint      NOT NULL,
    correlation     jsonb       NOT NULL DEFAULT '{}'::jsonb,
    created_at      timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT pk_transition PRIMARY KEY (tenant_id, agent_id, seq),
    CONSTRAINT uq_transition_id UNIQUE (transition_id)
) PARTITION BY RANGE (created_at);

-- Partições mensais (retenção via DROP PARTITION — ver §6).
CREATE TABLE lifecycle.transition_2026_07 PARTITION OF lifecycle.transition
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
CREATE TABLE lifecycle.transition_2026_08 PARTITION OF lifecycle.transition
    FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
-- Partições futuras criadas por job agendado (pg_partman ou migração periódica).

CREATE INDEX ix_transition_agent_seq   ON lifecycle.transition (tenant_id, agent_id, seq DESC);
CREATE INDEX ix_transition_trigger     ON lifecycle.transition (tenant_id, trigger, created_at DESC);
CREATE INDEX ix_transition_correlation ON lifecycle.transition USING gin (correlation jsonb_path_ops);

ALTER TABLE lifecycle.transition ENABLE ROW LEVEL SECURITY;
ALTER TABLE lifecycle.transition FORCE ROW LEVEL SECURITY;

CREATE POLICY transition_tenant_isolation ON lifecycle.transition
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

> `seq` é um contador monotônico **por agente** (não global), incrementado
> pelo `LifecycleCoordinator` dentro da mesma transação que grava o novo
> estado do ACB — garante ordenação total do histórico de um agente
> (`GET /v1/agents/{id}/lifecycle`) sem depender de `created_at` (que pode
> ter granularidade insuficiente sob alta concorrência). A tabela é
> **append-only**: nenhuma linha é `UPDATE`d ou `DELETE`d fora do processo
> de expurgo do `TombstoneManager` (§6).

### 2.4 Tabela `lifecycle.checkpoint`

```sql
CREATE TABLE lifecycle.checkpoint (
    checkpoint_id      char(26)    NOT NULL CHECK (lifecycle.is_valid_ulid(checkpoint_id)),
    agent_id           char(26)    NOT NULL,
    tenant_id          text        NOT NULL,
    generation         bigint      NOT NULL,
    storage_uri        text        NOT NULL,
    size_bytes         bigint      NOT NULL CHECK (size_bytes > 0),
    content_hash       text        NOT NULL,  -- sha-256 hex, 64 chars
    codec              text        NOT NULL CHECK (codec IN ('json+zstd','msgpack+zstd')),
    encryption         text        NOT NULL DEFAULT 'aes-256-gcm' CHECK (encryption IN ('aes-256-gcm')),
    working_memory_ref text        NOT NULL,
    consistency        text        NOT NULL CHECK (consistency IN ('consistent','crash')),
    created_at         timestamptz NOT NULL DEFAULT now(),
    expires_at         timestamptz NULL,

    CONSTRAINT pk_checkpoint PRIMARY KEY (tenant_id, checkpoint_id),
    CONSTRAINT ck_checkpoint_hash_format CHECK (content_hash ~ '^[0-9a-f]{64}$')
);

CREATE INDEX ix_checkpoint_agent      ON lifecycle.checkpoint (tenant_id, agent_id, created_at DESC);
CREATE INDEX ix_checkpoint_expires    ON lifecycle.checkpoint (expires_at) WHERE expires_at IS NOT NULL;
CREATE INDEX ix_checkpoint_hash       ON lifecycle.checkpoint (content_hash);

ALTER TABLE lifecycle.checkpoint ENABLE ROW LEVEL SECURITY;
ALTER TABLE lifecycle.checkpoint FORCE ROW LEVEL SECURITY;

CREATE POLICY checkpoint_tenant_isolation ON lifecycle.checkpoint
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

> `storage_uri` segue o padrão `minio://<bucket>/<tenant>/<agent>/<ckpt>.zst`
> (brief §3.3). A FK `lifecycle.agent_control_block.last_checkpoint_id →
> lifecycle.checkpoint.checkpoint_id` (declarada em §2.2) exige que
> `lifecycle.checkpoint` seja criada **antes** de `agent_control_block` na
> ordem real de migração (ver §7).

### 2.5 Tabela `lifecycle.hibernation_record`

```sql
CREATE TABLE lifecycle.hibernation_record (
    agent_id             char(26)    NOT NULL,
    tenant_id            text        NOT NULL,
    cold_since           timestamptz NOT NULL,
    storage_tier         text        NOT NULL DEFAULT 'warm' CHECK (storage_tier IN ('hot','warm','cold')),
    checkpoint_id        char(26)    NOT NULL,
    last_wake_latency_ms int         NULL,
    wake_count           int         NOT NULL DEFAULT 0 CHECK (wake_count >= 0),
    updated_at           timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT pk_hibernation_record PRIMARY KEY (tenant_id, agent_id),
    CONSTRAINT fk_hibernation_checkpoint FOREIGN KEY (tenant_id, checkpoint_id)
        REFERENCES lifecycle.checkpoint (tenant_id, checkpoint_id) ON DELETE RESTRICT
);

CREATE INDEX ix_hibernation_tier ON lifecycle.hibernation_record (tenant_id, storage_tier);
CREATE INDEX ix_hibernation_cold_since ON lifecycle.hibernation_record (cold_since);

ALTER TABLE lifecycle.hibernation_record ENABLE ROW LEVEL SECURITY;
ALTER TABLE lifecycle.hibernation_record FORCE ROW LEVEL SECURITY;

CREATE POLICY hibernation_tenant_isolation ON lifecycle.hibernation_record
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

> Esta tabela existe como **projeção especializada** de agentes atualmente
> `Hibernated`, otimizada para consultas operacionais (dashboards de
> `NFR-004`: `≤ 5%` ativos em RAM) sem varrer `agent_control_block`
> completo. Uma linha é inserida em `hibernate` (T6) e removida em `wake`
> (T7) — a fonte de verdade do estado permanece
> `agent_control_block.state`.

### 2.6 Tabela `lifecycle.migration_job`

```sql
CREATE TABLE lifecycle.migration_job (
    migration_id   char(26)    NOT NULL CHECK (lifecycle.is_valid_ulid(migration_id)),
    agent_id       char(26)    NOT NULL,
    tenant_id      text        NOT NULL,
    source_shard   int         NOT NULL,
    source_node    text        NULL,
    target_shard   int         NOT NULL,
    target_node    text        NULL,
    phase          text        NOT NULL DEFAULT 'quiesce' CHECK (phase IN (
                        'quiesce','checkpoint','transfer','materialize','cutover','cleanup')),
    status         text        NOT NULL DEFAULT 'pending' CHECK (status IN (
                        'pending','running','succeeded','failed','compensated')),
    checkpoint_id  char(26)    NULL,
    reason         text        NOT NULL DEFAULT 'manual' CHECK (reason IN ('rebalance','node_draining','manual')),
    started_at     timestamptz NOT NULL DEFAULT now(),
    finished_at    timestamptz NULL,

    CONSTRAINT pk_migration_job PRIMARY KEY (tenant_id, migration_id),
    CONSTRAINT fk_migration_checkpoint FOREIGN KEY (tenant_id, checkpoint_id)
        REFERENCES lifecycle.checkpoint (tenant_id, checkpoint_id) ON DELETE SET NULL,
    -- Apenas uma migração ativa (pending/running) por agente (regra de negócio
    -- reforçada por índice único parcial abaixo, além do lease em Redis).
    CONSTRAINT ck_migration_finished_scope CHECK (
        (status IN ('succeeded','failed','compensated') AND finished_at IS NOT NULL)
        OR (status IN ('pending','running') AND finished_at IS NULL)
    )
);

CREATE UNIQUE INDEX uq_migration_active_per_agent ON lifecycle.migration_job (tenant_id, agent_id)
    WHERE status IN ('pending','running');  -- gera AIOS-LIFECYCLE-0009 em violação
CREATE INDEX ix_migration_agent    ON lifecycle.migration_job (tenant_id, agent_id, started_at DESC);
CREATE INDEX ix_migration_status   ON lifecycle.migration_job (tenant_id, status);

ALTER TABLE lifecycle.migration_job ENABLE ROW LEVEL SECURITY;
ALTER TABLE lifecycle.migration_job FORCE ROW LEVEL SECURITY;

CREATE POLICY migration_tenant_isolation ON lifecycle.migration_job
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

> `uq_migration_active_per_agent` traduz a regra de negócio "migração já em
> andamento" (`AIOS-LIFECYCLE-0009`) em uma **constraint de banco**,
> complementando — não substituindo — o `LeaseManager` (Redis), que é a
> primeira linha de defesa contra concorrência (mais rápida, evita round-trip
> ao Postgres antes de sequer iniciar a saga).

### 2.7 Tabela `lifecycle.outbox`

```sql
CREATE TABLE lifecycle.outbox (
    event_id        char(26)    NOT NULL CHECK (lifecycle.is_valid_ulid(event_id)),
    tenant_id       text        NOT NULL,
    subject         text        NOT NULL,
    event_type      text        NOT NULL,
    payload         jsonb       NOT NULL,
    published       boolean     NOT NULL DEFAULT false,
    publish_attempts int        NOT NULL DEFAULT 0,
    last_attempt_at  timestamptz NULL,
    created_at      timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT pk_outbox PRIMARY KEY (event_id)
);

-- Índice parcial: o relay só varre mensagens pendentes (hot path pequeno,
-- mesmo com milhões de linhas históricas na tabela).
CREATE INDEX ix_outbox_pending ON lifecycle.outbox (created_at) WHERE published = false;

ALTER TABLE lifecycle.outbox ENABLE ROW LEVEL SECURITY;
ALTER TABLE lifecycle.outbox FORCE ROW LEVEL SECURITY;

CREATE POLICY outbox_tenant_isolation ON lifecycle.outbox
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

### 2.8 Tabela `lifecycle.idempotency_result`

```sql
CREATE TABLE lifecycle.idempotency_result (
    tenant_id       text        NOT NULL,
    idempotency_key text        NOT NULL,
    request_hash    text        NOT NULL,  -- SHA-256 do corpo normalizado
    operation       text        NOT NULL,  -- spawn|suspend|resume|hibernate|wake|migrate|checkpoint|restore|terminate
    http_status     int         NOT NULL,
    response_body   jsonb       NOT NULL,
    created_at      timestamptz NOT NULL DEFAULT now(),
    expires_at      timestamptz NOT NULL,

    CONSTRAINT pk_idempotency PRIMARY KEY (tenant_id, idempotency_key)
);

CREATE INDEX ix_idempotency_expires ON lifecycle.idempotency_result (expires_at);

ALTER TABLE lifecycle.idempotency_result ENABLE ROW LEVEL SECURITY;
ALTER TABLE lifecycle.idempotency_result FORCE ROW LEVEL SECURITY;

CREATE POLICY idempotency_tenant_isolation ON lifecycle.idempotency_result
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

### 2.9 Tabela `lifecycle.schema_migrations`

```sql
CREATE TABLE lifecycle.schema_migrations (
    version     text        NOT NULL,
    description text        NOT NULL,
    applied_at  timestamptz NOT NULL DEFAULT now(),
    checksum    text        NOT NULL,

    CONSTRAINT pk_schema_migrations PRIMARY KEY (version)
);
```

---

## 3. Índices — resumo consolidado

| Tabela | Índice | Tipo | Finalidade |
|--------|--------|------|------------|
| `agent_control_block` | `pk_acb (tenant_id, agent_id)` | B-tree (PK) | Lookup direto por agente (caminho quente). |
| `agent_control_block` | `ix_acb_tenant_state` | B-tree composto | Consultas administrativas por estado. |
| `agent_control_block` | `ix_acb_shard` | B-tree | Suporte a roteamento por shard lógico de aplicação. |
| `agent_control_block` | `ix_acb_hibernation_idle` (parcial) | B-tree parcial | Varredura de candidatos à hibernação (T6), restrita a `Suspended`. |
| `agent_control_block` | `ix_acb_cold` (parcial) | B-tree parcial | Estatística/roteamento de cold agents (NFR-004), restrita a `Hibernated`. |
| `agent_control_block` | `ix_acb_desired_mismatch` (parcial) | B-tree parcial | Fila de trabalho do `ReconciliationController` (deriva estado×desejado). |
| `agent_control_block` | `ix_acb_labels_gin` | GIN (jsonb_path_ops) | Consultas por metadados livres (`labels`). |
| `transition` | `pk_transition (tenant_id, agent_id, seq)` | B-tree (PK) | Ordenação total por agente, leitura de histórico paginado. |
| `transition` | `ix_transition_trigger` | B-tree composto | Análise operacional por tipo de gatilho. |
| `transition` | `ix_transition_correlation` | GIN (jsonb_path_ops) | Busca por `traceparent`/`idempotency_key` em auditoria. |
| `checkpoint` | `ix_checkpoint_agent` | B-tree composto (DESC) | Listagem de checkpoints por agente, mais recente primeiro. |
| `checkpoint` | `ix_checkpoint_expires` (parcial) | B-tree parcial | Job de expurgo por retenção. |
| `checkpoint` | `ix_checkpoint_hash` | B-tree | Verificação de integridade/deduplicação de blobs. |
| `hibernation_record` | `ix_hibernation_tier` | B-tree | Distribuição por camada de storage (hot/warm/cold). |
| `migration_job` | `uq_migration_active_per_agent` (parcial único) | B-tree parcial único | Impede migração concorrente para o mesmo agente (`AIOS-LIFECYCLE-0009`). |
| `migration_job` | `ix_migration_status` | B-tree | Painel operacional de sagas em andamento. |
| `outbox` | `ix_outbox_pending` (parcial) | B-tree parcial | Relay varre só `published=false` — custo O(pendentes), não O(total). |
| `idempotency_result` | `ix_idempotency_expires` | B-tree | Job de expurgo por `expires_at`. |

---

## 4. Particionamento

| Tabela | Estratégia | Justificativa |
|--------|-----------|----------------|
| `lifecycle.agent_control_block` | **HASH(tenant_id)**, 32 partições. | Distribui I/O entre tenants sem depender do volume individual; independe do `placement_shard` lógico de aplicação (usado para afinidade de execução, não para particionamento físico). Alinhado à meta `≥ 10⁶` agentes por cluster (NFR-004), com manutenção paralela (VACUUM/REINDEX por partição). |
| `lifecycle.transition` | **RANGE(created_at)**, mensal. | Log event-sourced de altíssimo volume (NFR-008: `≥ 50.000 transições/s`); permite expurgo por `DROP PARTITION` (O(1), sem `DELETE` em massa), alinhado à política de retenção (§6). |
| `lifecycle.checkpoint` | Não particionada (candidato a RANGE(created_at) se volume de checkpoints por agente crescer além do previsto). | Cardinalidade moderada — número de checkpoints é limitado por `checkpoint.interval_s` e retenção (`retention.checkpoint_ttl_days`, default 7 dias). |
| `lifecycle.hibernation_record` | Não particionada. | Uma linha por agente **enquanto** `Hibernated`; removida no wake — tamanho da tabela é proporcional a agentes frios ativos, não ao histórico. |
| `lifecycle.migration_job` | Não particionada. | Volume limitado por `migration.max_concurrent_per_node`; histórico antigo é candidato a arquivamento frio, não particionamento físico. |
| `lifecycle.outbox` | Não particionada (índice parcial supre o caso de uso). | Volume por réplica é limitado pelo relay (`outbox.publish_batch`); registros publicados são purgados periodicamente. |
| `lifecycle.idempotency_result` | Não particionada. | TTL curto (24–720h, default conforme brief) mantém a tabela pequena via expurgo contínuo. |

**Criação de partições futuras.** Um job agendado
(`lifecycle-partition-manager`, via CronJob/`pg_partman`) cria a partição do
mês seguinte de `lifecycle.transition` com **7 dias de antecedência**,
evitando falha de inserção por ausência de partição-alvo — mesmo padrão
operacional adotado por `006-Kernel`.

---

## 5. Row-Level Security (RLS) — política operacional

- Toda sessão de aplicação do `AcbStore`/`LifecycleCoordinator` **DEVE**
  executar `SET aios.tenant_id = '<tenant>'` (via `SET LOCAL` dentro da
  transação) imediatamente após obter a conexão do pool, com o valor
  extraído do claim autenticado — **nunca** de um parâmetro de URL não
  validado.
- `FORCE ROW LEVEL SECURITY` garante que **mesmo o owner da tabela** (a role
  de aplicação) está sujeito à política — não há bypass implícito por
  privilégio de owner.
- Roles administrativas de suporte (`lifecycle_dba_readonly`) usadas por
  operação/observabilidade **DEVEM** ter política RLS equivalente ou acesso
  somente via views agregadas sem PII, nunca `BYPASSRLS`.
- Testes de RLS (contract tests) **DEVEM** verificar que uma sessão com
  `aios.tenant_id = 'acme'` não enxerga nem consegue escrever linhas de
  `tenant_id = 'globex'`, para todas as 8 tabelas de negócio deste schema.

---

## 6. Retenção e expurgo

| Tabela | Política de retenção | Mecanismo |
|--------|------------------------|-----------|
| `agent_control_block` | Agentes `Terminated`/`Failed` são tombstoned após `retention.terminated_ttl_days` (default 30 dias). | `TombstoneManager` marca `labels.tombstoned=true` e agenda expurgo coordenado de checkpoints/snapshots; `agent_id` **NÃO É** reutilizado (RFC-0001 §5.1). |
| `transition` | **90 dias** (default) via `DROP PARTITION` mensal fora da janela; retenção estendida configurável por tenant regulado. | Job `lifecycle-partition-manager`. |
| `checkpoint` | `retention.checkpoint_ttl_days` (default 7 dias) — expiração calculada em `expires_at` na criação. | Job `lifecycle-checkpoint-gc` remove metadados + aciona remoção do blob em MinIO via `SnapshotStore`. |
| `hibernation_record` | Vida útil do agente frio; removida no `wake` (T7) ou no expurgo LGPD do agente. | Sem TTL próprio — segue o ciclo de vida do agente. |
| `migration_job` | Retida por auditoria operacional por 90 dias; sagas concluídas (`succeeded`/`failed`/`compensated`) são candidatas a arquivamento frio. | Job de arquivamento periódico (não destrutivo — copia para armazenamento frio antes de eventual purge). |
| `outbox` | Linhas com `published=true` e `created_at` > 7 dias são arquivadas/purgadas (job `lifecycle-outbox-gc`). | `DELETE` em lote com `LIMIT`/batching para evitar bloqueio longo. |
| `idempotency_result` | `expires_at` = `created_at + retenção configurada` (mínimo 24h por RFC-0001 §5.5). | Job `lifecycle-idempotency-gc` remove expirados a cada 15 min. |
| `schema_migrations` | Permanente (histórico de auditoria de schema). | Sem expurgo. |

**Direito ao esquecimento (LGPD/GDPR).** Conforme `_DESIGN_BRIEF.md` §1.1
(R12) e §12.3, o `TombstoneManager` executa expurgo rastreável de ACB,
transições, checkpoints e snapshots ao término da retenção ou sob
solicitação, emitindo evento auditável consumido por `025-Audit`. IDs de
agente **NÃO DEVEM** ser reutilizados após expurgo — uma linha
"tombstone" mínima (`agent_id`, `tenant_id`, `terminated_at`,
`purged_at`) é preservada indefinidamente em tabela de anti-ressurreição
para impedir colisão de ULID reciclado (mecanismo detalhado em
`Security.md`).

---

## 7. Migrações

| Ferramenta | **DEVE** ser aplicada via *migration runner* versionado (Flyway/EF Core Migrations no control plane .NET 10), nunca `psql` manual em produção. |
|---|---|

| Arquivo | Conteúdo |
|---------|----------|
| `migrations/0001_init.sql` | Cria schema `lifecycle`, funções de validação, tabela `checkpoint` (pré-requisito de FK), tabela `agent_control_block` (32 partições), índices, RLS. |
| `migrations/0002_transition.sql` | Cria `transition` particionada + partições dos 2 meses correntes + índices + RLS. |
| `migrations/0003_hibernation_migration.sql` | Cria `hibernation_record` e `migration_job` + índices + RLS. |
| `migrations/0004_outbox.sql` | Cria `outbox` + índice parcial + RLS. |
| `migrations/0005_idempotency.sql` | Cria `idempotency_result` + índice + RLS. |
| `migrations/0006_schema_migrations.sql` | Cria tabela de controle de migração (bootstrap, normalmente criada primeiro pela ferramenta). |

**Regras de migração:**

- Toda migração **DEVE** ser aditiva e retrocompatível dentro de uma mesma
  major de API (RFC-0001 §5.7): novas colunas `NULL`-áveis ou com
  `DEFAULT`; nunca `DROP COLUMN`/`ALTER TYPE` destrutivo sem uma migração
  de duas fases (deprecate → backfill → remove em release subsequente).
- Alteração do número de partições hash de `agent_control_block` (hoje 32,
  fixo no DDL) exige migração de **reparticionamento lógico** controlada
  (redistribuir linhas, invalidar afinidade de cache Redis) — documentada
  como procedimento operacional em `../029-Operations/`, não como
  `ALTER TABLE` trivial.
- Toda migração **DEVE** ser testada em ambiente de staging com volume
  representativo (idealmente ≥ 10⁶ ACBs sintéticos) antes de produção
  (ver `Testing.md`).
- Rollback: migrações destrutivas **DEVEM** ter script de reversão par
  (`_down.sql`); migrações aditivas puras não exigem rollback formal
  (basta não usar a nova coluna).

---

## 8. Relação com estado quente (Redis) e blobs (MinIO)

> O PostgreSQL é a **fonte da verdade**; Redis é **projeção quente** — nunca
> o inverso. MinIO armazena o **payload binário** de checkpoints; PostgreSQL
> armazena apenas metadados/índice. Em caso de divergência entre Redis e
> PostgreSQL, PostgreSQL vence.

| Dado | Onde vive | Chave/URI | TTL | Reconciliação |
|------|-----------|-----------|-----|-----------------|
| ACB quente (`Running`/`Suspended`) | Redis (`AcbStore` hot cache) | `aios:{tenant}:lc:acb:{agent_id}` (hash) | sem TTL fixo — invalidado em toda mutação; recarregado *lazy* em cache-miss | Leitura *read-your-writes*: toda escrita no PG atualiza (ou invalida) a chave na mesma transação lógica da aplicação. |
| Índice de ACB frio | Redis (índice mínimo, NFR-011: `≤ 4 KiB`) | `aios:{tenant}:lc:cold:{agent_id}` (hash reduzido: `state`, `last_checkpoint_id`, `placement_shard`) | sem TTL — mantido enquanto `Hibernated` | Atualizado em `hibernate`/`wake`; usado para roteamento e wake rápido sem tocar o PG no caminho quente. |
| Lease de transição | Redis (`LeaseManager`) | `aios:{tenant}:lc:lease:{agent_id}` → `{holder, fencing_token, expires_at}` | `lease.ttl_ms` (default 5000ms), renovação a cada `lease.renew_ms` | `fencing_token` monotônico persiste em `agent_control_block.fencing_token`; escrita com token obsoleto é rejeitada (`AIOS-LIFECYCLE-0013`). |
| Resultado idempotente | Redis (cache) | `aios:{tenant}:lc:idemp:{key}` | retenção configurada (`idempotency`, ver `Configuration.md`) | Fallback para `lifecycle.idempotency_result` em cache-miss. |
| Payload de checkpoint | MinIO (blob) | `minio://<bucket>/<tenant>/<agent>/<checkpoint_id>.zst` | conforme `retention.checkpoint_ttl_days` | `lifecycle.checkpoint.storage_uri` referencia o objeto; `content_hash` valida integridade no restore. |

---

## 9. Referências

- Modelo de dados canônico (fonte): `./_DESIGN_BRIEF.md` §3.
- Máquina de estados que impõe as constraints de `state`/`desired_state`/`runtime_instance_id`/`cold_since`: `./StateMachine.md`.
- Estratégia de sharding e concorrência (contexto arquitetural): `./Scalability.md`.
- Convenções de identidade (URN) e idempotência: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.1, §5.5.
- Padrões físicos gerais do banco AIOS (pgvector/AGE, convenções): `../005-Database/`.
- Modos de falha relacionados a persistência/Outbox/lease: `./FailureRecovery.md`.
- Chaves de configuração citadas (`lease.ttl_ms`, `checkpoint.interval_s`, `retention.*`): `./Configuration.md`.
