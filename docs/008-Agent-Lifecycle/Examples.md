---
Documento: Examples
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0084, ADR-0085, ADR-0086
RFCs relacionados: RFC-0001, RFC-0080
Depende de: 008-Agent-Lifecycle/API.md, 008-Agent-Lifecycle/Events.md, 008-Agent-Lifecycle/StateMachine.md, 003-RFC/RFC-0001-Architecture-Baseline.md
---

# 008-Agent-Lifecycle — Exemplos Executáveis

> Todos os exemplos assumem: base REST `https://gateway.aios.internal/v1/agents`
> (via API Gateway YARP, `001-Architecture`), pacote gRPC `aios.lifecycle.v1`,
> tenant `acme`, e um token OAuth2/OIDC já emitido pelo `021-Security` (o módulo
> apenas consome as claims, conforme `_DESIGN_BRIEF.md` §12.1). Cabeçalhos de
> correlação (`traceparent`, `X-AIOS-Tenant`, `Idempotency-Key`) seguem
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6. Os exemplos progridem do
> "hello world" a fluxos avançados de recuperação e migração (feliz **e**
> falha). Snippets são ilustrativos — DEVEM ser adaptados ao SDK/versão vigente
> (`../031-SDK/`).

---

## Índice

1. [Pré-requisitos e convenções](#1-pré-requisitos-e-convenções)
2. [Exemplo 1 — Hello World: `spawn` via REST (curl)](#2-exemplo-1--hello-world-spawn-via-rest-curl)
3. [Exemplo 2 — `Spawn` via gRPC (cliente C# / .NET 10)](#3-exemplo-2--spawn-via-grpc-cliente-c--net-10)
4. [Exemplo 3 — SDK Python: ciclo `suspend` → `hibernate` (automático) → `wake`](#4-exemplo-3--sdk-python-ciclo-suspend--hibernate-automático--wake)
5. [Exemplo 4 — Idempotência: reenvio seguro de `hibernate`](#5-exemplo-4--idempotência-reenvio-seguro-de-hibernate)
6. [Exemplo 5 — Checkpoint sob demanda e `restore` após falha de runtime](#6-exemplo-5--checkpoint-sob-demanda-e-restore-após-falha-de-runtime)
7. [Exemplo 6 — Migração entre shards (caminho feliz)](#7-exemplo-6--migração-entre-shards-caminho-feliz)
8. [Exemplo 7 — Migração compensada (falha do nó destino)](#8-exemplo-7--migração-compensada-falha-do-nó-destino)
9. [Exemplo 8 — Consultando o histórico de transições (event-sourced)](#9-exemplo-8--consultando-o-histórico-de-transições-event-sourced)
10. [Exemplo 9 — `StreamTransitions` via gRPC (cliente Python)](#10-exemplo-9--streamtransitions-via-grpc-cliente-python)
11. [Exemplo 10 — Consumindo eventos de ciclo de vida via NATS/JetStream](#11-exemplo-10--consumindo-eventos-de-ciclo-de-vida-via-natsjetstream)
12. [Exemplo 11 — Tratando lease conflict (`AIOS-LIFECYCLE-0003`) com retry](#12-exemplo-11--tratando-lease-conflict-aios-lifecycle-0003-com-retry)
13. [Exemplo 12 — Tratando transição inválida (`AIOS-LIFECYCLE-0002`)](#13-exemplo-12--tratando-transição-inválida-aios-lifecycle-0002)
14. [Exemplo 13 — Migração em massa disparada por drenagem de nó](#14-exemplo-13--migração-em-massa-disparada-por-drenagem-de-nó)
15. [Boas práticas ao consumir a API do Agent Lifecycle](#15-boas-práticas-ao-consumir-a-api-do-agent-lifecycle)

---

## 1. Pré-requisitos e convenções

| Item | Valor de exemplo | Referência |
|------|-------------------|------------|
| Base REST | `https://gateway.aios.internal/v1/agents` | `./API.md` |
| Pacote gRPC | `aios.lifecycle.v1` (serviço `LifecycleService`) | `./API.md`, `_DESIGN_BRIEF.md` §5.2 |
| Tenant de exemplo | `acme` | `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.1 |
| Formato de URN de agente | `urn:aios:acme:agent:<ULID>` | RFC-0001 §5.1 |
| Formato de URN de checkpoint | `urn:aios:acme:checkpoint:<ULID>` | `_DESIGN_BRIEF.md` §0 |
| Header de tenant | `X-AIOS-Tenant: acme` | RFC-0001 §5.6 |
| Header de correlação | `traceparent: 00-<trace_id>-<span_id>-01` | RFC-0001 §5.6, W3C Trace Context |
| Header de idempotência (mutações) | `Idempotency-Key: <ULID/UUID>` | RFC-0001 §5.5, `_DESIGN_BRIEF.md` §5.1 |
| Autenticação | `Authorization: Bearer <token OIDC>` (validado no Gateway) | `_DESIGN_BRIEF.md` §12.1 |

> Em todos os exemplos, `<ULID>` denota um identificador ULID gerado pelo
> chamador (ex.: `01J9Z8Q6H7K2M4N6P8R0S2T4V6`). Substitua por valores reais em
> produção — nunca reutilize um ULID entre entidades distintas (RFC-0001 §5.1),
> e lembre-se de que o `TombstoneManager` NÃO reutiliza IDs de agentes
> expurgados (`_DESIGN_BRIEF.md` §12.3).

---

## 2. Exemplo 1 — Hello World: `spawn` via REST (curl)

Cria/materializa um agente novo, disparando a saga de spawn: o Kernel (006)
registra o ACB, o Scheduler (009) admite, e o `SpawnManager` negocia o slot
de execução com o Runtime Supervisor (007).

```bash
curl -sS -X POST \
  "https://gateway.aios.internal/v1/agents" \
  -H "Authorization: Bearer $AIOS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  -H "Idempotency-Key: 01J9Z8QA1B2C3D4E5F6G7H8J9K" \
  -d '{
        "agentType": "research-assistant",
        "priorityClass": "interactive",
        "labels": { "team": "growth", "slaClass": "standard" }
      }'
```

Resposta esperada (`202 Accepted` — a materialização é assíncrona):

```json
{
  "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "tenantId": "acme",
  "state": "Created",
  "desiredState": "Running",
  "generation": 1,
  "priorityClass": "interactive",
  "placementShard": 42,
  "createdAt": "2026-07-20T12:34:56.789Z"
}
```

**O que aconteceu:** o `LifecycleApiSurface` validou o schema e a correlação,
o `LifecyclePolicyEnforcer` consultou o PDP para a ação `lifecycle:spawn`
(allow), o `LifecycleCoordinator` adquiriu a lease do novo `agent_id`,
persistiu a transição T1 (`∅ → Created`) e publicou
`aios.acme.agent.lifecycle.created` via Outbox. O agente segue
assincronamente para `Ready` assim que o `009-Scheduler` confirmar admissão
(T2) e para `Running` assim que o `007-Agent-Runtime` iniciar o loop (T3) —
ver `./StateMachine.md` e `./SequenceDiagrams.md`.

---

## 3. Exemplo 2 — `Spawn` via gRPC (cliente C# / .NET 10)

Uso interno típico: outro serviço do control plane (por exemplo, o próprio
Kernel, 006) chamando o Agent Lifecycle diretamente via gRPC, com mTLS.

```csharp
using Aios.Lifecycle.V1;
using Grpc.Net.Client;

// Canal com mTLS gerenciado pela malha de serviço (021-Security).
using var channel = GrpcChannel.ForAddress("https://lifecycle.internal.aios:6070");
var client = new LifecycleService.LifecycleServiceClient(channel);

var request = new SpawnRequest
{
    TenantId = "acme",
    AgentType = "research-assistant",
    PriorityClass = PriorityClass.Interactive,
    IdempotencyKey = "01J9Z8QA1B2C3D4E5F6G7H8J9K",
    Labels = { ["team"] = "growth", ["slaClass"] = "standard" }
};

var metadata = new Grpc.Core.Metadata
{
    { "traceparent", "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" },
    { "x-aios-tenant", "acme" }
};

try
{
    LifecycleAck ack = await client.SpawnAsync(request, metadata,
        deadline: DateTime.UtcNow.AddMilliseconds(300));
    Console.WriteLine($"ACB criado: {ack.AgentId} estado={ack.State} generation={ack.Generation}");
}
catch (Grpc.Core.RpcException ex) when (ex.StatusCode == Grpc.Core.StatusCode.PermissionDenied)
{
    // Mapeado de AIOS-LIFECYCLE-0012 via google.rpc.Status + ErrorInfo (RFC-0001 §5.4).
    Console.WriteLine($"Política negou o spawn: {ex.Status.Detail}");
}
```

> Note o `deadline` explícito de 300 ms — margem acima da meta p99 de 250 ms
> para spawn/materialização (NFR-001), evitando que o cliente aborte
> prematuramente uma chamada saudável, mas ainda protegendo contra
> travamento indefinido.

---

## 4. Exemplo 3 — SDK Python: ciclo `suspend` → `hibernate` (automático) → `wake`

O `007-Agent-Runtime` (Python) — ou um orquestrador externo — usa o SDK
(`031-SDK`) para pausar um agente que ficará ocioso, deixá-lo hibernar
automaticamente, e depois acordá-lo sob demanda.

```python
import uuid
from aios_sdk.lifecycle import LifecycleClient
from aios_sdk.errors import AiosApiError

lifecycle = LifecycleClient(
    base_url="https://gateway.aios.internal/v1/agents",
    tenant="acme",
    token_provider=lambda: get_cached_oidc_token(),
)

agent_id = "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6"

# 1) Suspende explicitamente (Running -> Suspended, T4). Working memory
#    permanece em RAM/Redis; retomada é rápida se necessária logo em seguida.
suspend_ack = lifecycle.suspend(agent_id=agent_id, idempotency_key=str(uuid.uuid4()))
print("Estado após suspend:", suspend_ack.state)  # "Suspended"

# 2) Nenhuma chamada explícita é necessária para hibernar: se o agente
#    permanecer ocioso além de hibernation.idle_ttl_s (padrão 300s), o
#    HibernationController dispara T6 (Suspended -> Hibernated) automaticamente,
#    grava um checkpoint "consistent" e libera 100% da RAM.
#    O evento aios.acme.agent.lifecycle.hibernated é publicado — basta assinar
#    (ver Exemplo 10) em vez de fazer polling.

# 3) Semanas depois, o orquestrador decide retomar o trabalho: `wake`
#    materializa o cold agent a partir do último checkpoint válido.
try:
    wake_ack = lifecycle.wake(agent_id=agent_id, idempotency_key=str(uuid.uuid4()))
    print(f"Acordado: estado={wake_ack.state} generation={wake_ack.generation}")
except AiosApiError as e:
    # Envelope RFC-7807 mapeado para exceção tipada pelo SDK.
    print(f"Erro {e.code} (status={e.status}, retriable={e.retriable}): {e.detail}")
```

`wake_ack.generation` é sempre **maior** que o `generation` anterior ao
`spawn` original — cada materialização a partir de frio incrementa a
encarnação do agente (`_DESIGN_BRIEF.md` §3).

---

## 5. Exemplo 4 — Idempotência: reenvio seguro de `hibernate`

Cenário: o cliente enviou `hibernate` sob demanda, a conexão caiu antes da
resposta chegar, e ele **não sabe** se a hibernação foi efetivada. Ele reenvia
com a **mesma** `Idempotency-Key`.

```bash
# Primeira tentativa (timeout de rede antes da resposta chegar ao cliente)
curl -sS -X POST \
  "https://gateway.aios.internal/v1/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6/hibernate" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QC3D4E5F6G7H8J9K0L1M"
# (sem resposta recebida pelo cliente — timeout)

# Reenvio idêntico, mesma Idempotency-Key
curl -sS -X POST \
  "https://gateway.aios.internal/v1/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6/hibernate" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QC3D4E5F6G7H8J9K0L1M"
```

**Resultado:** a segunda chamada reconhece a chave já processada e retorna
**exatamente o mesmo** `202 Accepted` com o mesmo `checkpointId`, sem criar um
segundo checkpoint nem publicar um evento duplicado (NFR-013). O outbox já
havia marcado `published=true` na primeira execução efetiva.

```json
{
  "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "state": "Hibernated",
  "checkpointId": "urn:aios:acme:checkpoint:01J9Z8QD4E5F6G7H8J9K0L1M2N",
  "coldSince": "2026-07-20T12:40:03.512Z"
}
```

---

## 6. Exemplo 5 — Checkpoint sob demanda e `restore` após falha de runtime

### 6.1 Checkpoint explícito antes de uma operação longa

```bash
curl -sS -X POST \
  "https://gateway.aios.internal/v1/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6/checkpoint" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QG7H8J9K0L1M2N3P4Q5R"
```

Resposta (`201 Created`):

```json
{
  "checkpointId": "urn:aios:acme:checkpoint:01J9Z8QH8J9K0L1M2N3P4Q5R6S",
  "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "generation": 3,
  "consistency": "consistent",
  "contentHash": "sha256:8f14e45fceea167a5a36dedd4bea2543",
  "sizeBytes": 4718592,
  "codec": "msgpack+zstd",
  "createdAt": "2026-07-20T12:50:00.045Z"
}
```

### 6.2 Falha de runtime e recuperação automática

Se o runtime falhar depois (heartbeat perdido além de
`reconcile.runtime_dead_after_ms`), o `ReconciliationController` detecta a
deriva e materializa automaticamente a partir do último checkpoint
`consistent`, **sem** ação do cliente — a recuperação faz parte do loop de
reconciliação, não é uma chamada explícita:

```bash
# Consulta apenas para observar o resultado da reconciliação automática
curl -sS -X GET \
  "https://gateway.aios.internal/v1/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6/lifecycle?limit=3" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme"
```

```json
{
  "items": [
    { "seq": 41, "fromState": "Running", "toState": "Failed", "trigger": "fail", "reason": "runtime_heartbeat_lost" },
    { "seq": 42, "fromState": "Failed", "toState": "Ready", "trigger": "reconcile", "reason": "materialized_from_checkpoint:01J9Z8QH8J9K0L1M2N3P4Q5R6S" },
    { "seq": 43, "fromState": "Ready", "toState": "Running", "trigger": "start", "reason": "scheduler_admitted" }
  ]
}
```

> Este exemplo ilustra o caminho de **falha e recuperação automática**: nem
> todo `Failed` é definitivo do ponto de vista operacional — quando um
> checkpoint `consistent` recente existe, a reconciliação restabelece o
> agente dentro da janela **RTO ≤ 15 min** (NFR-005) sem intervenção manual.
> Um `restore` explícito (`POST .../restore` com `checkpointId`) também está
> disponível para recuperação assistida, por exemplo, apontando
> deliberadamente para um checkpoint anterior ao mais recente.

```bash
curl -sS -X POST \
  "https://gateway.aios.internal/v1/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6/restore" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QJ9K0L1M2N3P4Q5R6S7T" \
  -d '{"checkpointId": "urn:aios:acme:checkpoint:01J9Z8QH8J9K0L1M2N3P4Q5R6S"}'
```

---

## 7. Exemplo 6 — Migração entre shards (caminho feliz)

Cenário: o Scheduler (009)/Cluster (027) decide rebalancear carga, movendo um
agente `Running` do shard 42 para o shard 17.

```bash
curl -sS -X POST \
  "https://gateway.aios.internal/v1/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6/migrate" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QK0L1M2N3P4Q5R6S7T8U" \
  -d '{"targetShard": 17}'
```

Resposta inicial (`202 Accepted`):

```json
{
  "migrationId": "urn:aios:acme:migration:01J9Z8QL1M2N3P4Q5R6S7T8U9V",
  "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "sourceShard": 42,
  "targetShard": 17,
  "phase": "quiesce",
  "status": "running"
}
```

O cliente acompanha a saga consultando o estado do ACB (ou assinando os
eventos `migrating`/`migrated` — ver Exemplo 10):

```bash
curl -sS -X GET \
  "https://gateway.aios.internal/v1/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme"
```

Após `cutover` bem-sucedido (dentro da meta `p99 ≤ 2 s`, NFR-010):

```json
{
  "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "state": "Running",
  "generation": 4,
  "placementShard": 17,
  "placementNode": "node-shard17-b",
  "updatedAt": "2026-07-20T13:05:02.331Z"
}
```

**O que a saga fez** (fases do `MigrationJob`, `_DESIGN_BRIEF.md` §3.5):
`quiesce` (drena operações não-preemptíveis) → `checkpoint` (captura
`consistent`) → `transfer` (blob para o novo shard) → `materialize` (novo
runtime no destino) → `cutover` (origem invalidada, só após confirmação) →
`cleanup` (libera recursos da origem). O evento
`aios.acme.agent.lifecycle.migrated` é publicado ao final.

---

## 8. Exemplo 7 — Migração compensada (falha do nó destino)

Mesmo cenário do Exemplo 6, mas o nó destino torna-se indisponível durante a
fase `materialize`.

```bash
curl -sS -X POST \
  "https://gateway.aios.internal/v1/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6/migrate" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QM2N3P4Q5R6S7T8U9V0W" \
  -d '{"targetShard": 17}'
# -> 202 { "migrationId": "...", "phase": "quiesce", "status": "running" }
```

Internamente, o nó destino cai antes da confirmação de materialização.
Após `migration.saga_timeout_ms` (padrão 120000 ms) sem confirmação, o
`MigrationOrchestrator` compensa automaticamente:

```bash
curl -sS -X GET \
  "https://gateway.aios.internal/v1/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6/lifecycle?limit=2" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme"
```

```json
{
  "items": [
    { "seq": 51, "fromState": "Running", "toState": "Migrating", "trigger": "migrate", "reason": "rebalance:shard17" },
    { "seq": 52, "fromState": "Migrating", "toState": "Running", "trigger": "cutover", "reason": "compensated:target_unhealthy", "placement": { "shard": 42 } }
  ]
}
```

**Resultado:** o agente permanece `Running` no shard/nó de **origem** (42),
sem perda de trabalho aceito — a invariante de migração compensável
(`_DESIGN_BRIEF.md` §9, "Migração travada") é a garantia estrutural aqui.
Se o cliente consultar `MigrationJob` diretamente durante a janela de
detecção de falha, recebe `AIOS-LIFECYCLE-0010` (502, retriable):

```json
{
  "type": "https://docs.aios/errors/migration-target-unavailable",
  "title": "Migration Target Unavailable",
  "status": 502,
  "code": "AIOS-LIFECYCLE-0010",
  "detail": "Nó de destino do shard 17 tornou-se indisponível durante a migração.",
  "instance": "urn:aios:acme:migration:01J9Z8QN3P4Q5R6S7T8U9V0W1X",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T13:10:00.500Z",
  "retriable": true,
  "retryAfterMs": 5000
}
```

Nova tentativa de migração (com nova `Idempotency-Key`, para um shard/nó
saudável) pode ser emitida normalmente assim que o operador confirmar a
disponibilidade do destino.

---

## 9. Exemplo 8 — Consultando o histórico de transições (event-sourced)

O histórico completo de um agente é sempre reconstruível a partir do log
append-only de `LifecycleTransition`:

```bash
curl -sS -X GET \
  "https://gateway.aios.internal/v1/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6/lifecycle?limit=10&offset=0" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme"
```

```json
{
  "items": [
    { "seq": 1, "fromState": null,       "toState": "Created",   "trigger": "spawn",    "actor": "api:kernel", "createdAt": "2026-07-20T12:34:56.789Z" },
    { "seq": 2, "fromState": "Created",  "toState": "Ready",     "trigger": "admit",    "actor": "scheduler",  "createdAt": "2026-07-20T12:34:57.012Z" },
    { "seq": 3, "fromState": "Ready",    "toState": "Running",   "trigger": "start",    "actor": "reconciler", "createdAt": "2026-07-20T12:34:57.240Z" },
    { "seq": 4, "fromState": "Running",  "toState": "Suspended", "trigger": "suspend",  "actor": "api:sub-xyz","createdAt": "2026-07-20T13:00:00.010Z" },
    { "seq": 5, "fromState": "Suspended","toState": "Hibernated","trigger": "hibernate","actor": "reconciler", "createdAt": "2026-07-20T13:05:00.300Z" }
  ],
  "pagination": { "totalCount": 5, "hasMore": false }
}
```

A tabela `guard_result` de cada transição (não incluída na resposta paginada
resumida acima, mas disponível no registro completo) permite auditar
exatamente qual guarda foi avaliada e o resultado — útil para investigar por
que uma transição foi ou não aceita.

---

## 10. Exemplo 9 — `StreamTransitions` via gRPC (cliente Python)

Um serviço de observabilidade acompanha em tempo real todas as transições de
um agente específico, via streaming server-side gRPC.

```python
import grpc
from aios.lifecycle.v1 import lifecycle_pb2, lifecycle_pb2_grpc

channel = grpc.secure_channel("lifecycle.internal.aios:6070", grpc.ssl_channel_credentials())
stub = lifecycle_pb2_grpc.LifecycleServiceStub(channel)

request = lifecycle_pb2.AgentRef(
    tenant_id="acme",
    agent_id="urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
)
metadata = (
    ("x-aios-tenant", "acme"),
    ("traceparent", "00-4bf92f3577b34da6a3ce929d0e0e4736-9c7b1e2f3a4b5c6d-01"),
)

for transition in stub.StreamTransitions(request, metadata=metadata):
    print(f"[seq={transition.seq}] {transition.from_state} -> {transition.to_state} "
          f"(trigger={transition.trigger}, actor={transition.actor})")
    if transition.to_state in ("Terminated", "Failed"):
        break  # estado terminal: nenhuma transição futura é esperada
```

---

## 11. Exemplo 10 — Consumindo eventos de ciclo de vida via NATS/JetStream

Um serviço externo (por exemplo, o orquestrador de workflows, `014`) assina
eventos de ciclo de vida de agentes de um tenant.

```python
import asyncio
import json
from nats.aio.client import Client as NATS

async def main():
    nc = NATS()
    await nc.connect(servers=["tls://nats.internal.aios:4222"])
    js = nc.jetstream()

    async def handler(msg):
        event = json.loads(msg.data)
        print(f"[{event['type']}] subject={event['subject']} tenant={event['tenant']}")
        # Deduplicação por event.id, conforme RFC-0001 §5.2/§5.5:
        if not already_seen(event["id"]):
            process_lifecycle_event(event)
            mark_seen(event["id"])
        await msg.ack()

    # Assinatura durável, filtrando pelo tenant acme, todos os eventos de ciclo de vida.
    await js.subscribe(
        "aios.acme.agent.lifecycle.>",
        durable="workflow-lifecycle-consumer",
        cb=handler,
        manual_ack=True,
    )

    await asyncio.Event().wait()  # mantém o processo vivo

asyncio.run(main())
```

Exemplo de payload recebido (`agent.lifecycle.hibernated`):

```json
{
  "specversion": "1.0",
  "id": "01J9Z8QO4Q5R6S7T8U9V0W1X2Y",
  "source": "urn:aios:acme:service:lifecycle",
  "type": "aios.agent.lifecycle.hibernated",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T13:05:00.300Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-9c7b1e2f3a4b5c6d-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.hibernated/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "generation": 3,
    "fromState": "Suspended",
    "toState": "Hibernated",
    "trigger": "hibernate",
    "checkpointId": "urn:aios:acme:checkpoint:01J9Z8QD4E5F6G7H8J9K0L1M2N"
  }
}
```

Note que **nenhum conteúdo de working memory** aparece no payload — apenas
metadados de estado, conforme a exigência de minimização de dados
(`_DESIGN_BRIEF.md` §12.3).

---

## 12. Exemplo 11 — Tratando lease conflict (`AIOS-LIFECYCLE-0003`) com retry

Duas requisições concorrentes tentam transicionar o mesmo agente
(`resume` disparado pelo orquestrador ao mesmo tempo em que o
`ReconciliationController` tenta corrigir uma deriva detectada).

```python
import time
import random
import uuid
from aios_sdk.lifecycle import LifecycleClient
from aios_sdk.errors import AiosApiError

lifecycle = LifecycleClient(base_url="https://gateway.aios.internal/v1/agents",
                             tenant="acme", token_provider=get_cached_oidc_token)

agent_id = "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6"

def resume_with_retry(max_attempts=5):
    for attempt in range(1, max_attempts + 1):
        try:
            return lifecycle.resume(agent_id=agent_id, idempotency_key=str(uuid.uuid4()))
        except AiosApiError as e:
            if e.code == "AIOS-LIFECYCLE-0003" and e.retriable:
                backoff = min(2 ** attempt, 8) + random.uniform(0, 0.5)
                time.sleep(backoff)
                continue
            raise
    raise RuntimeError("resume falhou após esgotar tentativas de retry")

ack = resume_with_retry()
print("Estado após resume:", ack.state)
```

> Note que uma **nova** `Idempotency-Key` é gerada a cada tentativa — a
> requisição anterior nunca foi aceita pelo servidor (foi rejeitada antes de
> qualquer efeito colateral), portanto não há risco de duplicação ao usar
> chaves distintas neste laço de retry específico.

---

## 13. Exemplo 12 — Tratando transição inválida (`AIOS-LIFECYCLE-0002`)

Tentativa de `resume` em um agente que já está `Terminated` (estado
terminal, sem transições de saída — invariante INV3).

```bash
curl -sS -X POST \
  "https://gateway.aios.internal/v1/agents/urn:aios:acme:agent:01J9Z8QP5R6S7T8U9V0W1X2Y3Z/resume" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QQ6S7T8U9V0W1X2Y3Z4A"
```

Resposta (`409 Conflict`, não retriável):

```json
{
  "type": "https://docs.aios/errors/lifecycle-invalid-transition",
  "title": "Invalid Lifecycle Transition",
  "status": 409,
  "code": "AIOS-LIFECYCLE-0002",
  "detail": "Transição 'resume' inválida a partir do estado atual 'Terminated'.",
  "instance": "urn:aios:acme:agent:01J9Z8QP5R6S7T8U9V0W1X2Y3Z",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T13:15:00.900Z",
  "retriable": false
}
```

Tratamento recomendado: não retentar. Um agente `Terminated` não pode ser
reanimado — para retomar trabalho equivalente, emita um novo `spawn`,
opcionalmente referenciando o último `checkpointId` conhecido:

```python
try:
    lifecycle.resume(agent_id=agent_id, idempotency_key=str(uuid.uuid4()))
except AiosApiError as e:
    if e.code == "AIOS-LIFECYCLE-0002":
        log.info("agente em estado terminal; emitindo novo spawn a partir do último checkpoint")
        new_ack = lifecycle.spawn(
            agent_type="research-assistant",
            priority_class="interactive",
            restore_from_checkpoint=last_known_checkpoint_id,
            idempotency_key=str(uuid.uuid4()),
        )
    else:
        raise
```

---

## 14. Exemplo 13 — Migração em massa disparada por drenagem de nó

Cenário operacional: o `027-Cluster` decide drenar um nó para manutenção e
publica `aios.acme.cluster.node.draining`. O módulo 008 consome esse evento e
dispara migração para **todos** os agentes hospedados naquele nó,
respeitando `migration.max_concurrent_per_node`.

```python
import asyncio
import json
from nats.aio.client import Client as NATS

async def main():
    nc = NATS()
    await nc.connect(servers=["tls://nats.internal.aios:4222"])
    js = nc.jetstream()

    async def handler(msg):
        event = json.loads(msg.data)
        if event["type"] == "aios.cluster.node.draining":
            node_id = event["data"]["nodeId"]
            print(f"Nó {node_id} entrando em drenagem — "
                  f"MigrationOrchestrator enfileirará migração para todos os "
                  f"agentes hospedados nele, até {8} migrações concorrentes "
                  f"por nó (migration.max_concurrent_per_node).")
        await msg.ack()

    await js.subscribe(
        "aios.acme.cluster.node.draining",
        durable="lifecycle-drain-consumer",
        cb=handler,
        manual_ack=True,
    )
    await asyncio.Event().wait()

asyncio.run(main())
```

> Este exemplo é ilustrativo do **consumo** do evento pelo próprio módulo 008
> (comportamento interno do `LifecycleEventConsumer`, `_DESIGN_BRIEF.md`
> §2, §6.2) — nenhuma chamada de API é necessária por parte de um operador
> humano: a drenagem do nó, uma vez sinalizada pelo Cluster (027), resulta em
> migrações automáticas e enfileiradas dos agentes afetados, com cada uma
> seguindo exatamente a saga do Exemplo 6 (ou a compensação do Exemplo 7, se
> um destino específico falhar).

---

## 15. Boas práticas ao consumir a API do Agent Lifecycle

- **Sempre gere uma nova `Idempotency-Key` por intenção de mutação** — nunca
  reaproveite entre operações logicamente distintas (ver `./FAQ.md` §7.5).
- **Trate `retriable: false` como definitivo** — não retente
  `AIOS-LIFECYCLE-0002`, `AIOS-LIFECYCLE-0004`, `AIOS-LIFECYCLE-0007`,
  `AIOS-LIFECYCLE-0012` ou `AIOS-LIFECYCLE-0013` sem antes reler o estado
  atual do agente e ajustar a requisição.
- **Respeite `retryAfterMs`/`Retry-After`** em respostas 429/502/503 —
  não implemente retry agressivo que ignore o sinal de backpressure
  (`AIOS-LIFECYCLE-0005`, `AIOS-LIFECYCLE-0010`, `AIOS-LIFECYCLE-0014`).
- **Prefira assinar eventos a fazer polling** de `GET /v1/agents/{id}` para
  descobrir mudanças de estado — reduz carga no módulo e latência de reação,
  especialmente relevante em escala de milhões de agentes (NFR-004).
- **Confie na reconciliação automática** antes de invocar `restore`
  manualmente — o `ReconciliationController` já corrige derivas de runtime
  morto dentro de `reconcile.interval_ms` (ver Exemplo 5).
- **Nunca trate `Hibernated` como equivalente a `Suspended` em latência** —
  planeje a UX/timeout do chamador considerando a materialização a frio
  (`p99 ≤ 250 ms`, mas ainda uma ordem de grandeza mais lenta que `resume` de
  um agente `Suspended`).
- **Propague sempre `traceparent`** recebido do chamador anterior, nunca
  gere um novo trace raiz no meio de um fluxo — isso preserva a correlação
  ponta a ponta exigida por NFR-012.
- **Correlacione `checkpointId` com o catálogo de checkpoints** ao investigar
  incidentes (`GET /v1/agents/{id}/checkpoints`), em vez de assumir qual
  snapshot foi usado em uma materialização.

---

## Referências

- Contratos completos: `./API.md`
- Catálogo de eventos: `./Events.md`
- Máquina de estados: `./StateMachine.md`
- Fluxos de sequência (feliz e falha): `./SequenceDiagrams.md`
- Perguntas frequentes: `./FAQ.md`
- Contratos centrais reutilizados: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- SDK: `../031-SDK/`

*Fim de `Examples.md`.*
