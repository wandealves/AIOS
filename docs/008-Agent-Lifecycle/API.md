---
Documento: API.md
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0082, ADR-0083, ADR-0084, ADR-0085
RFCs relacionados: RFC-0001 (baseline), RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore Protocol, a propor)
Depende de: 001-Architecture, 003-RFC/RFC-0001, 004-API, 006-Kernel, 007-Agent-Runtime, 009-Scheduler, 022-Policy
---

# 008-Agent-Lifecycle — API

> **Escopo.** Este documento especifica a superfície de API do módulo 008 —
> **Lifecycle Manager**: os contratos REST (`/v1/agents/**`) e gRPC
> (`aios.lifecycle.v1.LifecycleService`) para spawn, suspend, resume,
> hibernate, wake, migrate, checkpoint, restore e terminate de agentes, além
> da consulta do histórico de transições. Versionamento, autenticação,
> envelope de erro RFC 7807, `Idempotency-Key` e cabeçalhos de correlação
> **DEVEM** seguir `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7 e
> **NÃO são redefinidos aqui** — apenas instanciados para o domínio do ciclo
> de vida. Consistência absoluta com `_DESIGN_BRIEF.md` §5.

---

## 1. Convenções gerais da API do módulo 008

| Aspecto | Regra |
|---------|-------|
| Base REST | `/v1/agents` (versionado por caminho, RFC-0001 §5.7). Subrecursos de ciclo de vida sob `/v1/agents/{id}/...`. |
| Pacote gRPC | `aios.lifecycle.v1`. |
| Autenticação | OAuth2/OIDC validado no Gateway (YARP); claims assinadas propagadas. Interno: mTLS (`021-Security`). |
| Tenant | Cabeçalho `X-AIOS-Tenant` OBRIGATÓRIO; DEVE bater com o tenant autenticado, senão `AIOS-LIFECYCLE-0012`. |
| Correlação | `traceparent`/`tracestate` (W3C Trace Context) OBRIGATÓRIOS em toda requisição (NFR-012 exige 100% de cobertura). |
| Idempotência | Toda mutação (POST/DELETE de §2) DEVE aceitar `Idempotency-Key` (ULID/UUID); resultado persistido ≥ 24h. Ver §5. |
| Versão de API | `X-AIOS-Api-Version` opcional (default = última major compatível da rota). |
| Content-Type | `application/json` (REST); Protocol Buffers v3 (gRPC). |
| Paginação | `GET .../lifecycle` e `GET .../checkpoints` usam `page_token`/`page_size` (cursor opaco, max `page_size=200`, default `50`), ordenados por `seq`/`created_at` decrescente. |
| Erros | Envelope RFC 7807 + `code` `AIOS-LIFECYCLE-<NNNN>` (faixa 0001–0099, RFC-0001 §5.4). Ver §6. |
| PEP | Toda mutação passa por `LifecyclePolicyEnforcer` antes de qualquer efeito colateral (*default deny*, RFC-0001 §5.8). |
| Transporte interno vs externo | Chamadas de outros módulos do control/data plane usam gRPC diretamente; chamadas externas (CLI, SDK, WebConsole) passam pelo YARP (REST). |

---

## 2. Superfície de operações (REST + gRPC)

> Os 12 verbos definidos no `_DESIGN_BRIEF.md` §5.1/§5.2, mapeados 1:1 entre
> REST e gRPC. Cada verbo corresponde a uma transição (ou consulta) da FSM
> canônica descrita em `StateMachine.md` §4.

| # | Ação | REST | gRPC (`LifecycleService`) | Idempotente | Efeito na FSM | Sucesso |
|---|------|------|-----------------------------|-------------|----------------|---------|
| 1 | spawn/create | `POST /v1/agents` | `rpc Spawn(SpawnRequest) returns (LifecycleAck)` | sim (key) | T1 (`∅→Created`), delega admissão a T2 | 202 |
| 2 | get | `GET /v1/agents/{id}` | `rpc GetState(AgentRef) returns (AgentControlBlock)` | sim (leitura) | — | 200 |
| 3 | history | `GET /v1/agents/{id}/lifecycle` | `rpc StreamTransitions(AgentRef) returns (stream LifecycleTransition)` | sim (leitura) | — | 200 |
| 4 | suspend | `POST /v1/agents/{id}/suspend` | `rpc Suspend(AgentRef) returns (LifecycleAck)` | sim (key) | T4 (`Running→Suspended`) | 202 |
| 5 | resume | `POST /v1/agents/{id}/resume` | `rpc Resume(AgentRef) returns (LifecycleAck)` | sim (key) | T5 (`Suspended→Running`) | 202 |
| 6 | hibernate | `POST /v1/agents/{id}/hibernate` | `rpc Hibernate(AgentRef) returns (LifecycleAck)` | sim (key) | T6 (`Suspended→Hibernated`) | 202 |
| 7 | wake | `POST /v1/agents/{id}/wake` | `rpc Wake(WakeRequest) returns (LifecycleAck)` | sim (key) | T7 (`Hibernated→Ready/Running`) | 202 |
| 8 | migrate | `POST /v1/agents/{id}/migrate` | `rpc Migrate(MigrateRequest) returns (LifecycleAck)` | sim (key) | T8/T9 (`Running/Suspended→Migrating→...`) | 202 |
| 9 | checkpoint | `POST /v1/agents/{id}/checkpoint` | `rpc Checkpoint(AgentRef) returns (CheckpointRef)` | sim (key) | efeito colateral (sem transição de estado) | 201 |
| 10 | restore | `POST /v1/agents/{id}/restore` | `rpc Restore(RestoreRequest) returns (LifecycleAck)` | sim (key) | recomposição do ACB a partir de checkpoint | 202 |
| 11 | terminate | `DELETE /v1/agents/{id}` | `rpc Terminate(TerminateRequest) returns (LifecycleAck)` | sim (key) | T10 (`qualquer não-terminal→Terminated`) | 202 |
| 12 | list checkpoints | `GET /v1/agents/{id}/checkpoints` | — (leitura via `Checkpoint`/futuro `ListCheckpoints`) | sim (leitura) | — | 200 |

`{id}` é o URN completo *urlencoded* do agente
(`urn:aios:<tenant>:agent:<ULID>`), compartilhado com 006/007.

---

## 3. Contratos REST (trechos OpenAPI 3.1)

```yaml
openapi: 3.1.0
info:
  title: AIOS Agent Lifecycle API
  version: "1.0.0"
  description: >
    Superfície REST do Lifecycle Manager (módulo 008): spawn, suspend,
    resume, hibernate, wake, migrate, checkpoint, restore, terminate.
    Envelope de erro e idempotência conforme RFC-0001.
servers:
  - url: https://api.aios.internal/v1
paths:
  /agents:
    post:
      operationId: spawnAgent
      summary: "Cria/spawn de agente novo ou materialização de agente frio existente."
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
              priorityClass: "interactive"
              policyRef: "urn:aios:acme:policy:default-agent"
              quotaRef: "urn:aios:acme:quota:default"
              labels:
                slaClass: "standard"
      responses:
        "202":
          description: "Spawn aceito; ACB transita ∅→Created, admissão assíncrona (T1/T2)."
          headers:
            Location:
              schema: { type: string }
              description: "URL do recurso `/v1/agents/{id}`."
          content:
            application/json:
              schema: { $ref: '#/components/schemas/AgentControlBlockView' }
              example:
                agentId: "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6"
                tenantId: "acme"
                state: "Created"
                generation: 1
                priorityClass: "interactive"
                createdAt: "2026-07-20T12:00:00.000Z"
        "400": { $ref: '#/components/responses/BadRequest' }
        "403": { $ref: '#/components/responses/PolicyDenied' }
        "409": { $ref: '#/components/responses/IdempotencyConflict' }
        "429": { $ref: '#/components/responses/AdmissionDenied' }

  /agents/{id}:
    get:
      operationId: getAgentControlBlock
      summary: "Lê o ACB autoritativo (estado de ciclo de vida atual)."
      parameters:
        - $ref: '#/components/parameters/AgentIdPath'
        - $ref: '#/components/parameters/TenantHeader'
      responses:
        "200":
          description: "ACB atual."
          content:
            application/json:
              schema: { $ref: '#/components/schemas/AgentControlBlockView' }
        "404": { $ref: '#/components/responses/NotFound' }
    delete:
      operationId: terminateAgent
      summary: "Encerra o agente — qualquer estado não-terminal → Terminated (T10)."
      parameters:
        - $ref: '#/components/parameters/AgentIdPath'
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
                finalCheckpoint: { type: boolean, default: false }
      responses:
        "202":
          description: "Encerramento aceito; transição assíncrona para Terminated."
          content:
            application/json:
              schema: { $ref: '#/components/schemas/AgentControlBlockView' }
        "403": { $ref: '#/components/responses/PolicyDenied' }
        "404": { $ref: '#/components/responses/NotFound' }
        "423": { $ref: '#/components/responses/AlreadyTerminal' }

  /agents/{id}/lifecycle:
    get:
      operationId: getLifecycleHistory
      summary: "Histórico de transições do agente (event-sourced), paginado."
      parameters:
        - $ref: '#/components/parameters/AgentIdPath'
        - $ref: '#/components/parameters/TenantHeader'
        - name: pageToken
          in: query
          schema: { type: string }
        - name: pageSize
          in: query
          schema: { type: integer, default: 50, maximum: 200 }
      responses:
        "200":
          description: "Sequência de transições ordenadas por `seq` (crescente)."
          content:
            application/json:
              schema:
                type: object
                properties:
                  transitions:
                    type: array
                    items: { $ref: '#/components/schemas/LifecycleTransitionView' }
                  nextPageToken: { type: [string, "null"] }
        "404": { $ref: '#/components/responses/NotFound' }

  /agents/{id}/suspend:
    post:
      operationId: suspendAgent
      summary: "Running → Suspended (T4), preservando working memory."
      parameters:
        - $ref: '#/components/parameters/AgentIdPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      responses:
        "202": { description: "Suspensão aceita.", content: { application/json: { schema: { $ref: '#/components/schemas/AgentControlBlockView' } } } }
        "409": { $ref: '#/components/responses/InvalidTransition' }
        "423": { $ref: '#/components/responses/AlreadyTerminal' }

  /agents/{id}/resume:
    post:
      operationId: resumeAgent
      summary: "Suspended → Running (T5)."
      parameters:
        - $ref: '#/components/parameters/AgentIdPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      responses:
        "202": { description: "Retomada aceita.", content: { application/json: { schema: { $ref: '#/components/schemas/AgentControlBlockView' } } } }
        "409": { $ref: '#/components/responses/InvalidTransition' }
        "429": { $ref: '#/components/responses/AdmissionDenied' }

  /agents/{id}/hibernate:
    post:
      operationId: hibernateAgent
      summary: "Suspended → Hibernated (T6): checkpoint consistente + liberação de RAM."
      parameters:
        - $ref: '#/components/parameters/AgentIdPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      responses:
        "202": { description: "Hibernação aceita.", content: { application/json: { schema: { $ref: '#/components/schemas/AgentControlBlockView' } } } }
        "409": { $ref: '#/components/responses/InvalidTransition' }
        "500": { $ref: '#/components/responses/SerializationFailure' }
        "503": { $ref: '#/components/responses/StorageUnavailable' }

  /agents/{id}/wake:
    post:
      operationId: wakeAgent
      summary: "Hibernated → Ready/Running (T7): materialização sob demanda (p99 ≤ 250 ms)."
      parameters:
        - $ref: '#/components/parameters/AgentIdPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                targetState: { type: string, enum: [Ready, Running], default: Running }
      responses:
        "202": { description: "Materialização iniciada/concluída.", content: { application/json: { schema: { $ref: '#/components/schemas/AgentControlBlockView' } } } }
        "404": { $ref: '#/components/responses/CheckpointNotFound' }
        "409": { $ref: '#/components/responses/InvalidTransition' }
        "422": { $ref: '#/components/responses/CheckpointCorrupted' }
        "429": { $ref: '#/components/responses/AdmissionDenied' }
        "504": { $ref: '#/components/responses/MaterializationTimeout' }

  /agents/{id}/migrate:
    post:
      operationId: migrateAgent
      summary: "Running/Suspended → Migrating → destino (T8/T9), saga de migração."
      parameters:
        - $ref: '#/components/parameters/AgentIdPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                targetShard: { type: [integer, "null"] }
                targetNode: { type: [string, "null"] }
                reason: { type: string, enum: [rebalance, node_draining, manual], default: manual }
      responses:
        "202": { description: "Migração aceita; `MigrationJob` criado.", content: { application/json: { schema: { $ref: '#/components/schemas/MigrationJobView' } } } }
        "409": { $ref: '#/components/responses/MigrationInProgress' }
        "502": { $ref: '#/components/responses/TargetUnhealthy' }

  /agents/{id}/checkpoint:
    post:
      operationId: checkpointAgent
      summary: "Cria checkpoint sob demanda (ACB + working memory)."
      parameters:
        - $ref: '#/components/parameters/AgentIdPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      responses:
        "201": { description: "Checkpoint criado.", content: { application/json: { schema: { $ref: '#/components/schemas/CheckpointView' } } } }
        "500": { $ref: '#/components/responses/SerializationFailure' }
        "503": { $ref: '#/components/responses/StorageUnavailable' }

  /agents/{id}/restore:
    post:
      operationId: restoreAgent
      summary: "Restaura ACB + working memory a partir de um checkpoint específico."
      parameters:
        - $ref: '#/components/parameters/AgentIdPath'
        - $ref: '#/components/parameters/TenantHeader'
        - $ref: '#/components/parameters/IdempotencyKeyHeader'
          required: true
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [checkpointId]
              properties:
                checkpointId: { type: string }
      responses:
        "202": { description: "Restore concluído; `generation` incrementado.", content: { application/json: { schema: { $ref: '#/components/schemas/AgentControlBlockView' } } } }
        "404": { $ref: '#/components/responses/CheckpointNotFound' }
        "422": { $ref: '#/components/responses/CheckpointCorrupted' }

  /agents/{id}/checkpoints:
    get:
      operationId: listCheckpoints
      summary: "Lista checkpoints do agente, mais recente primeiro."
      parameters:
        - $ref: '#/components/parameters/AgentIdPath'
        - $ref: '#/components/parameters/TenantHeader'
        - name: pageToken
          in: query
          schema: { type: string }
        - name: pageSize
          in: query
          schema: { type: integer, default: 50, maximum: 200 }
      responses:
        "200":
          description: "Checkpoints do agente."
          content:
            application/json:
              schema:
                type: object
                properties:
                  checkpoints:
                    type: array
                    items: { $ref: '#/components/schemas/CheckpointView' }
                  nextPageToken: { type: [string, "null"] }

components:
  parameters:
    AgentIdPath:
      name: id
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
        priorityClass: { type: string, enum: [system, interactive, batch, best_effort], default: interactive }
        policyRef: { type: string }
        quotaRef: { type: string }
        labels: { type: object }
    AgentControlBlockView:
      type: object
      description: "Projeção pública do ACB (§3.1 do brief)."
      properties:
        agentId: { type: string }
        tenantId: { type: string }
        state:
          type: string
          enum: [Created, Ready, Running, Suspended, Hibernated, Migrating, Terminated, Failed]
        generation: { type: integer }
        desiredState: { type: string }
        priorityClass: { type: string }
        placementShard: { type: integer }
        placementNode: { type: [string, "null"] }
        runtimeInstanceId: { type: [string, "null"] }
        lastCheckpointId: { type: [string, "null"] }
        coldSince: { type: [string, "null"], format: date-time }
        wakeCount: { type: integer }
        createdAt: { type: string, format: date-time }
        updatedAt: { type: string, format: date-time }
        terminatedAt: { type: [string, "null"], format: date-time }
        failureCode: { type: [string, "null"] }
    LifecycleTransitionView:
      type: object
      properties:
        transitionId: { type: string }
        seq: { type: integer }
        fromState: { type: string }
        toState: { type: string }
        trigger: { type: string }
        actor: { type: string }
        reason: { type: [string, "null"] }
        generation: { type: integer }
        createdAt: { type: string, format: date-time }
    CheckpointView:
      type: object
      properties:
        checkpointId: { type: string }
        agentId: { type: string }
        generation: { type: integer }
        sizeBytes: { type: integer }
        contentHash: { type: string }
        codec: { type: string, enum: ["json+zstd", "msgpack+zstd"] }
        consistency: { type: string, enum: [consistent, crash] }
        createdAt: { type: string, format: date-time }
        expiresAt: { type: [string, "null"], format: date-time }
    MigrationJobView:
      type: object
      properties:
        migrationId: { type: string }
        agentId: { type: string }
        sourceShard: { type: integer }
        targetShard: { type: integer }
        phase: { type: string, enum: [quiesce, checkpoint, transfer, materialize, cutover, cleanup] }
        status: { type: string, enum: [pending, running, succeeded, failed, compensated] }
        startedAt: { type: string, format: date-time }
    ProblemDetails:
      type: object
      description: "RFC 7807 + extensões AIOS (RFC-0001 §5.4)."
      properties:
        type: { type: string, format: uri }
        title: { type: string }
        status: { type: integer }
        code: { type: string, pattern: '^AIOS-LIFECYCLE-[0-9]{4}$' }
        detail: { type: string }
        instance: { type: string }
        traceId: { type: string }
        timestamp: { type: string, format: date-time }
        retriable: { type: boolean }
        retryAfterMs: { type: integer }

  responses:
    BadRequest:
      description: "Payload/schema inválido."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    NotFound:
      description: "Agente não encontrado (`AIOS-LIFECYCLE-0001`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    InvalidTransition:
      description: "Transição inválida para o estado atual (`AIOS-LIFECYCLE-0002`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    AlreadyTerminal:
      description: "Agente em estado terminal (`AIOS-LIFECYCLE-0004`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    AdmissionDenied:
      description: "Admissão negada pelo Scheduler (`AIOS-LIFECYCLE-0005`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    MaterializationTimeout:
      description: "Timeout de materialização (`AIOS-LIFECYCLE-0006`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    CheckpointCorrupted:
      description: "Checkpoint corrompido, hash divergente (`AIOS-LIFECYCLE-0007`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    CheckpointNotFound:
      description: "Checkpoint inexistente (`AIOS-LIFECYCLE-0008`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    MigrationInProgress:
      description: "Migração já em andamento (`AIOS-LIFECYCLE-0009`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    TargetUnhealthy:
      description: "Destino de migração indisponível/insalubre (`AIOS-LIFECYCLE-0010`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    SerializationFailure:
      description: "Falha de serialização/deserialização do snapshot (`AIOS-LIFECYCLE-0011`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    PolicyDenied:
      description: "Negado pelo PDP (`AIOS-LIFECYCLE-0012`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    StorageUnavailable:
      description: "Storage de snapshot indisponível (`AIOS-LIFECYCLE-0014`)."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
    IdempotencyConflict:
      description: "`Idempotency-Key` reusada com payload divergente."
      content: { application/problem+json: { schema: { $ref: '#/components/schemas/ProblemDetails' } } }
```

---

## 4. Contrato gRPC (trecho proto3 — `aios.lifecycle.v1`)

```protobuf
syntax = "proto3";

package aios.lifecycle.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/struct.proto";
import "google/rpc/status.proto";

option csharp_namespace = "Aios.Lifecycle.V1";

// ── Serviço principal ───────────────────────────────────────────────────────
service LifecycleService {
  rpc Spawn(SpawnRequest)          returns (LifecycleAck);
  rpc Suspend(AgentRef)            returns (LifecycleAck);
  rpc Resume(AgentRef)             returns (LifecycleAck);
  rpc Hibernate(AgentRef)          returns (LifecycleAck);
  rpc Wake(WakeRequest)            returns (LifecycleAck);   // materialize
  rpc Migrate(MigrateRequest)      returns (LifecycleAck);
  rpc Checkpoint(AgentRef)         returns (CheckpointRef);
  rpc Restore(RestoreRequest)      returns (LifecycleAck);
  rpc Terminate(TerminateRequest)  returns (LifecycleAck);
  rpc GetState(AgentRef)           returns (AgentControlBlock);
  rpc StreamTransitions(AgentRef)  returns (stream LifecycleTransition);
}

// ── Envelope de correlação comum ────────────────────────────────────────────
message RequestMeta {
  string tenant_id       = 1;  // == X-AIOS-Tenant; DEVE bater com claim
  string traceparent     = 2;  // W3C Trace Context
  string idempotency_key = 3;  // obrigatório em mutações
  string caller_urn      = 4;  // X-AIOS-Agent ou identidade de serviço chamador
}

enum LifecycleState {
  LIFECYCLE_STATE_UNSPECIFIED = 0;
  CREATED    = 1;
  READY      = 2;
  RUNNING    = 3;
  SUSPENDED  = 4;
  HIBERNATED = 5;
  MIGRATING  = 6;
  TERMINATED = 7;
  FAILED     = 8;
}

enum PriorityClass {
  PRIORITY_CLASS_UNSPECIFIED = 0;
  SYSTEM      = 1;
  INTERACTIVE = 2;
  BATCH       = 3;
  BEST_EFFORT = 4;
}

message AgentRef {
  RequestMeta meta   = 1;
  string agent_urn   = 2;
}

message AgentControlBlock {
  string agent_urn                = 1;
  string tenant_id                = 2;
  LifecycleState state            = 3;
  int64 generation                = 4;
  LifecycleState desired_state    = 5;
  PriorityClass priority_class    = 6;
  int32 placement_shard           = 7;
  string placement_node           = 8;  // vazio se não quente
  string runtime_instance_id      = 9;  // vazio se cold
  string last_checkpoint_id       = 10; // vazio se nunca houve checkpoint
  google.protobuf.Timestamp cold_since  = 11;
  int32 wake_count                = 12;
  google.protobuf.Timestamp created_at  = 13;
  google.protobuf.Timestamp updated_at  = 14;
  google.protobuf.Timestamp terminated_at = 15;
  string failure_code              = 16; // vazio se nenhuma falha
}

message LifecycleAck {
  AgentControlBlock acb = 1;
}

// ── spawn ────────────────────────────────────────────────────────────────
message SpawnRequest {
  RequestMeta meta            = 1;
  string parent_urn           = 2;
  PriorityClass priority_class = 3;
  string policy_ref           = 4;
  string quota_ref            = 5;
  google.protobuf.Struct labels = 6;
}

// ── wake ─────────────────────────────────────────────────────────────────
message WakeRequest {
  RequestMeta meta     = 1;
  string agent_urn     = 2;
  LifecycleState target_state = 3; // READY ou RUNNING
}

// ── migrate ──────────────────────────────────────────────────────────────
message MigrateRequest {
  RequestMeta meta       = 1;
  string agent_urn       = 2;
  int32 target_shard     = 3; // 0 = sem preferência (Scheduler decide)
  string target_node     = 4; // vazio = sem preferência
  string reason          = 5; // rebalance|node_draining|manual
}

// ── restore ──────────────────────────────────────────────────────────────
message RestoreRequest {
  RequestMeta meta       = 1;
  string agent_urn       = 2;
  string checkpoint_id   = 3;
}

// ── terminate ────────────────────────────────────────────────────────────
message TerminateRequest {
  RequestMeta meta        = 1;
  string agent_urn        = 2;
  string reason           = 3;
  bool final_checkpoint   = 4;
}

// ── checkpoint ───────────────────────────────────────────────────────────
message CheckpointRef {
  string checkpoint_id     = 1;
  string agent_urn         = 2;
  int64 generation         = 3;
  int64 size_bytes         = 4;
  string content_hash      = 5;
  string consistency       = 6; // consistent|crash
  google.protobuf.Timestamp created_at = 7;
}

// ── histórico de transições ──────────────────────────────────────────────
message LifecycleTransition {
  string transition_id    = 1;
  string agent_urn        = 2;
  string tenant_id        = 3;
  int64 seq                = 4;
  LifecycleState from_state = 5;
  LifecycleState to_state   = 6;
  string trigger           = 7; // spawn|ready|start|suspend|resume|hibernate|wake|migrate|terminate|fail|reconcile
  string actor             = 8; // api:<sub>|scheduler|reconciler|runtime
  string reason            = 9;
  int64 generation          = 10;
  google.protobuf.Timestamp created_at = 11;
}
```

Mapeamento de erro gRPC: todo `rpc` retorna `google.rpc.Status` com `code`
gRPC padrão e `google.rpc.ErrorInfo{ reason: "AIOS-LIFECYCLE-<NNNN>", domain:
"aios.lifecycle" }` — o mesmo `code` do envelope REST (RFC-0001 §5.4),
garantindo paridade de semântica entre os dois transportes.

| gRPC `code` | HTTP equivalente | Exemplo `AIOS-LIFECYCLE-<NNNN>` |
|-------------|-------------------|------------------------------------|
| `NOT_FOUND` | 404 | `-0001`, `-0008` |
| `FAILED_PRECONDITION` | 409/423 | `-0002`, `-0004`, `-0009`, `-0013` |
| `ABORTED` | 409 | `-0003` |
| `RESOURCE_EXHAUSTED` | 429 | `-0005` |
| `DEADLINE_EXCEEDED` | 504 | `-0006` |
| `INVALID_ARGUMENT` | 422/400 | `-0007` |
| `PERMISSION_DENIED` | 403 | `-0012` |
| `UNAVAILABLE` | 502/503 | `-0010`, `-0014` |
| `INTERNAL` | 500 | `-0011`, `-0015` |

---

## 5. Idempotência (aplicação ao módulo 008)

- Toda mutação (`spawn`, `suspend`, `resume`, `hibernate`, `wake`, `migrate`,
  `checkpoint`, `restore`, `terminate`) DEVE receber `Idempotency-Key`
  (RFC-0001 §5.5, NFR-013).
- `LifecycleApiSurface` persiste `(tenant_id, idempotency_key) → resultado`
  por, no mínimo, 24h (default 48h, ver `Configuration.md`).
- Repetição com **mesma chave e mesmo payload** (hash do corpo) DEVE
  retornar o **mesmo status HTTP e corpo** da primeira execução, sem
  reexecutar efeitos colaterais — nenhuma nova transição na FSM, nenhum
  novo checkpoint, nenhum evento duplicado.
- Repetição com **mesma chave e payload divergente** DEVE ser rejeitada com
  `409 Conflict` (mapeado ao domínio `AIOS-LIFECYCLE`, código de conflito de
  idempotência específico do gateway compartilhado — ver registro central
  em `../004-API/Errors.md`).
- Leituras (`GetState`, `StreamTransitions`, listagem de checkpoints) são
  naturalmente idempotentes e **NÃO exigem** `Idempotency-Key`.
- A combinação de `Idempotency-Key` (efeito único por requisição) com
  `generation`/`fencing_token` (§4.3 do brief) garante que mesmo sob
  reentrega de mensagens e falha do coordenador, nenhum efeito colateral é
  duplicado (INV2 do brief: transição + outbox na mesma transação).

### 5.1 Exemplo — repetição idempotente de `hibernate`

```
# 1ª chamada
POST /v1/agents/urn%3Aaios%3Aacme%3Aagent%3A01J9Z8Q6H7K2M4N6P8R0S2T4V6/hibernate
Idempotency-Key: 01J9ZH000000000000000001
X-AIOS-Tenant: acme

→ 202 Accepted
{ "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "state": "Hibernated", "generation": 3, "lastCheckpointId": "01J9ZH02...", ... }

# 2ª chamada (retry de rede, mesma chave)
POST /v1/agents/urn%3Aaios%3Aacme%3Aagent%3A01J9Z8Q6H7K2M4N6P8R0S2T4V6/hibernate
Idempotency-Key: 01J9ZH000000000000000001
X-AIOS-Tenant: acme

→ 202 Accepted   (mesmo corpo exato; nenhum novo checkpoint criado, nenhum novo evento emitido)
```

---

## 6. Catálogo de códigos de erro (`AIOS-LIFECYCLE-<NNNN>`, faixa 0001–0099)

| Código | HTTP | gRPC | `retriable` | Significado | Ação recomendada do cliente |
|--------|------|------|-------------|--------------|------------------------------|
| `AIOS-LIFECYCLE-0001` | 404 | `NOT_FOUND` | não | Agente não encontrado (`agent_id`/`tenant_id`). | Verificar URN; não repetir. |
| `AIOS-LIFECYCLE-0002` | 409 | `FAILED_PRECONDITION` | não | Transição inválida para o estado atual (viola FSM §4). | Consultar `GetState`, reavaliar fluxo. |
| `AIOS-LIFECYCLE-0003` | 409 | `ABORTED` | sim | Lease não adquirido (outra transição em curso). | Retry com backoff exponencial + jitter. |
| `AIOS-LIFECYCLE-0004` | 423 | `FAILED_PRECONDITION` | não | Agente em estado terminal (`Terminated`/`Failed`). | Não repetir; consultar histórico. |
| `AIOS-LIFECYCLE-0005` | 429 | `RESOURCE_EXHAUSTED` | sim | Admissão negada pelo Scheduler (cota/capacidade). | Respeitar `Retry-After`; aplicar backoff. |
| `AIOS-LIFECYCLE-0006` | 504 | `DEADLINE_EXCEEDED` | sim | Timeout de materialização (spawn/wake > alvo). | Retry; verificar se agente já ficou `Failed`. |
| `AIOS-LIFECYCLE-0007` | 422 | `INVALID_ARGUMENT` | não | Checkpoint corrompido (hash divergente). | Tentar checkpoint anterior; não repetir a mesma chave. |
| `AIOS-LIFECYCLE-0008` | 404 | `NOT_FOUND` | não | Checkpoint inexistente para restore. | Consultar `GET .../checkpoints`. |
| `AIOS-LIFECYCLE-0009` | 409 | `FAILED_PRECONDITION` | sim | Migração já em andamento para o agente. | Consultar status do `MigrationJob`; aguardar conclusão. |
| `AIOS-LIFECYCLE-0010` | 502 | `UNAVAILABLE` | sim | Destino de migração indisponível/insalubre. | Retry (novo destino escolhido pelo Scheduler/Cluster). |
| `AIOS-LIFECYCLE-0011` | 500 | `INTERNAL` | sim | Falha de serialização/deserialização do snapshot. | Retry com backoff; monitorar `CheckpointService`. |
| `AIOS-LIFECYCLE-0012` | 403 | `PERMISSION_DENIED` | não | Política negou a operação (PDP `deny`). | Solicitar concessão de permissão; não repetir. |
| `AIOS-LIFECYCLE-0013` | 409 | `FAILED_PRECONDITION` | não | `generation`/`fencing_token` obsoleto (escrita rejeitada). | Reler `GetState`, reaplicar com `generation` atual. |
| `AIOS-LIFECYCLE-0014` | 503 | `UNAVAILABLE` | sim | Storage de snapshot (MinIO/PG) indisponível. | Retry com backoff; monitorar `SnapshotStore`. |
| `AIOS-LIFECYCLE-0015` | 500 | `INTERNAL` | não | Runtime perdido e irrecuperável (→ `Failed`). | Não repetir; agente precisa de novo `spawn`. |

---

## 7. Exemplos completos de request/response

### 7.1 `wake` (feliz — materialização de cold agent)

```
POST /v1/agents/urn%3Aaios%3Aacme%3Aagent%3A01J9Z8Q6H7K2M4N6P8R0S2T4V6/wake
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZH000000000000000010
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
{ "targetState": "Running" }

→ 202 Accepted
{
  "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "tenantId": "acme",
  "state": "Running",
  "generation": 4,
  "wakeCount": 3,
  "placementShard": 17,
  "placementNode": "runtime-node-08",
  "runtimeInstanceId": "urn:aios:acme:runtime:01J9ZH01",
  "updatedAt": "2026-07-20T12:15:00.180Z"
}
```

### 7.2 `wake` (falha — checkpoint corrompido)

```
POST /v1/agents/urn%3Aaios%3Aacme%3Aagent%3A01J9Z8Q6H7K2M4N6P8R0S2T4V6/wake
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZH000000000000000011

→ 422 Unprocessable Entity
{
  "type": "https://docs.aios/errors/checkpoint-corrupted",
  "title": "Checkpoint Corrupted",
  "status": 422,
  "code": "AIOS-LIFECYCLE-0007",
  "detail": "content_hash mismatch for checkpoint '01J9ZH02...': integrity check failed.",
  "instance": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:15:01.000Z",
  "retriable": false
}
```

### 7.3 `migrate` (falha — destino insalubre)

```
POST /v1/agents/urn%3Aaios%3Aacme%3Aagent%3A01J9Z8Q6H7K2M4N6P8R0S2T4V6/migrate
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZH000000000000000012
{ "targetNode": "runtime-node-99", "reason": "manual" }

→ 502 Bad Gateway
{
  "type": "https://docs.aios/errors/migration-target-unhealthy",
  "title": "Migration Target Unhealthy",
  "status": 502,
  "code": "AIOS-LIFECYCLE-0010",
  "detail": "Target node 'runtime-node-99' failed health check during placement negotiation.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:16:00.000Z",
  "retriable": true,
  "retryAfterMs": 3000
}
```

### 7.4 `suspend` (falha — lease em disputa)

```
POST /v1/agents/urn%3Aaios%3Aacme%3Aagent%3A01J9Z8Q6H7K2M4N6P8R0S2T4V6/suspend
X-AIOS-Tenant: acme
Idempotency-Key: 01J9ZH000000000000000013

→ 409 Conflict
{
  "type": "https://docs.aios/errors/lease-not-acquired",
  "title": "Lease Not Acquired",
  "status": 409,
  "code": "AIOS-LIFECYCLE-0003",
  "detail": "Another transition is currently in progress for this agent (lease held by coordinator 'lc-07').",
  "instance": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:16:30.000Z",
  "retriable": true,
  "retryAfterMs": 200
}
```

### 7.5 `GetState` via gRPC (leitura)

```
grpcurl -H 'x-aios-tenant: acme' -d '{
  "meta": { "tenant_id": "acme", "traceparent": "00-4bf92f...-00f067aa0ba902b7-01" },
  "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6"
}' api.aios.internal:443 aios.lifecycle.v1.LifecycleService/GetState

→ {
  "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "tenantId": "acme",
  "state": "SUSPENDED",
  "generation": "3",
  "priorityClass": "INTERACTIVE",
  "placementShard": 17,
  "wakeCount": 2
}
```

---

## 8. Versionamento e compatibilidade

- REST: `/v1/agents/...`; mudanças incompatíveis DEVEM introduzir
  `/v2/agents/...` coexistindo por ≥ 2 majors (RFC-0001 §5.7, V-R9).
- gRPC: pacote `aios.lifecycle.v1`; nova versão incompatível cria
  `aios.lifecycle.v2` como novo serviço, mantendo `v1` em produção durante a
  janela de migração.
- Adições de campos opcionais (REST/proto) são **aditivas** e NÃO exigem
  major bump; clientes DEVEM ignorar campos desconhecidos.
- Novos estados/transições da FSM (§4 do brief) exigem revisão de
  ADR-0080 antes de qualquer alteração de contrato — a FSM é imutável sem
  nova ADR (o brief a trata como contrato canônico).

---

## 9. Referências

- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7.
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §5.
- Catálogo de eventos emitidos por estas operações: `./Events.md`.
- Modelo de dados subjacente (ACB, LifecycleTransition, Checkpoint, MigrationJob): `./Database.md`.
- Chaves de configuração citadas (`spawn.timeout_ms`, `lease.ttl_ms`, `migration.saga_timeout_ms`): `./Configuration.md`.
- Fluxos de sequência detalhados (feliz/falha): `./SequenceDiagrams.md`.
- Máquina de estados referenciada pelas operações: `./StateMachine.md`.
- Diagramas de componentes que implementam esta API: `./Architecture.md`.
