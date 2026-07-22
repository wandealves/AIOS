---
Documento: Database
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0005 (global), ADR-0043
RFCs relacionados: RFC-0001 (§5.1, §8)
Depende de: ../005-Database/README.md, ./StateMachine.md, ./ClassDiagrams.md
---

# 004-API — Modelo de Dados Físico (PostgreSQL)

> Alinhado à RFC-0001 §5.1. Reproduz e detalha `./_DESIGN_BRIEF.md` §3.
> **Nuance deste módulo**: os registros de contrato (`route`, `version`,
> `error_code`, `event_schema`, `resource_type`) são **metadados de
> plataforma**, não recursos de negócio de um tenant — por isso **NÃO
> carregam `tenant_id`/RLS**; sua escrita é restrita ao papel administrativo
> `api-registry-admin`, e sua leitura é pública. Já o estado de borda por
> tenant (`rate_limit_policy`, `idempotency_record`) **carrega `tenant_id` +
> RLS**, como exigido pela RFC-0001 §5.1 para todo recurso multi-tenant. Esta
> decisão de design (registro global vs. estado por tenant) DEVE ser
> validada no ADR-0043.

## 1. Schema `api` — visão geral

```
schema api
 ├─ route                    (RouteDefinition — global, sem RLS)
 ├─ version                  (ApiVersion — global, sem RLS)
 ├─ error_code                (ErrorCodeRegistryEntry — global, sem RLS)
 ├─ event_schema               (EventSchemaRegistryEntry — global, sem RLS)
 ├─ resource_type              (ResourceTypeRegistryEntry — global, sem RLS)
 ├─ rate_limit_policy           (RateLimitPolicy — por tenant, RLS)
 └─ idempotency_record          (IdempotencyRecord — por tenant, RLS)
```

## 2. DDL

```sql
CREATE SCHEMA IF NOT EXISTS api;

-- Registros globais de plataforma (sem tenant_id, sem RLS) --------------

CREATE TABLE api.route (
  id                    CHAR(26)     PRIMARY KEY,             -- ULID
  module_owner          TEXT         NOT NULL,
  path_pattern          TEXT         NOT NULL,
  methods               TEXT[],                                -- NULL = gRPC-only
  protocol              TEXT         NOT NULL
                         CHECK (protocol IN ('rest','grpc','sse')),
  upstream_cluster      TEXT         NOT NULL,
  api_major             INT          NOT NULL,
  auth_required         BOOLEAN      NOT NULL DEFAULT true,
  required_roles        TEXT[],
  rate_limit_policy_id  UUID,
  schema_ref            TEXT,
  status                TEXT         NOT NULL DEFAULT 'draft'
                         CHECK (status IN ('draft','active','disabled')),
  version               BIGINT       NOT NULL DEFAULT 0,       -- OCC
  created_at            TIMESTAMPTZ  NOT NULL DEFAULT now(),
  updated_at            TIMESTAMPTZ  NOT NULL DEFAULT now(),
  CONSTRAINT fk_route_version FOREIGN KEY (module_owner, api_major)
    REFERENCES api.version (module_owner, major)
);
CREATE UNIQUE INDEX ux_route_path ON api.route (path_pattern, methods, api_major);
CREATE INDEX ix_route_owner_status ON api.route (module_owner, status);
CREATE INDEX ix_route_major ON api.route (api_major);

CREATE TABLE api.version (
  module_owner            TEXT         NOT NULL,
  major                    INT          NOT NULL CHECK (major >= 1),
  state                    TEXT         NOT NULL
                            CHECK (state IN ('Draft','Published','Deprecated','Retired')),
  contract_ref             TEXT         NOT NULL,
  deprecation_window_days  INT          NOT NULL DEFAULT 180,
  published_at             TIMESTAMPTZ,
  deprecated_at            TIMESTAMPTZ,
  retired_at               TIMESTAMPTZ,
  version                  BIGINT       NOT NULL DEFAULT 0,   -- OCC
  PRIMARY KEY (module_owner, major)
);
CREATE INDEX ix_version_owner_state ON api.version (module_owner, state);

CREATE TABLE api.error_code (
  code            TEXT         PRIMARY KEY,       -- AIOS-<DOMINIO>-<NNNN>
  domain          TEXT         NOT NULL,
  http_status     SMALLINT     NOT NULL,
  retriable       BOOLEAN      NOT NULL,
  title           TEXT         NOT NULL,
  owner_module    TEXT         NOT NULL,
  registered_at   TIMESTAMPTZ  NOT NULL DEFAULT now()
);
CREATE INDEX ix_error_code_domain ON api.error_code (domain);

CREATE TABLE api.event_schema (
  dataschema      TEXT         PRIMARY KEY,       -- aios://schemas/<type>/<version>
  event_type      TEXT         NOT NULL,           -- aios.<dominio>.<entidade>.<acao>
  json_schema     JSONB        NOT NULL,
  owner_module    TEXT         NOT NULL,
  status          TEXT         NOT NULL DEFAULT 'draft'
                  CHECK (status IN ('draft','active','deprecated')),
  registered_at   TIMESTAMPTZ  NOT NULL DEFAULT now()
);
CREATE INDEX ix_event_schema_type ON api.event_schema (event_type);

CREATE TABLE api.resource_type (
  type            TEXT         PRIMARY KEY,       -- <tipo> do URN (RFC-0001 §5.1)
  description     TEXT         NOT NULL,
  owner_module    TEXT         NOT NULL,
  introduced_in   TEXT,                             -- ADR/RFC de introdução
  registered_at   TIMESTAMPTZ  NOT NULL DEFAULT now()
);

-- Estado de borda por tenant (com tenant_id + RLS) -----------------------

CREATE TABLE api.rate_limit_policy (
  id            UUID          PRIMARY KEY DEFAULT gen_random_uuid(),  -- nativo PG13+
  tenant_id     TEXT          NOT NULL,
  scope         TEXT          NOT NULL CHECK (scope IN ('tenant','route')),
  route_id      CHAR(26)      REFERENCES api.route (id),
  rps           INT           NOT NULL,
  burst         INT           NOT NULL,
  window        INTERVAL      NOT NULL DEFAULT INTERVAL '1 second',
  enforcement   TEXT          NOT NULL CHECK (enforcement IN ('hard','soft')),
  version       BIGINT        NOT NULL DEFAULT 0    -- OCC
);
CREATE INDEX ix_rlp_tenant_scope ON api.rate_limit_policy (tenant_id, scope);
ALTER TABLE api.rate_limit_policy ENABLE ROW LEVEL SECURITY;
CREATE POLICY rlp_tenant_isolation ON api.rate_limit_policy
  USING (tenant_id = current_setting('aios.tenant_id', true));

CREATE TABLE api.idempotency_record (
  idempotency_key   TEXT          NOT NULL,
  tenant_id         TEXT          NOT NULL,
  route_id          CHAR(26)      NOT NULL REFERENCES api.route (id),
  request_hash      TEXT          NOT NULL,
  response_snapshot JSONB         NOT NULL,
  status_code       INT           NOT NULL,
  expires_at        TIMESTAMPTZ   NOT NULL,
  PRIMARY KEY (tenant_id, idempotency_key),
  CONSTRAINT ck_idem_expiry CHECK (expires_at >= now())
);
CREATE INDEX ix_idem_expiry ON api.idempotency_record (expires_at);
ALTER TABLE api.idempotency_record ENABLE ROW LEVEL SECURITY;
CREATE POLICY idem_tenant_isolation ON api.idempotency_record
  USING (tenant_id = current_setting('aios.tenant_id', true));

-- Outbox transacional (registros de plataforma) ---------------------------
CREATE TABLE api.outbox (
  id              BIGSERIAL     PRIMARY KEY,
  event_id        CHAR(26)      NOT NULL UNIQUE,     -- ULID do evento (RFC-0001 §5.2)
  subject         TEXT          NOT NULL,
  payload         JSONB         NOT NULL,
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
  published_at    TIMESTAMPTZ
);
CREATE INDEX ix_outbox_unpublished ON api.outbox (created_at) WHERE published_at IS NULL;
```

## 3. Índices e racional de acesso

| Tabela | Índice | Racional |
|--------|--------|----------|
| `api.route` | `UNIQUE(path_pattern, methods, api_major)` | Impede duplicidade de rota (base do erro `AIOS-API-0015`, UC-009). |
| `api.route` | `btree(module_owner, status)` | Consulta administrativa `GET /v1/api/routes?module_owner=...&status=active`. |
| `api.version` | `btree(module_owner, state)` | `VersionNegotiator` consulta o estado corrente por módulo/major com alta frequência (via *snapshot* em memória, recarregado por evento). |
| `api.idempotency_record` | `btree(expires_at)` | Suporte ao job de expurgo periódico (retenção mínima 24h, RFC-0001 §5.5). |
| `api.outbox` | índice parcial `WHERE published_at IS NULL` | Relay do padrão Outbox varre apenas eventos pendentes, mantendo o *scan* pequeno mesmo com histórico grande. |

## 4. Particionamento

- `api.idempotency_record`: candidata a **particionamento por `expires_at`
  (mensal)** quando o volume justificar (o expurgo por partição é mais
  barato que `DELETE` em massa). Não é obrigatório na v1.0 dado o TTL curto
  (≥ 24h, default `api.idempotency.ttl_hours=24`).
- `api.outbox`: candidata a particionamento por tempo se o volume de
  registros de contrato crescer significativamente; volume esperado é baixo
  (mutações administrativas), portanto não particionada na v1.0.
- `api.route`, `api.version`, `api.error_code`, `api.event_schema`,
  `api.resource_type`: tabelas pequenas (dezenas a centenas de linhas);
  **não particionadas**.

## 5. Row-Level Security (RLS)

RLS é aplicado **exclusivamente** às tabelas de estado por tenant
(`api.rate_limit_policy`, `api.idempotency_record`), via
`current_setting('aios.tenant_id', true)` definido pela conexão de
aplicação a cada requisição (mesmo padrão transversal do AIOS — ver
`../005-Database/README.md`). As tabelas de registro global **NÃO DEVEM**
habilitar RLS: sua leitura é intencionalmente pública (contrato do sistema)
e sua escrita é controlada em nível de aplicação pelo papel
`api-registry-admin` verificado via `RouteAuthorizer`/PDP, não pelo banco.

## 6. Migrações

Migrações seguem o padrão *versioned migrations* do AIOS (ferramenta comum
ao control plane .NET). Toda migração que adiciona uma coluna a uma tabela
de registro global (`route`, `version`, `error_code`, `event_schema`,
`resource_type`) DEVE ser aditiva e retrocompatível (nunca remove ou
renomeia uma coluna consumida por consumidores externos do catálogo, sem
depreciação prévia via `./API.md`).

## 7. Retenção

| Tabela | Política de retenção |
|--------|------------------------|
| `api.idempotency_record` | Expurgo após `expires_at` (mínimo 24h, configurável até 720h via `api.idempotency.ttl_hours`). |
| `api.route` | Retido indefinidamente; rotas obsoletas são marcadas `status=disabled`, nunca deletadas fisicamente (auditabilidade). |
| `api.version` | Retido indefinidamente, inclusive em `Retired` (histórico de contrato — ver `./StateMachine.md` §2.5). |
| `api.error_code` / `api.event_schema` / `api.resource_type` | Retido indefinidamente; um código/schema/tipo nunca é removido, apenas marcado `deprecated` quando aplicável. |
| `api.outbox` | Linhas com `published_at` preenchido são elegíveis a expurgo após janela de auditoria (default 30 dias, configurável em `../024-Observability/`). |

## 8. Relações (ASCII, reproduzido do brief §3.8)

```
   ApiVersion(1)───<RouteDefinition(*)───1 RateLimitPolicy(scope=route)
        │  contract_ref                          │
        ▼                                         └───< IdempotencyRecord(*) (tenant, rota)
   OpenAPI/proto bundle (MinIO)

   ErrorCodeRegistry(*)   EventSchemaRegistry(*)   ResourceTypeRegistry(*)
   (registros globais de plataforma — referenciados, não possuídos, por todos os módulos)

   Tenant(1)───< RateLimitPolicy(scope=tenant)
        └───< IdempotencyRecord(*)
```

## 9. Consistência com o restante do módulo

Os nomes de tabela/coluna deste documento são **idênticos** aos usados em
`./ClassDiagrams.md`, `./API.md` e `./Events.md`. Qualquer renomeação
DEVE ser propagada a todos os 26 documentos e à `./_DESIGN_BRIEF.md`
(defeito se não o for).

*Fim de `Database.md`.*
