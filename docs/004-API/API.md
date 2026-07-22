---
Documento: API
Módulo: 004-API
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 004-API
ADRs relacionados: ADR-0040, ADR-0041, ADR-0042, ADR-0044, ADR-0046, ADR-0049
RFCs relacionados: RFC-0001 (§5.1–§5.8), RFC-0040
Depende de: ./Database.md, ./StateMachine.md, ./Security.md
---

# 004-API — Contratos de API (REST e gRPC)

> Autenticação, envelope de erro, idempotência, correlação e versionamento
> reutilizam **integralmente** `../003-RFC/RFC-0001-Architecture-Baseline.md`
> §5.4–§5.8 — não redefinidos aqui. Pacote gRPC: `aios.api.v1`. Base REST:
> `/v1/api` (endpoints de **administração e registro** do próprio Gateway). A
> **maioria** do tráfego do módulo é *pass-through* transparente: cada rota
> registrada preserva o contrato REST/gRPC do módulo de destino — o Gateway
> aplica apenas os *cross-cutting concerns* desta seção e não redefine o
> contrato de negócio de cada módulo (ver `API.md` de cada módulo upstream).

## 1. Superfície de administração/registro — `aios.api.v1.RegistryService`

| Operação | REST | gRPC (rpc) | Idempotente | Notas |
|----------|------|-----------|-------------|-------|
| Listar rotas | `GET /v1/api/routes` | `ListRoutes` | leitura | Filtra por `module_owner`, `status`, `api_major`; paginado (§4). |
| Registrar/atualizar rota | `POST /v1/api/routes` | `RegisterRoute` | sim (`Idempotency-Key`) | Papel `api-registry-admin`. |
| Desativar rota | `POST /v1/api/routes/{id}:disable` | `DisableRoute` | sim | → `status=disabled`. |
| Publicar contrato OpenAPI/proto agregado | `GET /v1/api/openapi.json` / `GET /v1/api/proto` | `GetAggregatedContract` | leitura | Servido por `OpenApiAggregator`. |
| Listar versões | `GET /v1/api/versions` | `ListVersions` | leitura | Estado da FSM (`./StateMachine.md`) por módulo. |
| Publicar versão | `POST /v1/api/versions/{module}/{major}:publish` | `PublishVersion` | sim | T-02. |
| Depreciar versão | `POST /v1/api/versions/{module}/{major}:deprecate` | `DeprecateVersion` | sim | T-04 (geralmente automática ao publicar `M+1`). |
| Retirar versão | `POST /v1/api/versions/{module}/{major}:retire` | `RetireVersion` | sim | T-05. |
| Consultar catálogo de erros | `GET /v1/api/errors` | `ListErrorCodes` | leitura | RFC-0001 §8; acesso público. |
| Registrar código de erro | `POST /v1/api/errors` | `RegisterErrorCode` | sim | Governado (ADR/RFC de módulo). |
| Consultar schemas de evento | `GET /v1/api/events/schemas` | `ListEventSchemas` | leitura | RFC-0001 §8; acesso público. |
| Registrar schema de evento | `POST /v1/api/events/schemas` | `RegisterEventSchema` | sim | Governado. |
| Consultar tipos de recurso (URN) | `GET /v1/api/resource-types` | `ListResourceTypes` | leitura | RFC-0001 §5.1/§8; acesso público. |
| Saúde do Gateway | `GET /v1/healthz` / `GET /v1/readyz` | `HealthCheck` | leitura | Consumido pela orquestração (`../028-Deployment/`). |

### 1.1 Descritor proto (esboço)

```protobuf
syntax = "proto3";
package aios.api.v1;

service RegistryService {
  rpc ListRoutes(ListRoutesRequest) returns (ListRoutesResponse);
  rpc RegisterRoute(RegisterRouteRequest) returns (RouteDefinition);
  rpc DisableRoute(DisableRouteRequest) returns (RouteDefinition);
  rpc GetAggregatedContract(GetAggregatedContractRequest) returns (AggregatedContract);
  rpc ListVersions(ListVersionsRequest) returns (ListVersionsResponse);
  rpc PublishVersion(PublishVersionRequest) returns (ApiVersion);
  rpc DeprecateVersion(DeprecateVersionRequest) returns (ApiVersion);
  rpc RetireVersion(RetireVersionRequest) returns (ApiVersion);
  rpc ListErrorCodes(ListErrorCodesRequest) returns (ListErrorCodesResponse);
  rpc RegisterErrorCode(RegisterErrorCodeRequest) returns (ErrorCodeRegistryEntry);
  rpc ListEventSchemas(ListEventSchemasRequest) returns (ListEventSchemasResponse);
  rpc RegisterEventSchema(RegisterEventSchemaRequest) returns (EventSchemaRegistryEntry);
  rpc ListResourceTypes(ListResourceTypesRequest) returns (ListResourceTypesResponse);
  rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse);
}

message RouteDefinition {
  string id = 1;
  string module_owner = 2;
  string path_pattern = 3;
  repeated string methods = 4;
  string protocol = 5;          // rest | grpc | sse
  string upstream_cluster = 6;
  int32 api_major = 7;
  bool auth_required = 8;
  repeated string required_roles = 9;
  string rate_limit_policy_id = 10;
  string schema_ref = 11;
  string status = 12;           // draft | active | disabled
  int64 version = 13;
}
```

## 2. Autenticação

Toda rota `auth_required=true` **DEVE** apresentar
`Authorization: Bearer <jwt>` válido conforme JWKS de `021-Security` (ver
`./Security.md` §"AuthN"). A superfície de administração/registro exige,
adicionalmente, o papel `required_roles` que contém `api-registry-admin`
para operações de escrita (`RegisterRoute`, `DisableRoute`,
`PublishVersion`, `DeprecateVersion`, `RetireVersion`, `RegisterErrorCode`,
`RegisterEventSchema`). Consultas de leitura do catálogo de contrato
(`ListErrorCodes`, `ListEventSchemas`, `ListResourceTypes`,
`GetAggregatedContract`) **NÃO exigem** autenticação — são o contrato
público do sistema (RFC-0001 §8).

## 3. Versionamento

Ver `./StateMachine.md` para a FSM completa de `ApiVersion`. Resumo
normativo (RFC-0001 §5.7, especializado por este módulo):

- Toda rota REST **DEVE** ser acessível por caminho `/v{major}/...` e por
  `X-AIOS-Api-Version` (o Gateway usa o caminho como fonte primária e o
  header como confirmação/override quando aplicável).
- O Gateway **DEVE** sustentar `api.version.coexistence_majors_min` (default
  2) majors coexistentes por módulo.
- Requisição para major `Deprecated` **DEVE** incluir `Deprecation: true` e
  `Sunset: <data>`.
- Requisição para major `Retired` **DEVE** retornar `410 Gone`
  (`AIOS-API-0011`).

## 4. Paginação

Todas as operações `List*` **DEVEM** suportar paginação por cursor:

```
GET /v1/api/routes?page_size=50&page_token=<opaco>
```

Resposta:
```json
{
  "items": [ ... ],
  "nextPageToken": "eyJvZmZzZXQiOjUwfQ==",
  "totalSizeEstimate": 214
}
```

`page_size` **DEVE** ter default `50` e máximo `500`. `page_token` **DEVE**
ser opaco ao cliente (não é um número de página literal), permitindo evoluir
a implementação de paginação sem quebrar o contrato.

## 5. Idempotência

Toda operação de mutação (`POST`) desta seção **DEVE** aceitar
`Idempotency-Key` conforme RFC-0001 §5.5, com retenção mínima de 24h
(`api.idempotency.ttl_hours`, default 24, configurável até 720h). Reuso da
mesma chave com corpo divergente **DEVE** retornar `AIOS-API-0012`.

## 6. Catálogo de códigos de erro (domínio `API`, faixa reservada `0001`–`0099`)

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-API-0001` | 404 | não | Nenhuma `RouteDefinition` ativa corresponde ao caminho/método/versão. |
| `AIOS-API-0002` | 401 | não | Token OAuth2/OIDC ausente, inválido ou expirado. |
| `AIOS-API-0003` | 403 | não | Autorização de rota negada pelo PDP (*default deny*). |
| `AIOS-API-0004` | 429 | sim | Rate limit de borda excedido (tenant ou rota). |
| `AIOS-API-0005` | 503 | sim | Upstream indisponível / circuit breaker aberto. |
| `AIOS-API-0006` | 504 | sim | Timeout aguardando resposta do upstream. |
| `AIOS-API-0007` | 400 | não | Payload não conforme ao schema registrado (OpenAPI/proto). |
| `AIOS-API-0008` | 415 | não | `Content-Type`/codec não suportado. |
| `AIOS-API-0009` | 413 | não | Corpo da requisição excede `api.gateway.max_body_size_bytes`. |
| `AIOS-API-0010` | 400 | não | Versão de API solicitada inválida ou não registrada. |
| `AIOS-API-0011` | 410 | não | Versão de API em estado `Retired` (`./StateMachine.md`). |
| `AIOS-API-0012` | 409 | não | `Idempotency-Key` reusada com payload divergente (`request_hash` diferente). |
| `AIOS-API-0013` | 400 | não | Cabeçalho de correlação obrigatório ausente (`traceparent`/`X-AIOS-Tenant`). |
| `AIOS-API-0014` | 502 | sim | Upstream retornou erro fora do envelope RFC-0001 §5.4 (não conforme). |
| `AIOS-API-0015` | 409 | não | Conflito ao registrar rota (duplicidade de `path_pattern`+`methods`+`api_major`). |
| `AIOS-API-0016` | 403 | não | `X-AIOS-Tenant` divergente do tenant no token autenticado. |

Este catálogo é o dono operacional do domínio `API` no registro central de
RFC-0001 §8 (também consultável via `GET /v1/api/errors`).

## 7. Exemplos de request/response

### 7.1 Requisição *pass-through* bem-sucedida

```
POST /v1/kernel/agents HTTP/1.1
Host: api.aios.example
Authorization: Bearer eyJhbGciOi...
X-AIOS-Tenant: acme
Idempotency-Key: 01J9Z8Q8XYZABC000000000001
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Content-Type: application/json

{ "agentType": "researcher", "priority": "normal" }
```

```
HTTP/1.1 201 Created
Content-Type: application/json
X-AIOS-Api-Version: 1

{ "id": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6", "state": "Pending" }
```

### 7.2 Erro de rate limit

```
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
Retry-After: 2

{ "type": "https://docs.aios/errors/api-rate-limited",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "code": "AIOS-API-0004",
  "detail": "Tenant 'acme' exceeded 100 rps on route 'POST /v1/kernel/agents'.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-21T12:00:00.000Z",
  "retriable": true,
  "retryAfterMs": 2000 }
```

### 7.3 Registrar rota (administração)

```
POST /v1/api/routes HTTP/1.1
Authorization: Bearer <token com role api-registry-admin>
Idempotency-Key: 01J9Z8Q9REGR0VTE0000000001
Content-Type: application/json

{
  "moduleOwner": "006-Kernel",
  "pathPattern": "/v1/kernel/agents/{urn}",
  "methods": ["GET", "DELETE"],
  "protocol": "rest",
  "upstreamCluster": "kernel-cluster",
  "apiMajor": 1,
  "authRequired": true,
  "requiredRoles": ["agent-operator"],
  "schemaRef": "kernel.v1#/paths/~1agents~1{urn}"
}
```

```
HTTP/1.1 201 Created
Content-Type: application/json

{ "id": "01J9Z8QAR0VTE0000000000001", "status": "active", "version": 0 }
```

### 7.4 Publicar versão (governança de FSM)

```
POST /v1/api/versions/006-Kernel/2:publish HTTP/1.1
Authorization: Bearer <token com role api-registry-admin>
Idempotency-Key: 01J9Z8QBP0BVER000000000001
```

```
HTTP/1.1 200 OK
Content-Type: application/json

{ "moduleOwner": "006-Kernel", "major": 2, "state": "Published",
  "publishedAt": "2026-07-21T12:05:00.000Z", "version": 1 }
```

## 8. Consistência com o restante do módulo

Nomes de operação, campos e códigos de erro deste documento são **idênticos**
aos usados em `./Database.md`, `./Events.md` e `./Examples.md`.

*Fim de `API.md`.*
