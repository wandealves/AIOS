---
Documento: Database
Módulo: 024-Observability
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 024-Observability
ADRs relacionados: ADR-0005, ADR-0241, ADR-0242, ADR-0245, ADR-0247, ADR-0248
RFCs relacionados: RFC-0001, RFC-0240, RFC-0241
Depende de: _DESIGN_BRIEF.md §3, 005-Database (convenções físicas CV-01..CV-12), StateMachine.md
---

# 024-Observability — Modelo Físico

Schema **`telemetry`** no PostgreSQL 16, sob as convenções físicas obrigatórias de
`../005-Database/Database.md` §2 (CV-01..CV-12).

> **Regra absoluta deste modelo.** O PostgreSQL guarda o **plano de controle da
> observabilidade** — descritores de sinal, orçamentos, SLOs, regras, alertas,
> silêncios, dashboards, runbooks e políticas de retenção. **Nenhuma série temporal,
> trace ou linha de log é armazenada aqui**: esses vivem no Prometheus, no Tempo/Jaeger
> e no Seq. Guardar telemetria bruta em banco relacional é o erro que transforma o
> observador no maior consumidor de I/O do sistema que ele deveria apenas observar.

---

## 1. Preparação

```sql
CREATE SCHEMA IF NOT EXISTS telemetry;

-- Variável de sessão usada pelas políticas de RLS (padrão do 005)
--   SET LOCAL app.tenant_id = 'acme';
```

Tabelas multi-tenant (RLS obrigatória): `cardinality_budget`, `slo_definition`,
`error_budget_ledger`, `alert_rule`, `alert_instance`, `silence`, `dashboard`,
`runbook`. Tabelas de plataforma: `signal_descriptor`, `retention_policy` (usam o
tenant reservado `_platform`).

---

## 2. `telemetry.signal_descriptor` (plataforma)

Registro canônico de todo sinal admitido — a entidade que torna a telemetria um
contrato declarado (ADR-0241).

```sql
CREATE TABLE telemetry.signal_descriptor (
    urn                text        PRIMARY KEY,
    signal_kind        text        NOT NULL
                       CHECK (signal_kind IN ('metric','span','log_event')),
    name               text        NOT NULL,
    descriptor_version int         NOT NULL DEFAULT 1,
    owner_module       text        NOT NULL,
    metric_type        text        CHECK (metric_type IN ('counter','gauge','histogram','updown_counter')),
    unit               text,
    allowed_labels     text[]      NOT NULL DEFAULT '{}',
    forbidden_labels   text[]      NOT NULL DEFAULT '{}',
    bucket_boundaries  numeric[],
    max_series         int         NOT NULL DEFAULT 10000 CHECK (max_series > 0),
    data_class         text        NOT NULL DEFAULT 'operational'
                       CHECK (data_class IN ('operational','pii','sensitive')),
    retention_tier     text        NOT NULL
                       CHECK (retention_tier IN ('raw','short','medium','long')),
    state              text        NOT NULL DEFAULT 'Proposed'
                       CHECK (state IN ('Proposed','Registered','Quarantined',
                                        'Rejected','Deprecated','Retired')),
    version            bigint      NOT NULL DEFAULT 0,
    created_at         timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_signal_name    UNIQUE (name, descriptor_version),
    CONSTRAINT ck_metric_fields  CHECK (signal_kind <> 'metric'
                                        OR (metric_type IS NOT NULL AND unit IS NOT NULL)),
    CONSTRAINT ck_buckets_hist   CHECK (metric_type IS DISTINCT FROM 'histogram'
                                        OR bucket_boundaries IS NOT NULL)
);

CREATE INDEX ix_signal_owner  ON telemetry.signal_descriptor (owner_module, state);
CREATE INDEX ix_signal_kind   ON telemetry.signal_descriptor (signal_kind, retention_tier);
CREATE INDEX ix_signal_quar   ON telemetry.signal_descriptor (state)
    WHERE state = 'Quarantined';
```

- `ck_metric_fields` impede métrica sem tipo ou sem unidade: uma série sem unidade é
  um número que ninguém consegue comparar entre serviços.
- `ck_buckets_hist` impede histograma sem buckets explícitos — buckets automáticos
  não caem nas fronteiras de SLO, e `histogram_quantile` sobre eles produz um número
  que **parece** um p99.
- `allowed_labels` é uma **lista fechada**: é o único ponto do sistema em que a
  explosão de cardinalidade pode ser impedida **antes** de acontecer. Depois que a
  série existe, o custo já foi pago.

---

## 3. `telemetry.cardinality_budget` (multi-tenant, RLS)

```sql
CREATE TABLE telemetry.cardinality_budget (
    urn                text        PRIMARY KEY,
    tenant_id          text        NOT NULL,
    scope              text        NOT NULL CHECK (scope IN ('tenant','service')),
    scope_ref          text        NOT NULL DEFAULT '*',
    max_active_series  bigint      NOT NULL DEFAULT 1000000 CHECK (max_active_series > 0),
    current_series     bigint      NOT NULL DEFAULT 0,
    soft_threshold     numeric(4,3) NOT NULL DEFAULT 0.800
                       CHECK (soft_threshold > 0 AND soft_threshold <= 1),
    hard_action        text        NOT NULL DEFAULT 'quarantine'
                       CHECK (hard_action IN ('quarantine','drop_labels','reject')),
    evaluated_at       timestamptz NOT NULL DEFAULT now(),
    version            bigint      NOT NULL DEFAULT 0,
    created_at         timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_budget_scope UNIQUE (tenant_id, scope, scope_ref)
);

CREATE INDEX ix_budget_eval ON telemetry.cardinality_budget (tenant_id, evaluated_at DESC);

ALTER TABLE telemetry.cardinality_budget ENABLE ROW LEVEL SECURITY;
ALTER TABLE telemetry.cardinality_budget FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_budget_tenant ON telemetry.cardinality_budget
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

O contador quente (`current_series`) vive em Redis; esta coluna é a **projeção
periódica** para consulta e histórico — não é a fonte de verdade do caminho quente.

---

## 4. `telemetry.slo_definition` (multi-tenant, RLS)

```sql
CREATE TABLE telemetry.slo_definition (
    urn           text        PRIMARY KEY,
    tenant_id     text        NOT NULL,
    slo_key       text        NOT NULL,
    slo_version   int         NOT NULL DEFAULT 1,
    owner_module  text        NOT NULL,
    sli_query     text        NOT NULL,
    objective     numeric(6,4) NOT NULL CHECK (objective > 0 AND objective < 1),
    window        interval    NOT NULL DEFAULT '30 days',
    budget_policy jsonb       NOT NULL DEFAULT '{}'::jsonb,
    status        text        NOT NULL DEFAULT 'draft'
                  CHECK (status IN ('draft','active','deprecated')),
    version       bigint      NOT NULL DEFAULT 0,
    created_at    timestamptz NOT NULL DEFAULT now(),
    updated_at    timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_slo_version UNIQUE (tenant_id, slo_key, slo_version)
);

CREATE UNIQUE INDEX uq_slo_one_active
    ON telemetry.slo_definition (tenant_id, slo_key)
    WHERE status = 'active';

CREATE INDEX ix_slo_owner ON telemetry.slo_definition (tenant_id, owner_module, status);

ALTER TABLE telemetry.slo_definition ENABLE ROW LEVEL SECURITY;
ALTER TABLE telemetry.slo_definition FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_slo_tenant ON telemetry.slo_definition
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

`objective` é `CHECK (> 0 AND < 1)` por desenho: um SLO de 100% não deixa error budget
— e um serviço sem error budget não pode fazer nenhuma mudança, o que na prática
significa que o objetivo será ignorado.

---

## 5. `telemetry.error_budget_ledger` (multi-tenant, RLS, particionada)

```sql
CREATE TABLE telemetry.error_budget_ledger (
    id              char(26)     NOT NULL,
    tenant_id       text         NOT NULL,
    slo_urn         text         NOT NULL,
    evaluated_at    timestamptz  NOT NULL DEFAULT now(),
    sli_value       numeric(9,6) NOT NULL,
    budget_consumed numeric(6,4) NOT NULL,
    burn_rate_1h    numeric(9,4) NOT NULL,
    burn_rate_6h    numeric(9,4) NOT NULL,
    breached        boolean      NOT NULL DEFAULT false,
    CONSTRAINT pk_budget_ledger PRIMARY KEY (tenant_id, evaluated_at, id)
) PARTITION BY RANGE (evaluated_at);

CREATE TABLE telemetry.error_budget_ledger_2026_07
    PARTITION OF telemetry.error_budget_ledger
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

CREATE INDEX ix_ledger_slo     ON telemetry.error_budget_ledger (tenant_id, slo_urn, evaluated_at DESC);
CREATE INDEX ix_ledger_breach  ON telemetry.error_budget_ledger (breached, evaluated_at DESC);

ALTER TABLE telemetry.error_budget_ledger ENABLE ROW LEVEL SECURITY;
ALTER TABLE telemetry.error_budget_ledger FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_ledger_tenant ON telemetry.error_budget_ledger
    USING (tenant_id = current_setting('app.tenant_id', true));
```

O ledger é **append-only**: alterar um lançamento passado reescreveria a história de
conformidade do serviço. Correções entram como novo lançamento.

---

## 6. `telemetry.alert_rule` e `telemetry.alert_instance` (multi-tenant, RLS)

```sql
CREATE TABLE telemetry.alert_rule (
    urn             text        PRIMARY KEY,
    tenant_id       text        NOT NULL,
    rule_key        text        NOT NULL,
    expr            text        NOT NULL,
    for_duration    interval    NOT NULL DEFAULT '5 minutes',
    severity        text        NOT NULL CHECK (severity IN ('critical','warning','info')),
    slo_urn         text        REFERENCES telemetry.slo_definition (urn),
    runbook_urn     text        NOT NULL REFERENCES telemetry.runbook (urn),
    owner_module    text        NOT NULL,
    notify_channels text[]      NOT NULL DEFAULT '{}',
    status          text        NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','disabled')),
    version         bigint      NOT NULL DEFAULT 0,
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_rule_key UNIQUE (tenant_id, rule_key)
);

CREATE INDEX ix_rule_sev     ON telemetry.alert_rule (tenant_id, severity, status);
CREATE INDEX ix_rule_runbook ON telemetry.alert_rule (runbook_urn);   -- CV-05
CREATE INDEX ix_rule_slo     ON telemetry.alert_rule (slo_urn);       -- CV-05

CREATE TABLE telemetry.alert_instance (
    id               char(26)    NOT NULL,
    tenant_id        text        NOT NULL,
    started_at       timestamptz NOT NULL DEFAULT now(),
    rule_urn         text        NOT NULL REFERENCES telemetry.alert_rule (urn),
    fingerprint      text        NOT NULL,
    state            text        NOT NULL DEFAULT 'Pending'
                     CHECK (state IN ('Pending','Firing','Acknowledged',
                                      'Silenced','Resolved','Expired')),
    severity         text        NOT NULL,
    labels           jsonb       NOT NULL DEFAULT '{}'::jsonb,
    value            numeric,
    notified_at      timestamptz,
    acknowledged_by  text,
    acknowledged_at  timestamptz,
    silenced_until   timestamptz,
    resolved_at      timestamptz,
    escalation_level int         NOT NULL DEFAULT 0 CHECK (escalation_level >= 0),
    version          bigint      NOT NULL DEFAULT 0,
    CONSTRAINT pk_alert_instance PRIMARY KEY (tenant_id, started_at, id),
    CONSTRAINT ck_ack_fields CHECK ((acknowledged_by IS NULL) = (acknowledged_at IS NULL))
) PARTITION BY RANGE (started_at);

CREATE TABLE telemetry.alert_instance_2026_07
    PARTITION OF telemetry.alert_instance
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

-- Invariante I1 materializada: no máximo UMA instância aberta por fingerprint
CREATE UNIQUE INDEX uq_alert_open
    ON telemetry.alert_instance (tenant_id, fingerprint)
    WHERE state IN ('Pending','Firing','Acknowledged','Silenced');

CREATE INDEX ix_alert_state ON telemetry.alert_instance (tenant_id, state, severity);
CREATE INDEX ix_alert_rule  ON telemetry.alert_instance (rule_urn, started_at DESC);

ALTER TABLE telemetry.alert_rule     ENABLE ROW LEVEL SECURITY;
ALTER TABLE telemetry.alert_rule     FORCE  ROW LEVEL SECURITY;
ALTER TABLE telemetry.alert_instance ENABLE ROW LEVEL SECURITY;
ALTER TABLE telemetry.alert_instance FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_rule_tenant  ON telemetry.alert_rule
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
CREATE POLICY p_alert_tenant ON telemetry.alert_instance
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

> `runbook_urn` é `NOT NULL` com FK **por desenho** (ADR-0247): um alerta sem runbook
> transfere ao plantonista, às 3h da manhã, o trabalho de descobrir o que fazer — e o
> resultado previsível é o alerta ser silenciado em vez de tratado.
>
> `uq_alert_open` é a invariante I1 no lugar certo. Duas instâncias abertas do mesmo
> alerta produzem notificação duplicada, e notificação duplicada é a causa mais comum
> de fadiga de alerta.

---

## 7. `telemetry.silence` (multi-tenant, RLS)

```sql
CREATE TABLE telemetry.silence (
    urn        text        PRIMARY KEY,
    tenant_id  text        NOT NULL,
    matcher    jsonb       NOT NULL,
    reason     text        NOT NULL CHECK (length(btrim(reason)) > 0),
    created_by text        NOT NULL,
    starts_at  timestamptz NOT NULL DEFAULT now(),
    expires_at timestamptz NOT NULL,
    revoked_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT ck_silence_window CHECK (expires_at > starts_at)
);

CREATE INDEX ix_silence_expiry ON telemetry.silence (tenant_id, expires_at);
CREATE INDEX ix_silence_active ON telemetry.silence (expires_at)
    WHERE revoked_at IS NULL;

ALTER TABLE telemetry.silence ENABLE ROW LEVEL SECURITY;
ALTER TABLE telemetry.silence FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_silence_tenant ON telemetry.silence
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

`expires_at` é `NOT NULL` pela mesma razão que o waiver do `../022-Policy/` tem prazo:
silêncio permanente é o alerta excluído sem que ninguém tenha decidido excluí-lo. O
teto de 72 h vem de `obs.alert.max_silence_h` (`AIOS-OBS-0013`).

---

## 8. `telemetry.dashboard` e `telemetry.runbook` (multi-tenant, RLS)

```sql
CREATE TABLE telemetry.runbook (
    urn              text        PRIMARY KEY,
    tenant_id        text        NOT NULL,
    runbook_key      text        NOT NULL,
    path             text        NOT NULL,
    steps_count      int         NOT NULL CHECK (steps_count > 0),
    last_reviewed_at timestamptz NOT NULL DEFAULT now(),
    owner_module     text        NOT NULL,
    created_at       timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_runbook_key UNIQUE (tenant_id, runbook_key)
);

CREATE INDEX ix_runbook_review ON telemetry.runbook (last_reviewed_at);

CREATE TABLE telemetry.dashboard (
    urn               text        PRIMARY KEY,
    tenant_id         text        NOT NULL,
    dashboard_key     text        NOT NULL,
    dashboard_version int         NOT NULL DEFAULT 1,
    definition        jsonb       NOT NULL,
    owner_module      text        NOT NULL,
    signals_used      text[]      NOT NULL DEFAULT '{}',
    status            text        NOT NULL DEFAULT 'draft'
                      CHECK (status IN ('draft','published','deprecated')),
    created_at        timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_dashboard_version UNIQUE (tenant_id, dashboard_key, dashboard_version)
);

CREATE INDEX ix_dashboard_status ON telemetry.dashboard (tenant_id, status);

ALTER TABLE telemetry.runbook   ENABLE ROW LEVEL SECURITY;
ALTER TABLE telemetry.runbook   FORCE  ROW LEVEL SECURITY;
ALTER TABLE telemetry.dashboard ENABLE ROW LEVEL SECURITY;
ALTER TABLE telemetry.dashboard FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_runbook_tenant   ON telemetry.runbook
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
CREATE POLICY p_dashboard_tenant ON telemetry.dashboard
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

`steps_count > 0` é literal: runbook vazio não é runbook. A validação de que os
`signals_used` estão `Registered` é da aplicação (`AIOS-OBS-0009`), pois cruza schemas.

---

## 9. `telemetry.retention_policy` (plataforma) e `telemetry.outbox`

```sql
CREATE TABLE telemetry.retention_policy (
    urn            text        PRIMARY KEY,
    tier           text        NOT NULL CHECK (tier IN ('raw','short','medium','long')),
    signal_kind    text        NOT NULL CHECK (signal_kind IN ('metric','span','log_event')),
    retain_for     interval    NOT NULL,
    downsample_to  interval,
    storage_target text        NOT NULL
                   CHECK (storage_target IN ('prometheus','object_store','tempo','seq')),
    legal_basis    text,
    created_at     timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_retention UNIQUE (tier, signal_kind)
);

CREATE TABLE telemetry.outbox (
    id           char(26)    PRIMARY KEY,
    tenant_id    text        NOT NULL,
    subject      text        NOT NULL,
    event_type   text        NOT NULL,
    payload      jsonb       NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    published_at timestamptz
);

CREATE INDEX ix_outbox_pending ON telemetry.outbox (created_at)
    WHERE published_at IS NULL;
```

Toda transição para `Firing` e para `Resolved` grava a mudança **e** o evento na mesma
transação (invariante I5).

---

## 10. Relações

```
   telemetry.signal_descriptor(*) ──referenciado por──▶ telemetry.slo_definition(*)
            │                                                    │
            │ retention_tier                                     │ slo_urn
            ▼                                                    ▼
   telemetry.retention_policy(1..*)              telemetry.error_budget_ledger(*)
                                                                 │
   telemetry.alert_rule(*) ──[slo_urn]──────────────────────────┘
        │  │
        │  └──[runbook_urn NOT NULL]──▶ telemetry.runbook(1)
        ▼
   telemetry.alert_instance(*) ──silenciada por──▶ telemetry.silence(*)

   telemetry.cardinality_budget(*) ──governa──▶ admissão de sinal
   telemetry.dashboard(*) ──[signals_used]────▶ telemetry.signal_descriptor(*)

   ══════════════ FORA do PostgreSQL ══════════════
   séries temporais → Prometheus (+ objeto em MinIO para longo prazo)
   traces           → Tempo / Jaeger
   linhas de log    → Seq
```

---

## 11. Particionamento e retenção

### 11.1 Tabelas de controle (PostgreSQL)

| Tabela | Estratégia | Retenção | Classificação |
|--------|-----------|----------|---------------|
| `alert_instance` | `RANGE(started_at)`, partições **mensais** | 400 dias (análise de recorrência) | `operational` |
| `error_budget_ledger` | `RANGE(evaluated_at)`, partições **mensais** | 400 dias (conformidade anual) | `operational` |
| `signal_descriptor` | sem partição | permanente (histórico de contrato) | `config` |
| `slo_definition`, `alert_rule`, `dashboard`, `runbook` | sem partição | permanente, versionado | `config` |
| `silence` | sem partição | 1 ano após `expires_at` | `operational` |
| `outbox` | sem partição | expurgo de publicados após 7 dias | `operational` |

### 11.2 Telemetria (fora do PostgreSQL)

| Sinal | Camada | Resolução | Retenção | Destino |
|-------|--------|-----------|----------|---------|
| Métricas | `raw` | nativa | **15 d** (`obs.retention.metrics_raw_days`) | Prometheus |
| Métricas | `short` | 5 min | **90 d** (`obs.retention.metrics_5m_days`) | Prometheus + objeto |
| Métricas | `medium` | 1 h | **400 d** (`obs.retention.metrics_1h_days`) | objeto (MinIO) |
| Traces | `short` | — | **7 d** (`obs.retention.traces_days`) | Tempo / Jaeger |
| Logs | `short` | — | **14 d** (`obs.retention.logs_days`) | Seq |

O *downsampling* em camadas é decisão de **custo tanto quanto de utilidade**
(ADR-0248): um p99 de 300 dias atrás é útil como tendência, não como amostra de
segundo. Reter tudo em resolução nativa faria o observador custar mais que o
observado.

---

## 12. LGPD/GDPR no modelo

| Item | Tratamento |
|------|-----------|
| Dados pessoais | Telemetria é dado **operacional**. Conteúdo de negócio não entra em sinal (`./Vision.md` §3.2). |
| Classificação | `signal_descriptor.data_class` ∈ {`operational`, `pii`, `sensitive`} dirige redação e retenção. |
| Redação | Aplicada na **borda** pelo `PiiRedactor` (ADR-0249) — nunca no armazenamento. |
| Base legal | `retention_policy.legal_basis` obrigatório quando o sinal é `pii` (CV-08); ausência → `AIOS-OBS-0015`. |
| Direito ao esquecimento | UC-015: pseudonimização/expurgo nas três lojas, preservando agregados sem identificação; comprovante em `../025-Audit/`. |
| Segregação | RLS em todas as tabelas multi-tenant; rótulo de tenant obrigatório nas séries (NFR-017). |

---

## 13. Migrações

Estratégia **expand/contract** (CV-11), com DDL bloqueante ≤ 200 ms e índices
`CONCURRENTLY` (CV-12).

| # | Migração | Tipo | Observação |
|---|----------|------|------------|
| M-001 | Schema, `signal_descriptor`, `retention_policy` | expand | Base do registro |
| M-002 | `runbook` antes de `alert_rule` | expand | Ordem obrigatória: a FK `runbook_urn` é `NOT NULL` |
| M-003 | `alert_instance`, `error_budget_ledger` + partições + job de rotação | expand | Job cria M+2 partições |
| M-004 | Índice `uq_alert_open` | expand | `CONCURRENTLY`; exige verificar duplicidade antes |
| M-005 | Nova coluna em `signal_descriptor` | expand → backfill → contract | Descritores `Registered` são imutáveis: backfill vale só para `Proposed` |
| M-006 | Expurgo de partições fora da retenção | contract | Executa `DROP PARTITION` |

> Uma migração deste schema **NÃO DEVE** alterar o significado de um
> `signal_descriptor` já `Registered`. Se a mudança implicar reinterpretação da
> métrica, o caminho correto é **registrar nova `descriptor_version`** — do contrário,
> séries históricas passariam a significar outra coisa retroativamente, e toda
> comparação temporal ficaria inválida.

---

## 14. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §3
- Convenções físicas: `../005-Database/Database.md` §2 (CV-01..CV-12)
- FSM e invariantes: `./StateMachine.md` §5 e §7
- Retenção operacional: `./Configuration.md` §5 · Escala: `./Scalability.md`
- Segurança e privacidade: `./Security.md`

*Fim de `Database.md`.*
