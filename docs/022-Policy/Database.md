---
Documento: Database
Módulo: 022-Policy
Status: Draft
Versão: 0.1
Última atualização: 2026-07-23
Responsável (RACI-A): Arquiteto do Módulo 022-Policy
ADRs relacionados: ADR-0005, ADR-0221, ADR-0227, ADR-0229
RFCs relacionados: RFC-0001, RFC-0220, RFC-0221
Depende de: _DESIGN_BRIEF.md §3, 005-Database (convenções físicas CV-01..CV-12), StateMachine.md
---

# 022-Policy — Modelo Físico

Schema **`policy`** no PostgreSQL 16, sob as convenções físicas obrigatórias de
`../005-Database/Database.md` §2 (CV-01..CV-12).

> **Regra absoluta deste modelo.** A fonte da verdade da autorização é o **bundle**,
> não uma tabela de permissões concedidas. Nenhuma decisão é persistida como
> autorização durável: decisão tem TTL, é efêmera e é sempre reproduzível a partir de
> (`bundle_version`, `DecisionRequest`, `attributes_digest`). Um dump deste banco
> descreve **qual é a política** — não "quem já pode o quê".

---

## 1. Preparação

```sql
CREATE SCHEMA IF NOT EXISTS policy;

-- Variável de sessão usada pelas políticas de RLS (padrão do 005)
--   SET LOCAL app.tenant_id = 'acme';
```

Tabelas multi-tenant (RLS obrigatória): `policy_bundle`, `policy_rule`, `role`,
`role_binding`, `policy_exception`, `policy_test`, `simulation_run`,
`decision_journal`. Tabela de plataforma: `attribute_source` (usa o tenant reservado
`_platform` e não é multi-tenant).

---

## 2. `policy.policy_bundle` (multi-tenant, RLS)

Unidade de release da política — entidade governada pela FSM de `./StateMachine.md`.

```sql
CREATE TABLE policy.policy_bundle (
    urn               text        PRIMARY KEY,
    tenant_id         text        NOT NULL,
    scope             text        NOT NULL DEFAULT 'tenant'
                      CHECK (scope IN ('tenant','platform')),
    bundle_version    int         NOT NULL,
    state             text        NOT NULL DEFAULT 'Draft'
                      CHECK (state IN ('Draft','Validating','Validated','Tested',
                                       'Simulated','Active','Superseded','RolledBack',
                                       'Rejected','Archived')),
    source_digest     text        NOT NULL,
    compiled_artifact bytea,
    compiled_digest   text,
    signature         bytea,
    signed_by         text,
    rule_count        int         NOT NULL DEFAULT 0,
    test_coverage     numeric(5,4),
    authored_by       text        NOT NULL,
    approved_by       text,
    activated_at      timestamptz,
    superseded_by     text        REFERENCES policy.policy_bundle (urn),
    rollback_of       text        REFERENCES policy.policy_bundle (urn),
    idempotency_key   text,
    version           bigint      NOT NULL DEFAULT 0,
    created_at        timestamptz NOT NULL DEFAULT now(),
    updated_at        timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_bundle_version   UNIQUE (tenant_id, scope, bundle_version),
    CONSTRAINT uq_bundle_idem      UNIQUE (tenant_id, idempotency_key),
    CONSTRAINT ck_bundle_approver  CHECK (approved_by IS NULL OR approved_by <> authored_by),
    CONSTRAINT ck_bundle_active    CHECK (state <> 'Active' OR
                                          (compiled_artifact IS NOT NULL AND
                                           signature IS NOT NULL AND
                                           activated_at IS NOT NULL))
);

-- Invariante I1 materializada: no máximo UM bundle Active por (tenant, escopo)
CREATE UNIQUE INDEX uq_bundle_one_active
    ON policy.policy_bundle (tenant_id, scope)
    WHERE state = 'Active';

CREATE INDEX ix_bundle_state   ON policy.policy_bundle (tenant_id, state);
CREATE INDEX ix_bundle_digest  ON policy.policy_bundle (compiled_digest);
CREATE INDEX ix_bundle_supers  ON policy.policy_bundle (superseded_by);   -- CV-05
CREATE INDEX ix_bundle_rollback ON policy.policy_bundle (rollback_of);    -- CV-05

ALTER TABLE policy.policy_bundle ENABLE ROW LEVEL SECURITY;
ALTER TABLE policy.policy_bundle FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_bundle_tenant ON policy.policy_bundle
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

> `uq_bundle_one_active` **não é otimização**: é a invariante I1 no lugar certo.
> Deixá-la apenas na aplicação é convidar duas verdades simultâneas de política — e
> duas políticas ativas significam que a resposta a "este sujeito pode?" depende de
> qual réplica atendeu.
>
> `ck_bundle_approver` implementa a separação de funções (FR-009) no nível do
> esquema: nenhum caminho de código consegue publicar um bundle aprovado pelo próprio
> autor.

---

## 3. `policy.policy_rule` (multi-tenant, RLS)

```sql
CREATE TABLE policy.policy_rule (
    urn              text        PRIMARY KEY,
    tenant_id        text        NOT NULL,
    bundle_urn       text        NOT NULL REFERENCES policy.policy_bundle (urn)
                                 ON DELETE CASCADE,
    rule_key         text        NOT NULL,
    effect           text        NOT NULL CHECK (effect IN ('allow','deny')),
    priority         int         NOT NULL DEFAULT 100 CHECK (priority BETWEEN 0 AND 1000),
    action_pattern   text        NOT NULL,
    resource_pattern text        NOT NULL,
    subject_match    jsonb       NOT NULL DEFAULT '{}'::jsonb,
    condition        jsonb       NOT NULL DEFAULT '{}'::jsonb,
    obligations      jsonb       NOT NULL DEFAULT '[]'::jsonb,
    decision_ttl_ms  int         CHECK (decision_ttl_ms IS NULL OR decision_ttl_ms BETWEEN 0 AND 60000),
    dispensable      boolean     NOT NULL DEFAULT true,
    description      text        NOT NULL CHECK (length(btrim(description)) > 0),
    created_at       timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_rule_key UNIQUE (bundle_urn, rule_key)
);

CREATE INDEX ix_rule_priority  ON policy.policy_rule (bundle_urn, priority);
CREATE INDEX ix_rule_condition ON policy.policy_rule USING gin (condition jsonb_path_ops);
CREATE INDEX ix_rule_action    ON policy.policy_rule (bundle_urn, action_pattern);

ALTER TABLE policy.policy_rule ENABLE ROW LEVEL SECURITY;
ALTER TABLE policy.policy_rule FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_rule_tenant ON policy.policy_rule
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

- `description` **NÃO DEVE** ser vazia (`CHECK`): uma regra sem intenção declarada é
  irrevisável, e a revisão humana é o único controle contra política errada em um
  bundle que compila.
- `dispensable = false` marca a regra `deny` que **nenhum waiver** pode converter em
  `allow` (`./StateMachine.md` §6, passo 4).
- `condition` guarda uma **AST** (não texto livre): expressões são compiladas pelo
  `BundleCompiler`, nunca interpretadas no caminho quente.

---

## 4. `policy.role` e `policy.role_binding` (multi-tenant, RLS)

```sql
CREATE TABLE policy.role (
    urn           text        PRIMARY KEY,
    tenant_id     text        NOT NULL,
    bundle_urn    text        NOT NULL REFERENCES policy.policy_bundle (urn)
                              ON DELETE CASCADE,
    role_key      text        NOT NULL,
    inherits      text[]      NOT NULL DEFAULT '{}',
    permissions   jsonb       NOT NULL DEFAULT '[]'::jsonb,
    is_privileged boolean     NOT NULL DEFAULT false,
    description   text        NOT NULL CHECK (length(btrim(description)) > 0),
    created_at    timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_role_key UNIQUE (bundle_urn, role_key)
);

CREATE INDEX ix_role_bundle     ON policy.role (bundle_urn);            -- CV-05
CREATE INDEX ix_role_privileged ON policy.role (tenant_id, is_privileged);

CREATE TABLE policy.role_binding (
    urn           text        PRIMARY KEY,
    tenant_id     text        NOT NULL,
    bundle_urn    text        NOT NULL REFERENCES policy.policy_bundle (urn)
                              ON DELETE CASCADE,
    role_key      text        NOT NULL,
    subject_urn   text        NOT NULL,
    subject_kind  text        NOT NULL
                  CHECK (subject_kind IN ('principal','group','agent','service')),
    scope_pattern text        NOT NULL DEFAULT '*',
    not_after     timestamptz,
    granted_by    text        NOT NULL,
    created_at    timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_binding UNIQUE (bundle_urn, role_key, subject_urn, scope_pattern)
);

CREATE INDEX ix_binding_subject ON policy.role_binding (tenant_id, subject_urn);
CREATE INDEX ix_binding_bundle  ON policy.role_binding (bundle_urn);     -- CV-05
CREATE INDEX ix_binding_expiry  ON policy.role_binding (not_after)
    WHERE not_after IS NOT NULL;

ALTER TABLE policy.role         ENABLE ROW LEVEL SECURITY;
ALTER TABLE policy.role         FORCE  ROW LEVEL SECURITY;
ALTER TABLE policy.role_binding ENABLE ROW LEVEL SECURITY;
ALTER TABLE policy.role_binding FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_role_tenant ON policy.role
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
CREATE POLICY p_binding_tenant ON policy.role_binding
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

A aciclicidade de `inherits` **NÃO** é verificável por constraint declarativa; é
validada pelo `BundleCompiler` (`AIOS-POL-0017`) durante T-03 — e por isso o bundle só
se torna `Validated` depois da compilação, nunca na escrita.

---

## 5. `policy.policy_exception` (multi-tenant, RLS)

```sql
CREATE TABLE policy.policy_exception (
    urn              text        PRIMARY KEY,
    tenant_id        text        NOT NULL,
    subject_urn      text        NOT NULL,
    action_pattern   text        NOT NULL,
    resource_pattern text        NOT NULL,
    state            text        NOT NULL DEFAULT 'Pending'
                     CHECK (state IN ('Pending','Active','Denied','Revoked','Expired')),
    justification    text        NOT NULL CHECK (length(btrim(justification)) > 0),
    requested_by     text        NOT NULL,
    approved_by      text        NOT NULL,
    effective_from   timestamptz NOT NULL DEFAULT now(),
    expires_at       timestamptz NOT NULL,
    revoked_at       timestamptz,
    version          bigint      NOT NULL DEFAULT 0,
    created_at       timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT ck_exc_window   CHECK (expires_at > effective_from),
    CONSTRAINT ck_exc_approver CHECK (approved_by <> requested_by)
);

CREATE INDEX ix_exc_subject ON policy.policy_exception (tenant_id, subject_urn, expires_at);
CREATE INDEX ix_exc_expiry  ON policy.policy_exception (expires_at)
    WHERE state = 'Active';

ALTER TABLE policy.policy_exception ENABLE ROW LEVEL SECURITY;
ALTER TABLE policy.policy_exception FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_exc_tenant ON policy.policy_exception
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

`expires_at` é `NOT NULL` **por desenho**, pela mesma razão que `expires_at` de
credencial é `NOT NULL` no `../021-Security/Database.md`: exceção sem prazo é política
nova aprovada por ninguém. O teto de 72 h vem de `pol.exception.max_ttl_h` e é
aplicado pelo `ExceptionManager` (`AIOS-POL-0013`).

---

## 6. `policy.attribute_source` (plataforma)

```sql
CREATE TABLE policy.attribute_source (
    urn                text        PRIMARY KEY,
    source_key         text        NOT NULL UNIQUE,
    provider_module    text        NOT NULL,
    criticality        text        NOT NULL CHECK (criticality IN ('required','optional')),
    resolve_timeout_ms int         NOT NULL DEFAULT 20  CHECK (resolve_timeout_ms BETWEEN 1 AND 200),
    cache_ttl_s        int         NOT NULL DEFAULT 30  CHECK (cache_ttl_s BETWEEN 0 AND 3600),
    sensitive          boolean     NOT NULL DEFAULT false,
    status             text        NOT NULL DEFAULT 'active'
                       CHECK (status IN ('active','degraded','disabled')),
    created_at         timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX ix_attrsrc_status ON policy.attribute_source (status);
```

Fontes registradas de origem: `principal` (`../021-Security/`), `budget`
(`../026-Cost-Optimizer/`), `threat` (`../021-Security/`, `../025-Audit/`),
`resource` (módulo dono do recurso), `request` (contexto local, sempre disponível).

---

## 7. `policy.decision_journal` (multi-tenant, RLS, particionada)

```sql
CREATE TABLE policy.decision_journal (
    decision_id       char(26)    NOT NULL,
    tenant_id         text        NOT NULL,
    decided_at        timestamptz NOT NULL DEFAULT now(),
    subject_urn       text        NOT NULL,
    action            text        NOT NULL,
    resource_urn      text        NOT NULL,
    effect            text        NOT NULL CHECK (effect IN ('allow','deny')),
    reason_code       text        NOT NULL,
    matched_rules     text[]      NOT NULL DEFAULT '{}',
    bundle_urn        text        NOT NULL,
    bundle_version    int         NOT NULL,
    attributes_digest text        NOT NULL,
    obligations       jsonb       NOT NULL DEFAULT '[]'::jsonb,
    latency_us        int         NOT NULL,
    pep_module        text        NOT NULL,
    trace_id          text        NOT NULL,
    CONSTRAINT pk_decision_journal PRIMARY KEY (tenant_id, decided_at, decision_id),
    CONSTRAINT ck_allow_has_rule CHECK (effect <> 'allow' OR cardinality(matched_rules) > 0)
) PARTITION BY RANGE (decided_at);

-- Partições diárias criadas com antecedência pelo job de manutenção
CREATE TABLE policy.decision_journal_2026_07_23
    PARTITION OF policy.decision_journal
    FOR VALUES FROM ('2026-07-23') TO ('2026-07-24');

CREATE INDEX ix_journal_subject ON policy.decision_journal (tenant_id, subject_urn, decided_at DESC);
CREATE INDEX ix_journal_effect  ON policy.decision_journal (tenant_id, effect, decided_at DESC);
CREATE INDEX ix_journal_bundle  ON policy.decision_journal (bundle_version, decided_at DESC);

ALTER TABLE policy.decision_journal ENABLE ROW LEVEL SECURITY;
ALTER TABLE policy.decision_journal FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_journal_tenant ON policy.decision_journal
    USING (tenant_id = current_setting('app.tenant_id', true));
```

> `ck_allow_has_rule` é o NFR-008 (*default deny* verificável) escrito como
> constraint: **não é possível gravar um `allow` sem regra casada**. Se o motor
> alguma vez produzir um, a escrita falha e o defeito aparece imediatamente, em vez de
> virar acesso silencioso.
>
> `attributes_digest` guarda o `sha256` do conjunto de atributos — nunca os valores
> (FR-023). É o suficiente para o `ExplainService` verificar que reproduziu a decisão
> com o mesmo insumo.

---

## 8. `policy.policy_test` e `policy.simulation_run` (multi-tenant, RLS)

```sql
CREATE TABLE policy.policy_test (
    urn             text        PRIMARY KEY,
    tenant_id       text        NOT NULL,
    test_key        text        NOT NULL,
    request         jsonb       NOT NULL,
    expected_effect text        NOT NULL CHECK (expected_effect IN ('allow','deny')),
    expected_reason text,
    is_mandatory    boolean     NOT NULL DEFAULT true,
    created_at      timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_test_key UNIQUE (tenant_id, test_key)
);

CREATE INDEX ix_test_mandatory ON policy.policy_test (tenant_id, is_mandatory);

CREATE TABLE policy.simulation_run (
    urn              text        PRIMARY KEY,
    tenant_id        text        NOT NULL,
    candidate_bundle text        NOT NULL REFERENCES policy.policy_bundle (urn),
    baseline_bundle  text        NOT NULL REFERENCES policy.policy_bundle (urn),
    window_from      timestamptz NOT NULL,
    window_to        timestamptz NOT NULL,
    evaluated_count  bigint      NOT NULL DEFAULT 0,
    flipped_to_deny  bigint      NOT NULL DEFAULT 0,
    flipped_to_allow bigint      NOT NULL DEFAULT 0,
    truncated        boolean     NOT NULL DEFAULT false,
    diff_by_rule     jsonb       NOT NULL DEFAULT '{}'::jsonb,
    status           text        NOT NULL DEFAULT 'running'
                     CHECK (status IN ('running','completed','failed')),
    completed_at     timestamptz,
    created_at       timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT ck_sim_window CHECK (window_to > window_from)
);

CREATE INDEX ix_sim_candidate ON policy.simulation_run (tenant_id, candidate_bundle);
CREATE INDEX ix_sim_baseline  ON policy.simulation_run (baseline_bundle);   -- CV-05
CREATE INDEX ix_sim_status    ON policy.simulation_run (status);

ALTER TABLE policy.policy_test    ENABLE ROW LEVEL SECURITY;
ALTER TABLE policy.policy_test    FORCE  ROW LEVEL SECURITY;
ALTER TABLE policy.simulation_run ENABLE ROW LEVEL SECURITY;
ALTER TABLE policy.simulation_run FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_test_tenant ON policy.policy_test
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
CREATE POLICY p_sim_tenant  ON policy.simulation_run
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

`truncated = true` marca a simulação que atingiu `pol.simulation.max_requests`: um
diferencial amostrado **DEVE** ser apresentado como tal na revisão de publicação —
silenciar a truncagem faria a amostra parecer cobertura completa.

---

## 9. `policy.outbox` (padrão transacional)

```sql
CREATE TABLE policy.outbox (
    id           char(26)    PRIMARY KEY,
    tenant_id    text        NOT NULL,
    subject      text        NOT NULL,
    event_type   text        NOT NULL,
    payload      jsonb       NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    published_at timestamptz
);

CREATE INDEX ix_outbox_pending ON policy.outbox (created_at)
    WHERE published_at IS NULL;
```

Toda transição para `Active` grava a mudança **e** o evento na mesma transação
(invariante I4). O `policy-outbox-relay` publica e marca `published_at`; a
deduplicação no consumidor é por `event.id` (RFC-0001 §5.5).

---

## 10. Relações

```
   policy.policy_bundle(1) ──< policy.policy_rule(*)
        │  │
        │  ├──< policy.role(*) ──[inherits]──▶ policy.role   (grafo acíclico)
        │  │         ▲ role_key
        │  └──< policy.role_binding(*) ──▶ subject_urn
        │
        ├──[superseded_by]──▶ policy.policy_bundle (sucessora)
        ├──[rollback_of]────▶ policy.policy_bundle (versão restaurada)
        └──< policy.simulation_run(*) ──[baseline_bundle]──▶ policy.policy_bundle

   policy.attribute_source(*) ──alimenta──▶ avaliação ──▶ policy.decision_journal(*)
   policy.policy_exception(*) ──sobrepõe (com prazo)──▶ decisão
   policy.policy_test(*)      ──barra publicação──────▶ policy.policy_bundle
```

---

## 11. Particionamento e retenção

| Tabela | Estratégia | Retenção | Classificação |
|--------|-----------|----------|---------------|
| `decision_journal` | `RANGE(decided_at)`, partições **diárias** | `pol.journal.retention_days` (30 d) — `DROP PARTITION` | `operational` (contém URN de sujeito; ver §12) |
| `policy_bundle` | sem partição | versões `Superseded` mantidas por `pol.bundle.rollback_retention_versions` (10); depois `Archived` (T-13) | `config` |
| `policy_rule`, `role`, `role_binding` | sem partição | vive com o bundle (`ON DELETE CASCADE`) | `config` |
| `policy_exception` | sem partição | 1 ano após `expires_at` (prova de governança) | `operational` |
| `simulation_run` | sem partição | 90 dias | `operational` |
| `outbox` | sem partição | expurgo de publicados após 7 dias | `operational` |

A prova de longo prazo da decisão é da trilha imutável de `../025-Audit/` (ADR-0229).
Duplicar retenção aqui criaria uma segunda fonte da verdade de auditoria, com regras
de expurgo divergentes — e, em uma investigação, duas versões do mesmo fato.

---

## 12. LGPD/GDPR no modelo

| Item | Tratamento |
|------|-----------|
| Dados pessoais | O módulo **não** guarda conteúdo pessoal. `subject_urn` é identificador pseudonimizado (URN, RFC-0001 §5.1); o vínculo com a pessoa vive no `../021-Security/`. |
| Atributos sensíveis | **Nunca** persistidos: apenas `attributes_digest` (`sha256`) — FR-023. |
| Direito ao esquecimento | Ao consumir `security.principal.disabled`, o módulo expira vínculos e exceções do titular e **pseudonimiza** `subject_urn` no journal, preservando as contagens agregadas necessárias à auditoria de segurança. |
| Base legal | Journal e exceções são dado operacional de segurança (legítimo interesse/obrigação legal), declarados na política de retenção (CV-07/CV-08). |
| Segregação | RLS em todas as tabelas multi-tenant; `AIOS-POL-0003` para tenant divergente (NFR-016). |

---

## 13. Migrações

Estratégia **expand/contract** (CV-11), com DDL bloqueante ≤ 200 ms e índices
`CONCURRENTLY` (CV-12).

| # | Migração | Tipo | Observação |
|---|----------|------|------------|
| M-001 | Criação do schema e tabelas de configuração de política | expand | Nenhum dado prévio |
| M-002 | `decision_journal` + partições iniciais + job de rotação | expand | Job cria D+7 partições |
| M-003 | Índice `uq_bundle_one_active` | expand | `CONCURRENTLY`; exige verificar duplicidade antes |
| M-004 | Adição de coluna a `policy_rule` | expand → backfill → contract | Regras são imutáveis: backfill vale só para bundles em `Draft` |
| M-005 | Expurgo/arquivamento de bundles fora da janela | contract | Executa T-13; **NÃO DEVE** remover artefato de versão referenciada por simulação ou explicação pendente (I6) |

> Uma migração deste schema **NÃO DEVE** alterar a semântica de um bundle já `Active`.
> Se a mudança implicar reinterpretação de regra, o caminho correto é **publicar uma
> nova versão de bundle**, não migrar o dado — do contrário, decisões passadas deixam
> de ser reproduzíveis (NFR-009).

---

## 14. Referências

- Fonte única de verdade: `./_DESIGN_BRIEF.md` §3
- Convenções físicas: `../005-Database/Database.md` §2 (CV-01..CV-12)
- FSM e invariantes: `./StateMachine.md` §5
- Retenção e trilha: `../025-Audit/`
- Segurança: `./Security.md` · Escala: `./Scalability.md`

*Fim de `Database.md`.*
