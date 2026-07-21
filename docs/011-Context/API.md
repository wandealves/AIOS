---
Documento: API
Módulo: 011-Context
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 011-Context
ADRs relacionados: ADR-0110, ADR-0111, ADR-0112, ADR-0113, ADR-0114, ADR-0117, ADR-0118, ADR-0119
RFCs relacionados: RFC-0001 (baseline), RFC-0011 (proposta)
Depende de: 003-RFC (RFC-0001), 004-API, 010-Memory, 017-Model-Router, 021-Security, 022-Policy, 040-Glossary
---

# 011-Context — API

> Contratos REST (OpenAPI) e gRPC (`aios.context.v1`) do `ContextService`.
> Este documento **deriva** da §5 do `_DESIGN_BRIEF.md` e **NÃO PODE
> contradizê-lo**. Os contratos centrais (URN, envelope de erro RFC 7807,
> idempotência, correlação, versionamento) são **reutilizados** de
> [RFC-0001](../003-RFC/RFC-0001-Architecture-Baseline.md) — aqui apenas os
> aplicamos, sem redefini-los. Palavras normativas conforme RFC 2119/8174.

## 1. Visão geral

O `ContextService` expõe a operação cognitiva canônica de montagem de janela
— **`assemble`** — além das operações auxiliares de compressão avulsa
(`compress`), cache semântico (`cache/lookup`, `cache/entries`,
`cache/invalidate`), estimativa de tokens (`tokens/estimate`) e gestão de
`BudgetProfile` (`budgets/{scope}`). A superfície é oferecida em duas fachadas
equivalentes:

- **REST/JSON** sob a base `/v1/context`, para o `007-Agent-Runtime` e
  clientes de automação.
- **gRPC** no pacote `aios.context.v1`, serviço `ContextService`, para o
  caminho quente de baixa latência (chamado a cada turno de raciocínio de
  agente).

Ambas as fachadas atravessam o `ContextApiGateway`, que valida DTOs, aplica
versionamento e traduz cada chamada em comandos internos para o
`ContextAssembler`. Toda operação passa antes pelo `ContextPolicyGuard`
(Policy Enforcement Point), que autoriza junto ao PDP
([022-Policy](../022-Policy/Security.md)) sob *default deny*, impõe o
`tenant` do contexto autenticado e valida a `Idempotency-Key`.

```
        Agent Runtime (007) / Automação / WebConsole
                   │
             ┌─────▼──────┐  X-AIOS-Api-Version, traceparent, X-AIOS-Tenant
             │   YARP     │  (Gateway 028; TLS terminado; token OAuth2/OIDC → claims)
             └─────┬──────┘
        REST /v1/context      gRPC aios.context.v1
                   │                   │
             ┌─────▼───────────────────▼─────┐
             │       ContextApiGateway        │  valida DTO + versão
             └─────┬──────────────────────────┘
                   │  → ContextPolicyGuard (PDP 022, tenant guard, Idempotency-Key)
                   ▼  comando validado → ContextAssembler (budget→retrieve→rank→dedup→compress→emit)
```

## 2. Versionamento e autenticação

### 2.1 Versionamento (RFC-0001 §5.7)

- REST DEVE ser versionada por caminho (`/v1/...`) **e** pelo header
  `X-AIOS-Api-Version`. O caminho fixa a major; o header PODE selecionar uma
  minor compatível.
- gRPC DEVE usar pacote versionado (`aios.context.v1`). Uma mudança
  incompatível incrementa a major do pacote (`v2`) e coexiste com a anterior
  por **≥ 2 majors**.
- Evolução de payload DEVE ser **aditiva**; clientes DEVEM tolerar campos
  desconhecidos.

### 2.2 Autenticação e cabeçalhos obrigatórios (RFC-0001 §5.6)

O token OAuth2/OIDC é validado no Gateway
([021-Security](../021-Security/Security.md)); as claims assinadas (incluindo
`tenant`) são propagadas ao serviço. Entre serviços internos usa-se **mTLS**.

| Cabeçalho | Obrigatório em | Descrição |
|-----------|----------------|-----------|
| `Authorization: Bearer <jwt>` | todas as chamadas externas | Token OIDC validado no Gateway. |
| `X-AIOS-Tenant` | todas (multi-tenant) | Tenant do chamador; DEVE coincidir com a claim. Divergência → `AIOS-CTX-0011`. |
| `traceparent` / `tracestate` | todas | W3C Trace Context (OTel). |
| `Idempotency-Key` | **toda mutação** (`POST`/`PUT`/`DELETE`) | ULID/UUID do cliente (RFC-0001 §5.5). |
| `X-AIOS-Agent` | quando aplicável | URN do agente que origina a requisição de `assemble`. |
| `X-AIOS-Api-Version` | recomendado | Versão de API solicitada. |

## 3. Recursos e endpoints (mapa REST ↔ gRPC)

Derivado da §5.1/§5.2 do brief.

| Operação | REST | gRPC (`aios.context.v1.ContextService`) | Mutação | Idempotente |
|----------|------|-------------------------------------------|---------|-------------|
| **assemble** | `POST /v1/context/assemble` | `Assemble(AssembleRequest)` | sim | por `Idempotency-Key` |
| get bundle | `GET /v1/context/bundles/{bundle_id}` | `GetBundle(GetBundleRequest)` | não | — |
| **compress** | `POST /v1/context/compress` | `Compress(CompressRequest)` | sim | por `Idempotency-Key` |
| cache lookup | `POST /v1/context/cache/lookup` | `CacheLookup(CacheLookupRequest)` | não | leitura (naturalmente) |
| cache store | `POST /v1/context/cache/entries` | `CacheStore(CacheStoreRequest)` | sim | por `Idempotency-Key` |
| invalidar entrada | `DELETE /v1/context/cache/entries/{cache_id}` | `InvalidateCache(InvalidateCacheRequest)` | sim | por `Idempotency-Key` |
| invalidar por escopo | `POST /v1/context/cache/invalidate` | `InvalidateCache(InvalidateCacheRequest)` | sim | por `Idempotency-Key` |
| estimar tokens | `POST /v1/context/tokens/estimate` | `EstimateTokens(EstimateTokensRequest)` | não | — |
| ler budget | `GET /v1/context/budgets/{scope}` | `GetBudget(GetBudgetRequest)` | não | — |
| upsert budget | `PUT /v1/context/budgets/{scope}` | `UpsertBudget(UpsertBudgetRequest)` | sim | por `Idempotency-Key` |

`{bundle_id}` e `{cache_id}` são ULIDs (URN completo
`urn:aios:<tenant>:context:<id>` / `urn:aios:<tenant>:ctxcache:<id>`,
RFC-0001 §5.1). `{scope}` no path aceita `tenant`, `agent:<urn>` ou
`task_type:<rotulo>`, alinhado a `BudgetProfile.scope`/`scope_ref` (brief §3.4).

## 4. Idempotência e paginação

### 4.1 Idempotência (RFC-0001 §5.5)

Toda mutação (`assemble`, `compress`, `cache/entries` POST, invalidação,
`budgets` PUT) DEVE enviar `Idempotency-Key`. O `IdempotencyStore` persiste o
resultado por chave por **≥ 24 h**; uma repetição com a **mesma** chave e
**mesmo** payload retorna o resultado memoizado (sem reexecutar o pipeline
`budget→retrieve→rank→dedup→compress`). A mesma chave com payload divergente
DEVE retornar `AIOS-CTX-0012` (409). Isto é especialmente crítico para
`assemble`: reexecutar o pipeline duas vezes para a mesma chamada de LLM
duplicaria custo de recall/embedding/sumarização.

### 4.2 Paginação por cursor

Nenhum endpoint de leitura deste módulo retorna coleções grandes por padrão
(`GetBundle` retorna um único bundle com seus fragmentos embutidos, limitado
pelo próprio `token_budget`). Quando um cliente solicitar auditoria de
fragmentos de um bundle além do embutido na resposta (uso interno/depuração),
a listagem DEVE paginar por **cursor opaco**:

```json
"page": { "size": 50, "cursor": "eyJvIjo0MH0=" }
```

## 5. Códigos de erro (`AIOS-CTX-NNNN`)

Erros seguem o envelope RFC 7807 de [RFC-0001 §5.4](../003-RFC/RFC-0001-Architecture-Baseline.md).
gRPC mapeia `status` → `google.rpc.Status` + `ErrorInfo` com o mesmo `code`.
Catálogo canônico (brief §5.3); registro central em
[004-API/Errors.md](../004-API/Errors.md). Faixa reservada:
`AIOS-CTX-0001`..`AIOS-CTX-0099`.

| Código | HTTP | gRPC (canonical) | Retriable | Significado |
|--------|------|-------------------|-----------|--------------|
| `AIOS-CTX-0001` | 422 | FAILED_PRECONDITION | não | `BudgetInfeasible` — contexto mínimo (system+instruction) não cabe no orçamento do modelo. |
| `AIOS-CTX-0002` | 503 | UNAVAILABLE | sim | `ModelLimitUnavailable` — `017` não retornou limites do modelo. |
| `AIOS-CTX-0003` | 503 | UNAVAILABLE | sim | `TokenizerUnavailable` — tokenizer do modelo indisponível. |
| `AIOS-CTX-0004` | 500 | INTERNAL | sim | `CompressionFailed` — falha na sumarização; fallback truncate aplicado. |
| `AIOS-CTX-0005` | 504 | DEADLINE_EXCEEDED | sim | `MemoryRecallTimeout` — `010` excedeu o deadline de recall. |
| `AIOS-CTX-0006` | 503 | UNAVAILABLE | sim | `CacheBackendUnavailable` — Redis/pgvector indisponível; bypass de cache. |
| `AIOS-CTX-0007` | 404 | NOT_FOUND | não | `BundleNotFound` — `bundle_id` inexistente/expirado. |
| `AIOS-CTX-0008` | 422 | INVALID_ARGUMENT | não | `InvalidBudgetProfile` — pesos de alocação não somam 1.0 ou faixa inválida. |
| `AIOS-CTX-0009` | 503 | UNAVAILABLE | sim | `EmbeddingServiceUnavailable` — modelo de embedding indisponível (`017`). |
| `AIOS-CTX-0010` | 502 | UNAVAILABLE | sim | `SummarizationModelError` — erro no modelo de sumarização. |
| `AIOS-CTX-0011` | 403 | PERMISSION_DENIED | não | `TenantMismatch` — `tenant` do token ≠ contexto (isolamento). |
| `AIOS-CTX-0012` | 409 | ABORTED | não | `IdempotencyConflict` — `Idempotency-Key` reusada com payload divergente. |
| `AIOS-CTX-0013` | 413 | RESOURCE_EXHAUSTED | não | `PayloadTooLarge` — payload de entrada excede `context.limits.max_payload_bytes`. |
| `AIOS-CTX-0014` | 400 | INVALID_ARGUMENT | não | `ValidationError` — requisição malformada. |
| `AIOS-CTX-0015` | 429 | RESOURCE_EXHAUSTED | sim | `RateLimited` — limite de taxa por tenant excedido (`context.ratelimit.assemble_rps_per_tenant`). |

Exemplo de corpo de erro (RFC 7807):

```json
{
  "type": "https://docs.aios/errors/ctx-budget-infeasible",
  "title": "Budget Infeasible",
  "status": 422,
  "code": "AIOS-CTX-0001",
  "detail": "Minimum context (system+instruction) exceeds token budget for model 'gpt-x-mini'.",
  "instance": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:34:56.789Z",
  "retriable": false
}
```

> O campo `detail` NÃO DEVE vazar conteúdo de fragmentos ou PII (RFC-0001 §6;
> ver [Security.md](./Security.md)).

## 6. Especificação OpenAPI (REST)

Fragmento normativo (OpenAPI 3.1). Campos de `ContextBundle`/`ContextFragment`/
`BudgetProfile` derivam da §3 do brief.

```yaml
openapi: 3.1.0
info:
  title: AIOS Context Service API
  version: "1.0"
servers:
  - url: https://gw.aios.local/v1/context
security:
  - oidc: []
paths:
  /assemble:
    post:
      operationId: assemble
      summary: assemble(agent, task, model, query) — monta ContextBundle ótimo
      parameters:
        - { $ref: '#/components/parameters/IdempotencyKey' }
        - { $ref: '#/components/parameters/Tenant' }
        - { $ref: '#/components/parameters/Traceparent' }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/AssembleRequest' }
      responses:
        "200": { description: Bundle montado (inclui HIT de cache), content: { application/json: { schema: { $ref: '#/components/schemas/AssembleResponse' } } } }
        "201": { description: Bundle montado (MISS, novo cálculo), content: { application/json: { schema: { $ref: '#/components/schemas/AssembleResponse' } } } }
        "409": { $ref: '#/components/responses/Problem' }   # AIOS-CTX-0012
        "422": { $ref: '#/components/responses/Problem' }   # AIOS-CTX-0001 / 0008
        "429": { $ref: '#/components/responses/Problem' }   # AIOS-CTX-0015
  /bundles/{bundle_id}:
    get:
      operationId: getBundle
      parameters:
        - { name: bundle_id, in: path, required: true, schema: { type: string } }
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: '#/components/schemas/AssembleResponse' } } } }
        "404": { $ref: '#/components/responses/Problem' }   # AIOS-CTX-0007
  /compress:
    post:
      operationId: compress
      summary: compress(fragments, target) — sumarização hierárquica avulsa
      parameters: [ { $ref: '#/components/parameters/IdempotencyKey' } ]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/CompressRequest' }
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: '#/components/schemas/CompressResponse' } } } }
        "500": { $ref: '#/components/responses/Problem' }   # AIOS-CTX-0004 (fallback truncate ainda retorna 200)
  /cache/lookup:
    post:
      operationId: cacheLookup
      summary: cache/lookup(prompt|embedding) — consulta cache semântico (L1+L2)
      requestBody:
        content: { application/json: { schema: { $ref: '#/components/schemas/CacheLookupRequest' } } }
      responses:
        "200": { description: OK (hit ou miss no corpo), content: { application/json: { schema: { $ref: '#/components/schemas/CacheLookupResponse' } } } }
  /cache/entries:
    post:
      operationId: cacheStore
      summary: cache/store(prompt, response) — grava entrada no cache semântico
      parameters: [ { $ref: '#/components/parameters/IdempotencyKey' } ]
      requestBody:
        content: { application/json: { schema: { $ref: '#/components/schemas/CacheStoreRequest' } } }
      responses:
        "201": { description: Criado, content: { application/json: { schema: { $ref: '#/components/schemas/CacheStoreResponse' } } } }
        "200": { description: Idempotente (resultado memoizado) }
  /cache/entries/{cache_id}:
    delete:
      operationId: invalidateCacheEntry
      summary: Invalida uma entrada específica
      parameters:
        - { name: cache_id, in: path, required: true, schema: { type: string } }
        - { $ref: '#/components/parameters/IdempotencyKey' }
      responses:
        "202": { description: Invalidação aceita }
        "404": { $ref: '#/components/responses/Problem' }
  /cache/invalidate:
    post:
      operationId: invalidateCacheByScope
      summary: Invalida por scope/source_urn (manual ou event-driven)
      parameters: [ { $ref: '#/components/parameters/IdempotencyKey' } ]
      requestBody:
        content: { application/json: { schema: { $ref: '#/components/schemas/InvalidateCacheRequest' } } }
      responses:
        "202": { description: Invalidação em lote aceita, content: { application/json: { schema: { $ref: '#/components/schemas/InvalidateCacheResponse' } } } }
  /tokens/estimate:
    post:
      operationId: estimateTokens
      summary: tokens/estimate(payload, model) — conta/estima tokens
      requestBody:
        content: { application/json: { schema: { $ref: '#/components/schemas/EstimateTokensRequest' } } }
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: '#/components/schemas/EstimateTokensResponse' } } } }
  /budgets/{scope}:
    get:
      operationId: getBudget
      parameters: [ { name: scope, in: path, required: true, schema: { type: string } } ]
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: '#/components/schemas/BudgetProfile' } } } }
        "404": { $ref: '#/components/responses/Problem' }
    put:
      operationId: upsertBudget
      parameters:
        - { name: scope, in: path, required: true, schema: { type: string } }
        - { $ref: '#/components/parameters/IdempotencyKey' }
      requestBody:
        content: { application/json: { schema: { $ref: '#/components/schemas/UpsertBudgetRequest' } } }
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: '#/components/schemas/BudgetProfile' } } } }
        "422": { $ref: '#/components/responses/Problem' }   # AIOS-CTX-0008
components:
  securitySchemes:
    oidc: { type: openIdConnect, openIdConnectUrl: https://idp.aios.local/.well-known/openid-configuration }
  parameters:
    IdempotencyKey: { name: Idempotency-Key, in: header, required: true, schema: { type: string } }
    Tenant: { name: X-AIOS-Tenant, in: header, required: true, schema: { type: string } }
    Traceparent: { name: traceparent, in: header, required: true, schema: { type: string } }
  responses:
    Problem:
      description: Erro RFC 7807 (AIOS-CTX-NNNN)
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/Problem' } } }
  schemas:
    AssembleRequest:
      type: object
      required: [agent_urn, model_id, query]
      properties:
        agent_urn: { type: string }
        task_urn: { type: string, nullable: true }
        model_id: { type: string, description: "Modelo alvo (rótulo/URN reconhecido por 017)." }
        query: { type: string, description: "Consulta usada pelo SelectiveRetriever no recall a 010-Memory." }
        system: { type: string, nullable: true }
        instructions: { type: string, nullable: true }
        history:
          type: array
          items:
            type: object
            properties:
              role: { type: string, enum: [system, user, assistant, tool] }
              content: { type: string }
        tool_results:
          type: array
          items:
            type: object
            properties:
              tool_urn: { type: string }
              content: { type: string }
        recall:
          type: object
          properties:
            layer: { type: array, items: { type: string } }
            kind: { type: array, items: { type: string } }
            top_k: { type: integer, minimum: 1, maximum: 256 }
            min_score: { type: number, minimum: 0, maximum: 1 }
        budget_profile_scope: { type: string, enum: [tenant, agent, task_type], default: agent }
        bypass_cache: { type: boolean, default: false }
        reserved_output_tokens: { type: integer, nullable: true }
    AssembleResponse:
      type: object
      properties:
        bundle_urn: { type: string }
        status: { type: string, enum: [ASSEMBLED, SERVED, DEGRADED] }
        cache_outcome: { type: string, enum: [HIT, MISS, BYPASS] }
        model_id: { type: string }
        model_max_tokens: { type: integer }
        token_budget: { type: integer }
        reserved_output_tokens: { type: integer }
        tokens_raw: { type: integer }
        tokens_in: { type: integer }
        compression_ratio: { type: number }
        fragments:
          type: array
          items: { $ref: '#/components/schemas/ContextFragment' }
        assembled_messages:
          type: array
          description: "Sequência final pronta para a chamada de inferência (repassada a 017 pelo caller)."
          items:
            type: object
            properties: { role: { type: string }, content: { type: string } }
        expires_at: { type: string, format: date-time }
        trace_id: { type: string }
    ContextFragment:
      type: object
      properties:
        fragment_urn: { type: string }
        source_kind: { type: string, enum: [system, instruction, memory, knowledge, history, tool_result] }
        source_urn: { type: string, nullable: true }
        role: { type: string, enum: [system, user, assistant, tool] }
        ordinal: { type: integer }
        tokens_raw: { type: integer }
        tokens_final: { type: integer }
        relevance_score: { type: number }
        compression_method: { type: string, enum: [none, extractive, abstractive, truncate, dedup] }
        included: { type: boolean }
        content_inline: { type: string, nullable: true }
        content_ref: { type: string, nullable: true }
    CompressRequest:
      type: object
      required: [fragments]
      properties:
        fragments:
          type: array
          items:
            type: object
            properties:
              source_kind: { type: string }
              role: { type: string }
              content: { type: string }
        target_tokens: { type: integer, nullable: true }
        target_ratio: { type: number, nullable: true, minimum: 0, maximum: 0.95 }
        method_hint: { type: string, enum: [hierarchical, extractive, truncate], default: hierarchical }
    CompressResponse:
      type: object
      properties:
        fragments: { type: array, items: { $ref: '#/components/schemas/ContextFragment' } }
        summary_nodes:
          type: array
          items:
            type: object
            properties: { node_id: { type: string }, level: { type: integer }, tokens: { type: integer } }
        tokens_before: { type: integer }
        tokens_after: { type: integer }
        ratio: { type: number }
    CacheLookupRequest:
      type: object
      required: [model_id]
      properties:
        model_id: { type: string }
        prompt: { type: string, nullable: true }
        prompt_embedding: { type: array, items: { type: number }, nullable: true }
        scope: { type: string, enum: [tenant, agent, task_type], default: tenant }
        scope_ref: { type: string, nullable: true }
        similarity_threshold: { type: number, nullable: true, minimum: 0.80, maximum: 0.99 }
    CacheLookupResponse:
      type: object
      properties:
        hit: { type: boolean }
        cache_id: { type: string, nullable: true }
        similarity: { type: number, nullable: true }
        response_inline: { type: object, nullable: true }
        response_ref: { type: string, nullable: true }
        tokens_saved: { type: integer }
        cost_saved_usd: { type: number }
    CacheStoreRequest:
      type: object
      required: [model_id, response]
      properties:
        model_id: { type: string }
        prompt: { type: string, nullable: true }
        prompt_embedding: { type: array, items: { type: number }, nullable: true }
        response: { type: object }
        scope: { type: string, enum: [tenant, agent, task_type], default: tenant }
        scope_ref: { type: string, nullable: true }
        similarity_threshold: { type: number, nullable: true }
        ttl_seconds: { type: integer, nullable: true }
    CacheStoreResponse:
      type: object
      properties: { cache_id: { type: string }, state: { type: string, enum: [PENDING, INDEXED] } }
    InvalidateCacheRequest:
      type: object
      properties:
        scope: { type: string, enum: [tenant, agent, task_type], nullable: true }
        scope_ref: { type: string, nullable: true }
        source_urn: { type: string, nullable: true }
        reason: { type: string }
    InvalidateCacheResponse:
      type: object
      properties: { affected_count: { type: integer } }
    EstimateTokensRequest:
      type: object
      required: [model_id]
      properties:
        model_id: { type: string }
        text: { type: string, nullable: true }
        fragments: { type: array, items: { type: string }, nullable: true }
    EstimateTokensResponse:
      type: object
      properties:
        tokens: { type: integer }
        method: { type: string, enum: [exact, estimated] }
        tokenizer_id: { type: string }
    BudgetProfile:
      type: object
      properties:
        profile_id: { type: string }
        scope: { type: string, enum: [tenant, agent, task_type] }
        scope_ref: { type: string, nullable: true }
        reserved_output_ratio: { type: number }
        alloc_system: { type: number }
        alloc_instruction: { type: number }
        alloc_memory: { type: number }
        alloc_history: { type: number }
        alloc_tools: { type: number }
        min_compression_ratio: { type: number }
        updated_at: { type: string, format: date-time }
    UpsertBudgetRequest:
      allOf:
        - $ref: '#/components/schemas/BudgetProfile'
    Problem:
      type: object
      properties:
        type: { type: string }
        title: { type: string }
        status: { type: integer }
        code: { type: string, pattern: "^AIOS-CTX-[0-9]{4}$" }
        detail: { type: string }
        instance: { type: string }
        traceId: { type: string }
        timestamp: { type: string, format: date-time }
        retriable: { type: boolean }
        retryAfterMs: { type: integer }
```

## 7. Especificação gRPC (proto)

Pacote reservado `aios.context.v1` (brief §5.2). Erros propagados via
`google.rpc.Status` + `ErrorInfo{ reason = "AIOS-CTX-NNNN" }`.

```proto
syntax = "proto3";
package aios.context.v1;

import "google/protobuf/struct.proto";
import "google/protobuf/timestamp.proto";

service ContextService {
  rpc Assemble        (AssembleRequest)        returns (AssembleResponse);
  rpc GetBundle        (GetBundleRequest)        returns (AssembleResponse);
  rpc Compress         (CompressRequest)         returns (CompressResponse);
  rpc CacheLookup      (CacheLookupRequest)      returns (CacheLookupResponse);
  rpc CacheStore       (CacheStoreRequest)       returns (CacheStoreResponse);
  rpc InvalidateCache  (InvalidateCacheRequest)  returns (InvalidateCacheResponse);
  rpc EstimateTokens   (EstimateTokensRequest)   returns (EstimateTokensResponse);
  rpc GetBudget        (GetBudgetRequest)        returns (BudgetProfile);
  rpc UpsertBudget     (UpsertBudgetRequest)     returns (BudgetProfile);
}

enum SourceKind { SOURCE_KIND_UNSPECIFIED=0; SYSTEM=1; INSTRUCTION=2; MEMORY=3; KNOWLEDGE=4; HISTORY=5; TOOL_RESULT=6; }
enum MessageRole { ROLE_UNSPECIFIED=0; ROLE_SYSTEM=1; ROLE_USER=2; ROLE_ASSISTANT=3; ROLE_TOOL=4; }
enum CompressionMethod { METHOD_UNSPECIFIED=0; NONE=1; EXTRACTIVE=2; ABSTRACTIVE=3; TRUNCATE=4; DEDUP=5; }
enum CacheOutcome { OUTCOME_UNSPECIFIED=0; HIT=1; MISS=2; BYPASS=3; }
enum BudgetScope { SCOPE_UNSPECIFIED=0; TENANT=1; AGENT=2; TASK_TYPE=3; }
enum BundleStatus {
  BUNDLE_STATUS_UNSPECIFIED=0; RECEIVED=1; POLICY_CHECK=2; CACHE_LOOKUP=3;
  BUDGET_ALLOCATED=4; RETRIEVING=5; RANKING=6; COMPRESSING=7; ASSEMBLED=8;
  SERVED=9; REJECTED=10; DEGRADED=11; FAILED=12; EXPIRED=13;
}

message HistoryTurn { MessageRole role = 1; string content = 2; }
message ToolResultInput { string tool_urn = 1; string content = 2; }
message RecallOverride {
  repeated string layer = 1; repeated string kind = 2;
  int32 top_k = 3; float min_score = 4;
}

message AssembleRequest {
  string idempotency_key = 1;           // obrigatório (mutação)
  string tenant = 2;                    // deve casar com contexto autenticado
  string agent_urn = 3;
  string task_urn = 4;
  string model_id = 5;
  string query = 6;
  string system = 7;
  string instructions = 8;
  repeated HistoryTurn history = 9;
  repeated ToolResultInput tool_results = 10;
  RecallOverride recall = 11;
  BudgetScope budget_profile_scope = 12;
  bool bypass_cache = 13;
  int32 reserved_output_tokens = 14;    // 0 = usar BudgetProfile
}

message ContextFragmentMsg {
  string fragment_urn = 1;
  SourceKind source_kind = 2;
  string source_urn = 3;
  MessageRole role = 4;
  int32 ordinal = 5;
  int32 tokens_raw = 6;
  int32 tokens_final = 7;
  double relevance_score = 8;
  CompressionMethod compression_method = 9;
  bool included = 10;
  string content_inline = 11;
  string content_ref = 12;
}

message AssembleResponse {
  string bundle_urn = 1;
  BundleStatus status = 2;
  CacheOutcome cache_outcome = 3;
  string model_id = 4;
  int32 model_max_tokens = 5;
  int32 token_budget = 6;
  int32 reserved_output_tokens = 7;
  int32 tokens_raw = 8;
  int32 tokens_in = 9;
  double compression_ratio = 10;
  repeated ContextFragmentMsg fragments = 11;
  repeated HistoryTurn assembled_messages = 12;
  google.protobuf.Timestamp expires_at = 13;
  string trace_id = 14;
}

message GetBundleRequest { string tenant = 1; string bundle_urn = 2; }

message CompressFragmentInput { SourceKind source_kind = 1; MessageRole role = 2; string content = 3; }
message CompressRequest {
  string idempotency_key = 1; string tenant = 2;
  repeated CompressFragmentInput fragments = 3;
  int32 target_tokens = 4; float target_ratio = 5;
  string method_hint = 6;   // hierarchical|extractive|truncate
}
message SummaryNodeRef { string node_id = 1; int32 level = 2; int32 tokens = 3; }
message CompressResponse {
  repeated ContextFragmentMsg fragments = 1;
  repeated SummaryNodeRef summary_nodes = 2;
  int32 tokens_before = 3; int32 tokens_after = 4; double ratio = 5;
}

message CacheLookupRequest {
  string tenant = 1; string model_id = 2; string prompt = 3;
  repeated float prompt_embedding = 4;
  BudgetScope scope = 5; string scope_ref = 6; float similarity_threshold = 7;
}
message CacheLookupResponse {
  bool hit = 1; string cache_id = 2; double similarity = 3;
  google.protobuf.Struct response_inline = 4; string response_ref = 5;
  int32 tokens_saved = 6; double cost_saved_usd = 7;
}

message CacheStoreRequest {
  string idempotency_key = 1; string tenant = 2; string model_id = 3;
  string prompt = 4; repeated float prompt_embedding = 5;
  google.protobuf.Struct response = 6;
  BudgetScope scope = 7; string scope_ref = 8;
  float similarity_threshold = 9; int32 ttl_seconds = 10;
}
message CacheStoreResponse { string cache_id = 1; string state = 2; }

message InvalidateCacheRequest {
  string idempotency_key = 1; string tenant = 2;
  string cache_id = 3; BudgetScope scope = 4; string scope_ref = 5;
  string source_urn = 6; string reason = 7;
}
message InvalidateCacheResponse { int64 affected_count = 1; }

message EstimateTokensRequest { string tenant = 1; string model_id = 2; string text = 3; repeated string fragments = 4; }
message EstimateTokensResponse { int32 tokens = 1; string method = 2; string tokenizer_id = 3; }

message BudgetProfile {
  string profile_id = 1; BudgetScope scope = 2; string scope_ref = 3;
  float reserved_output_ratio = 4; float alloc_system = 5; float alloc_instruction = 6;
  float alloc_memory = 7; float alloc_history = 8; float alloc_tools = 9;
  float min_compression_ratio = 10; google.protobuf.Timestamp updated_at = 11;
}
message GetBudgetRequest { string tenant = 1; BudgetScope scope = 2; string scope_ref = 3; }
message UpsertBudgetRequest {
  string idempotency_key = 1; string tenant = 2; BudgetProfile profile = 3;
}
```

## 8. Exemplos request/response

### 8.1 `assemble` (REST) — feliz, cache MISS

```http
POST /v1/context/assemble HTTP/1.1
Host: gw.aios.local
Authorization: Bearer <jwt>
X-AIOS-Tenant: acme
X-AIOS-Agent: urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6
Idempotency-Key: 01J9Z8QGASSEMBLE00000001
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
Content-Type: application/json

{
  "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "task_urn": "urn:aios:acme:task:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
  "model_id": "gpt-x-mini",
  "query": "resumo das preferências de faturamento do cliente para a próxima resposta",
  "system": "Você é um agente de atendimento da AIOS Corp.",
  "instructions": "Responda em português, seja objetivo.",
  "history": [ { "role": "user", "content": "Qual o status da minha fatura?" } ],
  "recall": { "layer": ["semantic", "episodic"], "top_k": 16, "min_score": 0.35 },
  "bypass_cache": false
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "bundle_urn": "urn:aios:acme:context:01J9Z8QHBUNDLE0000000001",
  "status": "SERVED",
  "cache_outcome": "MISS",
  "model_id": "gpt-x-mini",
  "model_max_tokens": 16000,
  "token_budget": 12000,
  "reserved_output_tokens": 4000,
  "tokens_raw": 5230,
  "tokens_in": 2380,
  "compression_ratio": 0.5450,
  "fragments": [
    { "fragment_urn": "urn:aios:acme:ctxfrag:01J9...A", "source_kind": "system", "role": "system",
      "ordinal": 0, "tokens_raw": 18, "tokens_final": 18, "relevance_score": 1.0,
      "compression_method": "none", "included": true, "content_inline": "Você é um agente..." },
    { "fragment_urn": "urn:aios:acme:ctxfrag:01J9...B", "source_kind": "memory", "role": "system",
      "ordinal": 1, "tokens_raw": 940, "tokens_final": 210, "relevance_score": 0.91,
      "compression_method": "abstractive", "included": true, "content_inline": "Cliente prefere faturas em PDF; histórico de 3 solicitações de 2ª via." }
  ],
  "assembled_messages": [
    { "role": "system", "content": "Você é um agente de atendimento da AIOS Corp." },
    { "role": "system", "content": "Cliente prefere faturas em PDF; histórico de 3 solicitações de 2ª via." },
    { "role": "user", "content": "Qual o status da minha fatura?" }
  ],
  "expires_at": "2026-07-20T12:44:56.789Z",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

Repetir a chamada com a **mesma** `Idempotency-Key` retorna `200 OK` com o
mesmo corpo (resultado memoizado, sem reexecutar o pipeline).

### 8.2 `assemble` — feliz, cache HIT

Mesma requisição do §8.1 com `bypass_cache: false` e um prompt
suficientemente similar a uma entrada existente:

```json
{
  "bundle_urn": "urn:aios:acme:context:01J9Z8QJBUNDLE0000000002",
  "status": "SERVED",
  "cache_outcome": "HIT",
  "model_id": "gpt-x-mini",
  "token_budget": 12000,
  "tokens_raw": 5230,
  "tokens_in": 0,
  "compression_ratio": 1.0,
  "fragments": [],
  "assembled_messages": [
    { "role": "assistant", "content": "Sua fatura de julho já foi enviada em PDF para o e-mail cadastrado." }
  ],
  "expires_at": "2026-07-20T12:34:56.789Z",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

> Em `HIT`, `ContextAssembler` **não** monta um novo bundle de entrada — a
> resposta cacheada é servida diretamente (`SemanticCacheEntry.response_ref`),
> economizando 100% dos tokens de entrada e a chamada de inferência.

### 8.3 `assemble` — falha, orçamento infeasível (422)

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://docs.aios/errors/ctx-budget-infeasible",
  "title": "Budget Infeasible",
  "status": 422,
  "code": "AIOS-CTX-0001",
  "detail": "Minimum context (system+instruction) exceeds token budget for model 'gpt-x-nano'.",
  "instance": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:34:56.789Z",
  "retriable": false
}
```

### 8.4 `cache/invalidate` — invalidação por escopo (202)

```http
POST /v1/context/cache/invalidate HTTP/1.1
X-AIOS-Tenant: acme
Idempotency-Key: 01J9Z8QKINVAL0000000001
Content-Type: application/json

{ "source_urn": "urn:aios:acme:memory:01J9Z8QCITEMULID0000000001",
  "reason": "memory.item.consolidated" }
```

```json
{ "affected_count": 3 }
```

### 8.5 `budgets/{scope}` PUT — perfil inválido (422)

```http
PUT /v1/context/budgets/tenant HTTP/1.1
X-AIOS-Tenant: acme
Idempotency-Key: 01J9Z8QLBUDGET0000000001
Content-Type: application/json

{ "scope": "tenant", "reserved_output_ratio": 0.25,
  "alloc_system": 0.10, "alloc_instruction": 0.15,
  "alloc_memory": 0.30, "alloc_history": 0.30, "alloc_tools": 0.10 }
```

```json
{
  "type": "https://docs.aios/errors/ctx-invalid-budget-profile",
  "title": "Invalid Budget Profile",
  "status": 422,
  "code": "AIOS-CTX-0008",
  "detail": "alloc_system+alloc_instruction+alloc_memory+alloc_history+alloc_tools must equal 1.0 (got 0.95).",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:34:56.789Z",
  "retriable": false
}
```

## 9. Riscos e alternativas

| Risco / decisão | Mitigação / alternativa |
|-----------------|--------------------------|
| `assemble` acoplado a um payload rico (system/instructions/history/tools) infla o contrato. | DTO único versionado, aditivo; overrides opcionais (`recall`, `budget_profile_scope`). Alternativa descartada: um endpoint por seção (fragmenta o pipeline atômico). |
| `compress` síncrono com sumarização LLM pode violar SLO (NFR-003, brief §7.2). | `method_hint` permite `truncate` determinístico como fallback; timeout do `SummarizationClient` aplica `AIOS-CTX-0004` sem bloquear o caller. |
| Repetir `assemble` idêntico gera custo de recall/embedding duplicado. | Idempotência obrigatória (§4.1) **e** cache semântico (§8.2) atacam o mesmo problema em camadas complementares. |
| Divergência entre proto e OpenAPI ao evoluir. | Fonte única = §5 do `_DESIGN_BRIEF.md`; geração checada no CI de documentação ([033](../033-DeveloperGuide/DocumentationCI.md)). |
| `GetBundle` após expiração (`expires_at`) retorna 404 inesperado ao cliente. | `AIOS-CTX-0007` documentado explicitamente; clientes DEVEM tratar bundles como efêmeros e não persistir `bundle_urn` além do TTL. |

Ver [ADR.md](./ADR.md) e [RFC.md](./RFC.md) para as decisões `ADR-0113`
(algoritmo de token budgeting), `ADR-0114` (registro de tokenizers) e
`RFC-0011` (contrato de assembly e cache semântico) que governam estes
contratos.
