---
Documento: Database
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0005, ADR-0200, ADR-0201, ADR-0203, ADR-0205, ADR-0207
RFCs relacionados: RFC-0001, RFC-0200
Depende de: _DESIGN_BRIEF.md §3, 005-Database (convenções físicas), StateMachine.md
---

# 020-Communication — Modelo Físico

Schema **`comm`** no PostgreSQL 16, sob as convenções físicas obrigatórias de
`../005-Database/Database.md` §2 (CV-01..CV-12): PK em URN, `tenant_id` + RLS em toda
tabela multi-tenant, `version bigint` para OCC, `timestamptz` em UTC, índice em toda FK.

> **Princípio deste modelo.** O **estado de runtime** do barramento (streams
> materializados, consumidores conectados, mensagens em voo) vive no **NATS**. O
> PostgreSQL guarda a **declaração** e o **histórico auditável**. A única exceção é a
> DLQ — e ela existe justamente porque aquelas mensagens **não** foram entregues.
> O barramento **não é banco**: nenhuma tabela aqui armazena tráfego normal.

---

## 1. Preparação

```sql
CREATE SCHEMA IF NOT EXISTS comm;

-- Variável de sessão usada pelas políticas de RLS (padrão do 005)
--   SET LOCAL app.tenant_id = 'acme';
```

Entidades de plataforma (`subject_registry`, `stream`, `consumer`) usam o tenant
reservado **`_platform`** e **não** são multi-tenant. As entidades que se referem a um
tenant real (`a2a_session`, `group_channel`, `dlq_entry`) **são** multi-tenant e
carregam RLS.

---

## 2. `comm.subject_registry`

```sql
CREATE TABLE comm.subject_registry (
    urn              text        PRIMARY KEY,
    domain           text        NOT NULL,
    entity           text        NOT NULL,
    action           text        NOT NULL,
    event_type       text        NOT NULL,
    producer_module  text        NOT NULL,
    dataschema       text        NOT NULL,
    stream_ref       uuid        NOT NULL REFERENCES comm.stream(id),
    traffic_class    text        NOT NULL
                     CHECK (traffic_class IN ('control','telemetry','bulk','a2a')),
    platform_scoped  boolean     NOT NULL DEFAULT false,
    deprecated_at    timestamptz,
    version          bigint      NOT NULL DEFAULT 0,
    created_at       timestamptz NOT NULL DEFAULT now(),
    updated_at       timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_subject UNIQUE (domain, entity, action),
    CONSTRAINT ck_event_type_shape
        CHECK (event_type = 'aios.' || domain || '.' || entity || '.' || action),
    CONSTRAINT ck_name_shape
        CHECK (domain ~ '^[a-z][a-z0-9]*$' AND entity ~ '^[a-z][a-z0-9]*$'
               AND action ~ '^[a-z][a-z0-9]*$')
);

CREATE INDEX ix_subject_producer ON comm.subject_registry (producer_module);
CREATE INDEX ix_subject_class    ON comm.subject_registry (traffic_class);
CREATE INDEX ix_subject_stream   ON comm.subject_registry (stream_ref);
```

`ck_event_type_shape` codifica no banco a coerência entre subject e `type` exigida pela
RFC-0001 §5.2–§5.3. `ck_name_shape` impede, entre outras coisas, que um ULID apareça em
um componente do subject — ou seja, impede **subject por instância** (regra RN-04 de
`./FunctionalRequirements.md`), que é a principal causa de explosão de cardinalidade em
barramentos.

---

## 3. `comm.stream`

```sql
CREATE TABLE comm.stream (
    id            uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    name          text        NOT NULL UNIQUE,
    subjects      text[]      NOT NULL,
    retention     text        NOT NULL CHECK (retention IN ('limits','interest','workqueue')),
    max_age       interval    NOT NULL,
    max_bytes     bigint      NOT NULL CHECK (max_bytes > 0),
    replicas      smallint    NOT NULL CHECK (replicas BETWEEN 1 AND 5),
    discard       text        NOT NULL CHECK (discard IN ('old','new')),
    dedupe_window interval    NOT NULL DEFAULT '2 minutes',
    owner_module  text        NOT NULL,
    state         text        NOT NULL
                  CHECK (state IN ('declared','active','draining','retired')),
    version       bigint      NOT NULL DEFAULT 0,
    created_at    timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT ck_subjects_not_empty CHECK (array_length(subjects, 1) >= 1)
);

CREATE INDEX ix_stream_owner ON comm.stream (owner_module);
CREATE INDEX ix_stream_state ON comm.stream (state);
```

**Não-sobreposição de subjects** (invariante C-02 de `./ClassDiagrams.md`) não é
expressável como `CHECK` de linha: é validada pelo `StreamManager` no `PUT`, consultando
os demais streams `active`. Uma mensagem capturada por dois streams seria entregue duas
vezes e teria duas políticas de retenção conflitantes.

### 3.1 Streams de referência

| Stream | Subjects | Retenção | Réplicas | Dono |
|--------|----------|----------|----------|------|
| `KERNEL_LIFECYCLE` | `aios.*.agent.lifecycle.>` | 30 d | 3 | `../006-Kernel/` |
| `KERNEL_QUOTA` | `aios.*.agent.quota.>` | 7 d | 3 | `../006-Kernel/` |
| `SCHED_DECISIONS` | `aios.*.task.execution.>`, `aios.*.scheduler.>` | 7 d | 3 | `../009-Scheduler/` |
| `MEMORY_EVENTS` | `aios.*.memory.>`, `aios.*.context.>` | 7 d | 3 | `../010-Memory/` |
| `POLICY_DECISIONS` | `aios.*.policy.>` | 30 d | 3 | `../022-Policy/` |
| `AUDIT_TRAIL` | `aios.*.audit.>` | 7 anos | 3 | `../025-Audit/` |
| `DB_PLATFORM` | `aios._platform.database.>` | 1 ano | 3 | `../005-Database/` |
| `COMM_PLATFORM` | `aios._platform.comm.subject.>`, `…stream.>`, `…account.>` | 1 ano | 3 | **020** |
| `COMM_HEALTH` | `aios._platform.comm.cluster.>`, `aios.*.comm.delivery.>`, `aios.*.comm.consumer.>`, `aios.*.comm.quota.>` | 30 d | 3 | **020** |
| `COMM_A2A` | `aios.*.comm.a2a.>` | 90 d | 3 | **020** |

A retenção de 7 anos do `AUDIT_TRAIL` é a exceção que confirma a regra de §0: o
barramento não é arquivo — quem precisa de retenção legal é o `../025-Audit/`, e ele a
declara explicitamente para o seu próprio stream.

---

## 4. `comm.consumer`

```sql
CREATE TABLE comm.consumer (
    id              uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       uuid        NOT NULL REFERENCES comm.stream(id) ON DELETE CASCADE,
    durable_name    text        NOT NULL,
    consumer_module text        NOT NULL,
    filter_subject  text,
    ack_policy      text        NOT NULL DEFAULT 'explicit'
                    CHECK (ack_policy IN ('explicit','none','all')),
    ack_wait        interval    NOT NULL DEFAULT '30 seconds',
    max_deliver     smallint    NOT NULL DEFAULT 5 CHECK (max_deliver BETWEEN 1 AND 100),
    backoff         interval[]  NOT NULL DEFAULT '{1 second,5 seconds,30 seconds,2 minutes}',
    max_ack_pending integer     NOT NULL DEFAULT 1000 CHECK (max_ack_pending BETWEEN 1 AND 100000),
    dlq_subject     text        NOT NULL,
    version         bigint      NOT NULL DEFAULT 0,
    created_at      timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_consumer UNIQUE (stream_id, durable_name),
    CONSTRAINT ck_backoff_fits CHECK (array_length(backoff, 1) <= max_deliver)
);

CREATE INDEX ix_consumer_stream ON comm.consumer (stream_id);
CREATE INDEX ix_consumer_module ON comm.consumer (consumer_module);
```

---

## 5. `comm.a2a_session` (multi-tenant, RLS)

```sql
CREATE TABLE comm.a2a_session (
    urn              text        PRIMARY KEY,
    tenant_id        text        NOT NULL,
    initiator_urn    text        NOT NULL,
    peer_urn         text        NOT NULL,
    peer_kind        text        NOT NULL CHECK (peer_kind IN ('internal','external')),
    state            text        NOT NULL
                     CHECK (state IN ('Requested','Negotiating','Established',
                                      'Suspended','Closing','Closed','Rejected','Failed')),
    channel_subject  text        NOT NULL,
    capabilities     text[]      NOT NULL DEFAULT '{}',
    message_count    bigint      NOT NULL DEFAULT 0,
    bytes_total      bigint      NOT NULL DEFAULT 0,
    idempotency_key  text,
    close_reason     text,
    version          bigint      NOT NULL DEFAULT 0,
    created_at       timestamptz NOT NULL DEFAULT now(),
    last_activity_at timestamptz NOT NULL DEFAULT now(),
    closed_at        timestamptz,
    CONSTRAINT uq_a2a_idem UNIQUE (tenant_id, idempotency_key),
    CONSTRAINT ck_terminal_closed_at
        CHECK ((state IN ('Closed','Rejected','Failed')) = (closed_at IS NOT NULL))
);

-- Invariante I1: canal exclusivo entre sessões não-terminais
CREATE UNIQUE INDEX uq_a2a_channel_active ON comm.a2a_session (channel_subject)
    WHERE state NOT IN ('Closed','Rejected','Failed');

CREATE INDEX ix_a2a_tenant_state ON comm.a2a_session (tenant_id, state);
CREATE INDEX ix_a2a_initiator    ON comm.a2a_session (tenant_id, initiator_urn);
CREATE INDEX ix_a2a_idle         ON comm.a2a_session (tenant_id, last_activity_at)
    WHERE state IN ('Established','Suspended');

ALTER TABLE comm.a2a_session ENABLE ROW LEVEL SECURITY;
ALTER TABLE comm.a2a_session FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_a2a_tenant ON comm.a2a_session
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

O índice único parcial `uq_a2a_channel_active` implementa a invariante I1 no banco: dois
canais idênticos simultâneos tornam-se impossíveis, e não apenas improváveis. O índice
parcial `ix_a2a_idle` sustenta a varredura de ociosidade (T-08) em O(sessões ativas),
não em O(histórico).

---

## 6. `comm.group_channel` (multi-tenant, RLS)

```sql
CREATE TABLE comm.group_channel (
    urn              text        PRIMARY KEY,
    tenant_id        text        NOT NULL,
    group_ref        text        NOT NULL,
    subject_prefix   text        NOT NULL,
    fanout_limit     integer     NOT NULL DEFAULT 256 CHECK (fanout_limit BETWEEN 1 AND 10000),
    scope            text        NOT NULL DEFAULT 'group'
                     CHECK (scope IN ('group','hierarchy','broadcast')),
    rate_limit_msg_s integer     NOT NULL DEFAULT 100 CHECK (rate_limit_msg_s > 0),
    version          bigint      NOT NULL DEFAULT 0,
    created_at       timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_group_prefix UNIQUE (tenant_id, subject_prefix)
);

CREATE INDEX ix_group_ref ON comm.group_channel (tenant_id, group_ref);

ALTER TABLE comm.group_channel ENABLE ROW LEVEL SECURITY;
ALTER TABLE comm.group_channel FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_group_tenant ON comm.group_channel
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

A **composição** do *Agent Group* (quem são os membros) pertence a
`../008-Agent-Lifecycle/` e `../009-Scheduler/`; aqui guarda-se apenas a referência e
os parâmetros de comunicação.

---

## 7. `comm.dlq_entry` (multi-tenant, RLS, particionada)

```sql
CREATE TABLE comm.dlq_entry (
    id               char(26)    NOT NULL,
    tenant_id        text        NOT NULL,
    original_subject text        NOT NULL,
    stream_name      text        NOT NULL,
    consumer_name    text        NOT NULL,
    delivery_count   smallint    NOT NULL,
    last_error       text        NOT NULL,
    envelope         jsonb       NOT NULL,
    status           text        NOT NULL DEFAULT 'quarantined'
                     CHECK (status IN ('quarantined','replayed','discarded')),
    replayed_at      timestamptz,
    created_at       timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE INDEX ix_dlq_tenant_status ON comm.dlq_entry (tenant_id, status);
CREATE INDEX ix_dlq_stream_time   ON comm.dlq_entry (stream_name, created_at DESC);

ALTER TABLE comm.dlq_entry ENABLE ROW LEVEL SECURITY;
ALTER TABLE comm.dlq_entry FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_dlq_tenant ON comm.dlq_entry
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));

-- Partição diária, pré-criada pelo PartitionManager do 005
CREATE TABLE comm.dlq_entry_2026_07_22 PARTITION OF comm.dlq_entry
    FOR VALUES FROM ('2026-07-22') TO ('2026-07-23');
```

Retenção: **30 dias** (`bus.dlq.retention_days`), método `drop_partition`, declarada em
`platform.retention_policy` do `../005-Database/`.

---

## 8. `comm.outbox`

Instância do contrato canônico definido em `../005-Database/Database.md` §3.9, usada
pelo `EventEmitter` deste módulo para publicar seus próprios eventos com atomicidade:

```sql
CREATE TABLE comm.outbox (
    id         char(26)    PRIMARY KEY,          -- ULID = event.id
    tenant_id  text        NOT NULL,
    subject    text        NOT NULL,
    payload    jsonb       NOT NULL,
    published  boolean     NOT NULL DEFAULT false,
    created_at timestamptz NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX ix_comm_outbox_pending ON comm.outbox (created_at) WHERE published = false;
```

Há uma circularidade aparente — o barramento usa o barramento para publicar seus
eventos — resolvida pelo Outbox: o estado é commitado no PostgreSQL primeiro, e a
publicação é uma consequência assíncrona. Se o NATS estiver indisponível, os eventos de
plataforma do próprio módulo aguardam no Outbox e são publicados ao religar.

---

## 9. Classificação e Retenção

| Tabela | `data_class` | Multi-tenant | Retenção | Método |
|--------|--------------|--------------|----------|--------|
| `comm.subject_registry` | `internal` | não | permanente | — (catálogo) |
| `comm.stream` | `internal` | não | permanente | — (histórico de declaração) |
| `comm.consumer` | `internal` | não | permanente | — |
| `comm.a2a_session` | `confidential` | **sim** | 90 dias | `delete_batch` |
| `comm.group_channel` | `internal` | **sim** | enquanto existir o grupo | — |
| `comm.dlq_entry` | `confidential` | **sim** | 30 dias | `drop_partition` |
| `comm.outbox` | `internal` | **sim** | 7 dias | `drop_partition` |

`comm.a2a_session` é `confidential` porque o **grafo de colaboração** entre agentes é
informação sensível de negócio, mesmo sem conter o conteúdo das mensagens: saber quem
fala com quem, com que frequência e sobre quais capacidades já revela muito.

`comm.dlq_entry` é `confidential` porque preserva o **envelope** de mensagens reais.
Por isso a retenção é curta e o `last_error` **NÃO DEVE** conter PII
(`./Logging.md` §5).

---

## 10. Consultas Operacionais Frequentes

```sql
-- Sessões A2A ativas há mais tempo sem atividade (candidatas a T-08)
SELECT urn, initiator_urn, peer_urn,
       now() - last_activity_at AS idle
  FROM comm.a2a_session
 WHERE state IN ('Established','Suspended')
   AND last_activity_at < now() - interval '5 minutes'
 ORDER BY idle DESC
 LIMIT 50;

-- Padrões de falha na DLQ nas últimas 24h (base para diagnóstico antes do replay)
SELECT stream_name, consumer_name, left(last_error, 80) AS erro, count(*)
  FROM comm.dlq_entry
 WHERE created_at > now() - interval '24 hours'
   AND status = 'quarantined'
 GROUP BY 1, 2, 3
 ORDER BY count(*) DESC;

-- Subjects registrados sem stream ativo (defeito de declaração)
SELECT s.event_type, s.producer_module, st.name, st.state
  FROM comm.subject_registry s
  JOIN comm.stream st ON st.id = s.stream_ref
 WHERE st.state <> 'active';
```

---

## 11. Referências

- Brief: `./_DESIGN_BRIEF.md` §3
- Convenções físicas e RLS: `../005-Database/Database.md` §2 e §4
- FSM: `./StateMachine.md` · Eventos: `./Events.md` · API: `./API.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
