---
Documento: API
Módulo: 010-Memory
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 010-Memory
ADRs relacionados: ADR-0100, ADR-0101, ADR-0102, ADR-0103, ADR-0104, ADR-0105, ADR-0106, ADR-0109
RFCs relacionados: RFC-0001 (baseline), RFC-0100 (a propor)
Depende de: 003-RFC (RFC-0001), 004-API, 011-Context, 017-Model-Router, 021-Security, 022-Policy, 026-Cost-Optimizer, 040-Glossary
---

# 010-Memory — API

> Contratos REST (OpenAPI) e gRPC (`aios.memory.v1`) do `MemoryService`.
> Este documento **deriva** da §5 do `_DESIGN_BRIEF.md` e **NÃO PODE contradizê-lo**.
> Os contratos centrais (URN, envelope de erro RFC 7807, idempotência, correlação,
> versionamento) são **reutilizados** de [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md)
> — aqui apenas os aplicamos, sem redefini-los. Palavras normativas conforme RFC 2119/8174.

## 1. Visão geral

O `MemoryService` expõe as quatro operações cognitivas canônicas — `remember`,
`recall`, `consolidate`, `forget` — além de operações auxiliares de leitura, expurgo
(RTBF), estatísticas, gestão de camadas e ciclo de vida de jobs de consolidação. A
superfície é oferecida em duas fachadas equivalentes:

- **REST/JSON** sob a base `/v1/memory`, para clientes externos e o WebConsole.
- **gRPC** no pacote `aios.memory.v1`, serviço `MemoryService`, para o plano de
  controle e comunicação inter-serviços de baixa latência.

Ambas as fachadas atravessam o `MemoryApiFacade`, que valida DTOs, aplica
versionamento e traduz cada chamada em comandos internos. Toda operação passa antes
pelo `MemoryPep` (Policy Enforcement Point), que autoriza junto ao PDP
([022-Policy](../022-Policy/Security.md)) sob *default deny*, impõe o `tenant` do
contexto autenticado e valida a `Idempotency-Key`.

```
        Cliente / WebConsole / Runtime
                   │
             ┌─────▼──────┐  X-AIOS-Api-Version, traceparent, X-AIOS-Tenant
             │   YARP     │  (Gateway 028; TLS terminado; token OAuth2/OIDC → claims)
             └─────┬──────┘
        REST /v1/memory       gRPC aios.memory.v1
                   │                   │
             ┌─────▼───────────────────▼─────┐
             │        MemoryApiFacade         │  valida DTO + versão
             └─────┬──────────────────────────┘
                   │  → MemoryPep (PDP 022, tenant guard, Idempotency-Key)
                   ▼  comando validado → LayerRouter / RecallEngine / ...
```

## 2. Versionamento e autenticação

### 2.1 Versionamento (RFC-0001 §5.7)

- REST DEVE ser versionada por caminho (`/v1/...`) **e** pelo header
  `X-AIOS-Api-Version`. O caminho fixa a major; o header PODE selecionar uma minor
  compatível.
- gRPC DEVE usar pacote versionado (`aios.memory.v1`). Uma mudança incompatível
  incrementa a major do pacote (`v2`) e coexiste com a anterior por **≥ 2 majors**.
- Evolução de payload DEVE ser **aditiva**; clientes DEVEM tolerar campos
  desconhecidos.

### 2.2 Autenticação e cabeçalhos obrigatórios (RFC-0001 §5.6)

O token OAuth2/OIDC é validado no Gateway ([021-Security](../021-Security/Security.md));
as claims assinadas (incluindo `tenant`) são propagadas ao serviço. Entre serviços
internos usa-se **mTLS**.

| Cabeçalho | Obrigatório em | Descrição |
|-----------|----------------|-----------|
| `Authorization: Bearer <jwt>` | todas as chamadas externas | Token OIDC validado no Gateway. |
| `X-AIOS-Tenant` | todas (multi-tenant) | Tenant do chamador; DEVE coincidir com a claim. Divergência → `AIOS-MEM-0005`. |
| `traceparent` / `tracestate` | todas | W3C Trace Context (OTel). |
| `Idempotency-Key` | **toda mutação** (`POST`/`DELETE`) | ULID/UUID do cliente (RFC-0001 §5.5). |
| `X-AIOS-Agent` | quando aplicável | URN do agente que origina a operação. |
| `X-AIOS-Api-Version` | recomendado | Versão de API solicitada. |

## 3. Recursos e endpoints (mapa REST ↔ gRPC)

Derivado da §5.1 do brief. As quatro operações canônicas estão em **negrito**.

| Operação | REST | gRPC (`aios.memory.v1.MemoryService`) | Mutação | Idempotente |
|----------|------|----------------------------------------|---------|-------------|
| **remember** | `POST /v1/memory/items` | `Remember(RememberRequest)` | sim | por `Idempotency-Key` |
| **recall** | `POST /v1/memory/recall` | `Recall(RecallRequest)` | não | leitura (naturalmente) |
| **consolidate** | `POST /v1/memory/consolidate` | `Consolidate(ConsolidateRequest)` | sim | por `Idempotency-Key` |
| **forget** | `POST /v1/memory/forget` | `Forget(ForgetRequest)` | sim | por `Idempotency-Key` |
| get item | `GET /v1/memory/items/{id}` | `GetItem(GetItemRequest)` | não | — |
| purge item (RTBF) | `DELETE /v1/memory/items/{id}` | `PurgeItem(PurgeRequest)` | sim | por `Idempotency-Key` |
| stats/cotas | `GET /v1/memory/stats` | `GetStats(StatsRequest)` | não | — |
| listar camadas | `GET /v1/memory/layers` | `ListLayers(ListLayersRequest)` | não | — |
| status de job | `GET /v1/memory/jobs/{id}` | `GetJob(GetJobRequest)` | não | — |
| rollback | `POST /v1/memory/consolidate/{jobId}/rollback` | `RollbackConsolidation(RollbackRequest)` | sim | por `Idempotency-Key` |

`{id}` é o ULID do `MemoryItem` (URN completo `urn:aios:<tenant>:memory:<id>`, RFC-0001 §5.1).

## 4. Idempotência e paginação

### 4.1 Idempotência (RFC-0001 §5.5)

Toda mutação DEVE enviar `Idempotency-Key`. O `IdempotencyStore` persiste o resultado
por chave por **≥ 24 h**; uma repetição com a **mesma** chave e **mesmo** payload
retorna o resultado memoizado (sem reexecutar). A mesma chave com payload divergente
DEVE retornar `AIOS-MEM-0003` (409).

### 4.2 Paginação por cursor

`recall` e `stats` retornam listas potencialmente grandes e DEVEM paginar por
**cursor opaco** (nunca por offset numérico). O cliente repassa `page.cursor` da
resposta anterior; `page.size` (default 10, máx. 200 — alinhado a
`memory.recall.default_k`) limita a página.

```json
"page": { "size": 20, "cursor": "eyJvIjoxMjB9" }
```

A resposta traz `page.next_cursor` (ausente/nulo no fim) e `page.total_estimate`.

## 5. Códigos de erro (`AIOS-MEM-NNNN`)

Erros seguem o envelope RFC 7807 de [RFC-0001 §5.4](../003-RFC/RFC-0001-Architecture-Baseline.md).
gRPC mapeia `status` → `google.rpc.Status` + `ErrorInfo` com o mesmo `code`. Catálogo
canônico (brief §5.2); registro central em [004-API/Errors.md](../004-API/Errors.md).
Faixa reservada: `AIOS-MEM-0001`..`AIOS-MEM-0099`.

| Código | HTTP | gRPC (canonical) | Retriable | Significado |
|--------|------|------------------|-----------|-------------|
| `AIOS-MEM-0001` | 400 | INVALID_ARGUMENT | não | Requisição inválida (schema/DTO). |
| `AIOS-MEM-0002` | 404 | NOT_FOUND | não | Item de memória inexistente. |
| `AIOS-MEM-0003` | 409 | ABORTED | não | Conflito de idempotência (mesma key, payload divergente). |
| `AIOS-MEM-0004` | 403 | PERMISSION_DENIED | não | Autorização negada pelo PDP (*default deny*). |
| `AIOS-MEM-0005` | 403 | PERMISSION_DENIED | não | Tenant divergente do contexto autenticado. |
| `AIOS-MEM-0010` | 429 | RESOURCE_EXHAUSTED | sim | Cota de camada excedida (itens/bytes/vetores). |
| `AIOS-MEM-0011` | 429 | RESOURCE_EXHAUSTED | sim | Backpressure ativo (fila de escrita saturada). |
| `AIOS-MEM-0020` | 422 | FAILED_PRECONDITION | não | Camada inválida ou incompatível com o `kind`. |
| `AIOS-MEM-0021` | 422 | FAILED_PRECONDITION | não | Dimensão de embedding incompatível com o índice. |
| `AIOS-MEM-0030` | 502 | UNAVAILABLE | sim | Falha ao obter embedding do Model Router (017). |
| `AIOS-MEM-0031` | 503 | UNAVAILABLE | sim | Backend de camada indisponível (Redis/PG/AGE/MinIO). |
| `AIOS-MEM-0040` | 409 | ABORTED | não | Consolidação em conflito (versão concorrente). |
| `AIOS-MEM-0041` | 500 | INTERNAL | sim | Falha de consolidação (rollback aplicado). |
| `AIOS-MEM-0042` | 409 | ABORTED | não | Rollback impossível (versão inexistente/já ativa). |
| `AIOS-MEM-0050` | 423 | FAILED_PRECONDITION | não | Item sob `legal_hold`; esquecimento bloqueado. |
| `AIOS-MEM-0051` | 202 | OK (aceito) | — | Expurgo (RTBF) aceito e agendado. |
| `AIOS-MEM-0060` | 500 | INTERNAL | sim | Erro interno não classificado. |

Exemplo de corpo de erro (RFC 7807):

```json
{
  "type": "https://docs.aios/errors/mem-quota-exceeded",
  "title": "Memory Layer Quota Exceeded",
  "status": 429,
  "code": "AIOS-MEM-0010",
  "detail": "Layer 'semantic' item quota exceeded for tenant 'acme'.",
  "instance": "urn:aios:acme:memory:01J9Z8QATENTATIVA000000000",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:34:56.789Z",
  "retriable": true,
  "retryAfterMs": 1500
}
```

> O campo `detail` NÃO DEVE vazar PII nem conteúdo sensível (RFC-0001 §6; ver
> [Security.md](./Security.md)).

## 6. Especificação OpenAPI (REST)

Fragmento normativo (OpenAPI 3.1). Campos de `MemoryItem` derivam da §3.1 do brief.

```yaml
openapi: 3.1.0
info:
  title: AIOS Memory Service API
  version: "1.0"
servers:
  - url: https://gw.aios.local/v1/memory
security:
  - oidc: []
paths:
  /items:
    post:
      operationId: remember
      summary: remember(item, layer) — cria/atualiza item na camada
      parameters:
        - { $ref: '#/components/parameters/IdempotencyKey' }
        - { $ref: '#/components/parameters/Tenant' }
        - { $ref: '#/components/parameters/Traceparent' }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/RememberRequest' }
      responses:
        "201": { description: Criado, content: { application/json: { schema: { $ref: '#/components/schemas/MemoryItemRef' } } } }
        "200": { description: Idempotente (resultado memoizado) }
        "409": { $ref: '#/components/responses/Problem' }
        "422": { $ref: '#/components/responses/Problem' }
        "429": { $ref: '#/components/responses/Problem' }
  /recall:
    post:
      operationId: recall
      summary: recall(query, filtros) — busca híbrida rankeada
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/RecallRequest' }
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: '#/components/schemas/RecallResponse' } } } }
  /consolidate:
    post:
      operationId: consolidate
      summary: consolidate() — dispara ConsolidationJob (assíncrono)
      parameters: [ { $ref: '#/components/parameters/IdempotencyKey' } ]
      responses:
        "202": { description: Job aceito, content: { application/json: { schema: { $ref: '#/components/schemas/JobRef' } } } }
  /consolidate/{jobId}/rollback:
    post:
      operationId: rollbackConsolidation
      parameters:
        - { name: jobId, in: path, required: true, schema: { type: string } }
        - { $ref: '#/components/parameters/IdempotencyKey' }
      responses:
        "202": { description: Rollback agendado }
        "409": { $ref: '#/components/responses/Problem' }   # AIOS-MEM-0042
  /forget:
    post:
      operationId: forget
      summary: forget(policy) — aplica ttl/decay/lru/quota/rtbf
      parameters: [ { $ref: '#/components/parameters/IdempotencyKey' } ]
      requestBody:
        content: { application/json: { schema: { $ref: '#/components/schemas/ForgetRequest' } } }
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: '#/components/schemas/ForgetResponse' } } } }
        "423": { $ref: '#/components/responses/Problem' }   # AIOS-MEM-0050 legal_hold
  /items/{id}:
    get:
      operationId: getItem
      parameters: [ { name: id, in: path, required: true, schema: { type: string } } ]
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: '#/components/schemas/MemoryItem' } } } }
        "404": { $ref: '#/components/responses/Problem' }
    delete:
      operationId: purgeItem
      summary: Expurgo direcionado (RTBF)
      parameters:
        - { name: id, in: path, required: true, schema: { type: string } }
        - { $ref: '#/components/parameters/IdempotencyKey' }
      responses:
        "202": { description: Expurgo agendado (AIOS-MEM-0051) }
        "423": { $ref: '#/components/responses/Problem' }
  /stats:
    get: { operationId: getStats, responses: { "200": { description: OK } } }
  /layers:
    get: { operationId: listLayers, responses: { "200": { description: OK } } }
  /jobs/{id}:
    get:
      operationId: getJob
      parameters: [ { name: id, in: path, required: true, schema: { type: string } } ]
      responses: { "200": { description: OK }, "404": { $ref: '#/components/responses/Problem' } }
components:
  securitySchemes:
    oidc: { type: openIdConnect, openIdConnectUrl: https://idp.aios.local/.well-known/openid-configuration }
  parameters:
    IdempotencyKey: { name: Idempotency-Key, in: header, required: true, schema: { type: string } }
    Tenant: { name: X-AIOS-Tenant, in: header, required: true, schema: { type: string } }
    Traceparent: { name: traceparent, in: header, required: true, schema: { type: string } }
  responses:
    Problem:
      description: Erro RFC 7807 (AIOS-MEM-NNNN)
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/Problem' } } }
  schemas:
    RememberRequest:
      type: object
      required: [layer, kind, legal_basis, retention_class]
      properties:
        agent_id: { type: string, nullable: true }
        session_id: { type: string, nullable: true }
        layer: { type: string, enum: [working, short_term, long_term, semantic, procedural, episodic, kg] }
        kind: { type: string, enum: [fact, event, skill, observation, reflection, summary, edge] }
        content: { type: object, nullable: true }
        content_ref: { type: string, nullable: true }
        salience: { type: number, minimum: 0, maximum: 1, default: 0.5 }
        legal_basis: { type: string, enum: [consent, contract, legitimate_interest, legal_obligation] }
        retention_class: { type: string, enum: [ephemeral, standard, extended, legal_hold] }
        pii: { type: boolean, default: false }
        tags: { type: array, items: { type: string } }
        metadata: { type: object }
    RecallRequest:
      type: object
      required: [query]
      properties:
        query: { type: string }
        agent_id: { type: string }
        layer: { type: array, items: { type: string } }
        kind: { type: array, items: { type: string } }
        tags: { type: array, items: { type: string } }
        k: { type: integer, minimum: 1, maximum: 200, default: 10 }
        min_score: { type: number, minimum: 0, maximum: 1, default: 0.30 }
        time_range: { type: object, properties: { from: { type: string, format: date-time }, to: { type: string, format: date-time } } }
        include_forgotten: { type: boolean, default: false }
        mode: { type: string, enum: [vector, lexical, graph, hybrid], default: hybrid }
        page: { $ref: '#/components/schemas/PageRequest' }
    RecallResponse:
      type: object
      properties:
        results:
          type: array
          items:
            type: object
            properties:
              urn: { type: string }
              score: { type: number }
              layer: { type: string }
              content: { type: object }
        recall_rate_sample: { type: number }
        page: { $ref: '#/components/schemas/PageResponse' }
    ForgetRequest:
      type: object
      required: [strategy]
      properties:
        scope: { type: string, enum: [tenant, agent, layer] }
        strategy: { type: string, enum: [ttl, decay, lru, quota, rtbf] }
        params: { type: object }
    ForgetResponse:
      type: object
      properties: { affected_count: { type: integer } }
    MemoryItemRef:
      type: object
      properties: { urn: { type: string }, state: { type: string } }
    MemoryItem:
      allOf:
        - $ref: '#/components/schemas/MemoryItemRef'
        - type: object
          properties:
            layer: { type: string }
            kind: { type: string }
            salience: { type: number }
            decay_score: { type: number }
            created_at: { type: string, format: date-time }
    JobRef:
      type: object
      properties: { job_id: { type: string }, state: { type: string } }
    PageRequest:
      type: object
      properties: { size: { type: integer, default: 10, maximum: 200 }, cursor: { type: string } }
    PageResponse:
      type: object
      properties: { next_cursor: { type: string, nullable: true }, total_estimate: { type: integer } }
    Problem:
      type: object
      properties:
        type: { type: string }
        title: { type: string }
        status: { type: integer }
        code: { type: string, pattern: "^AIOS-MEM-[0-9]{4}$" }
        detail: { type: string }
        instance: { type: string }
        traceId: { type: string }
        timestamp: { type: string, format: date-time }
        retriable: { type: boolean }
        retryAfterMs: { type: integer }
```

## 7. Especificação gRPC (proto)

Pacote reservado `aios.memory.v1` (brief §0). Erros propagados via
`google.rpc.Status` + `ErrorInfo{ reason = "AIOS-MEM-NNNN" }`.

```proto
syntax = "proto3";
package aios.memory.v1;

import "google/protobuf/struct.proto";
import "google/protobuf/timestamp.proto";

service MemoryService {
  rpc Remember    (RememberRequest)    returns (MemoryItemRef);
  rpc Recall      (RecallRequest)      returns (RecallResponse);
  rpc Consolidate (ConsolidateRequest) returns (JobRef);
  rpc Forget      (ForgetRequest)      returns (ForgetResponse);
  rpc GetItem     (GetItemRequest)     returns (MemoryItem);
  rpc PurgeItem   (PurgeRequest)       returns (JobRef);
  rpc GetStats    (StatsRequest)       returns (StatsResponse);
  rpc ListLayers  (ListLayersRequest)  returns (ListLayersResponse);
  rpc GetJob      (GetJobRequest)      returns (JobRef);
  rpc RollbackConsolidation (RollbackRequest) returns (JobRef);
}

enum MemoryLayer {
  MEMORY_LAYER_UNSPECIFIED = 0;
  WORKING = 1; SHORT_TERM = 2; LONG_TERM = 3; SEMANTIC = 4;
  PROCEDURAL = 5; EPISODIC = 6; KG = 7;
}
enum MemoryKind { KIND_UNSPECIFIED=0; FACT=1; EVENT=2; SKILL=3; OBSERVATION=4; REFLECTION=5; SUMMARY=6; EDGE=7; }
enum RecallMode { MODE_UNSPECIFIED=0; VECTOR=1; LEXICAL=2; GRAPH=3; HYBRID=4; }

message RememberRequest {
  string idempotency_key = 1;            // obrigatório (mutação)
  string tenant = 2;                     // deve casar com contexto autenticado
  string agent_id = 3;
  string session_id = 4;
  MemoryLayer layer = 5;
  MemoryKind kind = 6;
  google.protobuf.Struct content = 7;
  float salience = 8;
  string legal_basis = 9;                // consent|contract|legitimate_interest|legal_obligation
  string retention_class = 10;           // ephemeral|standard|extended|legal_hold
  bool pii = 11;
  repeated string tags = 12;
  google.protobuf.Struct metadata = 13;
}
message MemoryItemRef { string urn = 1; string state = 2; }

message RecallRequest {
  string tenant = 1; string query = 2; string agent_id = 3;
  repeated MemoryLayer layer = 4; repeated MemoryKind kind = 5;
  repeated string tags = 6; int32 k = 7; float min_score = 8;
  google.protobuf.Timestamp time_from = 9; google.protobuf.Timestamp time_to = 10;
  bool include_forgotten = 11; RecallMode mode = 12; PageRequest page = 13;
}
message RecallHit { string urn = 1; double score = 2; MemoryLayer layer = 3; google.protobuf.Struct content = 4; }
message RecallResponse { repeated RecallHit results = 1; double recall_rate_sample = 2; PageResponse page = 3; }

message ConsolidateRequest { string idempotency_key=1; string tenant=2; string agent_id=3; string from_layer=4; string to_layer=5; string trigger=6; }
message ForgetRequest { string idempotency_key=1; string tenant=2; string scope=3; string strategy=4; google.protobuf.Struct params=5; }
message ForgetResponse { int64 affected_count = 1; }
message GetItemRequest { string tenant=1; string urn=2; }
message PurgeRequest { string idempotency_key=1; string tenant=2; string urn=3; }
message StatsRequest { string tenant=1; string agent_id=2; }
message StatsResponse { double memory_recall_rate=1; google.protobuf.Struct usage_by_layer=2; }
message ListLayersRequest { string tenant=1; }
message ListLayersResponse { google.protobuf.Struct layers=1; }
message GetJobRequest { string tenant=1; string job_id=2; }
message RollbackRequest { string idempotency_key=1; string tenant=2; string job_id=3; }
message JobRef { string job_id=1; string state=2; }
message MemoryItem { string urn=1; string state=2; MemoryLayer layer=3; MemoryKind kind=4; float salience=5; float decay_score=6; google.protobuf.Timestamp created_at=7; }

message PageRequest { int32 size=1; string cursor=2; }
message PageResponse { string next_cursor=1; int64 total_estimate=2; }
```

## 8. Exemplos request/response

### 8.1 `remember` (REST) — feliz

```http
POST /v1/memory/items HTTP/1.1
Host: gw.aios.local
Authorization: Bearer <jwt>
X-AIOS-Tenant: acme
Idempotency-Key: 01J9Z8QBREMEMBER00000000001
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Content-Type: application/json

{
  "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "layer": "semantic",
  "kind": "fact",
  "content": { "text": "O cliente prefere faturas em PDF." },
  "salience": 0.7,
  "legal_basis": "contract",
  "retention_class": "standard",
  "pii": false,
  "tags": ["preferencia", "faturamento"]
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json

{ "urn": "urn:aios:acme:memory:01J9Z8QCITEMULID0000000001", "state": "ACTIVE" }
```

Repetir a chamada com a **mesma** `Idempotency-Key` retorna `200 OK` com o mesmo
corpo (resultado memoizado, sem reexecutar).

### 8.2 `recall` (REST) — busca híbrida rankeada

```http
POST /v1/memory/recall HTTP/1.1
X-AIOS-Tenant: acme
Content-Type: application/json

{ "query": "preferências de faturamento do cliente",
  "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "layer": ["semantic", "episodic"], "k": 5, "min_score": 0.4, "mode": "hybrid" }
```

```json
{
  "results": [
    { "urn": "urn:aios:acme:memory:01J9Z8QCITEMULID0000000001",
      "score": 0.91, "layer": "semantic",
      "content": { "text": "O cliente prefere faturas em PDF." } }
  ],
  "recall_rate_sample": 0.97,
  "page": { "next_cursor": null, "total_estimate": 1 }
}
```

### 8.3 `consolidate` — resposta assíncrona (202)

```http
POST /v1/memory/consolidate HTTP/1.1
X-AIOS-Tenant: acme
Idempotency-Key: 01J9Z8QDCONSOL0000000000001
Content-Type: application/json

{ "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6", "from_layer": "short_term", "to_layer": "long_term", "trigger": "manual" }
```

```json
{ "job_id": "01J9Z8QEJOBULID00000000001", "state": "PENDING" }
```

O progresso é consultável por `GET /v1/memory/jobs/{id}` (estados da máquina do
`ConsolidationJob`, brief §4.2) e sinalizado por eventos (ver [Events.md](./Events.md)).

### 8.4 `forget(rtbf)` — direito ao esquecimento (202)

```http
POST /v1/memory/forget HTTP/1.1
X-AIOS-Tenant: acme
Idempotency-Key: 01J9Z8QFFORGET0000000000001
Content-Type: application/json

{ "scope": "agent", "strategy": "rtbf",
  "params": { "agent_id": "01J9Z8Q6H7K2M4N6P8R0S2T4V6", "reason": "data-subject-request" } }
```

```json
{ "affected_count": 42 }
```

Itens sob `legal_hold` NÃO são expurgados por `strategy != rtbf` e retornam
`AIOS-MEM-0050` (423) quando o alvo é exclusivamente `legal_hold`; um RTBF autorizado
PODE purgá-los (ver [Security.md](./Security.md) §LGPD e brief §4.1/§12.3).

## 9. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-----------------|--------------------------|
| Explosão de variações de `recall` (muitos filtros) sobre um único endpoint. | DTO único versionado; filtros opcionais com defaults do namespace `memory.recall.*` (ver [Configuration.md](./Configuration.md)). Alternativa descartada: um endpoint por `mode` (fragmenta contrato). |
| `consolidate` síncrono bloquearia o caller. | Contrato assíncrono (202 + `job_id`); status por polling e eventos. |
| Divergência entre proto e OpenAPI ao evoluir. | Fonte única = §5 do brief; geração checada no CI de documentação ([033](../033-DeveloperGuide/DocumentationCI.md)). |
| Paginação por offset degrada em grandes volumes. | Cursor opaco obrigatório (§4.2). |

Ver [ADR.md](./ADR.md) e [RFC.md](./RFC.md) para as decisões `ADR-0104` (fusão/rerank)
e `RFC-0100` (contrato de consolidação/esquecimento) que governam estes contratos.
