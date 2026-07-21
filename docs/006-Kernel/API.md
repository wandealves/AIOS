---
Documento: API.md
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0060, ADR-0063, ADR-0069
RFCs relacionados: RFC-0001 (baseline), RFC-0006 (Cognitive Syscall ABI, a propor)
Depende de: 001-Architecture, 003-RFC/RFC-0001, 004-API, 022-Policy, 009-Scheduler, 008-Agent-Lifecycle
---

# 006-Kernel — API

> **Escopo.** Este documento especifica os contratos de superfície do Kernel: a
> ABI de **syscalls cognitivas** expostas via REST (externo, através do API
> Gateway/YARP) e via gRPC (interno, service-to-service). Convenções de
> versionamento, autenticação, envelope de erro, idempotência e cabeçalhos de
> correlação **DEVEM** seguir `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7
> e **NÃO são redefinidas aqui** — apenas instanciadas para o domínio do Kernel.
> Consistência absoluta com `_DESIGN_BRIEF.md` §5.

---

## 1. Convenções gerais da API do Kernel

| Aspecto | Regra |
|---------|-------|
| Base REST | `/v1/kernel` (versionado por caminho, RFC-0001 §5.7). |
| Pacote gRPC | `aios.kernel.v1`. |
| Autenticação | OAuth2/OIDC validado no Gateway (YARP); Kernel consome claims assinadas. Interno: mTLS (`021-Security`). |
| Tenant | Cabeçalho `X-AIOS-Tenant` OBRIGATÓRIO; DEVE bater com o `tenant` do claim autenticado, senão `AIOS-CAP-0002`. |
| Correlação | `traceparent`/`tracestate` (W3C Trace Context) OBRIGATÓRIOS em toda requisição. |
| Idempotência | Toda mutação DEVE aceitar `Idempotency-Key` (ULID/UUID); resultado persistido ≥ 24h (`kernel.idempotency.retention_h`, default 48h). Ver §4. |
| Versão de API | `X-AIOS-Api-Version` opcional (default = última major compatível da rota). |
| Content-Type | `application/json` (REST); Protocol Buffers v3 (gRPC). |
| Paginação | Listagens (ex.: futura `ListAgents`) usam `page_token`/`page_size` (cursor opaco, max `page_size=200`). Nenhum endpoint do MVP (§2) expõe listagem paginada; reservado para extensão v1.1. |
| Erros | Envelope RFC 7807 + `code` `AIOS-<DOMINIO>-<NNNN>` (RFC-0001 §5.4). Ver §5. |
| Rate limit | Aplicado por `SyscallGateway` conforme `kernel.quota.syscalls_per_sec` (ver `Configuration.md`); excesso retorna `AIOS-QUOTA-0002` com `Retry-After`. |

---

## 2. Superfície de syscalls cognitivas (REST + gRPC)

> Os 11 verbos definidos no `_DESIGN_BRIEF.md` §5.1, mapeados 1:1 entre REST e
> gRPC. Toda syscall privilegiada passa por `CapabilityEnforcer` (PEP) antes de
> qualquer efeito colateral — ver `SequenceDiagrams.md` para o fluxo completo.

| # | Verbo | REST | gRPC (`aios.kernel.v1.KernelService`) | Idempotente | Privilegiada (PEP) |
|---|-------|------|-----------------------------------------|-------------|---------------------|
| 1 | `spawn` | `POST /v1/kernel/agents` | `rpc Spawn(SpawnRequest) returns (SpawnResponse)` | sim (key) | sim — `agent:spawn` |
| 2 | `kill` | `DELETE /v1/kernel/agents/{urn}` | `rpc Kill(KillRequest) returns (KillResponse)` | sim | sim — `agent:kill` |
| 3 | `suspend` | `POST /v1/kernel/agents/{urn}:suspend` | `rpc Suspend(SuspendRequest) returns (SuspendResponse)` | sim | sim — `agent:suspend` |
| 4 | `resume` | `POST /v1/kernel/agents/{urn}:resume` | `rpc Resume(ResumeRequest) returns (ResumeResponse)` | sim | sim — `agent:resume` |
| 5 | `remember` | `POST /v1/kernel/agents/{urn}/memory` | `rpc Remember(RememberRequest) returns (RememberResponse)` | sim (key) | sim — `memory:write` |
| 6 | `recall` | `POST /v1/kernel/agents/{urn}/memory:query` | `rpc Recall(RecallRequest) returns (RecallResponse)` | sim (leitura) | sim — `memory:read` |
| 7 | `plan` | `POST /v1/kernel/agents/{urn}/plans` | `rpc Plan(PlanRequest) returns (PlanResponse)` | sim (key) | sim — `plan:create` |
| 8 | `invoke_tool` | `POST /v1/kernel/agents/{urn}/tools:invoke` | `rpc InvokeTool(InvokeToolRequest) returns (InvokeToolResponse)` | sim (key) | sim — `tool:invoke:<tool_id>` |
| 9 | `route_model` | `POST /v1/kernel/agents/{urn}/inference:route` | `rpc RouteModel(RouteModelRequest) returns (RouteModelResponse)` | sim (key) | sim — `model:invoke` |
| 10 | `get_quota` | `GET /v1/kernel/agents/{urn}/quota` | `rpc GetQuota(GetQuotaRequest) returns (GetQuotaResponse)` | sim (leitura) | sim — `quota:read` |
| 11 | `checkpoint` | `POST /v1/kernel/agents/{urn}:checkpoint` | `rpc Checkpoint(CheckpointRequest) returns (CheckpointResponse)` | sim (key) | sim — `agent:checkpoint` |
| 12 (admin) | — | `GET /v1/kernel/agents/{urn}` | `rpc GetAcb(GetAcbRequest) returns (GetAcbResponse)` | sim (leitura) | sim — `agent:read` |

`{urn}` é o URN completo *urlencoded* do agente (`urn:aios:<tenant>:agent:<ULID>`).

---

## 3. Contratos REST (trechos OpenAPI 3.1)

```yaml
openapi: 3.1.0
info:
  title: AIOS Kernel API
  version: "1.0.0"
  description: >
    ABI de syscalls cognitivas do Kernel (módulo 006). Envelope de erro e
    idempotência conforme RFC-0001.
servers:
  - url: https://api.aios.internal/v1/kernel
paths:
  /agents:
    post:
      operationId: spawnAgent
      summary: "Syscall spawn — cria um novo Agent Control Block (ACB)."
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
              $ref: '#/components/schemas/SpawnRequest'
            example:
              parentUrn: null
              priority: 5
              policyRef: "urn:aios:acme:policy:default-agent"
              quotaOverride:
                tokensLimit: 500000
                costUsdLimit: 5.0
              labels:
                slaClass: "standard"
      responses:
        "201":
          description: ACB criado, estado inicial `Pending`.
          headers:
            Location:
              schema: { type: string }
              description: "URL do recurso `/v1/kernel/agents/{urn}`."
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AcbView'
              example:
                urn: "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6"
                tenantId: "acme"
                state: "Pending"
                priority: 5
                version: 0
                createdAt: "2026-07-20T12:00:00.000Z"
        "400": { $ref: '#/components/responses/BadRequest' }
        "403": { $ref: '#/components/responses/CapabilityDenied' }
        "409": { $ref: '#/components/responses/IdempotencyConflict' }
        "429": { $ref: '#/components/responses/QuotaExceeded' }

  /agents/{urn}:
    get:
      operationId: getAcb
      summary: "Lê o ACB autoritativo do agente."
      parameters:
        - $ref: '#/components/parameters/UrnPath'
        - $ref: '#/components/parameters/TenantHeader'
      responses:
        "200":
          description: ACB atual.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/AcbView' }
        "404": { $ref: '#/components/responses/NotFound' }
    delete:
      operationId: killAgent
      summary: "Syscall kill — inicia encerramento (Terminating → Terminated)."
      parameters:
        - $ref: '#/components/parameters/UrnPath'
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
                force: { type: boolean, default: false }
      responses:
        "202":
          description: Encerramento aceito; transição assíncrona para `Terminating`.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/AcbView' }
        "404": { $ref: '#/components/responses/NotFound' }
        "403": { $ref: '#/components/responses/CapabilityDenied' }
        "409": { $ref: '#/components/responses/InvalidTransition' }

  /agents/{urn}:suspend:
    post:
      operationId: suspendAgent
      summary: "Syscall suspend — Running → Suspended."
      parameters:
        - $ref: '#/components/parameters/UrnPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      responses:
        "200":
          description: Agente suspenso.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/AcbView' }
        "409": { $ref: '#/components/responses/InvalidTransition' }

  /agents/{urn}:resume:
    post:
      operationId: resumeAgent
      summary: "Syscall resume — Suspended|Hibernated → Admitted/Running."
      parameters:
        - $ref: '#/components/parameters/UrnPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      responses:
        "200":
          description: Agente em processo de retomada (síncrono se `Suspended`; assíncrono via `Admitted` se `Hibernated`).
          content:
            application/json:
              schema: { $ref: '#/components/schemas/AcbView' }
        "409": { $ref: '#/components/responses/InvalidTransition' }
        "503": { $ref: '#/components/responses/SchedulerUnavailable' }

  /agents/{urn}/memory:
    post:
      operationId: rememberItem
      summary: "Syscall remember — encaminha item ao 010-Memory após PEP+cota."
      parameters:
        - $ref: '#/components/parameters/UrnPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [namespace, content]
              properties:
                namespace: { type: string, enum: [working, short, long, semantic, episodic, procedural] }
                content: { type: object }
                ttlSeconds: { type: integer, minimum: 0 }
      responses:
        "201":
          description: Item persistido (delegado a 010-Memory).
          content:
            application/json:
              schema:
                type: object
                properties:
                  itemUrn: { type: string }
                  namespace: { type: string }
        "403": { $ref: '#/components/responses/CapabilityDenied' }
        "429": { $ref: '#/components/responses/QuotaExceeded' }
        "502": { $ref: '#/components/responses/BrokerFailure' }

  /agents/{urn}/memory:query:
    post:
      operationId: recallItems
      summary: "Syscall recall — consulta ao 010-Memory (leitura)."
      parameters:
        - $ref: '#/components/parameters/UrnPath'
        - $ref: '#/components/parameters/TenantHeader'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [query]
              properties:
                namespace: { type: string }
                query: { type: string, maxLength: 4096 }
                topK: { type: integer, default: 10, maximum: 100 }
      responses:
        "200":
          description: Itens recuperados (ordenados por relevância).
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items: { type: object }
        "403": { $ref: '#/components/responses/CapabilityDenied' }
        "502": { $ref: '#/components/responses/BrokerFailure' }

  /agents/{urn}/plans:
    post:
      operationId: createPlan
      summary: "Syscall plan — encaminha objetivo ao 012-Planning."
      parameters:
        - $ref: '#/components/parameters/UrnPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [goal]
              properties:
                goal: { type: object }
                constraints: { type: object }
      responses:
        "202":
          description: Plano solicitado (processamento assíncrono possível).
          content:
            application/json:
              schema:
                type: object
                properties:
                  planUrn: { type: string }
        "403": { $ref: '#/components/responses/CapabilityDenied' }
        "429": { $ref: '#/components/responses/QuotaExceeded' }

  /agents/{urn}/tools:invoke:
    post:
      operationId: invokeTool
      summary: "Syscall invoke_tool — encaminha ao 015-Tool-Manager."
      parameters:
        - $ref: '#/components/parameters/UrnPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [toolId, input]
              properties:
                toolId: { type: string }
                input: { type: object }
                timeoutMs: { type: integer, default: 10000 }
      responses:
        "200":
          description: Resultado da invocação da ferramenta.
          content:
            application/json:
              schema:
                type: object
                properties:
                  output: { type: object }
                  costUsd: { type: number }
        "403": { $ref: '#/components/responses/CapabilityDenied' }
        "429": { $ref: '#/components/responses/QuotaExceeded' }
        "502": { $ref: '#/components/responses/BrokerFailure' }

  /agents/{urn}/inference:route:
    post:
      operationId: routeModel
      summary: "Syscall route_model — encaminha ao 017-Model-Router."
      parameters:
        - $ref: '#/components/parameters/UrnPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [messages]
              properties:
                messages: { type: array, items: { type: object } }
                modelHint: { type: string }
                maxTokens: { type: integer }
      responses:
        "200":
          description: Resposta de inferência roteada.
          content:
            application/json:
              schema:
                type: object
                properties:
                  modelUsed: { type: string }
                  tokensUsed: { type: integer }
                  costUsd: { type: number }
                  output: { type: object }
        "403": { $ref: '#/components/responses/CapabilityDenied' }
        "429": { $ref: '#/components/responses/QuotaExceeded' }
        "502": { $ref: '#/components/responses/BrokerFailure' }

  /agents/{urn}/quota:
    get:
      operationId: getQuota
      summary: "Syscall get_quota — saldo corrente de cotas do agente."
      parameters:
        - $ref: '#/components/parameters/UrnPath'
        - $ref: '#/components/parameters/TenantHeader'
      responses:
        "200":
          description: Saldo de cotas.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/QuotaView' }
        "404": { $ref: '#/components/responses/NotFound' }

  /agents/{urn}:checkpoint:
    post:
      operationId: checkpointAgent
      summary: "Syscall checkpoint — snapshot durável para resume consistente."
      parameters:
        - $ref: '#/components/parameters/UrnPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      responses:
        "201":
          description: Checkpoint criado.
          content:
            application/json:
              schema:
                type: object
                properties:
                  checkpointRef: { type: string }
                  createdAt: { type: string, format: date-time }
        "403": { $ref: '#/components/responses/CapabilityDenied' }
        "409": { $ref: '#/components/responses/InvalidTransition' }

components:
  parameters:
    UrnPath:
      name: urn
      in: path
      required: true
      schema: { type: string, pattern: '^urn:aios:[a-z0-9-]+:agent:[0-9A-HJKMNP-TV-Z]{26}$' }
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
    SpawnRequest:
      type: object
      properties:
        parentUrn: { type: [string, "null"] }
        priority: { type: integer, minimum: 0, maximum: 9, default: 5 }
        policyRef: { type: string }
        quotaOverride: { type: object }
        labels: { type: object }
    AcbView:
      type: object
      properties:
        urn: { type: string }
        tenantId: { type: string }
        state:
          type: string
          enum: [Pending, Admitted, Running, Suspended, Hibernated, Terminating, Terminated, Failed]
        priority: { type: integer }
        parentUrn: { type: [string, "null"] }
        runtimeRef: { type: [string, "null"] }
        checkpointRef: { type: [string, "null"] }
        version: { type: integer }
        createdAt: { type: string, format: date-time }
        updatedAt: { type: string, format: date-time }
    QuotaView:
      type: object
      properties:
        subjectUrn: { type: string }
        tokensRemaining: { type: integer }
        cpuMsRemaining: { type: integer }
        memMibRemaining: { type: integer }
        costUsdRemaining: { type: number }
        toolsRemaining: { type: integer }
        windowResetAt: { type: string, format: date-time }
        enforcement: { type: string, enum: [hard, soft] }
    ProblemDetails:
      type: object
      description: "RFC 7807 + extensões AIOS (RFC-0001 §5.4)."
      properties:
        type: { type: string, format: uri }
        title: { type: string }
        status: { type: integer }
        code: { type: string, pattern: '^AIOS-[A-Z]+-[0-9]{4}$' }
        detail: { type: string }
        instance: { type: string }
        traceId: { type: string }
        timestamp: { type: string, format: date-time }
        retriable: { type: boolean }
        retryAfterMs: { type: integer }

  responses:
    BadRequest:
      description: Payload/schema inválido (`AIOS-KERNEL-0010` ou `AIOS-SYSCALL-0001`).
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    NotFound:
      description: ACB não encontrado (`AIOS-KERNEL-0001`).
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    CapabilityDenied:
      description: Negado pelo PDP (`AIOS-CAP-0001`) ou tenant divergente (`AIOS-CAP-0002`).
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    InvalidTransition:
      description: Transição de FSM inválida (`AIOS-KERNEL-0002`) ou conflito de OCC (`AIOS-KERNEL-0009`).
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    IdempotencyConflict:
      description: "`Idempotency-Key` reusada com payload divergente (`AIOS-SYSCALL-0002`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    QuotaExceeded:
      description: Cota excedida (`AIOS-QUOTA-0001`/`0002`/`0003`).
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    SchedulerUnavailable:
      description: Scheduler indisponível na admissão (`AIOS-KERNEL-0003`).
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    BrokerFailure:
      description: Falha de broker de recurso (`AIOS-KERNEL-0005`).
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
```

---

## 4. Contrato gRPC (trecho proto3 — `aios.kernel.v1`)

```protobuf
syntax = "proto3";

package aios.kernel.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/struct.proto";
import "google/rpc/status.proto";

option csharp_namespace = "Aios.Kernel.V1";

// ── Serviço principal: ABI de syscalls cognitivas ──────────────────────────
service KernelService {
  rpc Spawn(SpawnRequest) returns (SpawnResponse);
  rpc Kill(KillRequest) returns (KillResponse);
  rpc Suspend(SuspendRequest) returns (SuspendResponse);
  rpc Resume(ResumeRequest) returns (ResumeResponse);
  rpc Remember(RememberRequest) returns (RememberResponse);
  rpc Recall(RecallRequest) returns (RecallResponse);
  rpc Plan(PlanRequest) returns (PlanResponse);
  rpc InvokeTool(InvokeToolRequest) returns (InvokeToolResponse);
  rpc RouteModel(RouteModelRequest) returns (RouteModelResponse);
  rpc GetQuota(GetQuotaRequest) returns (GetQuotaResponse);
  rpc Checkpoint(CheckpointRequest) returns (CheckpointResponse);
  rpc GetAcb(GetAcbRequest) returns (GetAcbResponse);
}

// ── Envelope de correlação comum (todo request o inclui) ───────────────────
message RequestMeta {
  string tenant_id        = 1;  // == X-AIOS-Tenant; DEVE bater com claim
  string traceparent      = 2;  // W3C Trace Context
  string idempotency_key  = 3;  // obrigatório em mutações
  string agent_urn_caller = 4;  // URN do chamador (X-AIOS-Agent), quando aplicável
}

enum AcbState {
  ACB_STATE_UNSPECIFIED = 0;
  PENDING      = 1;
  ADMITTED     = 2;
  RUNNING      = 3;
  SUSPENDED    = 4;
  HIBERNATED   = 5;
  TERMINATING  = 6;
  TERMINATED   = 7;
  FAILED       = 8;
}

message AcbView {
  string urn                     = 1;
  string tenant_id               = 2;
  AcbState state                 = 3;
  int32 priority                 = 4;
  string parent_urn              = 5;  // vazio se raiz
  string runtime_ref             = 6;  // vazio se cold
  string checkpoint_ref          = 7;  // vazio se nunca houve checkpoint
  int64 version                  = 8;
  google.protobuf.Timestamp created_at = 9;
  google.protobuf.Timestamp updated_at = 10;
}

// ── spawn ────────────────────────────────────────────────────────────────
message SpawnRequest {
  RequestMeta meta       = 1;
  string parent_urn      = 2;
  int32 priority         = 3;
  string policy_ref      = 4;
  google.protobuf.Struct quota_override = 5;
  google.protobuf.Struct labels         = 6;
}
message SpawnResponse {
  AcbView acb = 1;
}

// ── kill ─────────────────────────────────────────────────────────────────
message KillRequest {
  RequestMeta meta = 1;
  string urn       = 2;
  string reason    = 3;
  bool force       = 4;
}
message KillResponse {
  AcbView acb = 1;
}

// ── suspend / resume ────────────────────────────────────────────────────
message SuspendRequest { RequestMeta meta = 1; string urn = 2; }
message SuspendResponse { AcbView acb = 1; }
message ResumeRequest  { RequestMeta meta = 1; string urn = 2; }
message ResumeResponse { AcbView acb = 1; }

// ── remember / recall ───────────────────────────────────────────────────
message RememberRequest {
  RequestMeta meta   = 1;
  string urn         = 2;
  string namespace   = 3; // working|short|long|semantic|episodic|procedural
  google.protobuf.Struct content = 4;
  int64 ttl_seconds  = 5;
}
message RememberResponse {
  string item_urn  = 1;
  string namespace = 2;
}
message RecallRequest {
  RequestMeta meta = 1;
  string urn       = 2;
  string namespace = 3;
  string query     = 4;
  int32 top_k      = 5;
}
message RecallResponse {
  repeated google.protobuf.Struct items = 1;
}

// ── plan ─────────────────────────────────────────────────────────────────
message PlanRequest {
  RequestMeta meta = 1;
  string urn       = 2;
  google.protobuf.Struct goal        = 3;
  google.protobuf.Struct constraints = 4;
}
message PlanResponse {
  string plan_urn = 1;
}

// ── invoke_tool ──────────────────────────────────────────────────────────
message InvokeToolRequest {
  RequestMeta meta = 1;
  string urn       = 2;
  string tool_id   = 3;
  google.protobuf.Struct input = 4;
  int32 timeout_ms = 5;
}
message InvokeToolResponse {
  google.protobuf.Struct output = 1;
  double cost_usd = 2;
}

// ── route_model ──────────────────────────────────────────────────────────
message RouteModelRequest {
  RequestMeta meta = 1;
  string urn       = 2;
  repeated google.protobuf.Struct messages = 3;
  string model_hint = 4;
  int32 max_tokens  = 5;
}
message RouteModelResponse {
  string model_used  = 1;
  int64 tokens_used  = 2;
  double cost_usd    = 3;
  google.protobuf.Struct output = 4;
}

// ── get_quota ────────────────────────────────────────────────────────────
message GetQuotaRequest {
  RequestMeta meta = 1;
  string urn       = 2;
}
message GetQuotaResponse {
  string subject_urn         = 1;
  int64 tokens_remaining     = 2;
  int64 cpu_ms_remaining     = 3;
  int32 mem_mib_remaining    = 4;
  double cost_usd_remaining  = 5;
  int32 tools_remaining      = 6;
  google.protobuf.Timestamp window_reset_at = 7;
  string enforcement         = 8; // hard|soft
}

// ── checkpoint ───────────────────────────────────────────────────────────
message CheckpointRequest {
  RequestMeta meta = 1;
  string urn       = 2;
}
message CheckpointResponse {
  string checkpoint_ref = 1;
  google.protobuf.Timestamp created_at = 2;
}

// ── get_acb (admin/leitura) ─────────────────────────────────────────────
message GetAcbRequest {
  RequestMeta meta = 1;
  string urn       = 2;
}
message GetAcbResponse {
  AcbView acb = 1;
}
```

Mapeamento de erro gRPC: todo `rpc` retorna `google.rpc.Status` com `code`
gRPC padrão (`PERMISSION_DENIED`, `NOT_FOUND`, `FAILED_PRECONDITION`,
`RESOURCE_EXHAUSTED`, `UNAVAILABLE`, `INVALID_ARGUMENT`) e um detalhe
`google.rpc.ErrorInfo{ reason: "AIOS-CAP-0001", domain: "aios.kernel" }` —
o mesmo `code` do envelope REST (RFC-0001 §5.4), garantindo paridade de
semântica entre os dois transportes.

| gRPC `code` | HTTP equivalente | Exemplo `AIOS-<DOMINIO>-<NNNN>` |
|-------------|-------------------|----------------------------------|
| `PERMISSION_DENIED` | 403 | `AIOS-CAP-0001`, `AIOS-CAP-0002` |
| `NOT_FOUND` | 404 | `AIOS-KERNEL-0001` |
| `FAILED_PRECONDITION` | 409 | `AIOS-KERNEL-0002`, `AIOS-KERNEL-0009` |
| `RESOURCE_EXHAUSTED` | 429 | `AIOS-QUOTA-0001`, `AIOS-QUOTA-0002`, `AIOS-QUOTA-0003` |
| `UNAVAILABLE` | 503 | `AIOS-KERNEL-0003`, `AIOS-CAP-0003` |
| `DEADLINE_EXCEEDED` | 504 | `AIOS-KERNEL-0004` |
| `INVALID_ARGUMENT` | 400 | `AIOS-KERNEL-0010`, `AIOS-SYSCALL-0001` |
| `ABORTED` | 409 | `AIOS-SYSCALL-0002` |

---

## 5. Catálogo de códigos de erro do Kernel

> Formato RFC-0001 §5.4: `AIOS-<DOMINIO>-<NNNN>`. Domínios reservados a este
> módulo: **`KERNEL`**, **`CAP`**, **`SYSCALL`**; faixa própria em **`QUOTA`**
> (`AIOS-QUOTA-0001..0020`, compartilhado com `026-Cost-Optimizer`). Ver
> ADR-0069 (a propor).

| Código | HTTP | gRPC | `retriable` | Significado | Ação recomendada do cliente |
|--------|------|------|-------------|--------------|------------------------------|
| `AIOS-KERNEL-0001` | 404 | `NOT_FOUND` | não | ACB/agente não encontrado. | Verificar URN; não repetir. |
| `AIOS-KERNEL-0002` | 409 | `FAILED_PRECONDITION` | não | Transição de estado inválida (viola FSM). | Consultar `GetAcb`, reavaliar fluxo. |
| `AIOS-KERNEL-0003` | 503 | `UNAVAILABLE` | sim | Scheduler indisponível na admissão. | Retry com backoff exponencial + jitter. |
| `AIOS-KERNEL-0004` | 504 | `DEADLINE_EXCEEDED` | sim | Timeout de materialização (boot). | Retry; agente pode já estar `Failed`. |
| `AIOS-KERNEL-0005` | 502 | `UNAVAILABLE` | sim | Falha de broker de recurso (010/011/012/015/017). | Retry com backoff; checar `Health` do broker. |
| `AIOS-KERNEL-0009` | 409 | `FAILED_PRECONDITION` | sim | Conflito de concorrência otimista (`version`). | Reler `GetAcb`, reaplicar com `version` atual. |
| `AIOS-KERNEL-0010` | 400 | `INVALID_ARGUMENT` | não | Payload/schema de syscall inválido. | Corrigir payload; validar contra proto/OpenAPI. |
| `AIOS-CAP-0001` | 403 | `PERMISSION_DENIED` | não | Capability negada pelo PDP (*default deny*). | Solicitar concessão de capability; não repetir. |
| `AIOS-CAP-0002` | 403 | `PERMISSION_DENIED` | não | Tenant divergente do contexto autenticado. | Corrigir `X-AIOS-Tenant`/claim. |
| `AIOS-CAP-0003` | 503 | `UNAVAILABLE` | sim | PDP indisponível e `fail-closed` ativo. | Retry após backoff; monitorar `022-Policy`. |
| `AIOS-QUOTA-0001` | 429 | `RESOURCE_EXHAUSTED` | sim | Orçamento de tokens/custo excedido (hard). | Aguardar reset de janela ou solicitar aumento. |
| `AIOS-QUOTA-0002` | 429 | `RESOURCE_EXHAUSTED` | sim | Rate-limit de syscalls excedido. | Respeitar `Retry-After`; aplicar backoff. |
| `AIOS-QUOTA-0003` | 429 | `RESOURCE_EXHAUSTED` | sim | Limite de ferramentas/inferências excedido. | Aguardar janela; reduzir concorrência. |
| `AIOS-SYSCALL-0001` | 400 | `INVALID_ARGUMENT` | não | Verbo desconhecido ou versão de ABI não suportada. | Atualizar cliente/SDK; checar `X-AIOS-Api-Version`. |
| `AIOS-SYSCALL-0002` | 409 | `ABORTED` | não | `Idempotency-Key` reusada com payload divergente. | Gerar nova chave para novo payload. |

---

## 6. Idempotência (aplicação ao Kernel)

- Toda mutação (`spawn`, `kill`, `suspend`, `resume`, `remember`, `plan`,
  `invoke_tool`, `route_model`, `checkpoint`) DEVE receber `Idempotency-Key`.
- O `IdempotencyStore` DEVE persistir `(tenant_id, idempotency_key) → resultado`
  por, no mínimo, `kernel.idempotency.retention_h` (default 48h ≥ 24h exigido
  pela RFC-0001 §5.5).
- Repetição com **mesma chave e mesmo payload** (hash do corpo) DEVE retornar
  o **mesmo status HTTP e corpo** da primeira execução, sem reexecutar efeitos
  colaterais (sem novo `spawn`, sem novo débito de cota, sem novo evento).
- Repetição com **mesma chave e payload divergente** DEVE retornar
  `AIOS-SYSCALL-0002` (409).
- Leituras (`recall`, `get_quota`, `GetAcb`) são naturalmente idempotentes e
  **NÃO exigem** `Idempotency-Key`, mas o campo é aceito e ignorado se enviado.

### 6.1 Exemplo — repetição idempotente de `spawn`

```
# 1ª chamada
POST /v1/kernel/agents
Idempotency-Key: 01J9ZC000000000000000001
X-AIOS-Tenant: acme
{ "priority": 5, "policyRef": "urn:aios:acme:policy:default-agent" }

→ 201 Created
{ "urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6", "state": "Pending", "version": 0, ... }

# 2ª chamada (retry de rede, mesma chave e mesmo corpo)
POST /v1/kernel/agents
Idempotency-Key: 01J9ZC000000000000000001
X-AIOS-Tenant: acme
{ "priority": 5, "policyRef": "urn:aios:acme:policy:default-agent" }

→ 201 Created  (mesmo corpo exato; nenhum novo ACB criado, nenhum novo evento emitido)
```

### 6.2 Exemplo — conflito de idempotência

```
POST /v1/kernel/agents
Idempotency-Key: 01J9ZC000000000000000001
{ "priority": 9, ... }   # payload diferente da 1ª chamada acima

→ 409 Conflict
{
  "type": "https://docs.aios/errors/idempotency-conflict",
  "title": "Idempotency Key Conflict",
  "status": 409,
  "code": "AIOS-SYSCALL-0002",
  "detail": "Idempotency-Key '01J9ZC000000000000000001' already used with a different payload.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:05:00.000Z",
  "retriable": false
}
```

---

## 7. Exemplos completos de request/response

### 7.1 `suspend` (feliz)

```
POST /v1/kernel/agents/urn%3Aaios%3Aacme%3Aagent%3A01J9Z8Q6H7K2M4N6P8R0S2T4V6:suspend
X-AIOS-Tenant: acme
X-AIOS-Agent: urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6
Idempotency-Key: 01J9ZC000000000000000099
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

→ 200 OK
{
  "urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "tenantId": "acme",
  "state": "Suspended",
  "priority": 5,
  "runtimeRef": null,
  "version": 3,
  "updatedAt": "2026-07-20T12:10:00.000Z"
}
```

### 7.2 `invoke_tool` (falha por capability negada)

```
POST /v1/kernel/agents/urn%3Aaios%3Aacme%3Aagent%3A01J9Z8Q6H7K2M4N6P8R0S2T4V6/tools:invoke
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZC0000000000000000AA
{ "toolId": "urn:aios:acme:tool:web-search", "input": { "query": "..." } }

→ 403 Forbidden
{
  "type": "https://docs.aios/errors/capability-denied",
  "title": "Capability Denied",
  "status": 403,
  "code": "AIOS-CAP-0001",
  "detail": "Agent lacks capability 'tool:invoke:web-search'.",
  "instance": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:11:00.000Z",
  "retriable": false
}
```

### 7.3 `get_quota` (feliz)

```
GET /v1/kernel/agents/urn%3Aaios%3Aacme%3Aagent%3A01J9Z8Q6H7K2M4N6P8R0S2T4V6/quota
X-AIOS-Tenant: acme

→ 200 OK
{
  "subjectUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "tokensRemaining": 482300,
  "cpuMsRemaining": 91200,
  "memMibRemaining": 512,
  "costUsdRemaining": 4.21,
  "toolsRemaining": 37,
  "windowResetAt": "2026-07-20T13:00:00.000Z",
  "enforcement": "hard"
}
```

### 7.4 `spawn` (falha por cota de tenant excedida)

```
POST /v1/kernel/agents
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZC0000000000000000BB
{ "priority": 3 }

→ 429 Too Many Requests
Retry-After: 2
{
  "type": "https://docs.aios/errors/quota-exceeded",
  "title": "Quota Exceeded",
  "status": 429,
  "code": "AIOS-QUOTA-0001",
  "detail": "Tenant 'acme' agent_count quota exhausted for current window.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:12:00.000Z",
  "retriable": true,
  "retryAfterMs": 2000
}
```

---

## 8. Versionamento e compatibilidade

- REST: `/v1/kernel/...`; mudanças incompatíveis DEVEM introduzir `/v2/kernel/...`
  coexistindo por ≥ 2 majors (RFC-0001 §5.7, V-R9).
- gRPC: pacote `aios.kernel.v1`; nova versão incompatível cria `aios.kernel.v2`
  como novo serviço, mantendo `v1` em produção durante a janela de migração.
- Adições de campos opcionais (REST/proto) são **aditivas** e NÃO exigem
  major bump; clientes DEVEM ignorar campos desconhecidos.
- Novo verbo de syscall (ex.: futura extensão da ABI) DEVE ser proposto via
  RFC-0006 (Cognitive Syscall ABI) antes de implementação, com endpoint em
  `/v1/kernel/...` aditivo — não reaproveitar código de erro de outro domínio.

---

## 9. Referências

- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7.
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §5.
- Catálogo de eventos emitidos por estas syscalls: `./Events.md`.
- Modelo de dados subjacente (ACB, Quota, SyscallRecord): `./Database.md`.
- Chaves de configuração citadas (`kernel.idempotency.retention_h`,
  `kernel.quota.syscalls_per_sec`): `./Configuration.md`.
- Fluxos de sequência detalhados (feliz/falha): `./SequenceDiagrams.md`.
- Máquina de estados do ACB referenciada pelos verbos: `./StateMachine.md`.
