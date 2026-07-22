---
Documento: API
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0072
RFCs relacionados: RFC-0001, RFC-0070
Depende de: ../001-Architecture/, ../003-RFC/RFC-0001, ../008-Agent-Lifecycle/, ../021-Security/, ../022-Policy/
---

# 007-Agent-Runtime — API

> **Escopo.** O Agent Runtime é um serviço **interno** do plano de dados,
> comandado pelo Runtime Supervisor e por serviços do control plane — nunca
> exposto ao usuário final (§N-08, `./Vision.md` §3.2). Sua superfície
> **gRPC** (`aios.runtime.v1`) é o contrato primário; a superfície **REST** é
> restrita a *health*/introspecção na rede de controle. Convenções de
> versionamento, autenticação, envelope de erro, idempotência e cabeçalhos
> de correlação seguem `../003-RFC/RFC-0001-Architecture-Baseline.md`
> §5.4–§5.7 e **não são redefinidas aqui** — apenas instanciadas para o
> domínio `RUNTIME`. Consistência absoluta com `_DESIGN_BRIEF.md` §5.

## Índice

1. Convenções gerais da API do Agent Runtime
2. Pacote gRPC `aios.runtime.v1` (contrato `.proto`)
3. Superfície REST (rede de controle)
4. Catálogo de códigos de erro (domínio `RUNTIME`)
5. Idempotência
6. Exemplos de request/response
7. Referências

---

## 1. Convenções Gerais da API do Agent Runtime

| Aspecto | Regra |
|---------|-------|
| Pacote gRPC | `aios.runtime.v1` (`RuntimeControl`, `AgentExecution`). |
| Base REST | `/v1/runtime` (rede de controle apenas; nunca via API Gateway externo). |
| Autenticação | mTLS interno obrigatório entre Runtime Supervisor/control plane e a instância de runtime (`021-Security`). Nenhum OAuth2/OIDC de usuário final — este módulo não é alcançado por chamador humano direto. |
| Tenant | Cabeçalho `X-AIOS-Tenant`/claim de serviço OBRIGATÓRIO; DEVE bater com o `tenant_id` do recurso-alvo, senão erro de isolamento (ver `./UseCases.md` UC-017). |
| Correlação | `traceparent`/`tracestate` (W3C Trace Context) OBRIGATÓRIOS em toda chamada (RFC-0001 §5.6). |
| Idempotência | `Boot` e `SubmitTask` DEVEM aceitar `Idempotency-Key`; resultado persistido ≥ 24h. Ver §5. |
| Versão de API | Pacote gRPC versionado (`aios.runtime.v1`); REST versionado por caminho (`/v1/...`). Coexistência ≥ 2 majors (NFR-014). |
| Content-Type | Protocol Buffers v3 (gRPC); `application/json` (REST). |
| Erros | Envelope RFC 7807 (REST) / `google.rpc.Status` + `ErrorInfo` (gRPC) com `code` `AIOS-RUNTIME-<NNNN>` (RFC-0001 §5.4). Ver §4. |

---

## 2. Pacote gRPC `aios.runtime.v1` (Contrato `.proto`)

### 2.1 `RuntimeControl` — Supervisor → Runtime

```protobuf
syntax = "proto3";
package aios.runtime.v1;

service RuntimeControl {
  rpc Boot(BootRequest) returns (BootReply);
  rpc Suspend(SuspendRequest) returns (SuspendReply);
  rpc Resume(ResumeRequest) returns (ResumeReply);
  rpc Kill(KillRequest) returns (KillReply);
  rpc Drain(DrainRequest) returns (DrainReply);
  rpc GetStatus(StatusRequest) returns (RuntimeStatus);
}

message BootRequest {
  string idempotency_key = 1;   // RFC-0001 §5.5, obrigatório
  string tenant_id       = 2;
  string agent_urn       = 3;
  bytes  agent_spec      = 4;   // AgentSpec serializado (JSON/Proto — ver 008-Agent-Lifecycle)
  string traceparent     = 5;
}

message BootReply {
  string runtime_id = 1;
  string session_id = 2;
  string state       = 3;   // AgentExecState — "ready" no caminho feliz
  int32  cold_start_ms = 4;
}

message SuspendRequest {
  string session_id = 1;
  string reason      = 2;   // "voluntary" | "preemption" | "quota_exceeded"
}
message SuspendReply {
  string checkpoint_id = 1;
  int32  step_cursor    = 2;
}

message ResumeRequest {
  string checkpoint_id = 1;
}
message ResumeReply {
  string session_id = 1;
  string state       = 2;
}

message KillRequest  { string session_id = 1; string reason = 2; }
message KillReply    { string terminal_state = 1; }

message DrainRequest { string runtime_id = 1; }
message DrainReply   { bool accepted = 1; }

message StatusRequest { string runtime_id = 1; }
message RuntimeStatus {
  string runtime_id       = 1;
  string state             = 2;  // RuntimeState
  string session_id         = 3;  // vazio se idle
  string agent_exec_state    = 4;  // AgentExecState, se aplicável
  ResourceUsage consumed      = 5;
  ResourceBudget budget         = 6;
  int64  last_heartbeat_unix_ms  = 7;
}

message ResourceUsage {
  int64   tokens_used         = 1;
  double  cost_usd_spent        = 2;
  int64   wall_clock_elapsed_ms   = 3;
  int32   steps_executed            = 4;
}
message ResourceBudget {
  int64  max_tokens             = 1;
  double max_cost_usd             = 2;
  int64  wall_clock_budget_ms       = 3;
  int32  max_steps                    = 4;
}
```

### 2.2 `AgentExecution` — Control Plane → Runtime

```protobuf
service AgentExecution {
  rpc SubmitTask(TaskRequest) returns (stream ExecutionEvent);
  rpc CancelTask(CancelRequest) returns (CancelReply);
  rpc StreamSteps(SessionRef) returns (stream ReActStepEvent);
}

message TaskRequest {
  string idempotency_key = 1;
  string tenant_id       = 2;
  string task_urn        = 3;
  string agent_urn       = 4;
  bytes  goal_payload    = 5;   // objetivo/critério de sucesso (ver 012-Planning)
  string traceparent     = 6;
}

message ExecutionEvent {
  oneof event {
    ReActStepEvent step       = 1;
    TaskCompleted  completed  = 2;
    TaskFailed     failed     = 3;
  }
}

message ReActStepEvent {
  string step_id    = 1;
  string session_id  = 2;
  int32  index         = 3;
  string phase           = 4;  // perceive|think|act|observe|reflect
  string status             = 5;  // ok|error|timeout|denied
  int32  latency_ms           = 6;
}

message TaskCompleted {
  string task_urn  = 1;
  string status      = 2;  // SUCCEEDED
  double cost_usd      = 3;
  int64  tokens          = 4;
}

message TaskFailed {
  string task_urn   = 1;
  string code         = 2;  // AIOS-RUNTIME-NNNN
  string reason         = 3;
  bool   retriable         = 4;
}

message CancelRequest { string task_urn = 1; string reason = 2; }
message CancelReply   { bool accepted = 1; }

message SessionRef { string session_id = 1; }
```

---

## 3. Superfície REST (Rede de Controle)

> Restrita a *health*, *readiness* e introspecção; **nunca** exposta pelo
> API Gateway externo (YARP) nem alcançável por usuário final.

```yaml
openapi: 3.1.0
info:
  title: Agent Runtime — Control Plane Introspection API
  version: "1.0"
paths:
  /v1/runtime/healthz:
    get:
      summary: Liveness probe.
      responses:
        "200": { description: Processo vivo. }
  /v1/runtime/readyz:
    get:
      summary: Readiness probe (pool quente ok, MCP host carregado).
      responses:
        "200": { description: Pronto para receber Boot/SubmitTask. }
        "503": { description: Ainda não pronto (ex.: MCP host em carregamento). }
  /v1/runtime/{runtime_id}/status:
    get:
      summary: Status/cotas da instância de runtime.
      parameters:
        - { name: runtime_id, in: path, required: true, schema: { type: string } }
      responses:
        "200":
          description: RuntimeStatus serializado.
          content:
            application/json:
              schema: { $ref: "#/components/schemas/RuntimeStatus" }
  /v1/runtime/sessions/{session_id}:
    get:
      summary: Snapshot da AgentSession (respeitando RLS por tenant_id).
      parameters:
        - { name: session_id, in: path, required: true, schema: { type: string } }
        - { name: X-AIOS-Tenant, in: header, required: true, schema: { type: string } }
      responses:
        "200": { description: Snapshot da sessão. }
        "403": { description: "Tenant divergente — AIOS-RUNTIME-0003-equivalente de isolamento." }
        "404": { description: Sessão não encontrada para este runtime/tenant. }
components:
  schemas:
    RuntimeStatus:
      type: object
      properties:
        runtime_id: { type: string }
        state: { type: string, enum: [warming, idle, busy, draining, dead] }
        session_id: { type: string, nullable: true }
        consumed: { type: object }
        budget: { type: object }
        last_heartbeat_unix_ms: { type: integer, format: int64 }
```

---

## 4. Catálogo de Códigos de Erro (Domínio `RUNTIME`)

> Faixa reservada **0001–0099** (`_DESIGN_BRIEF.md` §5.4/§11). `detail`
> **NÃO DEVE** vazar PII (RFC-0001 §6). gRPC mapeia `status` →
> `google.rpc.Status` + `ErrorInfo` com o mesmo `code`.

| Código | HTTP (equiv.) | `retriable` | Significado | Onde é emitido |
|--------|----------------|--------------|--------------|------------------|
| `AIOS-RUNTIME-0001` | 503 | true | Pool sem slot quente disponível (cold start indisponível). | Runtime Supervisor, antes de alcançar este módulo. |
| `AIOS-RUNTIME-0002` | 500 | false | Falha de boot do sandbox (seccomp/cgroups/netns). | `RuntimeBootstrapper`/`SandboxManager`. |
| `AIOS-RUNTIME-0003` | 403 | false | *Capability* ausente/negada (PEP *default deny*). | `CapabilityEnforcer`. |
| `AIOS-RUNTIME-0004` | 429 | true | Cota local esgotada (tokens/USD/wall-clock/passos). | `QuotaGovernor`. |
| `AIOS-RUNTIME-0005` | 504 | true | Timeout de invocação de tool (via `015`). | `ToolInvoker`. |
| `AIOS-RUNTIME-0006` | 502 | true | Falha do Model Router (inferência). | `ModelRouterClient`. |
| `AIOS-RUNTIME-0007` | 409 | false | Violação de idempotência (chave já usada com payload divergente). | `RuntimeControl.Boot` / `AgentExecution.SubmitTask`. |
| `AIOS-RUNTIME-0008` | 500 | false | Loop travado/deadlock (*watchdog* disparado). | `HealthProbe`. |
| `AIOS-RUNTIME-0009` | 422 | false | `AgentSpec` inválido/incompatível. | `RuntimeBootstrapper`. |
| `AIOS-RUNTIME-0010` | 500 | true | Falha ao gerar/restaurar checkpoint (integridade). | `CheckpointManager`. |
| `AIOS-RUNTIME-0011` | 403 | false | Violação de política de rede (egress fora da allowlist). | `SandboxManager`. |
| `AIOS-RUNTIME-0012` | 409 | false | Máximo de passos/profundidade de recursão do loop excedido. | `QuotaGovernor`/`CognitiveLoopEngine`. |
| `AIOS-RUNTIME-0013` | 503 | true | Runtime em `draining`; não aceita novas tarefas. | `RuntimeControl.Drain` / `AgentExecution.SubmitTask`. |
| `AIOS-RUNTIME-0014` | 400 | false | Replanejamento impossível (sem plano viável de `012`). | `PlanningClient`/`CognitiveLoopEngine`. |
| `AIOS-RUNTIME-0015` | 500 | false | Violação de sandbox detectada (syscall proibida) — sessão morta. | `SandboxManager` (monitor `seccomp`). |

---

## 5. Idempotência

- `Boot`: `idempotency_key` no escopo `(tenant_id, agent_urn)`; repetição
  com o mesmo payload retorna o mesmo `BootReply` (mesmo `runtime_id`/
  `session_id`); payload divergente → `AIOS-RUNTIME-0007`.
- `SubmitTask`: `idempotency_key` no escopo `(tenant_id, task_urn)`;
  comportamento idêntico. Ver `./SequenceDiagrams.md` §10 e `./UseCases.md`
  UC-014 para o fluxo completo.
- `Suspend`/`Resume`/`Kill`/`Drain`: idempotentes por `session_id` (não
  exigem `Idempotency-Key` explícita — o próprio identificador do recurso
  garante efeito único: `Kill` repetido em sessão já `terminated` é um
  no-op que retorna o `terminal_state` atual).
- Toda chave é retida por, no mínimo, 24h (RFC-0001 §5.5), com a mesma
  janela mínima usada em todo o AIOS.

---

## 6. Exemplos de Request/Response

### 6.1 `Boot` — sucesso (pool quente)

```json
// BootRequest
{
  "idempotency_key": "01J9Z9AABBCCDDEEFFGGHHJJKK",
  "tenant_id": "acme",
  "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "agent_spec": "<bytes: AgentSpec>",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
}
```

```json
// BootReply
{
  "runtime_id": "01J9ZA1RVNTMEXEC0000000001",
  "session_id": "01J9ZA1AGENTSESSN000000001",
  "state": "ready",
  "cold_start_ms": 187
}
```

### 6.2 `SubmitTask` — erro de cota (fluxo de falha)

```json
// TaskFailed (dentro do stream ExecutionEvent)
{
  "failed": {
    "task_urn": "urn:aios:acme:task:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
    "code": "AIOS-RUNTIME-0004",
    "reason": "Token budget exceeded (budget.max_tokens=200000, consumed=201455).",
    "retriable": true
  }
}
```

### 6.3 Erro REST equivalente (RFC 7807)

```json
{
  "type": "https://docs.aios/errors/runtime-quota-exceeded",
  "title": "Runtime Quota Exceeded",
  "status": 429,
  "code": "AIOS-RUNTIME-0004",
  "detail": "Local token budget exceeded for session.",
  "instance": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-21T12:34:56.789Z",
  "retriable": true,
  "retryAfterMs": 0
}
```

---

## 7. Referências

- Superfície completa (métodos, escopo, idempotência): `./_DESIGN_BRIEF.md` §5.
- Fluxos de chamada em sequência: `./SequenceDiagrams.md`.
- Interfaces internas equivalentes: `./ClassDiagrams.md` §3.
- Catálogo de eventos publicados/consumidos: `./Events.md`.
- Contratos centrais (envelope de erro, idempotência, correlação): `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.6.
- Protocolo formal a propor: RFC-0070 (`./RFC.md`).
