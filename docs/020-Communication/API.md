---
Documento: API
Módulo: 020-Communication
Status: Draft
Versão: 0.1
Última atualização: 2026-07-22
Responsável (RACI-A): Arquiteto do Módulo 020-Communication
ADRs relacionados: ADR-0200, ADR-0201, ADR-0203, ADR-0205, ADR-0207, ADR-0208, ADR-0209
RFCs relacionados: RFC-0001, RFC-0200, RFC-0201
Depende de: _DESIGN_BRIEF.md §5, 004-API (registro de erros), 021-Security, 022-Policy
---

# 020-Communication — Contratos de API

> **Esta é uma API de controle.** O caminho de mensagem **não passa** por ela:
> produtores e consumidores falam **NATS diretamente**, sob as contas, cotas e
> validações que este módulo governa (`./Architecture.md` §1). O protocolo de
> mensagem é o NATS + o envelope CloudEvents da RFC-0001 §5.2.

Autenticação, envelope de erro, idempotência, correlação e versionamento seguem
`../003-RFC/RFC-0001-Architecture-Baseline.md` §5.4–§5.7 e **não são redefinidos aqui**.

- Base REST: `/v1/comm` (exposta via `../004-API/`).
- Pacote gRPC: `aios.comm.v1`.
- Toda mutação **DEVE** enviar `Idempotency-Key` e os cabeçalhos de correlação.
- Operações de plataforma usam o tenant reservado `_platform`.

---

## 1. Superfície de Operações

| Operação | REST | gRPC (rpc) | Idempotente | Capability (PDP) |
|----------|------|-----------|-------------|------------------|
| Registrar subject | `POST /v1/comm/subjects` | `RegisterSubject` | sim (key) | `bus:subject:write` |
| Listar/ler subjects | `GET /v1/comm/subjects[/{urn}]` | `ListSubjects`/`GetSubject` | sim (leitura) | `bus:subject:read` |
| Depreciar subject | `POST /v1/comm/subjects/{urn}:deprecate` | `DeprecateSubject` | sim | `bus:subject:write` |
| Declarar stream | `PUT /v1/comm/streams/{name}` | `PutStream` | sim | `bus:stream:write` |
| Listar streams | `GET /v1/comm/streams` | `ListStreams` | sim (leitura) | `bus:stream:read` |
| Drenar stream | `POST /v1/comm/streams/{name}:drain` | `DrainStream` | sim | `bus:stream:write` |
| Declarar consumidor | `PUT /v1/comm/streams/{name}/consumers/{durable}` | `PutConsumer` | sim | `bus:consumer:write` |
| Estado de consumidor | `GET /v1/comm/streams/{name}/consumers/{durable}` | `GetConsumer` | sim (leitura) | `bus:consumer:read` |
| Listar DLQ | `GET /v1/comm/dlq` | `ListDeadLetters` | sim (leitura) | `bus:dlq:read` |
| Replay de DLQ | `POST /v1/comm/dlq/{id}:replay` | `ReplayDeadLetter` | sim (key) | `bus:dlq:replay` |
| Descartar da DLQ | `POST /v1/comm/dlq/{id}:discard` | `DiscardDeadLetter` | sim | `bus:dlq:discard` |
| Replay de stream | `POST /v1/comm/streams/{name}:replay` | `ReplayStream` | sim (key) | `bus:stream:replay` |
| Abrir sessão A2A | `POST /v1/comm/a2a/sessions` | `OpenSession` | sim (key) | `a2a:session:open` |
| Ler sessão A2A | `GET /v1/comm/a2a/sessions/{urn}` | `GetSession` | sim (leitura) | `a2a:session:read` |
| Encerrar sessão A2A | `POST /v1/comm/a2a/sessions/{urn}:close` | `CloseSession` | sim | `a2a:session:close` |
| Criar canal de grupo | `PUT /v1/comm/groups/{group}/channel` | `PutGroupChannel` | sim | `bus:group:write` |
| Provisionar conta | `POST /v1/comm/accounts` | `ProvisionAccount` | sim (key) | `bus:account:provision` |
| Exportar subject entre contas | `POST /v1/comm/accounts/{tenant}/exports` | `PutAccountExport` | sim | `bus:account:export` (**alta sensibilidade**) |
| Status do barramento | `GET /v1/comm/health` | `GetBusHealth` | sim (leitura) | `bus:health:read` |

---

## 2. OpenAPI (extrato normativo)

```yaml
openapi: 3.1.0
info:
  title: AIOS Communication Bus API
  version: "1.0.0"
servers:
  - url: https://api.aios.local/v1/comm
components:
  parameters:
    Traceparent:    { name: traceparent,     in: header, required: true, schema: { type: string } }
    Tenant:         { name: X-AIOS-Tenant,   in: header, required: true, schema: { type: string } }
    IdempotencyKey: { name: Idempotency-Key, in: header, required: true, schema: { type: string } }
  schemas:
    SubjectDefinition:
      type: object
      required: [domain, entity, action, producerModule, dataschema, streamRef, trafficClass]
      properties:
        urn:            { type: string, example: "urn:aios:_platform:subjectdef:01J9ZC5R8T0V2X4Z6B8D0F2H4K" }
        domain:         { type: string, example: "agent", pattern: "^[a-z][a-z0-9]*$" }
        entity:         { type: string, example: "lifecycle", pattern: "^[a-z][a-z0-9]*$" }
        action:         { type: string, example: "spawned", pattern: "^[a-z][a-z0-9]*$" }
        eventType:      { type: string, example: "aios.agent.lifecycle.spawned" }
        producerModule: { type: string, example: "006-Kernel" }
        dataschema:     { type: string, example: "aios://schemas/agent.lifecycle.spawned/1" }
        streamRef:      { type: string, format: uuid }
        trafficClass:   { type: string, enum: [control, telemetry, bulk, a2a] }
        deprecatedAt:   { type: string, format: date-time, nullable: true }
    A2ASession:
      type: object
      properties:
        urn:            { type: string }
        initiatorUrn:   { type: string }
        peerUrn:        { type: string }
        peerKind:       { type: string, enum: [internal, external] }
        state:          { type: string, enum: [Requested, Negotiating, Established, Suspended, Closing, Closed, Rejected, Failed] }
        channelSubject: { type: string }
        capabilities:   { type: array, items: { type: string } }
        messageCount:   { type: integer, format: int64 }
        bytesTotal:     { type: integer, format: int64 }
        closeReason:    { type: string, nullable: true }
    Problem:   # RFC 7807 / RFC-0001 §5.4 — NÃO redefinido aqui
      $ref: "https://docs.aios/schemas/problem.json"
paths:
  /subjects:
    post:
      summary: Registra um subject no namespace governado
      parameters: [ { $ref: '#/components/parameters/Traceparent' },
                    { $ref: '#/components/parameters/Tenant' },
                    { $ref: '#/components/parameters/IdempotencyKey' } ]
      responses:
        "201": { description: Registrado }
        "409": { description: "AIOS-BUS-0004 (forma inválida) ou já existente" }
        "422": { description: "AIOS-BUS-0015 (dataschema não registrado)" }
  /a2a/sessions:
    post:
      summary: Abre uma sessão A2A (T-01)
      responses:
        "201": { description: "Requested", content: { application/json: { schema: { $ref: '#/components/schemas/A2ASession' } } } }
        "403": { description: "AIOS-A2A-0001 (PDP negou)" }
        "429": { description: "AIOS-A2A-0006 (cota de sessões do agente)" }
        "504": { description: "AIOS-A2A-0002 (timeout de handshake)" }
```

---

## 3. gRPC (extrato do `.proto`)

```protobuf
syntax = "proto3";
package aios.comm.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";

service CommunicationBus {
  rpc RegisterSubject     (SubjectDefinition)     returns (SubjectDefinition);
  rpc ListSubjects        (ListSubjectsRequest)   returns (ListSubjectsResponse);
  rpc DeprecateSubject    (SubjectRef)            returns (SubjectDefinition);
  rpc PutStream           (StreamDefinition)      returns (StreamDefinition);
  rpc DrainStream         (StreamRef)             returns (StreamDefinition);
  rpc PutConsumer         (ConsumerDefinition)    returns (ConsumerDefinition);
  rpc GetConsumer         (ConsumerRef)           returns (ConsumerStatus);
  rpc ListDeadLetters     (ListDlqRequest)        returns (ListDlqResponse);
  rpc ReplayDeadLetter    (ReplayDlqRequest)      returns (ReplayOutcome);
  rpc DiscardDeadLetter   (DiscardDlqRequest)     returns (DeadLetterEntry);
  rpc ReplayStream        (ReplayStreamRequest)   returns (ReplayOutcome);
  rpc OpenSession         (OpenSessionRequest)    returns (A2ASession);
  rpc GetSession          (SessionRef)            returns (A2ASession);
  rpc CloseSession        (CloseSessionRequest)   returns (A2ASession);
  rpc PutGroupChannel     (GroupChannel)          returns (GroupChannel);
  rpc ProvisionAccount    (ProvisionRequest)      returns (AccountInfo);
  rpc PutAccountExport    (ExportSpec)            returns (ExportSpec);
  rpc GetBusHealth        (Empty)                 returns (BusHealth);
}

message OpenSessionRequest {
  string peer_urn = 1;
  string peer_kind = 2;                  // internal | external
  repeated string capabilities = 3;      // cada uma autorizada individualmente pelo PDP
  string idempotency_key = 4;            // RFC-0001 §5.5
}

message ConsumerDefinition {
  string stream_name = 1;
  string durable_name = 2;
  string consumer_module = 3;
  string filter_subject = 4;
  string ack_policy = 5;                                 // explicit | none | all
  google.protobuf.Duration ack_wait = 6;
  int32  max_deliver = 7;
  repeated google.protobuf.Duration backoff = 8;
  int32  max_ack_pending = 9;
  string dlq_subject = 10;
}

message BusHealth {
  int32  nodes_total = 1;
  int32  nodes_healthy = 2;
  bool   jetstream_quorum = 3;
  int64  streams_active = 4;
  int64  consumers_stalled = 5;
  int64  a2a_sessions_active = 6;
}
```

Erros gRPC mapeiam para `google.rpc.Status` + `ErrorInfo` com o mesmo `code`
`AIOS-<DOMINIO>-<NNNN>`, conforme RFC-0001 §5.4.

---

## 4. Protocolo de Mensagem (não é REST)

O tráfego real usa o cliente NATS. O contrato é:

| Aspecto | Regra |
|---------|-------|
| Autenticação | NKey/JWT da conta do tenant + TLS (`./Security.md` §1). |
| Subject | Conforme RFC-0001 §5.3 e registrado em `comm.subject_registry`. |
| Envelope | CloudEvents da RFC-0001 §5.2, validado no publish. |
| Correlação | `traceparent` obrigatório no envelope (RFC-0001 §5.6). |
| Idempotência | `event.id` (ULID) único; dedupe na janela do stream. |
| Ack | `explicit` em tráfego de controle; `nak` com atraso e `in-progress` suportados. |
| Request/reply | `reply-to` gerado pelo cliente; timeout `bus.request.timeout_ms` (5 s). |
| Payload | ≤ `bus.publish.max_payload_bytes` (1 MiB); acima disso, referência a objeto. |

---

## 5. Catálogo de Códigos de Erro

Domínios reservados por este módulo: **`BUS`** (0001–0099) e **`A2A`** (0001–0099),
registrados no catálogo mantido por `../004-API/` (RFC-0001 §8).

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-BUS-0001` | 404 | não | Subject, stream, consumidor ou sessão não encontrado. |
| `AIOS-BUS-0002` | 403 | não | Operação negada pelo PDP (*default deny*). |
| `AIOS-BUS-0003` | 403 | não | Tenant divergente do contexto autenticado (fronteira de conta). |
| `AIOS-BUS-0004` | 422 | não | Subject fora do registro ou fora da convenção RFC-0001 §5.3. |
| `AIOS-BUS-0005` | 422 | não | Envelope CloudEvents inválido. |
| `AIOS-BUS-0006` | 413 | não | Payload acima de `bus.publish.max_payload_bytes`. |
| `AIOS-BUS-0007` | 429 | sim | Cota de tráfego excedida. |
| `AIOS-BUS-0008` | 429 | sim | Limite de fan-out do canal de grupo excedido. |
| `AIOS-BUS-0009` | 409 | sim | Conflito de concorrência otimista (`version`). |
| `AIOS-BUS-0010` | 409 | não | Produtor não autorizado para o subject. |
| `AIOS-BUS-0011` | 503 | sim | Cluster NATS indisponível ou sem quórum de JetStream. |
| `AIOS-BUS-0012` | 504 | sim | Timeout em request/reply. |
| `AIOS-BUS-0013` | 507 | não | Stream atingiu `max_bytes` com `discard=new`. |
| `AIOS-BUS-0014` | 409 | sim | *Slow consumer*: `max_ack_pending` excedido. |
| `AIOS-BUS-0015` | 422 | não | `dataschema` não registrado ou versão desconhecida. |
| `AIOS-A2A-0001` | 403 | não | Sessão negada pelo PDP na negociação (T-05). |
| `AIOS-A2A-0002` | 504 | sim | Timeout de handshake (T-03). |
| `AIOS-A2A-0003` | 409 | não | Transição de estado inválida (viola a FSM). |
| `AIOS-A2A-0004` | 409 | sim | Sessão em `Suspended`: mensagem recusada. |
| `AIOS-A2A-0005` | 403 | não | Identidade do par não verificada ou revogada. |
| `AIOS-A2A-0006` | 429 | sim | Cota de sessões simultâneas do agente excedida. |
| `AIOS-A2A-0007` | 422 | não | Capacidade não suportada ou fora do perfil da RFC-0201. |

---

## 6. Exemplos

### 6.1 Registrar subject

```http
POST /v1/comm/subjects HTTP/1.1
Host: api.aios.local
X-AIOS-Tenant: _platform
Idempotency-Key: 01J9ZC6S9U1W3Y5A7C9E1G3J5L
Content-Type: application/json

{
  "domain": "agent",
  "entity": "lifecycle",
  "action": "spawned",
  "producerModule": "006-Kernel",
  "dataschema": "aios://schemas/agent.lifecycle.spawned/1",
  "streamRef": "6f1c2a10-3b4d-4e5f-8a9b-0c1d2e3f4a5b",
  "trafficClass": "control"
}
```

```http
HTTP/1.1 201 Created

{
  "urn": "urn:aios:_platform:subjectdef:01J9ZC5R8T0V2X4Z6B8D0F2H4K",
  "eventType": "aios.agent.lifecycle.spawned",
  "producerModule": "006-Kernel",
  "trafficClass": "control",
  "createdAt": "2026-07-22T15:02:11Z"
}
```

### 6.2 Subject por instância — recusado

```http
POST /v1/comm/subjects
{ "domain": "agent", "entity": "01J9Z8Q6H7K2M4N6P8R0S2T4V6", "action": "status", ... }
```

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://docs.aios/errors/invalid-subject",
  "title": "Invalid Subject Shape",
  "status": 422,
  "code": "AIOS-BUS-0004",
  "detail": "Componente 'entity' não corresponde a ^[a-z][a-z0-9]*$: subject deve identificar um TIPO de evento, nunca uma instância.",
  "retriable": false
}
```

### 6.3 Abrir sessão A2A

```bash
curl -sX POST https://api.aios.local/v1/comm/a2a/sessions \
  -H "X-AIOS-Tenant: acme" \
  -H "X-AIOS-Agent: urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6" \
  -H "Idempotency-Key: 01J9ZC7T0V2X4Z6B8D0F2H4K6M" \
  -H "Content-Type: application/json" \
  -d '{ "peerUrn": "urn:aios:acme:agent:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
        "peerKind": "internal",
        "capabilities": ["task.delegate", "memory.share"] }'
```

```json
{
  "urn": "urn:aios:acme:a2asession:01J9ZC8U1W3Y5A7C9E1G3J5L7N",
  "state": "Established",
  "channelSubject": "aios.acme.a2a.session.01J9ZC8U1W3Y5A7C9E1G3J5L7N",
  "capabilities": ["task.delegate", "memory.share"],
  "messageCount": 0,
  "createdAt": "2026-07-22T15:10:00Z"
}
```

### 6.4 Cota de tráfego excedida

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 2
Content-Type: application/problem+json

{
  "type": "https://docs.aios/errors/bus-quota-exceeded",
  "title": "Bus Quota Exceeded",
  "status": 429,
  "code": "AIOS-BUS-0007",
  "detail": "Tenant 'acme' excedeu 50000 msg/s (bus.quota.msgs_per_sec_tenant).",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "retriable": true,
  "retryAfterMs": 2000
}
```

### 6.5 Estado de consumidor (diagnóstico de lag)

```bash
curl -s https://api.aios.local/v1/comm/streams/KERNEL_LIFECYCLE/consumers/audit-lifecycle
```

```json
{
  "stream": "KERNEL_LIFECYCLE",
  "durableName": "audit-lifecycle",
  "consumerModule": "025-Audit",
  "numPending": 1842,
  "numAckPending": 87,
  "numRedelivered": 3,
  "maxAckPending": 1000,
  "stalled": false,
  "lastAckAt": "2026-07-22T15:11:47Z"
}
```

---

## 7. Versionamento e Compatibilidade

- Versionamento por caminho (`/v1/…`) e header `X-AIOS-Api-Version` (RFC-0001 §5.7).
- Mudanças incompatíveis incrementam a major e coexistem por ≥ 2 majors.
- **Depreciação de subject**: `:deprecate` marca `deprecated_at`; o subject continua
  publicável durante o período de coexistência, e a depreciação é anunciada em
  `aios._platform.comm.subject.deprecated` para que os consumidores migrem.
- Evolução de `dataschema` é **aditiva**; consumidores **DEVEM** tolerar campos
  desconhecidos.
- Códigos de erro são estáveis: um código **NÃO DEVE** mudar de significado.

---

## 8. Referências

- Brief: `./_DESIGN_BRIEF.md` §5 · Casos de uso: `./UseCases.md`
- FSM: `./StateMachine.md` · Eventos: `./Events.md` · Exemplos: `./Examples.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Registro de erros e schemas: `../004-API/`
