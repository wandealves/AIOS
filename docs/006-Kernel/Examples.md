---
Documento: Examples
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0060, ADR-0061, ADR-0062, ADR-0063, ADR-0064, ADR-0066, ADR-0067
RFCs relacionados: RFC-0001, RFC-0006, RFC-0007
Depende de: 006-Kernel/API.md, 006-Kernel/Events.md, 006-Kernel/StateMachine.md, 003-RFC/RFC-0001-Architecture-Baseline.md
---

# 006-Kernel — Exemplos Executáveis

> Todos os exemplos assumem: base REST `https://gateway.aios.internal/v1/kernel`
> (via API Gateway YARP, `001-Architecture`), pacote gRPC `aios.kernel.v1`, tenant
> `acme`, e um token OAuth2/OIDC já emitido pelo `021-Security` (o Kernel apenas
> consome as claims, conforme `_DESIGN_BRIEF.md` §12.1). Cabeçalhos de
> correlação (`traceparent`, `X-AIOS-Tenant`, `Idempotency-Key`) seguem
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.6. Os exemplos progridem do
> "hello world" a fluxos avançados de recuperação. Snippets são ilustrativos —
> DEVEM ser adaptados ao SDK/versão vigente (`../031-SDK/`).

---

## Índice

1. [Pré-requisitos e convenções](#1-pré-requisitos-e-convenções)
2. [Exemplo 1 — Hello World: `spawn` via REST (curl)](#2-exemplo-1--hello-world-spawn-via-rest-curl)
3. [Exemplo 2 — `spawn` via gRPC (cliente C# / .NET 10)](#3-exemplo-2--spawn-via-grpc-cliente-c--net-10)
4. [Exemplo 3 — SDK Python do Agent Runtime chamando syscalls](#4-exemplo-3--sdk-python-do-agent-runtime-chamando-syscalls)
5. [Exemplo 4 — Idempotência: reenvio seguro de `spawn`](#5-exemplo-4--idempotência-reenvio-seguro-de-spawn)
6. [Exemplo 5 — Verificando cota antes de uma operação cara (`get_quota`)](#6-exemplo-5--verificando-cota-antes-de-uma-operação-cara-get_quota)
7. [Exemplo 6 — Tratando negação de capability (`AIOS-CAP-0001`)](#7-exemplo-6--tratando-negação-de-capability-aios-cap-0001)
8. [Exemplo 7 — Ciclo `suspend` → `hibernate` (automático) → `resume`](#8-exemplo-7--ciclo-suspend--hibernate-automático--resume)
9. [Exemplo 8 — `checkpoint` e recuperação após falha](#9-exemplo-8--checkpoint-e-recuperação-após-falha)
10. [Exemplo 9 — Árvore de agentes: `spawn` de sub-agente com `parent_urn`](#10-exemplo-9--árvore-de-agentes-spawn-de-sub-agente-com-parent_urn)
11. [Exemplo 10 — Consumindo eventos de ciclo de vida via NATS/JetStream (Python)](#11-exemplo-10--consumindo-eventos-de-ciclo-de-vida-via-natsjetstream-python)
12. [Exemplo 11 — `kill` com compensação após falha de boot](#12-exemplo-11--kill-com-compensação-após-falha-de-boot)
13. [Exemplo 12 — Encadeando `invoke_tool` e `route_model` a partir do Agent Runtime](#13-exemplo-12--encadeando-invoke_tool-e-route_model-a-partir-do-agent-runtime)
14. [Boas práticas ao consumir a ABI do Kernel](#14-boas-práticas-ao-consumir-a-abi-do-kernel)

---

## 1. Pré-requisitos e convenções

| Item | Valor de exemplo | Referência |
|------|-------------------|------------|
| Base REST | `https://gateway.aios.internal/v1/kernel` | `./API.md` |
| Pacote gRPC | `aios.kernel.v1` | `./API.md` |
| Tenant de exemplo | `acme` | `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.1 |
| Formato de URN de agente | `urn:aios:acme:agent:<ULID>` | RFC-0001 §5.1 |
| Header de tenant | `X-AIOS-Tenant: acme` | RFC-0001 §5.6 |
| Header de correlação | `traceparent: 00-<trace_id>-<span_id>-01` | RFC-0001 §5.6, W3C Trace Context |
| Header de idempotência (mutações) | `Idempotency-Key: <ULID/UUID>` | RFC-0001 §5.5 |
| Autenticação | `Authorization: Bearer <token OIDC>` (validado no Gateway) | `_DESIGN_BRIEF.md` §12.1 |

> Em todos os exemplos, `<ULID>` denota um identificador ULID gerado pelo
> chamador (ex.: `01J9Z8Q6H7K2M4N6P8R0S2T4V6`). Substitua por valores reais em
> produção — nunca reutilize um ULID entre entidades distintas (RFC-0001 §5.1).

---

## 2. Exemplo 1 — Hello World: `spawn` via REST (curl)

Cria um novo agente (ACB em `Pending`) com prioridade default e cota
default do tenant.

```bash
curl -sS -X POST \
  "https://gateway.aios.internal/v1/kernel/agents" \
  -H "Authorization: Bearer $AIOS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  -H "Idempotency-Key: 01J9Z8QA1B2C3D4E5F6G7H8J9K" \
  -d '{
        "agentType": "research-assistant",
        "priority": 5,
        "labels": { "team": "growth", "slaClass": "standard" }
      }'
```

Resposta esperada (201 Created):

```json
{
  "urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "tenantId": "acme",
  "state": "Pending",
  "priority": 5,
  "quotaRef": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "createdAt": "2026-07-20T12:34:56.789Z",
  "version": 0
}
```

**O que aconteceu:** o `SyscallGateway` validou o schema e a correlação, o
`CapabilityEnforcer` consultou o PDP para `agent:spawn` (allow), o
`ResourceQuotaManager` reservou a cota `agent_count` do tenant, o
`LifecycleCoordinator` criou o ACB em `Pending` (T-01) e o `EventEmitter`
publicou `aios.acme.agent.lifecycle.spawned` via Outbox. O agente segue
assincronamente para `Admitted` assim que o `009-Scheduler` confirmar
admissão — ver `./StateMachine.md` e `./SequenceDiagrams.md`.

---

## 3. Exemplo 2 — `spawn` via gRPC (cliente C# / .NET 10)

Uso interno típico: outro serviço do control plane (ex.: `031-SDK` no modo
servidor) chamando o Kernel diretamente via gRPC, com mTLS.

```csharp
using Aios.Kernel.V1;
using Grpc.Net.Client;

// Canal com mTLS gerenciado pela malha de serviço (021-Security).
using var channel = GrpcChannel.ForAddress("https://kernel.internal.aios:6060");
var client = new KernelSyscalls.KernelSyscallsClient(channel);

var request = new SpawnRequest
{
    TenantId = "acme",
    AgentType = "research-assistant",
    Priority = 5,
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
    SpawnResponse response = await client.SpawnAsync(request, metadata, deadline: DateTime.UtcNow.AddMilliseconds(300));
    Console.WriteLine($"ACB criado: {response.Urn} estado={response.State}");
}
catch (Grpc.Core.RpcException ex) when (ex.StatusCode == Grpc.Core.StatusCode.PermissionDenied)
{
    // Mapeado de AIOS-CAP-0001 via google.rpc.Status + ErrorInfo (RFC-0001 §5.4).
    Console.WriteLine($"Capability negada: {ex.Status.Detail}");
}
```

> Note o `deadline` explícito de 300 ms — margem acima da meta p99 de 250 ms
> para `spawn` (NFR-001), evitando que o cliente aborte prematuramente uma
> chamada saudável, mas ainda protegendo contra travamento indefinido.

---

## 4. Exemplo 3 — SDK Python do Agent Runtime chamando syscalls

O `007-Agent-Runtime` (Python) nunca acessa memória/planejamento/ferramentas
diretamente — ele chama o Kernel, que broqueia. Exemplo simplificado usando
o SDK (`031-SDK`):

```python
from aios_sdk.kernel import KernelClient
from aios_sdk.errors import AiosApiError

kernel = KernelClient(
    base_url="https://gateway.aios.internal/v1/kernel",
    tenant="acme",
    token_provider=lambda: get_cached_oidc_token(),
)

# remember: grava um item de memória episódica via broker do Kernel -> 010-Memory
try:
    result = kernel.remember(
        agent_urn="urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
        memory_type="episodic",
        content={"event": "user_asked_refund_policy", "summary": "..."},
        idempotency_key="01J9Z8QB2C3D4E5F6G7H8J9K0L",
    )
    print("Item de memória gravado:", result.item_ref)
except AiosApiError as e:
    # Envelope RFC-7807 mapeado para exceção tipada pelo SDK.
    print(f"Erro {e.code} (status={e.status}, retriable={e.retriable}): {e.detail}")
```

```python
# recall: consulta a mesma memória (syscall de leitura, sem Idempotency-Key)
memories = kernel.recall(
    agent_urn="urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    query="qual é a política de reembolso?",
    top_k=5,
)
for m in memories.items:
    print(m.score, m.content)
```

---

## 5. Exemplo 4 — Idempotência: reenvio seguro de `spawn`

Cenário: o cliente enviou `spawn`, a conexão caiu antes da resposta chegar, e
o cliente **não sabe** se a criação foi efetivada. Ele reenvia a mesma
requisição com a **mesma** `Idempotency-Key`.

```bash
# Primeira tentativa (timeout de rede antes da resposta chegar ao cliente)
curl -sS -X POST "https://gateway.aios.internal/v1/kernel/agents" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QC3D4E5F6G7H8J9K0L1M" \
  -d '{"agentType":"research-assistant","priority":5}'
# (sem resposta recebida pelo cliente — timeout)

# Reenvio idêntico, mesma Idempotency-Key
curl -sS -X POST "https://gateway.aios.internal/v1/kernel/agents" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QC3D4E5F6G7H8J9K0L1M" \
  -d '{"agentType":"research-assistant","priority":5}'
```

**Resultado:** o `IdempotencyStore` reconhece a chave já processada e
retorna **exatamente o mesmo** `201 Created` com o mesmo `urn`, sem criar um
segundo ACB. Nenhum evento duplicado é publicado (o Outbox já havia marcado
`published=true` na primeira execução efetiva).

Se o corpo da segunda tentativa **divergir** do original (por exemplo,
`priority` diferente), o Kernel responde:

```json
{
  "type": "https://docs.aios/errors/idempotency-key-conflict",
  "title": "Idempotency Key Conflict",
  "status": 409,
  "code": "AIOS-SYSCALL-0002",
  "detail": "Idempotency-Key '01J9Z8QC3D4E5F6G7H8J9K0L1M' já foi usada com um payload diferente.",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:35:10.001Z",
  "retriable": false
}
```

---

## 6. Exemplo 5 — Verificando cota antes de uma operação cara (`get_quota`)

Boa prática: antes de disparar uma cadeia de `invoke_tool`/`route_model`
potencialmente cara, o Agent Runtime consulta o saldo de cota.

```bash
curl -sS -X GET \
  "https://gateway.aios.internal/v1/kernel/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6/quota" \
  -H "Authorization: Bearer $AIOS_TOKEN" \
  -H "X-AIOS-Tenant: acme"
```

```json
{
  "subjectUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "window": "PT1H",
  "tokens": { "limit": 1000000, "used": 812340, "remaining": 187660 },
  "costUsd": { "limit": 10.0, "used": 8.71, "remaining": 1.29 },
  "toolsInvocations": { "limit": 200, "used": 173, "remaining": 27 },
  "syscallsPerSec": { "limit": 200 },
  "enforcement": "hard",
  "asOf": "2026-07-20T12:40:03.512Z"
}
```

Com `costUsd.remaining` baixo, o agente PODE decidir rotear para um modelo
mais barato (via `route_model`) ou encerrar a tarefa graciosamente antes de
receber um `429`.

---

## 7. Exemplo 6 — Tratando negação de capability (`AIOS-CAP-0001`)

Um agente sem a capability `tool:invoke:payments-api` tenta invocar essa
ferramenta:

```bash
curl -sS -X POST \
  "https://gateway.aios.internal/v1/kernel/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6/tools:invoke" \
  -H "Authorization: Bearer $AIOS_TOKEN" \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QD4E5F6G7H8J9K0L1M2N" \
  -d '{"tool": "payments-api", "action": "refund", "params": {"orderId": "ORD-9921"}}'
```

Resposta (403):

```json
{
  "type": "https://docs.aios/errors/capability-denied",
  "title": "Capability Denied",
  "status": 403,
  "code": "AIOS-CAP-0001",
  "detail": "Capability 'tool:invoke:payments-api' não concedida para o agente.",
  "instance": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:41:00.128Z",
  "retriable": false
}
```

Tratamento recomendado no cliente (Python):

```python
from aios_sdk.errors import AiosApiError

try:
    kernel.invoke_tool(agent_urn=agent_urn, tool="payments-api", action="refund",
                        params={"orderId": "ORD-9921"},
                        idempotency_key="01J9Z8QD4E5F6G7H8J9K0L1M2N")
except AiosApiError as e:
    if e.code == "AIOS-CAP-0001":
        # Não é um erro transitório: não retentar. Registrar e escalar ao
        # planejamento (012) para replanejar sem essa ferramenta, ou solicitar
        # elevação de capability via fluxo humano (fora do escopo do Kernel).
        log.warning("capability negada", tool="payments-api", trace_id=e.trace_id)
    else:
        raise
```

O evento correlato `aios.acme.agent.syscall.denied` também é publicado,
disponível para auditoria e alertas (`025-Audit`, `024-Observability`).

---

## 8. Exemplo 7 — Ciclo `suspend` → `hibernate` (automático) → `resume`

### 8.1 Suspender explicitamente

```bash
curl -sS -X POST \
  "https://gateway.aios.internal/v1/kernel/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6:suspend" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QE5F6G7H8J9K0L1M2N3P"
```

Resposta: `200 OK`, `state: "Suspended"`.

### 8.2 Hibernação automática por ociosidade

Nenhuma chamada explícita é necessária: se `now - last_active_at >
kernel.hibernation.idle_ttl_ms` (padrão 300000 ms = 5 min), o
`LifecycleCoordinator` dispara a transição T-08 automaticamente, movendo o
estado para armazenamento durável e liberando RAM. O evento
`aios.acme.agent.lifecycle.hibernated` é publicado — nenhum polling é
necessário do lado do cliente; basta assinar o evento (ver Exemplo 10).

### 8.3 Retomar (de `Suspended` ou de `Hibernated`)

```bash
curl -sS -X POST \
  "https://gateway.aios.internal/v1/kernel/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6:resume" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QF6G7H8J9K0L1M2N3P4Q"
```

Se o ACB estava `Suspended`, a resposta chega em poucos milissegundos
(estado já quente). Se estava `Hibernated`, o Kernel solicita novo slot ao
Scheduler (T-09) e a materialização sob demanda ao Lifecycle — meta p99 <
250 ms, com o `checkpoint_ref` usado para restaurar ponteiros de
memória/contexto de forma consistente.

---

## 9. Exemplo 8 — `checkpoint` e recuperação após falha

Antes de uma operação longa e sensível, o agente solicita um checkpoint
explícito para garantir um ponto de retomada consistente:

```bash
curl -sS -X POST \
  "https://gateway.aios.internal/v1/kernel/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6:checkpoint" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QG7H8J9K0L1M2N3P4Q5R"
```

```json
{
  "urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "checkpointRef": "minio://kernel-checkpoints/acme/01J9Z8Q6.../ckpt-01J9Z8QH8J9K0L1M2N3P4Q5R6S.snap",
  "state": "Running",
  "createdAt": "2026-07-20T12:50:00.045Z"
}
```

Se o runtime falhar depois (heartbeat perdido além de
`kernel.runtime.heartbeat_deadline_ms`), o ACB transiciona para `Failed`
(T-12). A recuperação **não** é automática a partir de `Failed` (estado
terminal) — a prática recomendada é que o orquestrador/operador emita um
novo `spawn`, referenciando o `checkpointRef` anterior nos parâmetros de
criação, para que o novo agente restaure o estado de memória/contexto onde o
anterior parou:

```bash
curl -sS -X POST "https://gateway.aios.internal/v1/kernel/agents" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QJ9K0L1M2N3P4Q5R6S7T" \
  -d '{
        "agentType": "research-assistant",
        "priority": 5,
        "restoreFromCheckpoint": "minio://kernel-checkpoints/acme/01J9Z8Q6.../ckpt-01J9Z8QH8J9K0L1M2N3P4Q5R6S.snap"
      }'
```

---

## 10. Exemplo 9 — Árvore de agentes: `spawn` de sub-agente com `parent_urn`

Um agente orquestrador cria um sub-agente especializado para uma subtarefa.
O Kernel preenche `parent_urn` automaticamente a partir do contexto de
chamada autenticado (o agente chamador é identificado por `X-AIOS-Agent`):

```bash
curl -sS -X POST "https://gateway.aios.internal/v1/kernel/agents" \
  -H "Authorization: Bearer $AIOS_TOKEN" \
  -H "X-AIOS-Tenant: acme" \
  -H "X-AIOS-Agent: urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6" \
  -H "Idempotency-Key: 01J9Z8QK0L1M2N3P4Q5R6S7T8U" \
  -d '{
        "agentType": "web-search-worker",
        "priority": 6,
        "labels": { "purpose": "sub-task:citations" }
      }'
```

```json
{
  "urn": "urn:aios:acme:agent:01J9Z8QL1M2N3P4Q5R6S7T8U9V",
  "parentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "state": "Pending",
  "priority": 6,
  "createdAt": "2026-07-20T12:55:00.201Z"
}
```

> Fan-out de `spawn` a partir de um mesmo `parent_urn` é limitado por
> política (`022-Policy`) para conter o problema O(N²) de coordenação
> multi-agente identificado na Visão Global (RSK-03) — excedê-lo resulta em
> `AIOS-CAP-0001` para as tentativas além do limite.

---

## 11. Exemplo 10 — Consumindo eventos de ciclo de vida via NATS/JetStream (Python)

Um serviço de observabilidade (ou o próprio orquestrador) assina eventos de
ciclo de vida de agentes de um tenant:

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

    # Assinatura durável no stream KERNEL_LIFECYCLE, filtrando pelo tenant acme.
    await js.subscribe(
        "aios.acme.agent.lifecycle.>",
        durable="observability-lifecycle-consumer",
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
  "id": "01J9Z8QM2N3P4Q5R6S7T8U9V0W",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.lifecycle.hibernated",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T13:00:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-9c7b1e2f3a4b5c6d-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.hibernated/1",
  "data": {
    "urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "previousState": "Suspended",
    "checkpointRef": "minio://kernel-checkpoints/acme/01J9Z8Q6.../ckpt-latest.snap",
    "idleForMs": 300214
  }
}
```

---

## 12. Exemplo 11 — `kill` com compensação após falha de boot

Cenário de falha: `spawn` foi aceito, o Scheduler admitiu o agente
(`Admitted`), mas a materialização do runtime falhou por indisponibilidade
transitória do `008-Agent-Lifecycle`.

```bash
# 1) spawn (sucesso, ACB -> Pending -> Admitted)
curl -sS -X POST "https://gateway.aios.internal/v1/kernel/agents" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QN3P4Q5R6S7T8U9V0W1X" \
  -d '{"agentType":"research-assistant","priority":5}'
# -> 201 { "urn": "...", "state": "Pending" }

# 2) Internamente, T-04 falha: boot_timeout excedido (kernel.spawn.boot_timeout_ms=5000)
# O Kernel transiciona automaticamente para Failed (T-05), sem ação do cliente.

# 3) Cliente consulta o estado
curl -sS -X GET \
  "https://gateway.aios.internal/v1/kernel/agents/urn:aios:acme:agent:01J9Z8QO4Q5R6S7T8U9V0W1X2Y" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme"
```

```json
{
  "urn": "urn:aios:acme:agent:01J9Z8QO4Q5R6S7T8U9V0W1X2Y",
  "state": "Failed",
  "terminatedAt": "2026-07-20T13:05:12.900Z",
  "labels": { "failureReason": "boot_timeout_exceeded" }
}
```

**O que a saga de compensação fez automaticamente** (ADR-0064): reverteu a
reserva de slot no Scheduler, liberou a cota `agent_count` reservada para
este agente, e emitiu `aios.acme.agent.lifecycle.failed`. Nenhuma ação
manual de limpeza é necessária pelo cliente.

Se, em vez de falha de boot, o cliente decidir encerrar um agente saudável
manualmente:

```bash
curl -sS -X DELETE \
  "https://gateway.aios.internal/v1/kernel/agents/urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6" \
  -H "Authorization: Bearer $AIOS_TOKEN" -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QP5R6S7T8U9V0W1X2Y3Z"
```

O ACB transiciona `Running/Suspended/Hibernated → Terminating` (T-10) e, após
drenagem e liberação de cotas, `Terminating → Terminated` (T-11, terminal) —
com o evento `aios.acme.agent.lifecycle.terminated` publicado.

---

## 13. Exemplo 12 — Encadeando `invoke_tool` e `route_model` a partir do Agent Runtime

Fluxo típico do loop ReAct de um agente (executando em `007-Agent-Runtime`),
todo mediado pelo Kernel:

```python
from aios_sdk.kernel import KernelClient
from aios_sdk.errors import AiosApiError
import uuid

kernel = KernelClient(base_url="https://gateway.aios.internal/v1/kernel", tenant="acme",
                      token_provider=get_cached_oidc_token)

agent_urn = "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6"

# 1) Planejar o próximo passo (broker -> 012-Planning)
plan = kernel.plan(agent_urn=agent_urn, goal="responder ao ticket #4821",
                    idempotency_key=str(uuid.uuid4()))

# 2) Executar a ferramenta indicada pelo plano (broker -> 015-Tool-Manager)
try:
    tool_result = kernel.invoke_tool(
        agent_urn=agent_urn,
        tool=plan.next_step.tool,
        action=plan.next_step.action,
        params=plan.next_step.params,
        idempotency_key=str(uuid.uuid4()),
    )
except AiosApiError as e:
    if e.code.startswith("AIOS-QUOTA-"):
        # Cota de invocação de ferramentas esgotada: encerrar graciosamente.
        kernel.suspend(agent_urn=agent_urn, idempotency_key=str(uuid.uuid4()))
        raise
    raise

# 3) Rotear a próxima inferência para o modelo mais adequado (broker -> 017-Model-Router)
inference = kernel.route_model(
    agent_urn=agent_urn,
    prompt_tokens_estimate=1200,
    quality_tier="standard",
    idempotency_key=str(uuid.uuid4()),
)
print("Modelo selecionado:", inference.model_id, "custo estimado:", inference.estimated_cost_usd)
```

Cada uma dessas três chamadas passa individualmente pelo `CapabilityEnforcer`
e pelo `ResourceQuotaManager` antes de ser encaminhada ao módulo dono — o
Agent Runtime nunca precisa saber os detalhes de FSM, cotas ou política; ele
apenas reage a sucesso ou a um envelope de erro RFC-7807.

---

## 14. Boas práticas ao consumir a ABI do Kernel

- **Sempre gere uma nova `Idempotency-Key` por intenção de mutação** — nunca
  reaproveite entre operações logicamente distintas (ver `./FAQ.md` §7.2).
- **Trate `retriable: false` como definitivo** — não retente `AIOS-CAP-0001`,
  `AIOS-KERNEL-0002` ou `AIOS-SYSCALL-0002` sem alterar a requisição.
- **Respeite `retryAfterMs`/`Retry-After`** em respostas 429 de cota — não
  implemente retry agressivo que ignore o sinal de backpressure.
- **Prefira assinar eventos a fazer polling** de `GetAcb`/`get_quota` para
  descobrir mudanças de estado — reduz carga no Kernel e latência de reação.
- **Nunca acesse `010/011/012/015/017` diretamente a partir de um agente** —
  sempre via syscall do Kernel, mesmo quando tecnicamente acessível na rede
  interna; isso quebra capability enforcement, cota e auditoria.
- **Propague sempre `traceparent`** recebido do chamador anterior, nunca
  gere um novo trace raiz no meio de um fluxo — isso preserva a
  correlação ponta a ponta exigida por NFR-011.

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
