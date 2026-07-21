---
Documento: API.md
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0095, ADR-0096, ADR-0097 (a propor)
RFCs relacionados: RFC-0001 (baseline); RFC-0090, RFC-0091 (a propor)
Depende de: 001-Architecture, 003-RFC/RFC-0001, 004-API, 006-Kernel, 007-Agent-Runtime, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — API

> **Escopo.** Este documento especifica os contratos de superfície do Scheduler
> cognitivo: a API **gRPC** interna (`aios.scheduler.v1.SchedulerService`),
> consumida primariamente pelo Kernel (006) no caminho quente de admissão, e a
> API **REST** administrativa/observabilidade `/v1/scheduler/...`, exposta
> através do API Gateway (YARP). Convenções de versionamento, autenticação,
> envelope de erro RFC 7807, idempotência e cabeçalhos de correlação **DEVEM**
> seguir `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7 e **NÃO são
> redefinidas aqui** — apenas instanciadas para o domínio de scheduling.
> Consistência absoluta com `./_DESIGN_BRIEF.md` §5.

---

## 1. Convenções gerais da API do Scheduler

| Aspecto | Regra |
|---------|-------|
| Base REST | `/v1/scheduler` (versionado por caminho, RFC-0001 §5.7). |
| Pacote gRPC | `aios.scheduler.v1`. |
| Transporte preferencial | gRPC para o caminho quente (`Submit`, `ReportRuntimeSignal`) — sem overhead HTTP/JSON; REST reservado a administração/observabilidade (`_DESIGN_BRIEF.md` §5.1). |
| Autenticação | OAuth2/OIDC validado no Gateway (YARP) para chamadas externas/admin; interno (Kernel→Scheduler, Scheduler→Runtime Supervisor) via **mTLS** (`021-Security`). |
| Tenant | Cabeçalho `X-AIOS-Tenant` OBRIGATÓRIO; DEVE bater com o `tenant` do claim autenticado, senão `AIOS-SCHED-0004`. |
| Correlação | `traceparent`/`tracestate` (W3C Trace Context) OBRIGATÓRIOS em toda requisição (RFC-0001 §5.6). |
| Idempotência | Toda mutação (`Submit`, `Cancel`, `Preempt`, `ReportRuntimeSignal`) DEVE aceitar `Idempotency-Key`; resultado persistido ≥ `scheduler.idempotency.ttl_hours` (default 24h, ver `Configuration.md`). Ver §6. |
| Versão de API | `X-AIOS-Api-Version` opcional (default = última major compatível da rota). |
| Content-Type | `application/json` (REST); Protocol Buffers v3 (gRPC). |
| Paginação | `GET /v1/scheduler/queues` usa `page_token`/`page_size` (cursor opaco, `page_size` default 50, máx. 500). |
| Erros | Envelope RFC 7807 + `code` `AIOS-SCHED-<NNNN>` (RFC-0001 §5.4). Ver §5. |
| Rate limit / Backpressure | `Submit` sujeito ao `BackpressureController`: sob saturação retorna `AIOS-SCHED-0003` (503) com `Retry-After` (ver `scheduler.backpressure.*` em `Configuration.md`). |
| SLO do caminho quente | `Submit` DEVE responder em **p99 ≤ 20 ms** (NFR-001); nenhum I/O bloqueante síncrono a serviços externos além de Redis (cache local para orçamento/política, ver `Architecture.md`). |

---

## 2. Superfície de métodos (REST ↔ gRPC)

| # | Operação | REST | gRPC (`aios.scheduler.v1.SchedulerService`) | Idempotente | Caminho quente |
|---|----------|------|-----------------------------------------------|-------------|-----------------|
| 1 | Submeter `SchedulingRequest` | `POST /v1/scheduler/requests` | `rpc Submit(SchedulingRequest) returns (SchedulingDecision)` | sim (`request_id`/key) | **sim** — p99 ≤ 20 ms |
| 2 | Cancelar requisição | `DELETE /v1/scheduler/requests/{request_id}` | `rpc Cancel(CancelRequest) returns (Ack)` | sim | não |
| 3 | Preempção dirigida (admin) | `POST /v1/scheduler/preemptions` | `rpc Preempt(PreemptRequest) returns (PreemptResult)` | sim | não |
| 4 | Consultar decisão/proveniência | `GET /v1/scheduler/requests/{request_id}` | `rpc GetDecision(GetDecisionRequest) returns (SchedulingDecision)` | leitura | não |
| 5 | Runtime reporta sinal de execução | — (interno, apenas gRPC) | `rpc ReportRuntimeSignal(RuntimeSignal) returns (Ack)` | sim (`signal_id`) | **sim** — libera slot |
| 6 | Observar estado de filas (streaming) | `GET /v1/scheduler/queues` (snapshot paginado) | `rpc StreamQueueState(QueueFilter) returns (stream QueueStat)` | leitura | não |
| 7 | Ler política ativa (pesos) | `GET /v1/scheduler/policy` | — (apenas REST; governança via 022) | leitura | não |
| 8 | Atualizar política (pesos α,β,γ) | `PUT /v1/scheduler/policy` | — (apenas REST; PEP→PDP 022) | sim | não |
| 9 | Snapshot de capacidade/pressão | `GET /v1/scheduler/capacity` | — (apenas REST/observabilidade) | leitura | não |

`{request_id}` é o ULID do `request_id` (RFC-0001 §5.1). Operações 7–9 são
administrativas/observabilidade e **NÃO** participam do caminho quente de
decisão — não têm equivalente gRPC de baixa latência porque não são chamadas
por `SchedulingGateway` sob pressão de p99.

---

## 3. Contratos REST (trechos OpenAPI 3.1)

```yaml
openapi: 3.1.0
info:
  title: AIOS Scheduler API
  version: "1.0.0"
  description: >
    Superfície administrativa/observabilidade do Scheduler cognitivo (módulo 009).
    O caminho quente de decisão (Submit/ReportRuntimeSignal) é servido primariamente
    por gRPC (§4). Envelope de erro e idempotência conforme RFC-0001.
servers:
  - url: https://api.aios.internal/v1/scheduler
paths:
  /requests:
    post:
      operationId: submitSchedulingRequest
      summary: "Submete uma SchedulingRequest (admissão + enfileiramento)."
      parameters:
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/TraceparentHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SchedulingRequest'
            example:
              requestId: "01J9ZA000000000000000001"
              tenantId: "acme"
              taskUrn: "urn:aios:acme:task:01J9ZA100000000000000002"
              agentUrn: "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6"
              serviceClass: "INTERACTIVE"
              priorityHint: 60
              deadlineAt: "2026-07-20T12:05:00.000Z"
              qualityFloor: 0.80
              affinity: { "antiAffinityShards": [3] }
              payloadRef: "urn:aios:acme:tool-payload:01J9ZA2000000000000000AA"
              submittedAt: "2026-07-20T12:00:00.000Z"
      responses:
        "200":
          description: >
            Decisão de escalonamento produzida (admitido/rejeitado/deferido). O
            código HTTP é sempre 200 mesmo em rejeição de negócio — a semântica
            de sucesso/falha está em `outcome`; falhas de contrato/infra usam
            os códigos de erro de §5.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SchedulingDecision'
              example:
                decisionId: "01J9ZA300000000000000003"
                requestId: "01J9ZA000000000000000001"
                tenantId: "acme"
                outcome: "ADMITTED"
                effectivePriority: 42
                costScore: 0.3421
                alpha: 0.5
                beta: 0.3
                gamma: 0.2
                policyId: "heuristic-default"
                policyVersion: "heuristic@1"
                decidedAt: "2026-07-20T12:00:00.012Z"
                latencyDecisionMs: 8.4
        "400": { $ref: '#/components/responses/InvalidRequest' }
        "403": { $ref: '#/components/responses/PolicyDenied' }
        "409": { $ref: '#/components/responses/IdempotencyConflict' }
        "410": { $ref: '#/components/responses/DeadlineExpired' }
        "422": { $ref: '#/components/responses/QualityFloorUnattainable' }
        "429": { $ref: '#/components/responses/QuotaOrBudgetExceeded' }
        "503": { $ref: '#/components/responses/BackpressureOrNoPlacement' }

  /requests/{request_id}:
    get:
      operationId: getSchedulingDecision
      summary: "Consulta a decisão/proveniência mais recente de uma SchedulingRequest."
      parameters:
        - $ref: '#/components/parameters/RequestIdPath'
        - $ref: '#/components/parameters/TenantHeader'
      responses:
        "200":
          description: Decisão mais recente associada ao `request_id`.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/SchedulingDecision' }
        "404": { $ref: '#/components/responses/NotFound' }
    delete:
      operationId: cancelSchedulingRequest
      summary: "Cancela uma tarefa em QUEUED ou SCHEDULED."
      parameters:
        - $ref: '#/components/parameters/RequestIdPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                reason: { type: string, maxLength: 512 }
      responses:
        "200":
          description: Cancelamento aplicado; slot liberado se reservado.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Ack' }
        "404": { $ref: '#/components/responses/NotFound' }
        "409": { $ref: '#/components/responses/TerminalStateConflict' }

  /preemptions:
    post:
      operationId: requestPreemption
      summary: "Preempção dirigida (uso administrativo/Kernel). RBAC: scheduler-admin."
      parameters:
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/TraceparentHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [victimTaskUrn, reason]
              properties:
                victimTaskUrn: { type: string }
                initiatorTaskUrn: { type: string }
                reason: { type: string, maxLength: 512 }
                gracePeriodMsOverride: { type: integer, minimum: 0, maximum: 60000 }
      responses:
        "200":
          description: Resultado da preempção dirigida.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/PreemptResult' }
        "403": { $ref: '#/components/responses/PolicyDenied' }
        "404": { $ref: '#/components/responses/NotFound' }
        "409": { $ref: '#/components/responses/TerminalStateConflict' }

  /queues:
    get:
      operationId: listQueueState
      summary: "Snapshot paginado do estado das filas de prioridade por (tenant, classe, shard)."
      parameters:
        - $ref: '#/components/parameters/TenantHeader'
        - name: serviceClass
          in: query
          schema: { type: string, enum: [INTERACTIVE, BATCH, BACKGROUND, SYSTEM] }
        - name: shard
          in: query
          schema: { type: integer, minimum: 0 }
        - name: pageToken
          in: query
          schema: { type: string }
        - name: pageSize
          in: query
          schema: { type: integer, default: 50, maximum: 500 }
      responses:
        "200":
          description: Página de estatísticas de fila.
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items: { $ref: '#/components/schemas/QueueStat' }
                  nextPageToken: { type: string, nullable: true }

  /policy:
    get:
      operationId: getSchedulingPolicy
      summary: "Lê a política ativa (pesos α,β,γ, engine) por tenant."
      parameters:
        - $ref: '#/components/parameters/TenantHeader'
      responses:
        "200":
          description: Política ativa.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/PolicyWeights' }
    put:
      operationId: updateSchedulingPolicy
      summary: >
        Atualiza pesos/engine de política. Mutação governada: passa por PEP
        (`SchedulingGateway`) → PDP (022); requer role `scheduler-admin`.
      parameters:
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/PolicyWeights' }
      responses:
        "200":
          description: Política atualizada (normalizada `alpha+beta+gamma≈1`).
          content:
            application/json:
              schema: { $ref: '#/components/schemas/PolicyWeights' }
        "400": { $ref: '#/components/responses/InvalidRequest' }
        "403": { $ref: '#/components/responses/PolicyDenied' }

  /capacity:
    get:
      operationId: getCapacitySnapshot
      summary: "Snapshot de capacidade residual e nível de backpressure por shard."
      parameters:
        - $ref: '#/components/parameters/TenantHeader'
      responses:
        "200":
          description: Snapshot de capacidade/pressão.
          content:
            application/json:
              schema:
                type: object
                properties:
                  shards:
                    type: array
                    items: { $ref: '#/components/schemas/CapacitySnapshot' }
                  backpressureLevel: { type: string, enum: [accept, defer, reject] }
                  retryAfterMs: { type: integer, nullable: true }

components:
  parameters:
    RequestIdPath:
      name: request_id
      in: path
      required: true
      schema: { type: string, pattern: '^[0-9A-HJKMNP-TV-Z]{26}$' }
    TenantHeader:
      name: X-AIOS-Tenant
      in: header
      required: true
      schema: { type: string }
    TraceparentHeader:
      name: traceparent
      in: header
      required: true
      schema: { type: string }
    IdempotencyKeyHeader:
      name: Idempotency-Key
      in: header
      schema: { type: string }

  schemas:
    SchedulingRequest:
      type: object
      required: [requestId, tenantId, taskUrn, agentUrn, serviceClass, submittedAt, traceparent]
      properties:
        requestId: { type: string, description: "ULID; casado com Idempotency-Key." }
        tenantId: { type: string }
        taskUrn: { type: string }
        agentUrn: { type: string }
        serviceClass: { type: string, enum: [INTERACTIVE, BATCH, BACKGROUND, SYSTEM] }
        priorityHint: { type: integer, minimum: 0, maximum: 100, default: 50 }
        deadlineAt: { type: [string, "null"], format: date-time }
        estCostUsd: { type: number, minimum: 0 }
        estLatencyMs: { type: integer, minimum: 0 }
        estTokens: { type: integer, minimum: 0 }
        qualityFloor: { type: number, minimum: 0, maximum: 1 }
        affinity: { type: object, nullable: true }
        payloadRef: { type: [string, "null"] }
        submittedAt: { type: string, format: date-time }
        traceparent: { type: string }
    SchedulingDecision:
      type: object
      properties:
        decisionId: { type: string }
        requestId: { type: string }
        tenantId: { type: string }
        outcome: { type: string, enum: [ADMITTED, SCHEDULED, REJECTED, PREEMPTED, RESUMED, DEFERRED] }
        effectivePriority: { type: integer }
        costScore: { type: number }
        alpha: { type: number }
        beta: { type: number }
        gamma: { type: number }
        placementTarget:
          type: [object, "null"]
          properties:
            shard: { type: integer }
            nodeId: { type: string }
            runtimePool: { type: string }
        policyId: { type: string }
        policyVersion: { type: string }
        featureVector: { type: [object, "null"] }
        reasonCode: { type: [string, "null"] }
        decidedAt: { type: string, format: date-time }
        latencyDecisionMs: { type: number }
    PreemptResult:
      type: object
      properties:
        success: { type: boolean }
        victimTaskUrn: { type: string }
        gracePeriodMs: { type: integer }
        decisionId: { type: string }
    QueueStat:
      type: object
      properties:
        tenantId: { type: string }
        serviceClass: { type: string }
        shard: { type: integer }
        depth: { type: integer }
        oldestWaitMs: { type: integer }
        avgEffectivePriority: { type: number }
        pressure: { type: number }
    PolicyWeights:
      type: object
      required: [alpha, beta, gamma]
      properties:
        tenantId: { type: string }
        alpha: { type: number, minimum: 0, maximum: 1 }
        beta: { type: number, minimum: 0, maximum: 1 }
        gamma: { type: number, minimum: 0, maximum: 1 }
        qualityFloor: { type: number, minimum: 0, maximum: 1 }
        engine: { type: string, enum: [heuristic, rl], default: heuristic }
        updatedAt: { type: string, format: date-time }
    CapacitySnapshot:
      type: object
      properties:
        nodeId: { type: string }
        shard: { type: integer }
        freeSlots: { type: integer }
        pressure: { type: number }
        updatedAt: { type: string, format: date-time }
    Ack:
      type: object
      properties:
        success: { type: boolean }
        message: { type: string }
    ProblemDetails:
      type: object
      description: "RFC 7807 + extensões AIOS (RFC-0001 §5.4)."
      properties:
        type: { type: string, format: uri }
        title: { type: string }
        status: { type: integer }
        code: { type: string, pattern: '^AIOS-SCHED-[0-9]{4}$' }
        detail: { type: string }
        instance: { type: string }
        traceId: { type: string }
        timestamp: { type: string, format: date-time }
        retriable: { type: boolean }
        retryAfterMs: { type: integer, nullable: true }

  responses:
    InvalidRequest:
      description: "SchedulingRequest inválida — URN/campos (`AIOS-SCHED-0005`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    NotFound:
      description: "request_id/decisão não encontrada (`AIOS-SCHED-0009`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    PolicyDenied:
      description: "PDP (022) negou admissão/preempção/mutação de política (`AIOS-SCHED-0004`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    IdempotencyConflict:
      description: "Idempotency-Key reusada com payload divergente (`AIOS-SCHED-0006`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    DeadlineExpired:
      description: "deadline_at já vencido na admissão (`AIOS-SCHED-0008`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    QualityFloorUnattainable:
      description: "quality_floor inatingível sob restrições correntes (`AIOS-SCHED-0007`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    QuotaOrBudgetExceeded:
      description: "Cota de concorrência (`AIOS-SCHED-0001`) ou orçamento insuficiente (`AIOS-SCHED-0002`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    BackpressureOrNoPlacement:
      description: "Backpressure (`AIOS-SCHED-0003`) ou nenhum shard/nó elegível (`AIOS-SCHED-0010`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    TerminalStateConflict:
      description: "Tarefa em estado terminal; operação não aplicável (`AIOS-SCHED-0011`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
```

---

## 4. Contrato gRPC (trecho proto3 — `aios.scheduler.v1`)

```protobuf
syntax = "proto3";

package aios.scheduler.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/struct.proto";
import "google/rpc/status.proto";

option csharp_namespace = "Aios.Scheduler.V1";

// ── Serviço principal ───────────────────────────────────────────────────────
service SchedulerService {
  rpc Submit(SchedulingRequest) returns (SchedulingDecision);
  rpc Cancel(CancelRequest) returns (Ack);
  rpc Preempt(PreemptRequest) returns (PreemptResult);
  rpc GetDecision(GetDecisionRequest) returns (SchedulingDecision);
  rpc ReportRuntimeSignal(RuntimeSignal) returns (Ack);
  rpc StreamQueueState(QueueFilter) returns (stream QueueStat);
}

// ── Envelope de correlação comum (todo request o inclui) ───────────────────
message RequestMeta {
  string tenant_id       = 1;  // == X-AIOS-Tenant; DEVE bater com claim
  string traceparent     = 2;  // W3C Trace Context
  string idempotency_key = 3;  // obrigatório em mutações
  string agent_urn_caller = 4; // URN do chamador, quando aplicável
}

enum ServiceClass {
  SERVICE_CLASS_UNSPECIFIED = 0;
  INTERACTIVE = 1;
  BATCH       = 2;
  BACKGROUND  = 3;
  SYSTEM      = 4;
}

enum DecisionOutcome {
  DECISION_OUTCOME_UNSPECIFIED = 0;
  ADMITTED  = 1;
  SCHEDULED = 2;
  REJECTED  = 3;
  PREEMPTED = 4;
  RESUMED   = 5;
  DEFERRED  = 6;
}

enum RuntimeSignalType {
  RUNTIME_SIGNAL_UNSPECIFIED = 0;
  STARTED   = 1;
  COMPLETED = 2;
  FAILED    = 3;
  HEARTBEAT = 4;
}

message PlacementTarget {
  int32 shard         = 1;
  string node_id      = 2;
  string runtime_pool = 3;
}

// ── Submit ───────────────────────────────────────────────────────────────
message SchedulingRequest {
  RequestMeta meta        = 1;
  string request_id       = 2;  // ULID; == Idempotency-Key semanticamente
  string task_urn         = 3;
  string agent_urn        = 4;
  ServiceClass service_class = 5;
  int32 priority_hint     = 6;
  google.protobuf.Timestamp deadline_at = 7; // ausente == sem deadline
  double est_cost_usd     = 8;
  int32 est_latency_ms    = 9;
  int64 est_tokens        = 10;
  double quality_floor    = 11;
  google.protobuf.Struct affinity = 12;
  string payload_ref      = 13;
  google.protobuf.Timestamp submitted_at = 14;
}

message SchedulingDecision {
  string decision_id            = 1;
  string request_id             = 2;
  string tenant_id              = 3;
  DecisionOutcome outcome        = 4;
  int32 effective_priority       = 5;
  double cost_score              = 6;
  double alpha                   = 7;
  double beta                    = 8;
  double gamma                   = 9;
  PlacementTarget placement_target = 10; // ausente se REJECTED/DEFERRED
  string policy_id               = 11;
  string policy_version          = 12;
  google.protobuf.Struct feature_vector = 13;
  string reason_code             = 14;   // ex.: "AIOS-SCHED-0001" em REJECTED
  google.protobuf.Timestamp decided_at  = 15;
  double latency_decision_ms     = 16;
}

// ── Cancel ───────────────────────────────────────────────────────────────
message CancelRequest {
  RequestMeta meta = 1;
  string request_id = 2;
  string reason      = 3;
}

// ── Preempt ──────────────────────────────────────────────────────────────
message PreemptRequest {
  RequestMeta meta          = 1;
  string victim_task_urn    = 2;
  string initiator_task_urn = 3;
  string reason             = 4;
  int32 grace_period_ms_override = 5; // 0 == usar default de classe
}
message PreemptResult {
  bool success            = 1;
  string victim_task_urn  = 2;
  int32 grace_period_ms   = 3;
  string decision_id      = 4;
}

// ── GetDecision ──────────────────────────────────────────────────────────
message GetDecisionRequest {
  RequestMeta meta   = 1;
  string request_id  = 2;
}

// ── ReportRuntimeSignal (Runtime Supervisor → Scheduler) ────────────────────
message RuntimeSignal {
  RequestMeta meta        = 1;
  string signal_id        = 2;  // idempotência do sinal
  string task_urn         = 3;
  string decision_id       = 4;
  RuntimeSignalType type   = 5;
  google.protobuf.Timestamp occurred_at = 6;
  double cost_usd          = 7;  // preenchido em COMPLETED/FAILED
  int64 tokens              = 8;
  int32 latency_ms          = 9;
  string error_code         = 10; // preenchido em FAILED
  bool will_retry           = 11;
}

// ── Ack genérico ─────────────────────────────────────────────────────────
message Ack {
  bool success   = 1;
  string message = 2;
}

// ── StreamQueueState ─────────────────────────────────────────────────────
message QueueFilter {
  RequestMeta meta = 1;
  string tenant_id = 2;
  ServiceClass service_class = 3; // UNSPECIFIED == todas
  int32 shard = 4;                // -1 == todos
}
message QueueStat {
  string tenant_id        = 1;
  ServiceClass service_class = 2;
  int32 shard             = 3;
  int32 depth              = 4;
  int32 oldest_wait_ms     = 5;
  double avg_effective_priority = 6;
  double pressure          = 7;
}
```

Mapeamento de erro gRPC: todo `rpc` retorna `google.rpc.Status` com `code` gRPC
padrão e um detalhe `google.rpc.ErrorInfo{ reason: "AIOS-SCHED-NNNN", domain:
"aios.scheduler" }` — o mesmo `code` do envelope REST (RFC-0001 §5.4), garantindo
paridade de semântica entre os dois transportes.

| gRPC `code` | HTTP equivalente | `AIOS-SCHED-<NNNN>` |
|-------------|-------------------|----------------------|
| `RESOURCE_EXHAUSTED` | 429 | `0001` (cota concorrência), `0002` (orçamento) |
| `UNAVAILABLE` | 503 | `0003` (backpressure), `0010` (sem placement) |
| `PERMISSION_DENIED` | 403 | `0004` (PDP negou) |
| `INVALID_ARGUMENT` | 400 | `0005` (request inválida) |
| `ABORTED` | 409 | `0006` (conflito de idempotência) |
| `FAILED_PRECONDITION` | 422 / 409 | `0007` (quality_floor), `0011` (estado terminal) |
| `FAILED_PRECONDITION` | 410 | `0008` (deadline vencido na admissão) |
| `NOT_FOUND` | 404 | `0009` (request_id/decisão não encontrada) |
| `INTERNAL` | 500 | `0012` (falha interna, fallback aplicado) |

---

## 5. Catálogo de códigos de erro `AIOS-SCHED-<NNNN>`

> Formato RFC-0001 §5.4: `AIOS-<DOMINIO>-<NNNN>`. Domínio reservado a este
> módulo: **`SCHED`**, faixa **0001–0099** (`_DESIGN_BRIEF.md` §5.4). Esta lista
> é canônica e **NÃO PODE** ser estendida sem RFC/ADR de módulo.

| Código | HTTP | gRPC | `retriable` | Significado | Ação recomendada do cliente |
|--------|------|------|-------------|--------------|------------------------------|
| `AIOS-SCHED-0001` | 429 | `RESOURCE_EXHAUSTED` | sim | Cota de concorrência do tenant excedida (`QuotaLedger`). | Aguardar `Retry-After`; reduzir taxa de submissão; considerar aumento de cota. |
| `AIOS-SCHED-0002` | 429 | `RESOURCE_EXHAUSTED` | sim | Orçamento insuficiente para custo estimado (`BudgetClient`→026). | Aguardar reposição de orçamento; reduzir `est_tokens`/escopo da tarefa. |
| `AIOS-SCHED-0003` | 503 | `UNAVAILABLE` | sim | Backpressure: sistema saturado, sem capacidade (`BackpressureController`). | Retry com backoff exponencial + jitter respeitando `Retry-After`. |
| `AIOS-SCHED-0004` | 403 | `PERMISSION_DENIED` | não | Política negou admissão/preempção/mutação (PDP 022) ou `X-AIOS-Tenant` divergente. | Solicitar concessão de política; corrigir tenant; não repetir sem mudança. |
| `AIOS-SCHED-0005` | 400 | `INVALID_ARGUMENT` | não | `SchedulingRequest` inválida (URN malformada, campo obrigatório ausente). | Corrigir payload conforme schema OpenAPI/proto. |
| `AIOS-SCHED-0006` | 409 | `ABORTED` | não | Conflito de idempotência: `Idempotency-Key` reusada com payload divergente. | Gerar nova chave para novo payload. |
| `AIOS-SCHED-0007` | 422 | `FAILED_PRECONDITION` | não | `quality_floor` inatingível sob restrições de custo/latência correntes. | Reduzir `quality_floor` ou relaxar `deadline_at`/orçamento. |
| `AIOS-SCHED-0008` | 410 | `FAILED_PRECONDITION` | não | `deadline_at` já vencido no instante da admissão (tarefa nasce `EXPIRED`). | Resubmeter com novo `deadline_at`; não repetir a mesma requisição. |
| `AIOS-SCHED-0009` | 404 | `NOT_FOUND` | não | `request_id`/decisão não encontrada. | Verificar `request_id`; não repetir. |
| `AIOS-SCHED-0010` | 503 | `UNAVAILABLE` | sim | Nenhum shard/nó elegível para placement (afinidade/capacidade). | Retry com backoff; revisar afinidade/anti-afinidade. |
| `AIOS-SCHED-0011` | 409 | `FAILED_PRECONDITION` | não | Tarefa em estado terminal (`COMPLETED`/`FAILED`/`REJECTED`/`EXPIRED`/`CANCELLED`); operação não aplicável. | Consultar `GetDecision`; não repetir a operação. |
| `AIOS-SCHED-0012` | 500 | `INTERNAL` | sim | Falha interna de decisão; fallback conservador aplicado. | Retry com backoff; se persistir, abrir incidente (ver `Monitoring.md`). |

---

## 6. Idempotência (aplicação ao Scheduler)

- `Submit`, `Cancel`, `Preempt` DEVEM receber `Idempotency-Key`; `Submit` usa
  adicionalmente `request_id` como chave de correlação de negócio (o par
  `(tenant_id, idempotency_key)` **DEVE** mapear 1:1 para o mesmo `request_id`,
  conforme R-08/FR-008 do `_DESIGN_BRIEF.md`).
- `ReportRuntimeSignal` DEVE ser idempotente por `signal_id` (evita dupla
  liberação de slot/dupla contabilidade sob at-least-once do 007).
- O `IdempotencyStore` (Redis + PostgreSQL, ver `Database.md`) DEVE persistir
  `(tenant_id, idempotency_key) → SchedulingDecision` por, no mínimo,
  `scheduler.idempotency.ttl_hours` (default 24h — piso exigido pela RFC-0001
  §5.5).
- Repetição com **mesma chave e mesmo payload** (hash do corpo) DEVE retornar
  a **mesma** `SchedulingDecision` da primeira execução, sem reexecutar efeitos
  colaterais (sem nova reserva de slot, sem novo débito de cota, sem novo
  evento `admitted`/`rejected`/`scheduled`) — invariante NFR-011 (0 admissões
  duplicadas).
- Repetição com **mesma chave e payload divergente** DEVE retornar
  `AIOS-SCHED-0006` (409).
- Leituras (`GetDecision`, `StreamQueueState`, `GET /policy`, `GET /capacity`)
  são naturalmente idempotentes e **NÃO exigem** `Idempotency-Key`.

### 6.1 Exemplo — repetição idempotente de `Submit`

```
# 1ª chamada
POST /v1/scheduler/requests
Idempotency-Key: 01J9ZB000000000000000001
X-AIOS-Tenant: acme
{ "requestId":"01J9ZA000000000000000001", "taskUrn":"urn:aios:acme:task:01J9ZA100000000000000002",
  "agentUrn":"urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6", "serviceClass":"INTERACTIVE" }

→ 200 OK
{ "decisionId":"01J9ZA300000000000000003", "requestId":"01J9ZA000000000000000001",
  "outcome":"ADMITTED", "effectivePriority":42, ... }

# 2ª chamada (retry de rede, mesma chave e mesmo corpo)
POST /v1/scheduler/requests
Idempotency-Key: 01J9ZB000000000000000001
X-AIOS-Tenant: acme
{ "requestId":"01J9ZA000000000000000001", ... }   # corpo idêntico

→ 200 OK  (mesma SchedulingDecision exata; nenhuma nova reserva de slot, nenhum novo evento)
```

### 6.2 Exemplo — conflito de idempotência

```
POST /v1/scheduler/requests
Idempotency-Key: 01J9ZB000000000000000001
{ "requestId":"01J9ZA000000000000000001", "serviceClass":"BATCH", ... }  # classe divergente da 1ª chamada

→ 409 Conflict
{
  "type": "https://docs.aios/errors/idempotency-conflict",
  "title": "Idempotency Key Conflict",
  "status": 409,
  "code": "AIOS-SCHED-0006",
  "detail": "Idempotency-Key '01J9ZB000000000000000001' already used with a different payload.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:05:00.000Z",
  "retriable": false
}
```

---

## 7. Exemplos completos de request/response

### 7.1 `Submit` (feliz — admissão + agendamento)

```
POST /v1/scheduler/requests
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZB000000000000000010
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
{
  "requestId": "01J9ZA000000000000000010",
  "tenantId": "acme",
  "taskUrn": "urn:aios:acme:task:01J9ZA100000000000000011",
  "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "serviceClass": "INTERACTIVE",
  "priorityHint": 70,
  "qualityFloor": 0.75,
  "estCostUsd": 0.02,
  "estLatencyMs": 400,
  "estTokens": 1200,
  "submittedAt": "2026-07-20T12:00:00.000Z"
}

→ 200 OK
{
  "decisionId": "01J9ZA300000000000000012",
  "requestId": "01J9ZA000000000000000010",
  "tenantId": "acme",
  "outcome": "SCHEDULED",
  "effectivePriority": 68,
  "costScore": 0.2841,
  "alpha": 0.5, "beta": 0.3, "gamma": 0.2,
  "placementTarget": { "shard": 17, "nodeId": "node-07", "runtimePool": "python-default" },
  "policyId": "heuristic-default",
  "policyVersion": "heuristic@1",
  "decidedAt": "2026-07-20T12:00:00.009Z",
  "latencyDecisionMs": 9.1
}
```

### 7.2 `Submit` (falha — backpressure)

```
POST /v1/scheduler/requests
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZB000000000000000020
{ "requestId": "01J9ZA000000000000000020", "taskUrn": "urn:aios:acme:task:01J9ZA100000000000000021",
  "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V7", "serviceClass": "BATCH" }

→ 503 Service Unavailable
Retry-After: 2
{
  "type": "https://docs.aios/errors/scheduler-backpressure",
  "title": "Scheduler Backpressure",
  "status": 503,
  "code": "AIOS-SCHED-0003",
  "detail": "Shard pool saturated (pressure=0.97 ≥ reject_threshold=0.95).",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:01:00.000Z",
  "retriable": true,
  "retryAfterMs": 1500
}
```

### 7.3 `Preempt` (feliz — dirigido, admin)

```
POST /v1/scheduler/preemptions
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZB000000000000000030
{
  "victimTaskUrn": "urn:aios:acme:task:01J9ZA100000000000000030",
  "initiatorTaskUrn": "urn:aios:acme:task:01J9ZA100000000000000031",
  "reason": "SLA de tarefa interativa em risco"
}

→ 200 OK
{
  "success": true,
  "victimTaskUrn": "urn:aios:acme:task:01J9ZA100000000000000030",
  "gracePeriodMs": 2000,
  "decisionId": "01J9ZA300000000000000032"
}
```

### 7.4 `ReportRuntimeSignal` (gRPC — Runtime Supervisor informa falha)

```
rpc: ReportRuntimeSignal
{
  "meta": { "tenant_id": "acme", "traceparent": "00-...-01" },
  "signal_id": "01J9ZA400000000000000040",
  "task_urn": "urn:aios:acme:task:01J9ZA100000000000000041",
  "decision_id": "01J9ZA300000000000000042",
  "type": "FAILED",
  "occurred_at": "2026-07-20T12:03:00.000Z",
  "error_code": "AIOS-RUNTIME-0007",
  "will_retry": true
}

→ Ack { success: true, message: "slot liberado; tarefa reenfileirada (retry policy)" }
```

### 7.5 `GetDecision` (falha — não encontrada)

```
GET /v1/scheduler/requests/01J9ZA000000000000000999
X-AIOS-Tenant: acme

→ 404 Not Found
{
  "type": "https://docs.aios/errors/request-not-found",
  "title": "Request Not Found",
  "status": 404,
  "code": "AIOS-SCHED-0009",
  "detail": "No SchedulingDecision found for request_id '01J9ZA000000000000000999'.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:04:00.000Z",
  "retriable": false
}
```

---

## 8. Versionamento e compatibilidade

- REST: `/v1/scheduler/...`; mudanças incompatíveis DEVEM introduzir
  `/v2/scheduler/...` coexistindo por ≥ 2 majors (RFC-0001 §5.7, V-R9).
- gRPC: pacote `aios.scheduler.v1`; nova versão incompatível cria
  `aios.scheduler.v2` como novo serviço, mantendo `v1` em produção durante a
  janela de migração.
- Adição de campos opcionais (REST/proto) é **aditiva** e NÃO exige major
  bump; clientes DEVEM ignorar campos desconhecidos (RFC-0001 §5.7).
- A troca de `ISchedulingPolicy` de `heuristic` para `rl` (R-13/FR-013 do
  `_DESIGN_BRIEF.md`) **NÃO DEVE** alterar a superfície de API/eventos —
  apenas os valores de `policyId`/`policyVersion`/`featureVector` na
  `SchedulingDecision`. Ver RFC-0091 (a propor).
- Novo verbo/rota DEVE ser proposto via RFC-0090 (Scheduling Decision
  Contract) antes de implementação.

---

## 9. Referências

- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7.
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §5.
- Catálogo de eventos emitidos/consumidos por este serviço: `./Events.md`.
- Modelo de dados subjacente (`SchedulingRequest`, `SchedulingDecision`,
  `IdempotencyRecord`): `./Database.md`.
- Chaves de configuração citadas (`scheduler.idempotency.ttl_hours`,
  `scheduler.backpressure.*`, `scheduler.decision.timeout_ms`):
  `./Configuration.md`.
- Fluxos de sequência detalhados (feliz/falha): `./SequenceDiagrams.md`.
- Máquina de estados referenciada pelos outcomes/sinais: `./StateMachine.md`.
- Estrutura de componentes (`SchedulingGateway`, `AdmissionController`, ...):
  `./Architecture.md`.
