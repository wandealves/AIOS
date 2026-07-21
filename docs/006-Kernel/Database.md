---
Documento: Database.md
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0061, ADR-0062, ADR-0065, ADR-0066
RFCs relacionados: RFC-0001 (baseline), RFC-0007 (ACB & Quota Model, a propor)
Depende de: 001-Architecture, 003-RFC/RFC-0001, 005-Database, 021-Security
---

# 006-Kernel — Database

> **Escopo.** Modelo físico PostgreSQL 16 do Kernel: DDL completo, índices,
> particionamento, Row-Level Security (RLS) por tenant, política de retenção e
> estratégia de migração. Alinhado a `_DESIGN_BRIEF.md` §3 (Modelo de Dados) e
> §10 (Escalabilidade). O Kernel **NÃO usa** `pgvector`/Apache AGE diretamente
> (não armazena embeddings nem grafos — isso é responsabilidade de
> `010-Memory`/`019-GraphRAG`); usa PostgreSQL relacional puro + Redis para
> estado quente.

---

## 1. Visão geral do schema

| Tabela | Papel | Fonte da verdade para |
|--------|-------|------------------------|
| `kernel.acb` | Agent Control Block autoritativo. | Estado de controle do agente (§4 do brief). |
| `kernel.quota` | Limites de cota por tenant/agente. | Configuração de teto; contadores voláteis vivem em Redis. |
| `kernel.syscall_log` | Log de auditoria local de invocações de syscall. | Rastreabilidade de decisão PEP + resultado. |
| `kernel.outbox` | Mensagens pendentes de publicação no NATS/JetStream. | Padrão Outbox transacional (ADR-0066). |
| `kernel.idempotency_result` | Resultados persistidos por `Idempotency-Key`. | Deduplicação de mutações (RFC-0001 §5.5). |
| `kernel.schema_migrations` | Controle de versão de migrações aplicadas. | Auditoria de evolução do schema. |

Todas as tabelas de negócio (`acb`, `quota`, `syscall_log`, `outbox`,
`idempotency_result`) **DEVEM** ter `tenant_id` e Row-Level Security ativa
(RFC-0001 §5.1, §6). O schema PostgreSQL é `kernel` (schema dedicado, isolado
de outros módulos no mesmo cluster lógico, conforme `../005-Database/`).

---

## 2. DDL completo

### 2.1 Extensões e schema

```sql
CREATE SCHEMA IF NOT EXISTS kernel;

-- ULID como texto de 26 caracteres (Crockford base32); validação por CHECK.
-- Geração de ULID ocorre na aplicação (.NET 10 / NUlid) — não no banco.
CREATE OR REPLACE FUNCTION kernel.is_valid_ulid(v text) RETURNS boolean
LANGUAGE sql IMMUTABLE AS $$
  SELECT v ~ '^[0-9A-HJKMNP-TV-Z]{26}$';
$$;

CREATE OR REPLACE FUNCTION kernel.is_valid_urn(v text, tipo text) RETURNS boolean
LANGUAGE sql IMMUTABLE AS $$
  SELECT v ~ ('^urn:aios:[a-z0-9-]+:' || tipo || ':[0-9A-HJKMNP-TV-Z]{26}$');
$$;
```

### 2.2 Tabela `kernel.acb`

```sql
CREATE TABLE kernel.acb (
    urn               text        NOT NULL,
    tenant_id         text        NOT NULL,
    agent_id          char(26)    NOT NULL CHECK (kernel.is_valid_ulid(agent_id)),
    state             text        NOT NULL CHECK (state IN (
                          'Pending','Admitted','Running','Suspended',
                          'Hibernated','Terminating','Terminated','Failed')),
    priority          smallint    NOT NULL DEFAULT 5 CHECK (priority BETWEEN 0 AND 9),
    parent_urn        text        NULL,
    quota_ref         uuid        NOT NULL,
    memory_ptr        jsonb       NOT NULL DEFAULT '{}'::jsonb,
    context_ptr       jsonb       NOT NULL DEFAULT '{}'::jsonb,
    policy_ref        text        NOT NULL,
    capability_set    text[]      NOT NULL DEFAULT '{}',
    runtime_ref       text        NULL,
    checkpoint_ref    text        NULL,
    shard_key         int         GENERATED ALWAYS AS (
                          abs(hashtextextended(tenant_id || ':' || agent_id, 0))
                          % 64
                        ) STORED,
    version           bigint      NOT NULL DEFAULT 0,
    created_at        timestamptz NOT NULL DEFAULT now(),
    updated_at        timestamptz NOT NULL DEFAULT now(),
    last_active_at    timestamptz NULL,
    terminated_at     timestamptz NULL,
    labels            jsonb       NOT NULL DEFAULT '{}'::jsonb,

    CONSTRAINT pk_acb PRIMARY KEY (urn),
    CONSTRAINT uq_acb_tenant_agent UNIQUE (tenant_id, agent_id),
    CONSTRAINT ck_acb_urn_format CHECK (kernel.is_valid_urn(urn, 'agent')),
    CONSTRAINT fk_acb_parent FOREIGN KEY (parent_urn)
        REFERENCES kernel.acb (urn) ON DELETE SET NULL,
    CONSTRAINT fk_acb_quota FOREIGN KEY (quota_ref)
        REFERENCES kernel.quota (id) ON DELETE RESTRICT,
    -- I2 (invariante do brief §4.2): estado terminal DEVE ter terminated_at
    CONSTRAINT ck_acb_terminal_has_timestamp CHECK (
        (state IN ('Terminated','Failed') AND terminated_at IS NOT NULL)
        OR (state NOT IN ('Terminated','Failed'))
    ),
    -- I1: runtime_ref só é não-nulo em Running/Suspended
    CONSTRAINT ck_acb_runtime_ref_scope CHECK (
        (runtime_ref IS NOT NULL AND state IN ('Running','Suspended'))
        OR runtime_ref IS NULL
    )
) PARTITION BY HASH (tenant_id);

-- 16 partições hash por tenant_id (independente do shard_key lógico de
-- aplicação; particionamento físico existe para paralelismo de I/O e
-- manutenção — ver §5 Particionamento).
CREATE TABLE kernel.acb_p00 PARTITION OF kernel.acb FOR VALUES WITH (MODULUS 16, REMAINDER 0);
CREATE TABLE kernel.acb_p01 PARTITION OF kernel.acb FOR VALUES WITH (MODULUS 16, REMAINDER 1);
CREATE TABLE kernel.acb_p02 PARTITION OF kernel.acb FOR VALUES WITH (MODULUS 16, REMAINDER 2);
CREATE TABLE kernel.acb_p03 PARTITION OF kernel.acb FOR VALUES WITH (MODULUS 16, REMAINDER 3);
-- ... p04..p15 seguem o mesmo padrão (omitidos por brevidade; ver migração 0001).

CREATE INDEX ix_acb_tenant_state       ON kernel.acb (tenant_id, state);
CREATE INDEX ix_acb_shard_key          ON kernel.acb (shard_key);
CREATE INDEX ix_acb_tenant_last_active ON kernel.acb (tenant_id, last_active_at)
    WHERE state IN ('Suspended');   -- varredura de hibernação (T-08)
CREATE INDEX ix_acb_parent             ON kernel.acb (parent_urn) WHERE parent_urn IS NOT NULL;
CREATE INDEX ix_acb_labels_gin         ON kernel.acb USING gin (labels jsonb_path_ops);

ALTER TABLE kernel.acb ENABLE ROW LEVEL SECURITY;
ALTER TABLE kernel.acb FORCE ROW LEVEL SECURITY;

CREATE POLICY acb_tenant_isolation ON kernel.acb
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

> **Nota de projeto:** `parent_urn` referencia `kernel.acb(urn)`, mas a tabela
> é particionada por `tenant_id`; FKs entre partições de uma tabela
> particionada por hash funcionam normalmente no PostgreSQL 16 desde que a
> chave referenciada (`urn`, PK global) seja única — validado via índice único
> declarado na tabela-mãe (PostgreSQL propaga a constraint às partições).

### 2.3 Tabela `kernel.quota`

```sql
CREATE TABLE kernel.quota (
    id                uuid        NOT NULL DEFAULT gen_random_uuid(),
    tenant_id         text        NOT NULL,
    scope             text        NOT NULL CHECK (scope IN ('tenant','agent')),
    subject_urn       text        NOT NULL,
    tokens_limit      bigint      NOT NULL CHECK (tokens_limit >= 0),
    cpu_ms_limit      bigint      NOT NULL CHECK (cpu_ms_limit >= 0),
    mem_mib_limit     int         NOT NULL CHECK (mem_mib_limit >= 0),
    cost_usd_limit    numeric(12,4) NOT NULL CHECK (cost_usd_limit >= 0),
    tools_limit       int         NOT NULL CHECK (tools_limit >= 0),
    syscalls_per_sec  int         NOT NULL CHECK (syscalls_per_sec >= 0),
    window            interval    NOT NULL DEFAULT interval '1 hour',
    enforcement       text        NOT NULL DEFAULT 'hard' CHECK (enforcement IN ('hard','soft')),
    version           bigint      NOT NULL DEFAULT 0,
    created_at        timestamptz NOT NULL DEFAULT now(),
    updated_at        timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT pk_quota PRIMARY KEY (id),
    CONSTRAINT uq_quota_subject UNIQUE (tenant_id, subject_urn, scope)
);

CREATE INDEX ix_quota_tenant  ON kernel.quota (tenant_id);
CREATE INDEX ix_quota_subject ON kernel.quota (subject_urn);

ALTER TABLE kernel.quota ENABLE ROW LEVEL SECURITY;
ALTER TABLE kernel.quota FORCE ROW LEVEL SECURITY;

CREATE POLICY quota_tenant_isolation ON kernel.quota
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

> A FK `kernel.acb.quota_ref → kernel.quota.id` é declarada acima na criação
> de `acb`; por isso `kernel.quota` **DEVE** ser criada antes de `kernel.acb`
> na ordem de migração real (a ordem de exposição neste documento segue a
> narrativa do brief, não a ordem de execução — ver `migrations/0001_init.sql`
> no §7).

### 2.4 Tabela `kernel.syscall_log`

```sql
CREATE TABLE kernel.syscall_log (
    id                char(26)    NOT NULL CHECK (kernel.is_valid_ulid(id)),
    tenant_id         text        NOT NULL,
    agent_urn         text        NOT NULL,
    verb              text        NOT NULL CHECK (verb IN (
                          'spawn','kill','suspend','resume','remember','recall',
                          'plan','invoke_tool','route_model','get_quota','checkpoint')),
    idempotency_key   text        NULL,
    decision          text        NOT NULL CHECK (decision IN ('allow','deny')),
    deny_code         text        NULL,
    status            text        NOT NULL CHECK (status IN ('ok','error')),
    latency_ms        int         NOT NULL CHECK (latency_ms >= 0),
    trace_id          text        NOT NULL,
    created_at        timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT pk_syscall_log PRIMARY KEY (id, created_at),
    CONSTRAINT uq_syscall_idem UNIQUE (tenant_id, idempotency_key)
) PARTITION BY RANGE (created_at);

-- Partições mensais (retenção via DROP PARTITION — ver §6).
CREATE TABLE kernel.syscall_log_2026_07 PARTITION OF kernel.syscall_log
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
CREATE TABLE kernel.syscall_log_2026_08 PARTITION OF kernel.syscall_log
    FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
-- Partições futuras criadas por job agendado (pg_partman ou migração periódica).

CREATE INDEX ix_syscall_log_tenant_agent ON kernel.syscall_log (tenant_id, agent_urn, created_at DESC);
CREATE INDEX ix_syscall_log_trace        ON kernel.syscall_log (trace_id);
CREATE INDEX ix_syscall_log_deny         ON kernel.syscall_log (tenant_id, decision) WHERE decision = 'deny';

ALTER TABLE kernel.syscall_log ENABLE ROW LEVEL SECURITY;
ALTER TABLE kernel.syscall_log FORCE ROW LEVEL SECURITY;

CREATE POLICY syscall_log_tenant_isolation ON kernel.syscall_log
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

> `PRIMARY KEY (id, created_at)` é exigido pelo PostgreSQL para tabelas
> particionadas por `RANGE (created_at)` — a chave de partição DEVE fazer
> parte de toda constraint única. `id` (ULID) sozinho já é globalmente único
> por construção (ordenável no tempo), então a combinação não compromete a
> unicidade lógica de negócio.

### 2.5 Tabela `kernel.outbox`

```sql
CREATE TABLE kernel.outbox (
    id            char(26)    NOT NULL CHECK (kernel.is_valid_ulid(id)),
    tenant_id     text        NOT NULL,
    subject       text        NOT NULL,
    event_type    text        NOT NULL,
    payload       jsonb       NOT NULL,
    published     boolean     NOT NULL DEFAULT false,
    publish_attempts int      NOT NULL DEFAULT 0,
    last_attempt_at  timestamptz NULL,
    created_at    timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT pk_outbox PRIMARY KEY (id)
);

-- Índice parcial: o relay só varre mensagens pendentes (hot path pequeno,
-- mesmo com milhões de linhas históricas na tabela).
CREATE INDEX ix_outbox_pending ON kernel.outbox (created_at)
    WHERE published = false;

ALTER TABLE kernel.outbox ENABLE ROW LEVEL SECURITY;
ALTER TABLE kernel.outbox FORCE ROW LEVEL SECURITY;

CREATE POLICY outbox_tenant_isolation ON kernel.outbox
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

### 2.6 Tabela `kernel.idempotency_result`

```sql
CREATE TABLE kernel.idempotency_result (
    tenant_id         text        NOT NULL,
    idempotency_key   text        NOT NULL,
    request_hash      text        NOT NULL,  -- SHA-256 do corpo normalizado
    verb              text        NOT NULL,
    http_status       int         NOT NULL,
    response_body     jsonb       NOT NULL,
    created_at        timestamptz NOT NULL DEFAULT now(),
    expires_at        timestamptz NOT NULL,

    CONSTRAINT pk_idempotency PRIMARY KEY (tenant_id, idempotency_key)
);

CREATE INDEX ix_idempotency_expires ON kernel.idempotency_result (expires_at);

ALTER TABLE kernel.idempotency_result ENABLE ROW LEVEL SECURITY;
ALTER TABLE kernel.idempotency_result FORCE ROW LEVEL SECURITY;

CREATE POLICY idempotency_tenant_isolation ON kernel.idempotency_result
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

> **Nota:** o `IdempotencyStore` (componente do brief §2.1) mantém uma
> **projeção quente em Redis** (`idemp:{tenant}:{key}` → resultado serializado,
> TTL = `kernel.idempotency.retention_h`) para o caminho de leitura de alta
> frequência; esta tabela é a **fonte durável** usada em *cache miss* e para
> auditoria/replay além do TTL do Redis.

### 2.7 Tabela `kernel.schema_migrations`

```sql
CREATE TABLE kernel.schema_migrations (
    version       text        NOT NULL,
    description   text        NOT NULL,
    applied_at    timestamptz NOT NULL DEFAULT now(),
    checksum      text        NOT NULL,

    CONSTRAINT pk_schema_migrations PRIMARY KEY (version)
);
```

---

## 3. Índices — resumo consolidado

| Tabela | Índice | Tipo | Finalidade |
|--------|--------|------|------------|
| `acb` | `pk_acb (urn)` | B-tree (PK) | Lookup direto por URN (caminho quente). |
| `acb` | `uq_acb_tenant_agent` | B-tree único | Garantir unicidade lógica `(tenant, agent_id)`. |
| `acb` | `ix_acb_tenant_state` | B-tree composto | Consultas administrativas por estado (ex.: "todos os `Running` do tenant"). |
| `acb` | `ix_acb_shard_key` | B-tree | Suporte a roteamento por shard lógico de aplicação. |
| `acb` | `ix_acb_tenant_last_active` (parcial) | B-tree parcial | Varredura periódica de candidatos à hibernação (T-08), restrita a `Suspended`. |
| `acb` | `ix_acb_parent` (parcial) | B-tree parcial | Navegação da árvore de `spawn` (agentes filhos). |
| `acb` | `ix_acb_labels_gin` | GIN (jsonb_path_ops) | Consultas por metadados livres (`labels`, ex. `slaClass`). |
| `quota` | `uq_quota_subject` | B-tree único | Um único conjunto de cotas por `(tenant, subject_urn, scope)`. |
| `syscall_log` | `ix_syscall_log_tenant_agent` | B-tree composto (DESC) | Auditoria/timeline por agente, mais recente primeiro. |
| `syscall_log` | `ix_syscall_log_trace` | B-tree | Correlação por `trace_id` (OTel). |
| `syscall_log` | `ix_syscall_log_deny` (parcial) | B-tree parcial | Análise de negações (segurança/observabilidade) sem varrer linhas `allow`. |
| `outbox` | `ix_outbox_pending` (parcial) | B-tree parcial | Relay varre só `published = false` — custo O(pendentes), não O(total). |
| `idempotency_result` | `ix_idempotency_expires` | B-tree | Job de expurgo por `expires_at`. |

---

## 4. Particionamento

| Tabela | Estratégia | Justificativa |
|--------|-----------|----------------|
| `kernel.acb` | **HASH(tenant_id)**, 16 partições. | Distribui I/O entre tenants sem depender do volume de cada um; independe do `shard_key` lógico de aplicação (que é usado para afinidade de cache/réplica, não para particionamento físico). Alinhado à meta de suportar ≥ 10⁶ ACBs (NFR-006) com manutenção paralela (VACUUM/REINDEX por partição). |
| `kernel.syscall_log` | **RANGE(created_at)**, mensal. | Log de alto volume e vida útil limitada; permite expurgo por `DROP PARTITION` (O(1), sem `DELETE` em massa) alinhado à política de retenção (§6). |
| `kernel.outbox` | Não particionada (índice parcial supre o caso de uso). | Volume por réplica é limitado pelo relay (`kernel.outbox.relay_interval_ms`); registros publicados são arquivados/purgados periodicamente, mantendo a tabela pequena na prática. |
| `kernel.quota` | Não particionada. | Cardinalidade baixa (uma linha por tenant/agente com cota customizada; a maioria herda default). |
| `kernel.idempotency_result` | Não particionada (candidato a RANGE(created_at) se volume crescer além do previsto). | TTL curto (`kernel.idempotency.retention_h`, default 48h) mantém a tabela pequena via expurgo contínuo. |

**Criação de partições futuras.** Um job agendado (`kernel-partition-manager`,
executado via CronJob/`pg_partman`) cria a partição do mês seguinte de
`syscall_log` com **7 dias de antecedência**, evitando falha de inserção por
ausência de partição-alvo.

---

## 5. Row-Level Security (RLS) — política operacional

- Toda sessão de aplicação (pool de conexões do Kernel Service) **DEVE**
  executar `SET aios.tenant_id = '<tenant>'` (via `SET LOCAL` dentro da
  transação) imediatamente após obter a conexão do pool, com o valor extraído
  do claim autenticado — **nunca** de um parâmetro de URL não validado.
- `FORCE ROW LEVEL SECURITY` garante que **mesmo o owner da tabela** (a role
  de aplicação) está sujeito à política — não há bypass implícito por
  privilégio de owner.
- Roles administrativas de suporte (`kernel_dba_readonly`) usadas por
  operação/observabilidade **DEVEM** ter política RLS equivalente ou acesso
  somente via views agregadas sem PII, nunca `BYPASSRLS`.
- Testes de RLS (contract tests) **DEVEM** verificar que uma sessão com
  `aios.tenant_id = 'acme'` não enxerga nem consegue escrever linhas de
  `tenant_id = 'globex'`, para todas as 5 tabelas de negócio.

---

## 6. Retenção e expurgo

| Tabela | Política de retenção | Mecanismo |
|--------|------------------------|-----------|
| `kernel.acb` | Retida indefinidamente enquanto não há requisito legal de expurgo; estados terminais (`Terminated`/`Failed`) são candidatos a arquivamento frio após 180 dias (config futura `kernel.acb.archive_after_days`, não no MVP). | Nenhuma exclusão automática no MVP — auditoria depende do histórico. |
| `kernel.quota` | Vida útil do agente/tenant; removida apenas em expurgo de tenant (LGPD/GDPR, coordenado com `025-Audit`). | `ON DELETE RESTRICT` na FK de `acb` impede remoção acidental de cota em uso. |
| `kernel.syscall_log` | **90 dias** (default) via `DROP PARTITION` mensal fora da janela. | Job `kernel-partition-manager`; retenção configurável por tenant regulado (config futura). |
| `kernel.outbox` | Linhas com `published = true` e `created_at` > 7 dias são arquivadas/purgadas (job `kernel-outbox-gc`). | `DELETE` em lote com `LIMIT`/batching para evitar bloqueio longo. |
| `kernel.idempotency_result` | `expires_at` (= `created_at + kernel.idempotency.retention_h`, default 48h, mínimo 24h por RFC-0001 §5.5). | Job `kernel-idempotency-gc` remove expirados a cada 15 min. |
| `kernel.schema_migrations` | Permanente (histórico de auditoria de schema). | Sem expurgo. |

**Direito ao esquecimento (LGPD/GDPR).** Conforme `_DESIGN_BRIEF.md` §12.3, o
Kernel **NÃO** armazena conteúdo de memória (isso é de `010-Memory`, que
carrega base legal própria). O expurgo de um agente via `kill` **DEVE**
preservar apenas metadados mínimos de auditoria em `syscall_log`/`outbox`
conforme retenção legal, sem reter `memory_ptr`/`context_ptr` além do
necessário para rastreabilidade — a operação de purge é coordenada com
`025-Audit`.

---

## 7. Migrações

| Ferramenta | **DEVE** ser aplicada via *migration runner* versionado (ex.: Flyway/EF Core Migrations no control plane .NET 10), nunca `psql` manual em produção. |
|---|---|

| Arquivo | Conteúdo |
|---------|----------|
| `migrations/0001_init.sql` | Cria schema `kernel`, funções de validação, tabela `quota`, tabela `acb` (com 16 partições), índices, RLS. |
| `migrations/0002_syscall_log.sql` | Cria `syscall_log` particionada + partições dos 2 meses correntes + índices + RLS. |
| `migrations/0003_outbox.sql` | Cria `outbox` + índice parcial + RLS. |
| `migrations/0004_idempotency.sql` | Cria `idempotency_result` + índice + RLS. |
| `migrations/0005_schema_migrations.sql` | Cria tabela de controle de migração (bootstrap, normalmente criada primeiro pela ferramenta). |

**Regras de migração:**

- Toda migração **DEVE** ser aditiva e retrocompatível dentro de uma mesma
  major de API (RFC-0001 §5.7): novas colunas `NULL`-áveis ou com `DEFAULT`;
  nunca `DROP COLUMN`/`ALTER TYPE` destrutivo sem uma migração de duas fases
  (deprecate → backfill → remove em release subsequente).
- Alteração de `kernel.acb.shard_count` (config **não recarregável**, ver
  `Configuration.md`) exige migração de **reparticionamento lógico**
  controlada (recalcular `shard_key`, redistribuir chaves quentes no Redis) —
  documentada como procedimento operacional em `../029-Operations/`, não como
  `ALTER TABLE` trivial, pois `shard_key` é `GENERATED` a partir de constante
  `% 64` fixa no DDL acima (mudar `N` exige migração de dados, não apenas de
  schema).
- Toda migração **DEVE** ser testada em ambiente de staging com volume
  representativo antes de produção (ver `Testing.md`).
- Rollback: migrações destrutivas **DEVEM** ter script de reversão par
  (`_down.sql`); migrações aditivas puras não exigem rollback formal (basta
  não usar a nova coluna).

---

## 8. Relação com estado quente (Redis)

> O PostgreSQL é a **fonte da verdade**; Redis é **projeção quente** — nunca
> o inverso. Em caso de divergência, PostgreSQL vence.

| Dado | Chave Redis | TTL | Reconciliação |
|------|-------------|-----|-----------------|
| ACB (`Running`/`Suspended`) | `acb:{tenant}:{agent_id}` (hash) | sem TTL fixo — invalidado em toda mutação; recarregado *lazy* em cache-miss | Leitura *read-your-writes*: toda escrita no PG atualiza (ou invalida) a chave na mesma transação lógica da aplicação. |
| Contador de cota (token-bucket) | `quota:{tenant}:{subject_urn}:{dimensão}` | = `window` da cota (ex.: 1h) | Script Lua atômico incrementa/decrementa; reconciliação periódica contra `kernel.quota` para limites (não para saldo consumido, que é autoritativo no Redis durante a janela). |
| Resultado idempotente | `idemp:{tenant}:{key}` | `kernel.idempotency.retention_h` | Fallback para `kernel.idempotency_result` em cache-miss. |
| Lock distribuído de transição de FSM | `lock:acb:{urn}` | curto (ex.: 2s) | Usado apenas quando OCC sozinho não basta (concorrência de alta contenção pontual); preferência é sempre por OCC (`version`). |

---

## 9. Referências

- Modelo de dados canônico (fonte): `./_DESIGN_BRIEF.md` §3.
- Máquina de estados que impõe as constraints de `state`/`terminated_at`/`runtime_ref`: `./StateMachine.md`.
- Estratégia de sharding e concorrência (contexto arquitetural): `./Scalability.md`.
- Convenções de identidade (URN) e idempotência: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.1, §5.5.
- Padrões físicos gerais do banco AIOS (pgvector/AGE, convenções): `../005-Database/`.
- Modos de falha relacionados a persistência/Outbox: `./FailureRecovery.md`.
