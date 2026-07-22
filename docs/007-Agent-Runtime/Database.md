---
Documento: Database
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0076, ADR-0074
RFCs relacionados: RFC-0001
Depende de: ./_DESIGN_BRIEF.md, ../005-Database/, ../010-Memory/, ../025-Audit/, ../021-Security/
---

# 007-Agent-Runtime — Database

> **Escopo e fronteira crítica.** O processo Python do Agent Runtime **NÃO
> DEVE** conectar-se diretamente a PostgreSQL/pgvector/Apache AGE (regra de
> fronteira invioável, `./_DESIGN_BRIEF.md` §2.2, R-05). Este documento,
> portanto, tem duas partes com naturezas distintas: (1) o **schema lógico
> durável** (`runtime.*`) das entidades canônicas do módulo
> (`AgentSession`, `ReActStep`, `ToolInvocation`, `ModelCall`, `Checkpoint`) —
> escrito **por projeção de eventos**, nunca por SQL direto do runtime, pelos
> serviços donos de persistência (`010-Memory` para o caminho de working
> memory/observação, `025-Audit` para a trilha imutável) — e (2) o
> **armazenamento efêmero local** (SQLite/WAL) que vive **dentro do
> sandbox** de cada processo runtime, usado exclusivamente pelo `EventPublisher`
> como outbox transacional (ADR-0076) e pelo cursor de execução.

## Índice

1. Visão geral e responsabilidade de escrita
2. Schema durável `runtime.*` (projeção, escrito por 010/025)
3. Índices e particionamento (schema durável)
4. Row-Level Security (RLS)
5. Armazenamento efêmero local (SQLite/WAL, outbox do sandbox)
6. Retenção e expurgo
7. Migrações
8. Relação com estado quente (Redis)
9. Referências

---

## 1. Visão Geral e Responsabilidade de Escrita

| Tabela/Store | Papel | Quem escreve | Onde vive |
|--------------|-------|----------------|------------|
| `runtime.agent_session` | Sessão de execução de um agente materializado. | Projeção a partir de `aios.<t>.agent.runtime.*`/`task.execution.*`, consumida por `010-Memory` (working memory) e `025-Audit`. | PostgreSQL do control plane (RLS por `tenant_id`). |
| `runtime.react_step` | Passo do loop cognitivo, *event-sourced*, *append-only*. | Idem — projeção do evento `task.execution.step`. | PostgreSQL do control plane. |
| `runtime.tool_invocation` | Chamada de ferramenta via `015-Tool-Manager`. | Projeção a partir dos campos de `ReActStep`/eventos correlatos de `015`. | PostgreSQL do control plane. |
| `runtime.model_call` | Chamada de inferência via `017-Model-Router`. | Projeção a partir dos campos de `ReActStep`/eventos correlatos de `017`. | PostgreSQL do control plane. |
| `runtime.checkpoint` | Snapshot para hibernação/retomada (*cold agent*). | `CheckpointManager` do runtime grava o **blob** em MinIO via API do control plane; o **metadado** (linha da tabela) é projetado a partir de `agent.runtime.suspended`. | Metadado: PostgreSQL. Blob: MinIO (via control plane). |
| Outbox local (SQLite/WAL) | Fila transacional local de eventos pendentes de publicação. | `EventPublisher`, diretamente, dentro do sandbox do processo. | Disco efêmero (tmpfs/`sandbox.fs_writable_mib`) do processo runtime — **não sobrevive** à morte do processo/nó. |
| Cursor de execução local | Última posição confirmada do loop antes do próximo checkpoint. | `CognitiveLoopEngine`, diretamente, dentro do sandbox. | Idem outbox local. |

**Regra normativa central:** todo acesso do runtime a qualquer uma das
tabelas `runtime.*` **DEVE** ocorrer por chamada gRPC a `010-Memory`/
`025-Audit` (leitura) ou por publicação de evento NATS consumido por esses
serviços (escrita/projeção) — **nunca** por driver de banco de dados
embutido no processo Python. Esta seção documenta o **schema físico** dessas
tabelas como referência de contrato entre módulos (o schema é definido aqui
porque as entidades são conceitualmente donas deste módulo, ainda que a
persistência física seja delegada), mas a **implementação** do schema
pertence a `010-Memory`/`025-Audit`, sob as convenções gerais de
`../005-Database/`.

---

## 2. Schema Durável `runtime.*` (Projeção, Escrito por 010/025)

### 2.1 Extensões e schema

```sql
CREATE SCHEMA IF NOT EXISTS runtime;

CREATE OR REPLACE FUNCTION runtime.is_valid_ulid(v text) RETURNS boolean
LANGUAGE sql IMMUTABLE AS $$
  SELECT v ~ '^[0-9A-HJKMNP-TV-Z]{26}$';
$$;
```

### 2.2 Tabela `runtime.agent_session`

```sql
CREATE TABLE runtime.agent_session (
    session_id        char(26)    NOT NULL CHECK (runtime.is_valid_ulid(session_id)),
    agent_urn         text        NOT NULL,
    task_urn          text        NOT NULL,
    tenant_id         text        NOT NULL,
    runtime_id        char(26)    NOT NULL,
    state             text        NOT NULL CHECK (state IN (
                          'boot','ready','thinking','acting','waiting-io',
                          'reflecting','done','failed','suspending',
                          'suspended','resuming','terminating')),
    plan_urn          text        NULL,
    step_cursor       int         NOT NULL DEFAULT 0,
    budget            jsonb       NOT NULL,
    consumed          jsonb       NOT NULL DEFAULT '{}'::jsonb,
    idempotency_key   text        NOT NULL,
    traceparent       text        NOT NULL,
    created_at        timestamptz NOT NULL DEFAULT now(),
    updated_at        timestamptz NOT NULL DEFAULT now(),
    terminal_reason   text        NULL,

    CONSTRAINT pk_agent_session PRIMARY KEY (session_id),
    CONSTRAINT uq_agent_session_idem UNIQUE (tenant_id, task_urn, idempotency_key),
    -- estado terminal exige terminal_reason preenchido (I2 de ./ClassDiagrams.md, análogo)
    CONSTRAINT ck_session_terminal_reason CHECK (
        (state IN ('done','failed') AND terminal_reason IS NOT NULL)
        OR (state NOT IN ('done','failed'))
    )
) PARTITION BY HASH (tenant_id);

CREATE TABLE runtime.agent_session_p00 PARTITION OF runtime.agent_session FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE runtime.agent_session_p01 PARTITION OF runtime.agent_session FOR VALUES WITH (MODULUS 8, REMAINDER 1);
-- ... p02..p07 seguem o mesmo padrão (omitidos por brevidade; ver migração 0001).

CREATE INDEX ix_session_tenant_state ON runtime.agent_session (tenant_id, state);
CREATE INDEX ix_session_agent        ON runtime.agent_session (agent_urn);
CREATE INDEX ix_session_runtime      ON runtime.agent_session (runtime_id);
CREATE INDEX ix_session_suspended    ON runtime.agent_session (tenant_id, updated_at)
    WHERE state = 'suspended';   -- varredura de agentes cold-storable

ALTER TABLE runtime.agent_session ENABLE ROW LEVEL SECURITY;
ALTER TABLE runtime.agent_session FORCE ROW LEVEL SECURITY;
CREATE POLICY agent_session_tenant_isolation ON runtime.agent_session
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

### 2.3 Tabela `runtime.react_step` (event-sourced, append-only)

```sql
CREATE TABLE runtime.react_step (
    step_id           char(26)    NOT NULL CHECK (runtime.is_valid_ulid(step_id)),
    session_id        char(26)    NOT NULL,
    tenant_id         text        NOT NULL,
    index             int         NOT NULL,
    phase             text        NOT NULL CHECK (phase IN ('perceive','think','act','observe','reflect')),
    percept_ref       text        NULL,
    thought           text        NULL,   -- redigido para PII antes de chegar aqui (FR-015)
    action_type       text        NULL CHECK (action_type IN ('tool_call','final_answer','replan','noop')),
    tool_invocation_id char(26)   NULL,
    model_call_id     char(26)    NULL,
    observation       jsonb       NULL,
    status            text        NOT NULL CHECK (status IN ('ok','error','timeout','denied')),
    latency_ms        int         NOT NULL CHECK (latency_ms >= 0),
    created_at        timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT pk_react_step PRIMARY KEY (step_id, created_at),
    CONSTRAINT uq_react_step_order UNIQUE (session_id, index)
) PARTITION BY RANGE (created_at);

CREATE TABLE runtime.react_step_2026_07 PARTITION OF runtime.react_step
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
-- partições futuras criadas por job agendado (pg_partman), 7 dias de antecedência.

CREATE INDEX ix_step_session_index ON runtime.react_step (session_id, index);
CREATE INDEX ix_step_tenant_status ON runtime.react_step (tenant_id, status) WHERE status IN ('error','denied');

ALTER TABLE runtime.react_step ENABLE ROW LEVEL SECURITY;
ALTER TABLE runtime.react_step FORCE ROW LEVEL SECURITY;
CREATE POLICY react_step_tenant_isolation ON runtime.react_step
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

### 2.4 Tabelas `runtime.tool_invocation` e `runtime.model_call`

```sql
CREATE TABLE runtime.tool_invocation (
    tool_invocation_id char(26)   NOT NULL CHECK (runtime.is_valid_ulid(tool_invocation_id)),
    session_id          char(26)   NOT NULL,
    tenant_id           text       NOT NULL,
    tool_urn            text       NOT NULL,
    capability_id        text       NOT NULL,
    arguments_ref        text       NOT NULL,
    result_status         text       NOT NULL CHECK (result_status IN ('ok','error','timeout','denied','circuit_open')),
    retries               int        NOT NULL DEFAULT 0,
    cost_usd              numeric(12,6) NOT NULL DEFAULT 0,
    latency_ms             int        NOT NULL,
    idempotency_key        text       NOT NULL,
    created_at              timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT pk_tool_invocation PRIMARY KEY (tool_invocation_id),
    CONSTRAINT uq_tool_invocation_idem UNIQUE (tenant_id, idempotency_key)
);
CREATE INDEX ix_tool_invocation_session ON runtime.tool_invocation (session_id);
ALTER TABLE runtime.tool_invocation ENABLE ROW LEVEL SECURITY;
ALTER TABLE runtime.tool_invocation FORCE ROW LEVEL SECURITY;
CREATE POLICY tool_invocation_tenant_isolation ON runtime.tool_invocation
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));

CREATE TABLE runtime.model_call (
    model_call_id       char(26)   NOT NULL CHECK (runtime.is_valid_ulid(model_call_id)),
    session_id            char(26)   NOT NULL,
    tenant_id              text       NOT NULL,
    requested_model         text       NULL,
    resolved_model_urn        text       NOT NULL,
    prompt_tokens              int        NOT NULL,
    completion_tokens           int        NOT NULL,
    cost_usd                     numeric(12,6) NOT NULL,
    status                        text       NOT NULL CHECK (status IN ('ok','error','timeout','fallback')),
    latency_ms                    int        NOT NULL,
    created_at                     timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT pk_model_call PRIMARY KEY (model_call_id)
);
CREATE INDEX ix_model_call_session ON runtime.model_call (session_id);
ALTER TABLE runtime.model_call ENABLE ROW LEVEL SECURITY;
ALTER TABLE runtime.model_call FORCE ROW LEVEL SECURITY;
CREATE POLICY model_call_tenant_isolation ON runtime.model_call
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

### 2.5 Tabela `runtime.checkpoint`

```sql
CREATE TABLE runtime.checkpoint (
    checkpoint_id        char(26)   NOT NULL CHECK (runtime.is_valid_ulid(checkpoint_id)),
    session_id             char(26)   NOT NULL,
    tenant_id               text       NOT NULL,
    step_cursor              int        NOT NULL,
    working_memory_ref         text       NOT NULL,  -- referência de objeto MinIO
    plan_snapshot_ref            text       NULL,
    size_bytes                    bigint     NOT NULL,
    checksum                       text       NOT NULL,  -- sha256
    created_at                      timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT pk_checkpoint PRIMARY KEY (checkpoint_id)
);
CREATE INDEX ix_checkpoint_session_created ON runtime.checkpoint (session_id, created_at DESC);
ALTER TABLE runtime.checkpoint ENABLE ROW LEVEL SECURITY;
ALTER TABLE runtime.checkpoint FORCE ROW LEVEL SECURITY;
CREATE POLICY checkpoint_tenant_isolation ON runtime.checkpoint
    USING (tenant_id = current_setting('aios.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('aios.tenant_id', true));
```

> **Nota sobre `SandboxProfile`.** O catálogo de perfis de sandbox
> (`sandbox_profile_id`, `seccomp_profile`, `cpu_millicores`, `mem_mib`,
> `pids_max`, `egress_allowlist`, `fs_writable_mib`) é dado de
> **configuração**, não de execução — seu armazenamento físico (tabela ou
> config versionada) é responsabilidade de `../021-Security/` e é apenas
> **consumido** por este módulo no boot (`RuntimeBootstrapper`/`SandboxManager`).
> Ver `./Configuration.md` §`sandbox.profile`.

---

## 3. Índices e Particionamento (Schema Durável)

| Tabela | Estratégia | Justificativa |
|--------|-----------|----------------|
| `runtime.agent_session` | **HASH(tenant_id)**, 8 partições. | Distribui I/O entre tenants; volume moderado (uma linha por sessão, não por passo). |
| `runtime.react_step` | **RANGE(created_at)**, mensal. | Volume alto (um `INSERT` por passo do loop, potencialmente milhões/dia sob NFR-004); expurgo por `DROP PARTITION` alinhado à retenção (§6). |
| `runtime.tool_invocation` / `runtime.model_call` | Não particionadas no MVP (candidatas a `RANGE(created_at)` se o volume ultrapassar o previsto). | Cardinalidade proporcional ao volume de passos `act`/`think`; revisitar particionamento junto com `react_step` se necessário. |
| `runtime.checkpoint` | Não particionada; índice composto por `(session_id, created_at DESC)`. | Baixa cardinalidade relativa (um checkpoint a cada `checkpoint.interval_steps`, não por passo). |

---

## 4. Row-Level Security (RLS)

- Toda sessão de conexão dos serviços que escrevem em `runtime.*`
  (`010-Memory`, `025-Audit`) **DEVE** executar `SET LOCAL aios.tenant_id =
  '<tenant>'` dentro da transação, com o valor extraído do `tenant` do
  evento/claim autenticado — nunca de um campo não validado do payload.
- `FORCE ROW LEVEL SECURITY` garante que nem o *owner* da tabela escapa da
  política — alinhado à convenção de `../005-Database/` e ao padrão
  observado em `../006-Kernel/Database.md` §5.
- Testes de RLS (contract tests) **DEVEM** verificar que uma sessão
  `tenant_id = 'acme'` não enxerga linhas de `tenant_id = 'globex'` em
  nenhuma das 5 tabelas `runtime.*`.

---

## 5. Armazenamento Efêmero Local (SQLite/WAL, Outbox do Sandbox)

> Esta seção descreve o **único** armazenamento persistente que o processo
> Python do Agent Runtime escreve **diretamente**, e ele vive **dentro** do
> seu próprio sandbox — nunca é um banco de dados compartilhado do control
> plane, e nunca é acessado por outro processo.

```sql
-- SQLite (modo WAL), arquivo em tmpfs efêmero do sandbox
-- (sandbox.fs_writable_mib), path: /run/aios-runtime/outbox.db

CREATE TABLE outbox (
    id                TEXT PRIMARY KEY,          -- ULID do evento
    tenant_id          TEXT NOT NULL,
    subject             TEXT NOT NULL,             -- aios.<t>.<dominio>.<entidade>.<acao>
    event_type           TEXT NOT NULL,
    payload               TEXT NOT NULL,             -- JSON serializado (envelope CloudEvents)
    published              INTEGER NOT NULL DEFAULT 0, -- boolean
    publish_attempts        INTEGER NOT NULL DEFAULT 0,
    last_attempt_at           TEXT NULL,
    created_at                 TEXT NOT NULL
);
CREATE INDEX ix_outbox_pending ON outbox (created_at) WHERE published = 0;

CREATE TABLE execution_cursor (
    session_id     TEXT PRIMARY KEY,
    step_cursor     INTEGER NOT NULL,
    plan_urn         TEXT NULL,
    updated_at        TEXT NOT NULL
);
```

**Regras operacionais:**

- O `EventPublisher` grava em `outbox` **na mesma unidade de trabalho
  lógica** que a `ExecutionStateMachine` aplica a transição de estado (o
  runtime não tem transação distribuída real com o processo — a garantia é
  de ordem de operações single-process, não de 2PC), e só então tenta
  publicar no NATS/JetStream de forma assíncrona (`events.outbox.flush_interval_ms`).
- `execution_cursor` é o espelho local do `step_cursor`/`plan_urn`, usado
  para checkpoints incrementais (`checkpoint.interval_steps`) e para
  observabilidade local (`GetStatus`), **nunca** como fonte da verdade —
  a fonte durável é `runtime.agent_session.step_cursor` (projetada via
  evento).
- Este arquivo SQLite **NÃO sobrevive** à morte do processo/nó (é efêmero
  por design, dentro do `fs_writable_mib` do sandbox); sua perda é
  compensada pelo modelo de checkpoint durável (`runtime.checkpoint`) e pela
  meta de RPO ≤ 1 passo (NFR-006) — ver `./FailureRecovery.md`.

---

## 6. Retenção e Expurgo

| Tabela | Política de retenção | Mecanismo |
|--------|------------------------|-----------|
| `runtime.agent_session` | Retida enquanto o agente/tenant existir; expurgo apenas via solicitação LGPD/GDPR coordenada com `025-Audit`. | Sem exclusão automática no MVP. |
| `runtime.react_step` | **90 dias** (default) via `DROP PARTITION` mensal fora da janela; tenants regulados podem exigir retenção maior (config futura). | Job de particionamento periódico (mesma família de `pg_partman` usada em `../006-Kernel/Database.md` §6). |
| `runtime.tool_invocation` / `runtime.model_call` | Acompanha a retenção de `react_step` (referenciadas por `session_id`), tipicamente 90 dias. | Expurgo coordenado com `react_step`. |
| `runtime.checkpoint` | Metadado retido por 30 dias após a sessão atingir estado terminal ou o último `resume`; blob correspondente em MinIO segue a mesma política via `010-Memory`. | Job de expurgo periódico. |
| Outbox local (SQLite) | Linhas com `published = 1` são compactadas/removidas a cada `events.outbox.flush_interval_ms` × N; o arquivo inteiro desaparece com o processo. | Sem persistência além da vida do processo (por design). |

**Direito ao esquecimento (LGPD/GDPR).** Conforme `_DESIGN_BRIEF.md` §12.3,
o expurgo de `thought`/`observation` com PII é coordenado com `010-Memory`
(base legal/retenção nos metadados de memória); a linha de `react_step`
permanece para rastreabilidade de auditoria, mas com o campo redigido —
nunca com o conteúdo original.

---

## 7. Migrações

| Arquivo | Conteúdo | Aplicado por |
|---------|----------|----------------|
| `migrations/0001_runtime_schema.sql` | Cria schema `runtime`, funções de validação, `agent_session` (8 partições), índices, RLS. | Pipeline de `010-Memory` (dono físico do schema). |
| `migrations/0002_react_step.sql` | Cria `react_step` particionada + partições dos 2 meses correntes + índices + RLS. | Idem. |
| `migrations/0003_tool_model.sql` | Cria `tool_invocation` e `model_call` + índices + RLS. | Idem. |
| `migrations/0004_checkpoint.sql` | Cria `checkpoint` + índice + RLS. | Idem. |

**Regras de migração:** idênticas às de `../006-Kernel/Database.md` §7
(aditivas e retrocompatíveis dentro de uma major de API; migrações
destrutivas exigem duas fases deprecate→backfill→remove). Como este módulo
não é o dono físico do schema, toda migração **DEVE** ser coordenada e
revisada em conjunto com `010-Memory`/`025-Audit` antes de qualquer mudança
de contrato de evento que a alimente (`./Events.md`).

---

## 8. Relação com Estado Quente (Redis)

| Dado | Chave Redis | TTL | Reconciliação |
|------|-------------|-----|-----------------|
| *Lease* de sessão (exatamente-um-ativo) | `lease:session:{tenant}:{session_id}` | curto (ex.: 10s, renovado por heartbeat) | Adquirido em `resume`; perdido = candidato a nova retomada por outra instância (ADR-0079). |
| Cache de decisão do PEP local | `cap:{tenant}:{agent}:{action}:{resource}` | `runtime.pep.decision_cache_ttl_ms` (config, ver `Configuration.md`) | Invalidado por `aios.<t>.policy.decision.updated`. |
| Contadores de cota local (opcional, alta frequência) | `quota:{tenant}:{session_id}:{dimensão}` | janela de orçamento (`loop.wall_clock_budget_ms` ou equivalente) | `QuotaGovernor` reconcilia com `runtime.agent_session.consumed` (projeção durável) periodicamente. |

> O PostgreSQL (`runtime.*`) é a **fonte da verdade durável**; Redis é
> **projeção quente** operacional (lease, cache), nunca o inverso — mesmo
> padrão adotado em `../006-Kernel/Database.md` §8.

---

## 9. Referências

- Modelo de dados canônico (fonte): `./_DESIGN_BRIEF.md` §3.
- Máquina de estados que impõe `state`/`terminal_reason`: `./StateMachine.md`.
- Estratégia de sharding/escala: `./Scalability.md`.
- Convenções físicas gerais do banco AIOS: `../005-Database/`.
- Serviços donos da persistência física: `../010-Memory/`, `../025-Audit/`.
- Modos de falha relacionados a outbox/checkpoint: `./FailureRecovery.md`.
