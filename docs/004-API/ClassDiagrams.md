---
Documento: ClassDiagrams
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0041, ADR-0043, ADR-0044
RFCs relacionados: RFC-0001
Depende de: ./Architecture.md, ./Database.md
---

# 004-API — Diagramas de Classe / Estrutura

> Estruturas internas (ASCII), interfaces públicas, contratos e invariantes,
> derivados de `./_DESIGN_BRIEF.md` §2 e §3. Nomes de componente em
> `PascalCase`, conforme convenção global.

## 1. Estruturas de domínio (entidades canônicas)

```
┌────────────────────────┐        ┌────────────────────────┐
│    RouteDefinition       │        │       ApiVersion         │
├────────────────────────┤        ├────────────────────────┤
│ id: ULID (PK)             │  N   │ module_owner: text (PK1) │
│ module_owner: text        │─────▶│ major: int (PK2)          │
│ path_pattern: text        │  1   │ state: ApiVersionState    │
│ methods: text[]?          │      │ contract_ref: text         │
│ protocol: rest|grpc|sse   │      │ deprecation_window_days   │
│ upstream_cluster: text    │      │ published_at: ts?          │
│ api_major: int (FK)       │      │ deprecated_at: ts?         │
│ auth_required: bool       │      │ retired_at: ts?            │
│ required_roles: text[]?   │      │ version: bigint (OCC)      │
│ rate_limit_policy_id: uuid?│     └────────────────────────┘
│ schema_ref: text?         │
│ status: draft|active|     │
│         disabled          │
│ version: bigint (OCC)     │
└──────────┬────────────────┘
           │ 1
           │
           │ 0..1
┌──────────▼──────────────┐        ┌────────────────────────┐
│    RateLimitPolicy        │        │   IdempotencyRecord       │
├────────────────────────┤        ├────────────────────────┤
│ id: uuid (PK)              │        │ idempotency_key: text (PK1)│
│ tenant_id: text (RLS)      │        │ tenant_id: text (PK2, RLS) │
│ scope: tenant|route        │        │ route_id: ULID (FK)         │
│ route_id: ULID? (FK)       │        │ request_hash: text           │
│ rps: int                   │        │ response_snapshot: jsonb      │
│ burst: int                 │        │ status_code: int               │
│ window: interval           │        │ expires_at: ts (≥24h)         │
│ enforcement: hard|soft     │        └────────────────────────┘
│ version: bigint (OCC)      │
└────────────────────────┘

┌────────────────────────┐  ┌────────────────────────┐  ┌────────────────────────┐
│ ErrorCodeRegistryEntry    │  │EventSchemaRegistryEntry  │  │ResourceTypeRegistryEntry│
├────────────────────────┤  ├────────────────────────┤  ├────────────────────────┤
│ code: text (PK)            │  │ dataschema: text (PK)     │  │ type: text (PK)           │
│ domain: text                │  │ event_type: text          │  │ description: text          │
│ http_status: smallint       │  │ json_schema: jsonb        │  │ owner_module: text         │
│ retriable: bool             │  │ owner_module: text        │  │ introduced_in: text?       │
│ title: text                 │  │ status: draft|active|     │  │ registered_at: ts          │
│ owner_module: text          │  │         deprecated        │  └────────────────────────┘
│ registered_at: ts           │  │ registered_at: ts         │
└────────────────────────┘  └────────────────────────┘
```

**Invariantes de domínio** (não redefinem RFC-0001, apenas especializam):

- `RouteDefinition.path_pattern + methods + api_major` DEVE ser único
  (`UNIQUE` — ver `./Database.md`).
- `RouteDefinition.api_major` DEVE referenciar uma `ApiVersion` existente
  (FK), mas PODE estar em qualquer estado da FSM — a roteabilidade efetiva é
  decidida em tempo de requisição por `VersionNegotiator` lendo `state`.
- `IdempotencyRecord.expires_at` DEVE ser ≥ 24h após criação (RFC-0001 §5.5).
- `RateLimitPolicy`/`IdempotencyRecord` carregam `tenant_id` + RLS; os cinco
  registros de contrato (`RouteDefinition`, `ApiVersion`, `ErrorCodeRegistryEntry`,
  `EventSchemaRegistryEntry`, `ResourceTypeRegistryEntry`) **NÃO** carregam
  `tenant_id` — são metadados de plataforma (ADR-0043).

## 2. Componentes de runtime e suas interfaces públicas (ASCII, estilo UML simplificado)

```
┌───────────────────────────────┐
│        «interface»             │
│      IRouteResolver             │
├───────────────────────────────┤
│ + Resolve(path, method, major) │
│     : RouteDefinition?          │
└───────────────┬────────────────┘
                 │ implementa
┌───────────────▼────────────────┐        depende de       ┌───────────────────┐
│         EdgeRouter                │─────────────────────▶│ CircuitBreakerMgr   │
├───────────────────────────────┤                          ├───────────────────┤
│ - routeSnapshot: RouteDefinition[]│                        │ + IsOpen(cluster)   │
│ + Route(request): Response         │                        │ + RecordResult(...) │
└───────────────┬────────────────┘                          └───────────────────┘
                 │ usa
┌───────────────▼────────────────┐
│        «interface»               │
│     IAuthenticator                │
├───────────────────────────────┤
│ + Authenticate(token): Claims?    │
└───────────────┬────────────────┘
                 │ implementa
┌───────────────▼────────────────┐        consulta         ┌───────────────────┐
│        AuthNFilter                │────────────────────▶│  JwksCache          │
└───────────────────────────────┘                          │ + GetKey(kid)       │
                                                              └───────────────────┘

┌───────────────────────────────┐
│        «interface»               │
│      IPolicyDecisionPoint        │
├───────────────────────────────┤
│ + Decide(subject, action, res)   │
│     : Decision (allow|deny)       │
└───────────────┬────────────────┘
                 │ implementa (client-side)
┌───────────────▼────────────────┐        chama gRPC        ┌───────────────────┐
│         PolicyClient              │─────────────────────▶│ 022-Policy (PDP)    │
├───────────────────────────────┤                          └───────────────────┘
│ - decisionCache: Cache<Decision>  │
│ - circuitBreaker: CircuitBreaker  │
│ - failMode: closed|open           │
│ + Decide(...): Decision            │
└───────────────────────────────┘

┌───────────────────────────────┐
│        «interface»               │
│      IIdempotencyStore            │
├───────────────────────────────┤
│ + TryReserve(key, hash): Result   │
│ + GetCached(key): Response?       │
└───────────────┬────────────────┘
                 │ implementa
┌───────────────▼────────────────┐
│      IdempotencyRelay             │
├───────────────────────────────┤
│ - hotCache: Redis                  │
│ - durable: PostgreSQL               │
└───────────────────────────────┘
```

## 3. Relações de composição/agregação/dependência

| Relação | Tipo | Descrição |
|---------|------|-----------|
| `EdgeRouter` ◇── `CircuitBreakerManager` | Agregação | `EdgeRouter` consulta o estado do *breaker* antes de encaminhar; `CircuitBreakerManager` sobrevive independentemente. |
| `RouteAuthorizer` ●── `PolicyClient` | Composição | `PolicyClient` é instanciado e possuído pelo `RouteAuthorizer`; não é compartilhado fora do pipeline de AuthZ. |
| `ContractRegistry` ◇── `OpenApiAggregator` | Agregação | `OpenApiAggregator` lê do `ContractRegistry`, mas é um componente independente (pode ser reiniciado sem afetar o registro). |
| `IdempotencyRelay` ──▶ `RateLimiter` | Dependência | Ordem do pipeline: rate limit é avaliado antes da checagem de idempotência (§1 de `./StateMachine.md`). |
| `RouteDefinition` ──▶ `ApiVersion` (FK) | Dependência de dados | Toda rota referencia exatamente uma major; a FSM da major é lida, não modificada, pelo `EdgeRouter`. |
| `RateLimitPolicy` (`scope=route`) ──▶ `RouteDefinition` | Dependência de dados | Política de rota é opcional; ausência usa o default de tenant/global. |

## 4. Contratos de interface (assinatura conceitual)

```
interface IRouteResolver {
  RouteDefinition? Resolve(string path, string method, int apiMajor);
}

interface IPolicyDecisionPoint {
  Decision Decide(Subject subject, string action, string resource);
}

interface IIdempotencyStore {
  ReservationResult TryReserve(string idempotencyKey, string tenantId, string requestHash);
  CachedResponse? GetCached(string idempotencyKey, string tenantId);
}

interface IErrorTranslator {
  ErrorEnvelope Translate(Exception|UpstreamError source, string traceId);
}
```

Estas interfaces são conceituais (não um contrato de código executável — este
repositório é de documentação, não de implementação) e servem para fixar as
fronteiras testáveis do módulo, referenciadas em `./Testing.md`.

*Fim de `ClassDiagrams.md`.*
