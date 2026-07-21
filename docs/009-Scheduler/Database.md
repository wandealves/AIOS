---
Documento: Database.md
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0093, ADR-0097 (a propor); ADR-0005 (herdada), ADR-0006 (herdada)
RFCs relacionados: RFC-0001 (baseline); RFC-0090 (a propor)
Depende de: 003-RFC/RFC-0001, 005-Database, 021-Security, 025-Audit, 027-Cluster
---

# 009-Scheduler — Database

> **Escopo.** Modelo físico de persistência do Scheduler: DDL PostgreSQL
> (fonte da verdade de decisão, event-sourced/append-only) e estruturas de
> estado quente em Redis (filas de prioridade, cotas, capacidade,
> idempotência). Convenções de URN, ULID e RLS por tenant **DEVEM** seguir
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.1 e o padrão geral de
> `../005-Database/Architecture.md`. Consistência absoluta com
> `./_DESIGN_BRIEF.md` §3.

---

## 1. Justificativa de tecnologia e não-uso de pgvector/AGE

O Scheduler **NÃO** realiza busca por similaridade semântica nem consultas de
grafo de conhecimento — essas capacidades são do domínio de 010-Memory/
018-Knowledge/019-GraphRAG (ADR-0005). Portanto, **`pgvector` e Apache AGE NÃO
são utilizados** neste módulo; o modelo físico é puramente relacional
(tabelas normalizadas + `jsonb` para estruturas semiestruturadas como
`feature_vector` e `placement_target`), otimizado para **escrita
append-only de alta frequência** (proveniência de decisão) e leitura por
`request_id`/`tenant_id`/janela temporal.

O **estado quente** (filas de prioridade, cotas, leases, capacidade) vive em
**Redis** (ADR-0006), fora do escopo de DDL relacional, mas documentado aqui
(§5) por ser contratual ao módulo — sem ele, o comportamento do Scheduler não
pode ser entendido nem operado.

| Armazenamento | Papel | Por quê |
|----------------|-------|---------|
| PostgreSQL | Fonte da verdade de **decisão** (event sourcing), idempotência de longo prazo, proveniência para auditoria (025) e treino (023). | Durabilidade, consistência transacional (outbox), consulta analítica. |
| Redis | Estado **quente**: filas ZSET, contadores de cota, leases, snapshot de capacidade. | Latência sub-ms exigida por NFR-001 (p99 ≤ 20 ms no caminho `Submit`). |

---

## 2. Esquema PostgreSQL — DDL

### 2.1 Schema e extensões

```sql
CREATE SCHEMA IF NOT EXISTS scheduler;

-- Apenas para geração de UUID auxiliar em índices internos; IDs de negócio
-- são ULID gerados na aplicação (RFC-0001 §5.1), armazenados como TEXT/CHAR(26).
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

> **Nota sobre ULID:** o AIOS padroniza ULID (26 caracteres, Base32 Crockford)
> como identificador de recursos. Este módulo armazena ULIDs como
> `CHAR(26)` (comparável lexicograficamente, ordenável por tempo de criação,
> sem necessidade de extensão adicional). `tenant_id` é `TEXT` (slug
> `[a-z0-9-]+`).

### 2.2 Tabela `scheduler.scheduling_request` — log de entrada (append-only)

```sql
CREATE TABLE scheduler.scheduling_request (
    request_id       CHAR(26)        NOT NULL,
    tenant_id        TEXT            NOT NULL,
    task_urn         TEXT            NOT NULL,
    agent_urn        TEXT            NOT NULL,
    service_class    TEXT            NOT NULL
        CHECK (service_class IN ('INTERACTIVE','BATCH','BACKGROUND','SYSTEM')),
    priority_hint    SMALLINT        NOT NULL DEFAULT 50
        CHECK (priority_hint BETWEEN 0 AND 100),
    deadline_at      TIMESTAMPTZ,
    est_cost_usd     NUMERIC(12,6)   CHECK (est_cost_usd >= 0),
    est_latency_ms   INTEGER         CHECK (est_latency_ms >= 0),
    est_tokens       BIGINT          CHECK (est_tokens >= 0),
    quality_floor    NUMERIC(4,3)    CHECK (quality_floor BETWEEN 0 AND 1),
    affinity         JSONB,
    payload_ref      TEXT,
    submitted_at     TIMESTAMPTZ     NOT NULL,
    traceparent      TEXT            NOT NULL,
    created_at       TIMESTAMPTZ     NOT NULL DEFAULT now(),

    CONSTRAINT pk_scheduling_request PRIMARY KEY (tenant_id, request_id),
    CONSTRAINT chk_task_urn_format  CHECK (task_urn  ~ '^urn:aios:[a-z0-9-]+:task:[0-9A-HJKMNP-TV-Z]{26}$'),
    CONSTRAINT chk_agent_urn_format CHECK (agent_urn ~ '^urn:aios:[a-z0-9-]+:agent:[0-9A-HJKMNP-TV-Z]{26}$')
) PARTITION BY RANGE (submitted_at);

-- Partição mensal (exemplo; automatizado via job de manutenção, ver §4)
CREATE TABLE scheduler.scheduling_request_2026_07
    PARTITION OF scheduler.scheduling_request
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

CREATE INDEX idx_sched_request_tenant_submitted
    ON scheduler.scheduling_request (tenant_id, submitted_at DESC);
CREATE INDEX idx_sched_request_task_urn
    ON scheduler.scheduling_request (task_urn);
CREATE INDEX idx_sched_request_deadline
    ON scheduler.scheduling_request (deadline_at)
    WHERE deadline_at IS NOT NULL;

ALTER TABLE scheduler.scheduling_request ENABLE ROW LEVEL SECURITY;
ALTER TABLE scheduler.scheduling_request FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_scheduling_request ON scheduler.scheduling_request
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

**Invariantes:** `request_id` é ULID gerado pelo cliente/gateway; a PK
composta `(tenant_id, request_id)` reforça o isolamento por tenant mesmo em
caso de colisão improvável de ULID entre tenants. Tabela **append-only** —
nenhuma aplicação DEVE emitir `UPDATE`/`DELETE` fora de expurgo de retenção
(§4) ou direito ao esquecimento (§4.3).

### 2.3 Tabela `scheduler.scheduling_decision` — proveniência (event-sourced, append-only)

```sql
CREATE TABLE scheduler.scheduling_decision (
    decision_id           CHAR(26)        NOT NULL,
    request_id            CHAR(26)        NOT NULL,
    tenant_id             TEXT            NOT NULL,
    outcome                TEXT            NOT NULL
        CHECK (outcome IN ('ADMITTED','SCHEDULED','REJECTED','PREEMPTED','RESUMED','DEFERRED')),
    effective_priority     INTEGER         NOT NULL,
    cost_score             NUMERIC(12,6)   NOT NULL,
    alpha                  NUMERIC(6,4)    NOT NULL CHECK (alpha  BETWEEN 0 AND 1),
    beta                   NUMERIC(6,4)    NOT NULL CHECK (beta   BETWEEN 0 AND 1),
    gamma                  NUMERIC(6,4)    NOT NULL CHECK (gamma  BETWEEN 0 AND 1),
    placement_target       JSONB,                       -- {shard,node_id,runtime_pool}; NULL se REJECTED/DEFERRED
    policy_id              TEXT            NOT NULL,
    policy_version         TEXT            NOT NULL,     -- ex.: 'heuristic@1' | 'rl@<hash>'
    feature_vector         JSONB,
    reason_code            TEXT,                         -- ex.: 'AIOS-SCHED-0001' quando REJECTED
    decided_at             TIMESTAMPTZ     NOT NULL DEFAULT now(),
    latency_decision_ms    NUMERIC(8,3)    NOT NULL CHECK (latency_decision_ms >= 0),

    CONSTRAINT pk_scheduling_decision PRIMARY KEY (tenant_id, decision_id),
    CONSTRAINT fk_scheduling_decision_request
        FOREIGN KEY (tenant_id, request_id)
        REFERENCES scheduler.scheduling_request (tenant_id, request_id)
) PARTITION BY RANGE (decided_at);

CREATE TABLE scheduler.scheduling_decision_2026_07
    PARTITION OF scheduler.scheduling_decision
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

CREATE INDEX idx_sched_decision_request
    ON scheduler.scheduling_decision (tenant_id, request_id, decided_at DESC);
CREATE INDEX idx_sched_decision_outcome
    ON scheduler.scheduling_decision (tenant_id, outcome, decided_at DESC);
CREATE INDEX idx_sched_decision_policy_version
    ON scheduler.scheduling_decision (policy_version, decided_at DESC);
-- Índice parcial para consultas rápidas de decisões recentes de erro (dashboards)
CREATE INDEX idx_sched_decision_rejected
    ON scheduler.scheduling_decision (tenant_id, decided_at DESC)
    WHERE outcome = 'REJECTED';
-- Índice GIN para consultas ad hoc sobre feature_vector (auditoria/treino 023)
CREATE INDEX idx_sched_decision_feature_gin
    ON scheduler.scheduling_decision USING GIN (feature_vector jsonb_path_ops);

ALTER TABLE scheduler.scheduling_decision ENABLE ROW LEVEL SECURITY;
ALTER TABLE scheduler.scheduling_decision FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_scheduling_decision ON scheduler.scheduling_decision
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

**Invariantes (RN-01..RN-04 do `_DESIGN_BRIEF.md` §3):** (i) toda linha
carrega `tenant_id` com RLS ativo; (ii) tabela **append-only** — uma nova
decisão para o mesmo `request_id` (re-schedule, preempção, resume) É UMA NOVA
LINHA, nunca um `UPDATE`; (iii) o *outbox* (`DecisionJournal`) grava esta
linha e publica o evento correspondente na mesma transação lógica (padrão
Outbox, ver `Events.md` §2); (iv) **uma `SchedulingRequest` NÃO DEVE possuir
mais de uma decisão `SCHEDULED` ativa simultaneamente** — invariante
verificada pela aplicação via lease Redis (§5.1) antes do `INSERT`, não por
constraint SQL (o histórico de `SCHEDULED`s ao longo do tempo é legítimo após
preempção/resume).

### 2.4 Tabela `scheduler.idempotency_record`

```sql
CREATE TABLE scheduler.idempotency_record (
    tenant_id          TEXT            NOT NULL,
    idempotency_key    TEXT            NOT NULL,
    request_id         CHAR(26)        NOT NULL,
    response_snapshot  JSONB           NOT NULL,
    created_at         TIMESTAMPTZ     NOT NULL DEFAULT now(),
    expires_at         TIMESTAMPTZ     NOT NULL,

    CONSTRAINT pk_idempotency_record PRIMARY KEY (tenant_id, idempotency_key)
);

CREATE INDEX idx_idempotency_expires
    ON scheduler.idempotency_record (expires_at);

ALTER TABLE scheduler.idempotency_record ENABLE ROW LEVEL SECURITY;
ALTER TABLE scheduler.idempotency_record FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_idempotency ON scheduler.idempotency_record
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

**Uso:** camada de persistência durável do `IdempotencyStore` — Redis é o
caminho quente (lookup sub-ms), PostgreSQL é o *fallback*/registro de longo
prazo quando Redis expira a chave antes do TTL de negócio (`expires_at` ≥
`scheduler.idempotency.ttl_hours`, default 24h). Expurgo por job (§4.2).

### 2.5 Tabela `scheduler.policy_weight` — governança de pesos (α, β, γ) por tenant/classe

```sql
CREATE TABLE scheduler.policy_weight (
    tenant_id       TEXT            NOT NULL,
    service_class   TEXT            NOT NULL
        CHECK (service_class IN ('INTERACTIVE','BATCH','BACKGROUND','SYSTEM','*')),
    alpha           NUMERIC(6,4)    NOT NULL CHECK (alpha BETWEEN 0 AND 1),
    beta            NUMERIC(6,4)    NOT NULL CHECK (beta  BETWEEN 0 AND 1),
    gamma           NUMERIC(6,4)    NOT NULL CHECK (gamma BETWEEN 0 AND 1),
    quality_floor   NUMERIC(4,3)    NOT NULL DEFAULT 0.7 CHECK (quality_floor BETWEEN 0 AND 1),
    engine          TEXT            NOT NULL DEFAULT 'heuristic'
        CHECK (engine IN ('heuristic','rl')),
    updated_by      TEXT            NOT NULL,           -- claim/ator autorizado pelo PDP (022)
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    version         BIGINT          NOT NULL DEFAULT 0,  -- controle de concorrência otimista

    CONSTRAINT pk_policy_weight PRIMARY KEY (tenant_id, service_class),
    CONSTRAINT chk_weights_normalized
        CHECK (abs((alpha + beta + gamma) - 1.0) <= 0.01)
);

ALTER TABLE scheduler.policy_weight ENABLE ROW LEVEL SECURITY;
ALTER TABLE scheduler.policy_weight FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_policy_weight ON scheduler.policy_weight
    USING (tenant_id = current_setting('aios.tenant_id', true));
```

`service_class = '*'` representa o *default* do tenant quando não há
sobreposição por classe. Toda escrita (`PUT /v1/scheduler/policy`, `API.md`
§3) passa por PEP→PDP (022) e é auditada (025); `version` habilita
*optimistic concurrency control* (OCC) para evitar corrida entre operadores.

### 2.6 Tabela `scheduler.policy_weight_history` — auditoria de mudança de pesos (append-only)

```sql
CREATE TABLE scheduler.policy_weight_history (
    tenant_id       TEXT            NOT NULL,
    service_class   TEXT            NOT NULL,
    alpha           NUMERIC(6,4)    NOT NULL,
    beta            NUMERIC(6,4)    NOT NULL,
    gamma           NUMERIC(6,4)    NOT NULL,
    quality_floor   NUMERIC(4,3)    NOT NULL,
    engine          TEXT            NOT NULL,
    changed_by      TEXT            NOT NULL,
    changed_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    previous_version BIGINT
) PARTITION BY RANGE (changed_at);

CREATE TABLE scheduler.policy_weight_history_2026_07
    PARTITION OF scheduler.policy_weight_history
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

CREATE INDEX idx_policy_weight_history_tenant
    ON scheduler.policy_weight_history (tenant_id, service_class, changed_at DESC);
```

Populada via *trigger* `AFTER UPDATE OR INSERT ON scheduler.policy_weight`
(implementação na aplicação/migração, não reproduzida aqui) — garante trilha
completa de quem mudou o quê e quando, consumida por 025-Audit.

### 2.7 Diagrama relacional (ASCII)

```
┌───────────────────────────┐        1        N ┌────────────────────────────┐
│ scheduling_request        │───────────────────▶│ scheduling_decision        │
│ PK (tenant_id,request_id) │                    │ PK (tenant_id,decision_id) │
│ task_urn, agent_urn       │                    │ FK (tenant_id,request_id)  │
│ service_class, deadline_at│                    │ outcome, effective_prio    │
└───────────────────────────┘                    │ placement_target(jsonb)    │
                                                  │ policy_id, policy_version  │
                                                  │ feature_vector (jsonb)     │
                                                  └────────────────────────────┘

┌────────────────────────────┐                   ┌────────────────────────────┐
│ idempotency_record         │                   │ policy_weight               │
│ PK (tenant_id,idem_key)    │                   │ PK (tenant_id,service_class)│
│ request_id, response_snap  │                   │ alpha,beta,gamma,engine     │
└────────────────────────────┘                   └──────────────┬─────────────┘
                                                                  │ trigger
                                                                  ▼
                                                  ┌────────────────────────────┐
                                                  │ policy_weight_history       │
                                                  │ (append-only, particionada) │
                                                  └────────────────────────────┘
```

---

## 3. Particionamento

| Tabela | Estratégia | Granularidade | Justificativa |
|--------|-----------|----------------|----------------|
| `scheduling_request` | `RANGE (submitted_at)` | Mensal | Alto volume de escrita (throughput ≥ 50k decisões/s agregadas, NFR-002); poda de partições antigas é O(1) (`DETACH PARTITION`). |
| `scheduling_decision` | `RANGE (decided_at)` | Mensal | Mesmo racional; é a tabela de maior volume (1 request pode gerar N decisões — admissão, preempção, resume). |
| `policy_weight_history` | `RANGE (changed_at)` | Mensal | Volume baixo, mas mantém consistência de padrão operacional com as demais. |
| `idempotency_record` | Sem particionamento (TTL curto, expurgo frequente por `expires_at`) | — | Volume limitado pelo TTL (24–168h); índice em `expires_at` é suficiente. |

Partições futuras (mês seguinte) DEVEM ser criadas com antecedência mínima de
7 dias por job agendado (`pg_partman` ou equivalente, operado por
`027-Cluster`/`005-Database`); ausência de partição futura é alarme de
severidade alta (ver `Monitoring.md`).

---

## 4. Políticas de retenção e expurgo

### 4.1 Retenção padrão

| Tabela | Retenção "quente" (partição ativa) | Retenção total antes de arquivar/expurgar | Destino pós-retenção |
|--------|--------------------------------------|---------------------------------------------|------------------------|
| `scheduling_request` | 1 mês corrente | 13 meses (alinhado a auditoria 025) | Arquivo frio (MinIO, formato Parquet) via job de exportação; partição então `DROP`. |
| `scheduling_decision` | 1 mês corrente | 13 meses | Idem — é a fonte primária de proveniência para 025/023; exportação precede o `DROP`. |
| `idempotency_record` | — | `expires_at` (24–168h, `scheduler.idempotency.ttl_hours`) | `DELETE` direto por job periódico (`DELETE ... WHERE expires_at < now()`), sem necessidade de arquivo frio. |
| `policy_weight_history` | 1 mês corrente | 24 meses (governança/compliance) | Arquivo frio; partição então `DROP`. |

### 4.2 Job de expurgo de idempotência (exemplo)

```sql
-- Executado periodicamente (ex.: a cada 15 min) por worker do Scheduler.
DELETE FROM scheduler.idempotency_record
WHERE expires_at < now()
LIMIT 10000; -- em lotes, para não competir com o caminho quente
```

### 4.3 Direito ao esquecimento (LGPD/GDPR)

Operação de expurgo por `tenant_id`/`agent_urn`, coordenada com 025-Audit:

```sql
-- Executado sob autorização auditada; nunca por chamada direta de API pública.
DELETE FROM scheduler.scheduling_decision
WHERE tenant_id = $1 AND request_id IN (
  SELECT request_id FROM scheduler.scheduling_request WHERE tenant_id = $1 AND agent_urn = $2
);
DELETE FROM scheduler.scheduling_request WHERE tenant_id = $1 AND agent_urn = $2;
DELETE FROM scheduler.idempotency_record WHERE tenant_id = $1
  AND request_id IN (SELECT request_id FROM scheduler.scheduling_request WHERE tenant_id=$1 AND agent_urn=$2);
```

`feature_vector` **NÃO DEVE** conter PII bruta (apenas features numéricas
derivadas); `payload_ref` aponta para MinIO e é expurgado pelo dono do dado
(007/015), não pelo Scheduler — este módulo não persiste payload de tarefa
(RFC-0001 §7, `_DESIGN_BRIEF.md` §12.3).

---

## 5. Redis — Estruturas de Estado Quente

> Fora do escopo de DDL relacional, mas **contratual** ao módulo — o
> comportamento do caminho quente (`Submit`, `Dispatch`, preempção) depende
> inteiramente destas estruturas. Prefixo de chave configurável
> (`scheduler.redis.zset_prefix`, default `sched`, ver `Configuration.md`).

### 5.1 Filas de prioridade — `sched:{tenant}:{class}:{shard}` (ZSET)

| Campo | Tipo | Notas |
|-------|------|-------|
| Chave | `sched:{tenant}:{class}:{shard}` | Uma ZSET por (tenant, classe, shard) — ver `Scalability.md`. |
| Membro | `task_urn` | Elemento da fila. |
| Score | `double` | `effective_priority` com componente de aging; menor valor = mais urgente (convenção EDF-aware). |
| Hash lateral | `sched:meta:{task_urn}` | `{request_id, deadline_at, submitted_at, lease_owner, lease_until}` — TTL amarrado a `scheduler.lease.ttl_ms`. |

```
ZADD sched:acme:INTERACTIVE:17 68 "urn:aios:acme:task:01J9ZA100000000000000010"
HSET sched:meta:urn:aios:acme:task:01J9ZA100000000000000010 \
     request_id 01J9ZA000000000000000010 \
     deadline_at "" \
     submitted_at "2026-07-20T12:00:00.000Z" \
     lease_owner "" \
     lease_until 0
```

### 5.2 Cotas — `sched:quota:*` (contadores atômicos)

| Chave | Tipo | Notas |
|-------|------|-------|
| `sched:quota:{tenant}:concurrency` | `INT` (atômico, script Lua) | Slots concorrentes em uso vs. `scheduler.max_concurrency.per_tenant`. |
| `sched:quota:{tenant}:{class}:inflight` | `INT` | Concorrência por classe (ex.: `scheduler.max_concurrency.per_class.INTERACTIVE`). |
| `sched:reservation:{request_id}` | `HASH` + TTL | Reserva atômica durante admissão; liberada em confirm (`INSERT` da decisão) ou abort (rollback). TTL de segurança = `scheduler.decision.timeout_ms` × fator de margem. |

Reserva/confirmação usa script Lua (CAS atômico) para impedir overcommit
(NFR-005): incremento condicional ao limite, com rollback automático por TTL
se a decisão não for confirmada.

### 5.3 Capacidade — `sched:capacity:*`

| Chave | Tipo | Notas |
|-------|------|-------|
| `sched:capacity:{shard}` | `HASH` | `{node_id, free_slots, pressure, updated_at}` — atualizado por heartbeat do Runtime Supervisor (`scheduler.capacity.heartbeat_ttl_ms`). |
| `sched:shard:owner:{shard}` | `STRING` + TTL (lease) | Réplica *owner* do shard para `DispatchLoop` (evita contenção cross-réplica, ver `Scalability.md`). |

### 5.4 Idempotência quente — `sched:idem:*`

| Chave | Tipo | Notas |
|-------|------|-------|
| `sched:idem:{tenant}:{idempotency_key}` | `STRING` (JSON) + TTL | Espelha `scheduler.idempotency_record` para lookup sub-ms; TTL = `scheduler.idempotency.ttl_hours`. Miss em Redis cai para consulta em PostgreSQL antes de tratar como chave nova. |

---

## 6. Migrações

- Ferramenta: migrações versionadas (`Flyway`/EF Core Migrations, a definir em
  `Deployment.md`), nomeadas `V<NNNN>__<descricao>.sql`, aplicadas de forma
  **aditiva e retrocompatível** (nunca `DROP COLUMN`/`ALTER TYPE` destrutivo
  sem janela de coexistência ≥ 2 versões, espelhando RFC-0001 §5.7).
- Toda migração DEVE ser idempotente (`IF NOT EXISTS`/`IF EXISTS`) e
  reversível (script de rollback correspondente).
- Criação de partições futuras é migração recorrente automatizada, não manual
  (ver §3).
- Mudança em `CHECK` constraints (ex.: novo `service_class`) exige RFC/ADR de
  módulo (consistência com `_DESIGN_BRIEF.md` §11) antes de aplicar.

---

## 7. Referências

- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.1, §5.7.
- Padrão de banco do AIOS: `../005-Database/Architecture.md`.
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §3.
- Superfície de API que lê/escreve estas tabelas: `./API.md`.
- Eventos publicados a partir do outbox: `./Events.md`.
- Chaves de configuração citadas (`scheduler.redis.zset_prefix`,
  `scheduler.idempotency.ttl_hours`, `scheduler.lease.ttl_ms`,
  `scheduler.max_concurrency.*`): `./Configuration.md`.
- Modos de falha de Redis/PostgreSQL: `./FailureRecovery.md`.
- Modelo de sharding/particionamento de carga: `./Scalability.md`.
