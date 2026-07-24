---
Documento: Database
Módulo: 025-Audit
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 025-Audit
ADRs relacionados: ADR-0005, ADR-0251, ADR-0252, ADR-0253, ADR-0254, ADR-0258
RFCs relacionados: RFC-0001, RFC-0250, RFC-0251
Depende de: _DESIGN_BRIEF.md §3, 005-Database (convenções físicas CV-01..CV-12), StateMachine.md
---

# 025-Audit — Modelo Físico

Schema **`audit`** no PostgreSQL 16, sob as convenções físicas obrigatórias de
`../005-Database/Database.md` §2 (CV-01..CV-12).

> **Regra absoluta deste modelo.** A tabela `audit.record` é **append-only**. Não
> existe caminho de `UPDATE` sobre conteúdo, e o `DELETE` **nunca remove a linha** —
> remove apenas `payload_cipher`. `seq`, `record_hash`, `prev_hash` e os metadados
> mínimos permanecem para sempre, sob pena de quebrar a cadeia. Correção de erro é
> **novo registro de compensação**, nunca alteração do original.

---

## 1. Preparação

```sql
CREATE SCHEMA IF NOT EXISTS audit;

-- Variável de sessão usada pelas políticas de RLS (padrão do 005)
--   SET LOCAL app.tenant_id = 'acme';
```

Tabelas multi-tenant (RLS obrigatória): `record`, `seal`, `legal_hold`, `subject_key`,
`erasure_receipt`, `export`, `anomaly`. Tabela de plataforma: `auditable_class`.

---

## 2. `audit.record` (multi-tenant, RLS, particionada) — a cadeia

```sql
CREATE TABLE audit.record (
    id             char(26)    NOT NULL,
    tenant_id      text        NOT NULL,
    partition_key  text        NOT NULL,
    seq            bigint      NOT NULL,
    occurred_at    timestamptz NOT NULL,
    recorded_at    timestamptz NOT NULL DEFAULT now(),
    record_class   text        NOT NULL,
    producer_module text       NOT NULL,
    actor          jsonb       NOT NULL,
    action         text        NOT NULL,
    resource_urn   text        NOT NULL,
    outcome        text        NOT NULL
                   CHECK (outcome IN ('allowed','denied','succeeded','failed','compensated')),
    decision_ref   text,
    subject_ref    text,
    payload_cipher bytea,
    payload_digest text        NOT NULL,
    prev_hash      text        NOT NULL,
    record_hash    text        NOT NULL,
    seal_ref       text,
    state          text        NOT NULL DEFAULT 'Received'
                   CHECK (state IN ('Received','Sealed','Held','Shredded','Expired')),
    trace_id       text        NOT NULL,
    event_id       text,
    CONSTRAINT pk_audit_record PRIMARY KEY (tenant_id, occurred_at, id),
    CONSTRAINT ck_sealed_has_ref CHECK (state <> 'Sealed' OR seal_ref IS NOT NULL),
    CONSTRAINT ck_shredded_has_subject CHECK (state <> 'Shredded' OR subject_ref IS NOT NULL)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE audit.record_2026_07
    PARTITION OF audit.record
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');

-- Cadeia: sequência monotônica por partição lógica (invariante I3)
CREATE UNIQUE INDEX uq_record_seq  ON audit.record (partition_key, seq);
-- Encadeamento: nenhum hash se repete (invariante I2)
CREATE UNIQUE INDEX uq_record_hash ON audit.record (record_hash);
-- Idempotência (invariante I7)
CREATE UNIQUE INDEX uq_record_event ON audit.record (tenant_id, event_id)
    WHERE event_id IS NOT NULL;

CREATE INDEX ix_record_subject  ON audit.record (tenant_id, subject_ref)
    WHERE subject_ref IS NOT NULL;
CREATE INDEX ix_record_class    ON audit.record (tenant_id, record_class, occurred_at DESC);
CREATE INDEX ix_record_decision ON audit.record (decision_ref)
    WHERE decision_ref IS NOT NULL;
CREATE INDEX ix_record_trace    ON audit.record (trace_id);
CREATE INDEX ix_record_seal     ON audit.record (seal_ref);            -- CV-05
CREATE INDEX ix_record_unsealed ON audit.record (partition_key, recorded_at)
    WHERE state = 'Received';

ALTER TABLE audit.record ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit.record FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_record_tenant ON audit.record
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

### 2.1 Imutabilidade imposta pelo banco

A ausência de operação na API não basta: um `UPDATE` direto por quem tem acesso ao
banco precisa falhar **no banco**.

```sql
CREATE OR REPLACE FUNCTION audit.deny_record_mutation() RETURNS trigger AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        RAISE EXCEPTION 'AIOS-AUD-0016: audit.record é append-only (DELETE proibido)';
    END IF;

    -- Campos que sustentam a cadeia e o fato: IMUTÁVEIS em qualquer estado
    IF NEW.id            IS DISTINCT FROM OLD.id            OR
       NEW.seq           IS DISTINCT FROM OLD.seq           OR
       NEW.partition_key IS DISTINCT FROM OLD.partition_key OR
       NEW.occurred_at   IS DISTINCT FROM OLD.occurred_at   OR
       NEW.recorded_at   IS DISTINCT FROM OLD.recorded_at   OR
       NEW.record_class  IS DISTINCT FROM OLD.record_class  OR
       NEW.actor         IS DISTINCT FROM OLD.actor         OR
       NEW.action        IS DISTINCT FROM OLD.action        OR
       NEW.resource_urn  IS DISTINCT FROM OLD.resource_urn  OR
       NEW.outcome       IS DISTINCT FROM OLD.outcome       OR
       NEW.payload_digest IS DISTINCT FROM OLD.payload_digest OR
       NEW.prev_hash     IS DISTINCT FROM OLD.prev_hash     OR
       NEW.record_hash   IS DISTINCT FROM OLD.record_hash
    THEN
        RAISE EXCEPTION 'AIOS-AUD-0016: campo imutável de audit.record';
    END IF;

    -- Permitido: state, seal_ref e a REMOÇÃO de payload_cipher (expurgo/apagamento)
    IF NEW.payload_cipher IS NOT NULL
       AND NEW.payload_cipher IS DISTINCT FROM OLD.payload_cipher THEN
        RAISE EXCEPTION 'AIOS-AUD-0016: payload só pode ser removido, nunca alterado';
    END IF;

    RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER tr_record_immutable
    BEFORE UPDATE OR DELETE ON audit.record
    FOR EACH ROW EXECUTE FUNCTION audit.deny_record_mutation();
```

> Este *trigger* é a materialização das invariantes I1, I4 e I6. As **únicas** mutações
> permitidas são: transição de `state`, atribuição de `seal_ref` e **remoção** de
> `payload_cipher`. Tudo o mais falha — inclusive para quem tem acesso direto ao banco.
> A cópia WORM (§9) cobre o caso em que até o *trigger* é removido.

---

## 3. `audit.seal` (multi-tenant, RLS) — a cadeia de selos

```sql
CREATE TABLE audit.seal (
    urn             text        PRIMARY KEY,
    tenant_id       text        NOT NULL,
    partition_key   text        NOT NULL,
    seq_from        bigint      NOT NULL,
    seq_to          bigint      NOT NULL,
    merkle_root     text        NOT NULL,
    prev_seal_hash  text        NOT NULL,
    seal_hash       text        NOT NULL,
    signature       bytea       NOT NULL,
    signed_by       text        NOT NULL,
    sealed_at       timestamptz NOT NULL DEFAULT now(),
    external_anchor text,
    worm_object     text,
    CONSTRAINT ck_seal_range  CHECK (seq_to >= seq_from),
    CONSTRAINT uq_seal_hash   UNIQUE (seal_hash),
    CONSTRAINT uq_seal_start  UNIQUE (partition_key, seq_from)
);

CREATE INDEX ix_seal_time  ON audit.seal (tenant_id, sealed_at DESC);
CREATE INDEX ix_seal_range ON audit.seal (partition_key, seq_to DESC);

ALTER TABLE audit.seal ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit.seal FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_seal_tenant ON audit.seal
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));

CREATE TRIGGER tr_seal_immutable
    BEFORE UPDATE OR DELETE ON audit.seal
    FOR EACH ROW EXECUTE FUNCTION audit.deny_seal_mutation();
```

O `audit.deny_seal_mutation()` segue o mesmo padrão de §2.1, permitindo apenas o
preenchimento de `worm_object` e `external_anchor` após a criação.

> Os selos formam uma **segunda cadeia** (`prev_seal_hash`), sobre a cadeia de
> registros. Adulterar um registro exige recomputar sua cadeia, o selo que o cobre,
> **todos os selos seguintes** — e forjar assinaturas cuja chave privada está no
> `../021-Security/`, fora do alcance de quem opera o banco.

---

## 4. `audit.auditable_class` (plataforma)

```sql
CREATE TABLE audit.auditable_class (
    urn                  text        PRIMARY KEY,
    class_key            text        NOT NULL UNIQUE,
    producer_module      text        NOT NULL,
    source_subjects      text[]      NOT NULL DEFAULT '{}',
    retention_days       int         NOT NULL CHECK (retention_days > 0),
    legal_basis          text        NOT NULL CHECK (length(btrim(legal_basis)) > 0),
    contains_pii         boolean     NOT NULL DEFAULT false,
    expected_rate_per_day bigint,
    status               text        NOT NULL DEFAULT 'active'
                         CHECK (status IN ('active','deprecated')),
    created_at           timestamptz NOT NULL DEFAULT now(),
    updated_at           timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX ix_class_producer ON audit.auditable_class (producer_module, status);
CREATE INDEX ix_class_subjects ON audit.auditable_class USING gin (source_subjects);
```

- `legal_basis` é `NOT NULL` **para toda classe**, não apenas para as que contêm PII
  (CV-08 exige para `pii`; aqui o requisito é mais forte): reter dado indefinidamente
  "porque sim" é o que um auditor pergunta primeiro.
- `retention_days` é validado pela aplicação contra `aud.retention.min_days` (365)
  — `AIOS-AUD-0010`.
- `expected_rate_per_day` alimenta a detecção de **classe silenciosa**: sem uma
  expectativa declarada, ausência de registro é indistinguível de ausência de
  atividade.

---

## 5. `audit.legal_hold` (multi-tenant, RLS)

```sql
CREATE TABLE audit.legal_hold (
    urn          text        PRIMARY KEY,
    tenant_id    text        NOT NULL,
    case_ref     text        NOT NULL CHECK (length(btrim(case_ref)) > 0),
    scope        jsonb       NOT NULL,
    legal_basis  text        NOT NULL CHECK (length(btrim(legal_basis)) > 0),
    requested_by text        NOT NULL,
    approved_by  text        NOT NULL,
    applied_at   timestamptz NOT NULL DEFAULT now(),
    expires_at   timestamptz,                      -- PODE ser nulo (StateMachine §6.1)
    released_at  timestamptz,
    released_by  text,
    version      bigint      NOT NULL DEFAULT 0,
    CONSTRAINT ck_hold_approver CHECK (approved_by <> requested_by),
    CONSTRAINT ck_hold_release  CHECK ((released_at IS NULL) = (released_by IS NULL))
);

CREATE INDEX ix_hold_active ON audit.legal_hold (tenant_id)
    WHERE released_at IS NULL;
CREATE INDEX ix_hold_scope  ON audit.legal_hold USING gin (scope jsonb_path_ops);
CREATE INDEX ix_hold_review ON audit.legal_hold (applied_at)
    WHERE released_at IS NULL AND expires_at IS NULL;

ALTER TABLE audit.legal_hold ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit.legal_hold FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_hold_tenant ON audit.legal_hold
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

`expires_at` **PODE ser nulo** — a única entidade do AIOS em que isso é permitido, ao
contrário do waiver do `../022-Policy/` e do silêncio do `../024-Observability/`. Um
*hold* não expira porque um prazo técnico venceu; expira quando o **caso encerra**. O
índice `ix_hold_review` existe exatamente para alimentar a revisão periódica dos
*holds* sem prazo (`aud.hold.review_interval_days`).

---

## 6. `audit.subject_key` (multi-tenant, RLS)

```sql
CREATE TABLE audit.subject_key (
    urn             text        PRIMARY KEY,
    tenant_id       text        NOT NULL,
    subject_ref     text        NOT NULL,
    key_ref         text        NOT NULL,     -- referência no 021; NUNCA o material
    state           text        NOT NULL DEFAULT 'active'
                    CHECK (state IN ('active','destroyed')),
    destroyed_at    timestamptz,
    destroy_receipt text,
    version         bigint      NOT NULL DEFAULT 0,
    created_at      timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_subject_key UNIQUE (tenant_id, subject_ref),
    CONSTRAINT ck_destroyed   CHECK ((state = 'destroyed') = (destroyed_at IS NOT NULL))
);

CREATE INDEX ix_subject_key_state ON audit.subject_key (state);
```

`key_ref` é uma **referência**: o material da chave vive no `../021-Security/`. Guardar
a chave ao lado do dado cifrado tornaria a cifragem decorativa.

---

## 7. `audit.erasure_receipt` (multi-tenant, RLS)

```sql
CREATE TABLE audit.erasure_receipt (
    urn              text        PRIMARY KEY,
    tenant_id        text        NOT NULL,
    kind             text        NOT NULL
                     CHECK (kind IN ('crypto_shred','payload_purge','external')),
    subject_ref      text,
    origin_module    text        NOT NULL,
    scope            jsonb       NOT NULL,
    records_affected bigint      NOT NULL DEFAULT 0,
    executed_at      timestamptz NOT NULL DEFAULT now(),
    executed_by      text        NOT NULL,
    receipt_hash     text        NOT NULL UNIQUE,
    chain_record_id  char(26)    NOT NULL
);

CREATE INDEX ix_receipt_subject ON audit.erasure_receipt (tenant_id, subject_ref);
CREATE INDEX ix_receipt_origin  ON audit.erasure_receipt (origin_module, executed_at DESC);

ALTER TABLE audit.erasure_receipt ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit.erasure_receipt FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_receipt_tenant ON audit.erasure_receipt
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

`chain_record_id` aponta para o registro em `audit.record` que **prova** o comprovante:
apagar sem deixar prova do apagamento seria indistinguível de nunca ter apagado.
Comprovantes de `../005-Database/`, `../010-Memory/` e `../024-Observability/` entram
por esta tabela com `origin_module` preservado e `kind = 'external'`.

---

## 8. `audit.export` e `audit.anomaly` (multi-tenant, RLS)

```sql
CREATE TABLE audit.export (
    urn          text        PRIMARY KEY,
    tenant_id    text        NOT NULL,
    scope        jsonb       NOT NULL,
    requested_by text        NOT NULL,
    approved_by  text        NOT NULL,
    record_count bigint      NOT NULL DEFAULT 0,
    seal_count   int         NOT NULL DEFAULT 0,
    package_hash text        UNIQUE,
    package_object text,
    status       text        NOT NULL DEFAULT 'requested'
                 CHECK (status IN ('requested','building','ready','failed','expired')),
    expires_at   timestamptz NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT ck_export_approver CHECK (approved_by <> requested_by)
);

CREATE INDEX ix_export_status ON audit.export (tenant_id, status);
CREATE INDEX ix_export_expiry ON audit.export (expires_at);

CREATE TABLE audit.anomaly (
    id          char(26)    PRIMARY KEY,
    tenant_id   text        NOT NULL,
    kind        text        NOT NULL
                CHECK (kind IN ('chain_break','sequence_gap','seal_overdue',
                                'class_silent','volume_anomaly','query_pattern')),
    severity    text        NOT NULL CHECK (severity IN ('critical','high','medium')),
    detail      jsonb       NOT NULL,
    detected_at timestamptz NOT NULL DEFAULT now(),
    resolved_at timestamptz,
    resolution  text,
    CONSTRAINT ck_anomaly_resolution CHECK ((resolved_at IS NULL) = (resolution IS NULL))
);

CREATE INDEX ix_anomaly_kind ON audit.anomaly (tenant_id, kind, detected_at DESC);
CREATE INDEX ix_anomaly_open ON audit.anomaly (severity)
    WHERE resolved_at IS NULL;

ALTER TABLE audit.export  ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit.export  FORCE  ROW LEVEL SECURITY;
ALTER TABLE audit.anomaly ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit.anomaly FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_export_tenant  ON audit.export
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
CREATE POLICY p_anomaly_tenant ON audit.anomaly
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

`ck_anomaly_resolution` impede fechar uma anomalia sem registrar a conclusão: uma
quebra de cadeia marcada como "resolvida" sem explicação é pior que uma aberta.

---

## 9. `audit.outbox` e o arquivamento WORM

```sql
CREATE TABLE audit.outbox (
    id           char(26)    PRIMARY KEY,
    tenant_id    text        NOT NULL,
    subject      text        NOT NULL,
    event_type   text        NOT NULL,
    payload      jsonb       NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    published_at timestamptz
);

CREATE INDEX ix_outbox_pending ON audit.outbox (created_at)
    WHERE published_at IS NULL;
```

O arquivamento WORM **não é uma tabela**: é o objeto gravado no MinIO com *object
lock* por `aud.worm.object_lock_days` (1825), referenciado por `audit.seal.worm_object`.
É a **terceira fonte independente** da verificação (`./StateMachine.md` §7):

```
   fonte 1: PostgreSQL     (pode ser adulterado por quem opera o banco)
   fonte 2: selo assinado  (a chave privada está no 021-Security)
   fonte 3: objeto WORM    (object lock impede sobrescrita no armazenamento)

   adulterar sem detecção exige comprometer as TRÊS.
```

---

## 10. Relações

```
   audit.auditable_class(1) ──rege──< audit.record(*)
                                          │  │
                     seq/prev_hash ───────┘  │ seal_ref
                     (cadeia por partição)   ▼
                                        audit.seal(*) ──[prev_seal_hash]──▶ audit.seal
                                             │                              (cadeia de selos)
                                             └──[worm_object]──▶ MinIO (object lock)

   audit.record(*) ──[subject_ref]──▶ audit.subject_key(1) ──destruída──▶ Shredded
        │
        ├──[decision_ref]──▶ 022-Policy (journal, retenção curta)
        └──[chain_record_id]◀── audit.erasure_receipt(*)

   audit.legal_hold(*) ══sobrepõe══╳ retenção e apagamento de audit.record
   audit.export(*) ──inclui──▶ audit.record(*) + audit.seal(*) + provas de Merkle
   audit.anomaly(*) ◀──detecta── IntegrityVerifier · CompletenessMonitor
```

---

## 11. Particionamento e retenção

| Tabela | Estratégia | Retenção | Observação |
|--------|-----------|----------|------------|
| `record` | `RANGE(occurred_at)` **mensal** + partição lógica de cadeia (`partition_key`) | **Linha: permanente.** Payload: `retention_days` da classe (default 1825) | O expurgo remove `payload_cipher`, nunca a linha (invariante I1) |
| `seal` | sem partição | Permanente | Sem selo, os registros do intervalo tornam-se inverificáveis |
| `auditable_class` | sem partição | Permanente | Histórico de contrato |
| `legal_hold` | sem partição | Permanente | Prova de governança |
| `subject_key` | sem partição | Permanente (com `state = destroyed`) | Provar que a chave foi destruída é parte do comprovante |
| `erasure_receipt` | sem partição | Permanente | O comprovante é a prova do apagamento |
| `export` | sem partição | 5 anos | Quem exportou o quê |
| `anomaly` | sem partição | Permanente | Histórico de integridade |
| `outbox` | sem partição | Expurgo de publicados após 7 dias | Operacional |

> **Nenhuma linha de `audit.record` é jamais removida.** Uma partição mensal antiga
> continua existindo com registros `Expired`: `seq`, hashes e metadados mínimos, sem
> payload. É o que permite verificar a cadeia de 2026 em 2032 — sem guardar o conteúdo
> de 2026 além do prazo legal.

### 11.1 O que sobrevive ao expurgo

```
   Registro em Received/Sealed        →  Registro em Expired/Shredded
   ─────────────────────────────         ────────────────────────────
   id, seq, partition_key             →  id, seq, partition_key        ✔
   occurred_at, recorded_at           →  occurred_at, recorded_at      ✔
   record_class, producer_module      →  record_class, producer_module ✔
   actor, action, resource_urn        →  actor, action, resource_urn   ✔
   outcome, decision_ref              →  outcome, decision_ref         ✔
   prev_hash, record_hash             →  prev_hash, record_hash        ✔
   payload_digest                     →  payload_digest                ✔
   payload_cipher                     →  NULL                          ✘
```

Prova-se **que** a ação ocorreu, **quem** a fez, **sobre o quê** e **com que
resultado** — sem preservar o detalhe além do prazo legal ou após o direito ao
esquecimento.

---

## 12. LGPD/GDPR no modelo

| Item | Tratamento |
|------|-----------|
| **A tensão central** | Imutabilidade × direito ao esquecimento: resolvida por **apagamento criptográfico** (ADR-0253), não por remoção de linha. |
| Cifragem | `payload_cipher` cifrado com a chave do titular quando `contains_pii`; com a chave do tenant caso contrário. |
| Apagamento | Destruição da chave em `audit.subject_key`; o registro permanece `Shredded` com cadeia íntegra. |
| Prova de existência | `payload_digest` sobrevive: prova que **houve** conteúdo, sem revelá-lo. |
| Base legal | `legal_basis` obrigatória em **toda** classe (§4), não só nas com PII. |
| *Legal hold* | **Prevalece** sobre o RTBF (invariante I5); o titular é informado da obrigação legal. |
| Comprovante | `audit.erasure_receipt` com `receipt_hash` e `chain_record_id`. |
| Segregação | RLS em todas as tabelas multi-tenant; partição lógica de cadeia por tenant. |
| Minimização | Payload contém o mínimo para provar a ação; conteúdo de negócio é do módulo dono. |

---

## 13. Migrações

Estratégia **expand/contract** (CV-11), com DDL bloqueante ≤ 200 ms e índices
`CONCURRENTLY` (CV-12).

| # | Migração | Tipo | Observação |
|---|----------|------|------------|
| M-001 | Schema, `auditable_class`, `subject_key` | expand | Base do registro |
| M-002 | `record` + partições + *trigger* de imutabilidade | expand | O *trigger* entra **junto** com a tabela, nunca depois |
| M-003 | `seal` + *trigger*; índices `uq_record_seq`, `uq_record_hash` | expand | `CONCURRENTLY` |
| M-004 | `legal_hold`, `erasure_receipt`, `export`, `anomaly`, `outbox` | expand | — |
| M-005 | Nova coluna em `record` | expand → backfill **apenas em novos registros** | Registros existentes **não** podem ser alterados (I6); a coluna nasce nula no histórico |
| M-006 | Rotação de partições mensais | expand | Job cria M+2 partições |

> **Uma migração deste schema NÃO DEVE alterar registros existentes.** Adicionar coluna
> é permitido; preencher retroativamente **não** é — mudaria o `record_hash` esperado e
> quebraria a cadeia. Se um campo novo é necessário para registros antigos, o caminho é
> um **registro de compensação** que o adiciona como fato novo.
>
> Uma migração também **NÃO DEVE** remover o *trigger* de imutabilidade. Sua remoção é
> exatamente o primeiro passo de uma adulteração — e é por isso que a cópia WORM existe
> independentemente dele.

---

## 14. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §3
- Convenções físicas: `../005-Database/Database.md` §2 (CV-01..CV-12)
- FSM e invariantes: `./StateMachine.md` §5
- Retenção operacional: `./Configuration.md` §5 · Escala: `./Scalability.md`
- Segurança e privacidade: `./Security.md`

*Fim de `Database.md`.*
