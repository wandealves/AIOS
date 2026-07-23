---
Documento: Database
Módulo: 021-Security
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 021-Security
ADRs relacionados: ADR-0005, ADR-0212, ADR-0213, ADR-0216, ADR-0217
RFCs relacionados: RFC-0001, RFC-0210
Depende de: _DESIGN_BRIEF.md §3, 005-Database (convenções físicas), StateMachine.md
---

# 021-Security — Modelo Físico

Schema **`security`** no PostgreSQL 16, sob as convenções físicas obrigatórias de
`../005-Database/Database.md` §2 (CV-01..CV-12).

> **Regra absoluta deste modelo.** Nenhuma tabela armazena **material secreto em
> claro**. Segredos e chaves privadas são gravados **cifrados por envelope** (DEK do
> KMS) ou permanecem no HSM. O que a tabela guarda é **metadado, referência e
> `fingerprint`** — nunca o segredo. Um dump do banco `security` não deve conceder
> acesso a nada.

---

## 1. Preparação

```sql
CREATE SCHEMA IF NOT EXISTS security;

-- Variável de sessão usada pelas políticas de RLS (padrão do 005)
--   SET LOCAL app.tenant_id = 'acme';
```

Tabelas multi-tenant: `principal`, `credential`, `secret`, `revocation`,
`federation_trust`. Tabelas de plataforma (`key`, `sandbox_profile`) usam o tenant
reservado `_platform` e não são multi-tenant.

---

## 2. `security.principal` (multi-tenant, RLS)

```sql
CREATE TABLE security.principal (
    urn           text        PRIMARY KEY,
    tenant_id     text        NOT NULL,
    kind          text        NOT NULL CHECK (kind IN ('human','service','agent','external')),
    external_ref  text,
    display_name  text        NOT NULL,
    attributes    jsonb       NOT NULL DEFAULT '{}'::jsonb,
    status        text        NOT NULL DEFAULT 'active'
                  CHECK (status IN ('active','suspended','disabled')),
    mfa_required  boolean     NOT NULL DEFAULT false,
    last_authn_at timestamptz,
    version       bigint      NOT NULL DEFAULT 0,
    created_at    timestamptz NOT NULL DEFAULT now(),
    updated_at    timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_principal_external UNIQUE (tenant_id, kind, external_ref),
    CONSTRAINT ck_mfa_only_human CHECK (NOT mfa_required OR kind = 'human')
);

CREATE INDEX ix_principal_status ON security.principal (tenant_id, status);
CREATE INDEX ix_principal_authn  ON security.principal (tenant_id, last_authn_at DESC);

ALTER TABLE security.principal ENABLE ROW LEVEL SECURITY;
ALTER TABLE security.principal FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_principal_tenant ON security.principal
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

`attributes` são os **atributos assertáveis** consumidos pelo `../022-Policy/`. O 021
os declara; o PDP decide o que significam — a fronteira de ADR-0210 expressa em uma
coluna.

---

## 3. `security.credential` (multi-tenant, RLS, particionada)

```sql
CREATE TABLE security.credential (
    urn               text        NOT NULL,
    tenant_id         text        NOT NULL,
    principal_urn     text        NOT NULL,
    kind              text        NOT NULL
                      CHECK (kind IN ('access_token','refresh_token','workload_cert',
                                      'db_role','nats_account','object_key',
                                      'provider_key','signing_key')),
    state             text        NOT NULL
                      CHECK (state IN ('Requested','Issued','Active','Rotating',
                                       'Superseded','Revoked','Expired','Failed')),
    purpose           text        NOT NULL,
    material_ref      text,
    fingerprint       text        NOT NULL,
    issued_at         timestamptz NOT NULL DEFAULT now(),
    not_before        timestamptz NOT NULL,
    expires_at        timestamptz NOT NULL,          -- ⚠ invariante I1
    rotated_to        text,
    revoked_at        timestamptz,
    revocation_reason text,
    issued_by         text        NOT NULL,
    idempotency_key   text,
    version           bigint      NOT NULL DEFAULT 0,
    PRIMARY KEY (urn, expires_at),
    CONSTRAINT ck_validity_window CHECK (not_before < expires_at),
    CONSTRAINT ck_revoked_consistency
        CHECK ((state = 'Revoked') = (revoked_at IS NOT NULL)),
    CONSTRAINT ck_reason_when_revoked
        CHECK (state <> 'Revoked' OR revocation_reason IS NOT NULL)
) PARTITION BY RANGE (expires_at);

CREATE UNIQUE INDEX uq_credential_fingerprint ON security.credential (fingerprint, expires_at);
CREATE UNIQUE INDEX uq_credential_idem
    ON security.credential (tenant_id, idempotency_key, expires_at)
    WHERE idempotency_key IS NOT NULL;
CREATE INDEX ix_credential_principal ON security.credential (tenant_id, principal_urn, state);
CREATE INDEX ix_credential_expiring  ON security.credential (expires_at)
    WHERE state IN ('Issued','Active','Rotating');
CREATE INDEX ix_credential_kind      ON security.credential (kind, state);

ALTER TABLE security.credential ENABLE ROW LEVEL SECURITY;
ALTER TABLE security.credential FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_credential_tenant ON security.credential
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));

-- Partições mensais, pré-criadas pelo PartitionManager do 005
CREATE TABLE security.credential_2026_08 PARTITION OF security.credential
    FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
```

Três decisões de esquema merecem justificativa:

1. **`expires_at NOT NULL`** codifica a invariante I1 no banco. Não é possível gravar
   credencial eterna nem por engano nem por urgência operacional.
2. **Particionamento por `expires_at`** (e não por `created_at`) torna o expurgo de
   credenciais vencidas um `DROP` de partição: as vencidas ficam **juntas**.
3. **`ck_reason_when_revoked`** impede revogação sem motivo registrado — o que
   inviabilizaria a análise posterior de um incidente.

---

## 4. `security.secret` (multi-tenant, RLS)

```sql
CREATE TABLE security.secret (
    urn             text        PRIMARY KEY,
    tenant_id       text        NOT NULL,
    path            text        NOT NULL,
    version_no      integer     NOT NULL,
    ciphertext      bytea       NOT NULL,           -- ⚠ envelope; nunca em claro
    dek_ref         text        NOT NULL REFERENCES security.key(urn),
    lease_ttl       interval    NOT NULL DEFAULT '1 hour',
    rotation_period interval,
    last_rotated_at timestamptz,
    created_at      timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_secret_version UNIQUE (tenant_id, path, version_no),
    CONSTRAINT ck_version_positive CHECK (version_no > 0)
);

CREATE INDEX ix_secret_current ON security.secret (tenant_id, path, version_no DESC);

ALTER TABLE security.secret ENABLE ROW LEVEL SECURITY;
ALTER TABLE security.secret FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_secret_tenant ON security.secret
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

Versões são **imutáveis**: escrever um segredo cria `version_no + 1`, nunca atualiza a
linha existente. Isso permite reverter uma rotação problemática e auditar o que estava
vigente em qualquer momento.

---

## 5. `security.key`

```sql
CREATE TABLE security.key (
    urn              text        PRIMARY KEY,
    role             text        NOT NULL
                     CHECK (role IN ('kek','dek','signing','ca_root','ca_intermediate')),
    algorithm        text        NOT NULL,
    state            text        NOT NULL
                     CHECK (state IN ('pending','active','rotating','retired','destroyed')),
    hsm_backed       boolean     NOT NULL DEFAULT false,
    public_material  bytea,
    wrapped_material bytea,                          -- cifrado pela KEK (NULL se HSM)
    parent_key       text        REFERENCES security.key(urn),
    activated_at     timestamptz,
    retire_after     timestamptz,
    destroyed_at     timestamptz,
    version          bigint      NOT NULL DEFAULT 0,
    created_at       timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT ck_material_present
        CHECK (hsm_backed OR wrapped_material IS NOT NULL OR role = 'ca_root'),
    CONSTRAINT ck_destroyed_consistency
        CHECK ((state = 'destroyed') = (destroyed_at IS NOT NULL))
);

-- Apenas UMA chave ativa por papel: impede assinar com chave ambígua
CREATE UNIQUE INDEX uq_key_active_per_role ON security.key (role)
    WHERE state = 'active';

CREATE INDEX ix_key_state  ON security.key (role, state);
CREATE INDEX ix_key_retire ON security.key (retire_after) WHERE state = 'rotating';
```

`ck_material_present` permite `ca_root` sem material no banco: a chave da CA raiz vive
**offline**, em custódia física ou HSM, e a linha aqui existe apenas para referência de
cadeia e auditoria.

O índice único parcial `uq_key_active_per_role` garante que nunca haja duas chaves
`active` para o mesmo papel — situação em que a escolha de qual usar para assinar seria
não-determinística.

---

## 6. `security.sandbox_profile`

```sql
CREATE TABLE security.sandbox_profile (
    urn              text        PRIMARY KEY,
    name             text        NOT NULL,
    profile_version  integer     NOT NULL,
    seccomp          jsonb       NOT NULL,
    cgroups          jsonb       NOT NULL,
    namespaces       text[]      NOT NULL,
    egress_allowlist text[]      NOT NULL DEFAULT '{}',
    signature        bytea       NOT NULL,
    signed_by        text        NOT NULL REFERENCES security.key(urn),
    status           text        NOT NULL DEFAULT 'draft'
                     CHECK (status IN ('draft','published','deprecated')),
    created_at       timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_profile_version UNIQUE (name, profile_version),
    CONSTRAINT ck_namespaces_minimum
        CHECK (namespaces @> ARRAY['pid','net','mnt','ipc']::text[])
);

CREATE INDEX ix_profile_status ON security.sandbox_profile (status);
```

`ck_namespaces_minimum` impede publicar um perfil sem os namespaces mínimos de
isolamento — um perfil sem `net` ou `pid`, por exemplo, não isolaria efetivamente um
agente executando código de terceiros.

**Imutabilidade:** linhas com `status = 'published'` **NÃO DEVEM** ser atualizadas
(exceto para `deprecated`); mudança gera nova `profile_version`
(`AIOS-SEC-0012`). Isso preserva a capacidade de auditar sob qual isolamento um agente
executou em qualquer momento do passado.

---

## 7. `security.revocation` (multi-tenant, RLS)

```sql
CREATE TABLE security.revocation (
    id             char(26)    PRIMARY KEY,
    tenant_id      text        NOT NULL,
    credential_urn text        NOT NULL,
    fingerprint    text        NOT NULL UNIQUE,
    reason         text        NOT NULL
                   CHECK (reason IN ('compromise','superseded','principal_disabled',
                                     'policy','operator')),
    effective_at   timestamptz NOT NULL DEFAULT now(),
    expires_at     timestamptz NOT NULL,
    propagated_at  timestamptz
);

CREATE INDEX ix_revocation_tenant ON security.revocation (tenant_id, effective_at DESC);
CREATE INDEX ix_revocation_expiry ON security.revocation (expires_at);
CREATE INDEX ix_revocation_pending ON security.revocation (effective_at)
    WHERE propagated_at IS NULL;
```

`expires_at` da revogação é igual à expiração natural da credencial: a entrada só pode
sair da lista **depois** que a credencial expiraria de qualquer forma. Removê-la antes
equivaleria a desfazer a revogação.

O índice parcial `ix_revocation_pending` sustenta o monitoramento de NFR-006
(propagação ≤ 30 s) em O(pendentes).

---

## 8. `security.federation_trust` (multi-tenant, RLS)

```sql
CREATE TABLE security.federation_trust (
    urn            text        PRIMARY KEY,
    tenant_id      text        NOT NULL,
    kind           text        NOT NULL CHECK (kind IN ('idp','a2a_peer')),
    issuer         text        NOT NULL,
    jwks_uri       text,
    trust_anchor   bytea,
    allowed_scopes text[]      NOT NULL DEFAULT '{}',
    status         text        NOT NULL DEFAULT 'active'
                   CHECK (status IN ('active','suspended','revoked')),
    version        bigint      NOT NULL DEFAULT 0,
    created_at     timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_federation_issuer UNIQUE (tenant_id, issuer),
    CONSTRAINT ck_trust_material CHECK (jwks_uri IS NOT NULL OR trust_anchor IS NOT NULL)
);

CREATE INDEX ix_federation_kind ON security.federation_trust (tenant_id, kind, status);

ALTER TABLE security.federation_trust ENABLE ROW LEVEL SECURITY;
ALTER TABLE security.federation_trust FORCE  ROW LEVEL SECURITY;
CREATE POLICY p_federation_tenant ON security.federation_trust
    USING      (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));
```

`allowed_scopes` vazio (o default) significa **nenhum escopo interno concedido** — a
identidade externa é reconhecida, mas não recebe privilégio algum até que a lista seja
preenchida deliberadamente (ADR-0218).

---

## 9. `security.outbox`

Instância do contrato canônico de `../005-Database/Database.md` §3.9:

```sql
CREATE TABLE security.outbox (
    id         char(26)    PRIMARY KEY,
    tenant_id  text        NOT NULL,
    subject    text        NOT NULL,
    payload    jsonb       NOT NULL,
    published  boolean     NOT NULL DEFAULT false,
    created_at timestamptz NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX ix_sec_outbox_pending ON security.outbox (created_at) WHERE published = false;
```

É o Outbox que garante a invariante I4: revogação e evento commitam juntos.

---

## 10. Roles de Banco e Acesso

| *Role* | Concessões | Quem usa |
|--------|-----------|----------|
| `security_rw` | `SELECT/INSERT/UPDATE` no schema `security` | `aios-security-svc` |
| `security_ca_rw` | `SELECT/INSERT/UPDATE` em `credential` e `key` (papéis de CA) | `aios-security-ca` (processo isolado) |
| `security_ro` | `SELECT` em `revocation` | Verificadores que consultam a CRL |

Nenhuma outra *role* do AIOS tem acesso ao schema `security`. Note a circularidade
resolvida: as credenciais dessas *roles* são as **únicas** de bootstrap, provisionadas
na instalação e rotacionadas pela cerimônia de `../029-Operations/` — o módulo que
emite credenciais precisa de uma credencial inicial que não pode emitir para si mesmo.

---

## 11. Classificação e Retenção

| Tabela | `data_class` | Multi-tenant | Retenção | Método |
|--------|--------------|--------------|----------|--------|
| `security.principal` | **`pii`** | sim | enquanto ativo + 5 anos (obrigação legal) | `delete_batch` + tokenização |
| `security.credential` | `confidential` | sim | 1 ano após `expires_at` | `drop_partition` |
| `security.secret` | `confidential` | sim | versões: últimas 10 ou 1 ano | `delete_batch` |
| `security.key` | `confidential` | não | permanente até `destroyed` | — |
| `security.sandbox_profile` | `internal` | não | permanente (auditoria de isolamento) | — |
| `security.revocation` | `confidential` | sim | até `expires_at` + 90 dias | `drop_partition` |
| `security.federation_trust` | `internal` | sim | enquanto ativa | — |
| `security.outbox` | `internal` | sim | 7 dias | `drop_partition` |

`security.principal` é a única tabela `pii` do módulo e, conforme CV-08 do
`../005-Database/`, **DEVE** declarar `legal_basis` em sua política de retenção —
tipicamente execução de contrato e obrigação legal de registro de acesso.

---

## 12. Consultas Operacionais Frequentes

```sql
-- Credenciais ativas sem expiração (DEVE retornar zero linhas — invariante I1)
SELECT count(*) FROM security.credential
 WHERE state IN ('Issued','Active','Rotating') AND expires_at IS NULL;

-- Credenciais próximas da expiração sem rotação programada (FR-019)
SELECT urn, principal_urn, kind, purpose, expires_at - now() AS restante
  FROM security.credential
 WHERE state = 'Active'
   AND rotated_to IS NULL
   AND expires_at < now() + interval '6 hours'
 ORDER BY expires_at;

-- Revogações ainda não propagadas além da meta (NFR-006 = 30 s)
SELECT id, credential_urn, now() - effective_at AS atraso
  FROM security.revocation
 WHERE propagated_at IS NULL
   AND effective_at < now() - interval '30 seconds';

-- Violação da invariante I2: mais de duas credenciais utilizáveis por purpose
SELECT principal_urn, purpose, count(*)
  FROM security.credential
 WHERE state IN ('Active','Rotating')
 GROUP BY 1, 2
HAVING count(*) > 2;

-- Chaves fora do período de rotação
SELECT urn, role, algorithm, now() - activated_at AS idade
  FROM security.key
 WHERE state = 'active' AND activated_at < now() - interval '90 days';
```

A primeira e a quarta consultas são **verificações de invariante** e rodam como job
contínuo: qualquer linha retornada é incidente, não relatório.

---

## 13. Referências

- Brief: `./_DESIGN_BRIEF.md` §3
- Convenções físicas e RLS: `../005-Database/Database.md` §2 e §4
- FSM: `./StateMachine.md` · API: `./API.md` · Segurança: `./Security.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
