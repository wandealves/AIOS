---
Documento: Examples
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0095, ADR-0096, ADR-0097
RFCs relacionados: RFC-0001, RFC-0090, RFC-0091
Depende de: 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — Examples

> Todos os exemplos assumem o Gateway YARP em `https://api.aios.local` (REST) e
> o serviço gRPC `aios.scheduler.v1.SchedulerService` acessível internamente em
> `scheduler.aios.internal:7009`. Endpoints, campos e erros seguem
> `./API.md`/`./_DESIGN_BRIEF.md` §5. Toda mutação usa `Idempotency-Key` e
> propaga `traceparent`/`X-AIOS-Tenant` conforme RFC-0001 §5.6. Os `tenant_id`
> usados (`acme`) e IDs ULID são ilustrativos.

---

## 1. Hello World — submeter uma tarefa (REST)

O exemplo mais simples: submeter uma `SchedulingRequest` mínima para uma tarefa
`INTERACTIVE` e receber a `SchedulingDecision`.

```bash
# Hello world: submissão mínima via REST
curl -sS -X POST https://api.aios.local/v1/scheduler/requests \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QA000000000000000A" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  -d '{
        "task_urn": "urn:aios:acme:task:01J9Z8QB1C2D3E4F5G6H7J8K9L",
        "agent_urn": "urn:aios:acme:agent:01J9Z8QB1C2D3E4F5G6H7J8K9M",
        "service_class": "INTERACTIVE",
        "priority_hint": 60,
        "quality_floor": 0.7,
        "submitted_at": "2026-07-20T12:00:00.000Z"
      }'
```

Resposta esperada (caminho feliz — admissão + enfileiramento imediato):

```json
{
  "decision_id": "01J9Z8QC2D3E4F5G6H7J8K9L0M",
  "request_id": "01J9Z8QB1C2D3E4F5G6H7J8K9N",
  "tenant_id": "acme",
  "outcome": "ADMITTED",
  "effective_priority": 58,
  "cost_score": 0.4123,
  "policy_id": "heuristic",
  "policy_version": "heuristic@1",
  "decided_at": "2026-07-20T12:00:00.012Z",
  "latency_decision_ms": 4.7
}
```

> Nota: `outcome=ADMITTED` significa que a tarefa entrou na fila
> (`QUEUED`); o despacho efetivo (`SCHEDULED`) é comunicado de forma
> assíncrona pelo evento `aios.acme.task.execution.scheduled` (ver §5).

---

## 2. Hello World — mesma submissão via gRPC (grpcurl)

```bash
# Equivalente ao exemplo 1, via gRPC (grpcurl para fins de exemplo/CLI)
grpcurl -plaintext \
  -H "x-aios-tenant: acme" \
  -H "idempotency-key: 01J9Z8QA000000000000000A" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  -d '{
        "task_urn": "urn:aios:acme:task:01J9Z8QB1C2D3E4F5G6H7J8K9L",
        "agent_urn": "urn:aios:acme:agent:01J9Z8QB1C2D3E4F5G6H7J8K9M",
        "service_class": "INTERACTIVE",
        "priority_hint": 60,
        "quality_floor": 0.7
      }' \
  scheduler.aios.internal:7009 aios.scheduler.v1.SchedulerService/Submit
```

---

## 3. SDK Python — submissão com tratamento de erro

```python
# Exemplo de uso do AIOS SDK (031-SDK) para submeter uma tarefa ao Scheduler.
# O SDK encapsula geração de Idempotency-Key, propagação de traceparent e
# parsing do envelope de erro RFC 7807 (RFC-0001 §5.4).

from aios_sdk import SchedulerClient, SchedulingRequest, SchedulerError

client = SchedulerClient(tenant="acme")

request = SchedulingRequest(
    task_urn="urn:aios:acme:task:01J9Z8QD3E4F5G6H7J8K9L0M1N",
    agent_urn="urn:aios:acme:agent:01J9Z8QD3E4F5G6H7J8K9L0M1P",
    service_class="BATCH",
    priority_hint=40,
    quality_floor=0.6,
)

try:
    decision = client.submit(request)  # Idempotency-Key gerado automaticamente
    print(f"outcome={decision.outcome} effective_priority={decision.effective_priority}")
except SchedulerError as err:
    # err.code segue o padrão AIOS-SCHED-<NNNN> (ver ./API.md §4)
    if err.code == "AIOS-SCHED-0003":
        print(f"Sistema saturado; retry em {err.retry_after_ms} ms")
    elif err.code == "AIOS-SCHED-0002":
        print("Orçamento insuficiente para o custo estimado da tarefa.")
    else:
        raise
```

---

## 4. Tarefa com deadline (EDF) e classe `SYSTEM`

Tarefas de manutenção interna com prazo rígido usam `deadline_at` para ativar
a lógica *Earliest Deadline First* (FR-012).

```bash
curl -sS -X POST https://api.aios.local/v1/scheduler/requests \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QE4F5G6H7J8K9L0M1N2P" \
  -d '{
        "task_urn": "urn:aios:acme:task:01J9Z8QE4F5G6H7J8K9L0M1N3Q",
        "agent_urn": "urn:aios:acme:agent:01J9Z8QE4F5G6H7J8K9L0M1N4R",
        "service_class": "SYSTEM",
        "priority_hint": 80,
        "deadline_at": "2026-07-20T12:05:00.000Z",
        "quality_floor": 0.5
      }'
```

Se o `deadline_at` já tiver vencido no instante da admissão, a resposta é um
erro (não uma decisão de sucesso):

```json
{
  "type": "https://docs.aios/errors/deadline-expired",
  "title": "Deadline Already Expired",
  "status": 410,
  "code": "AIOS-SCHED-0008",
  "detail": "deadline_at is in the past at admission time.",
  "instance": "urn:aios:acme:task:01J9Z8QE4F5G6H7J8K9L0M1N3Q",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-07-20T12:05:01.203Z",
  "retriable": false
}
```

---

## 5. Consumindo eventos de ciclo de decisão (Python + NATS)

```python
# Assina os eventos de execução de tarefa do tenant "acme" para acompanhar
# o ciclo de vida de uma SchedulingRequest de ponta a ponta.
import asyncio
import json
from nats.aio.client import Client as NATS

async def main():
    nc = NATS()
    await nc.connect("nats://nats.aios.internal:4222")

    async def handler(msg):
        event = json.loads(msg.data.decode())
        # Consumidores DEVEM deduplicar por event.id (at-least-once, RFC-0001 §5.2)
        print(f"[{event['type']}] taskUrn={event['data'].get('taskUrn')} "
              f"tenant={event['tenant']} id={event['id']}")

    # Wildcard: todos os eventos de execução de tarefa do tenant acme
    await nc.subscribe("aios.acme.task.execution.*", cb=handler)
    # Backpressure do próprio Scheduler
    await nc.subscribe("aios.acme.scheduler.backpressure.changed", cb=handler)

    await asyncio.sleep(3600)

asyncio.run(main())
```

Saída típica ao longo do ciclo de vida de uma tarefa:

```
[aios.task.execution.admitted]   taskUrn=urn:aios:acme:task:01J9...L id=01J9Z8QF...
[aios.task.execution.scheduled]  taskUrn=urn:aios:acme:task:01J9...L id=01J9Z8QG...
[aios.task.execution.completed]  taskUrn=urn:aios:acme:task:01J9...L id=01J9Z8QH...
```

---

## 6. Cancelando uma tarefa em fila

```bash
# Cancela uma tarefa ainda em QUEUED ou SCHEDULED (idempotente).
curl -sS -X DELETE \
  https://api.aios.local/v1/scheduler/requests/01J9Z8QB1C2D3E4F5G6H7J8K9N \
  -H "X-AIOS-Tenant: acme" \
  -H "Idempotency-Key: 01J9Z8QI5G6H7J8K9L0M1N2P3Q"
```

Se a tarefa já estiver em estado terminal (`COMPLETED`, `FAILED`, `REJECTED`,
`EXPIRED`, `CANCELLED`), o cancelamento retorna erro de conflito, não sucesso
silencioso:

```json
{
  "type": "https://docs.aios/errors/terminal-state-conflict",
  "title": "Task In Terminal State",
  "status": 409,
  "code": "AIOS-SCHED-0011",
  "detail": "Task is already in a terminal state; cancel is not applicable.",
  "traceId": "9f1e5c2a7b3d4e6f8091a2b3c4d5e6f7",
  "timestamp": "2026-07-20T12:10:00.000Z",
  "retriable": false
}
```

---

## 7. Preempção dirigida (administrativa)

Requer a role `scheduler-admin`; toda preempção passa pelo PDP (022) antes de
ser executada.

```bash
curl -sS -X POST https://api.aios.local/v1/scheduler/preemptions \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -H "Authorization: Bearer <token-com-role-scheduler-admin>" \
  -H "Idempotency-Key: 01J9Z8QJ6H7J8K9L0M1N2P3Q4R" \
  -d '{
        "victim_task_urn": "urn:aios:acme:task:01J9Z8QK7J8K9L0M1N2P3Q4R5S",
        "reason": "incident-mitigation",
        "grace_period_ms": 5000
      }'
```

Resposta:

```json
{
  "decision_id": "01J9Z8QL8K9L0M1N2P3Q4R5S6T",
  "outcome": "PREEMPTED",
  "victim_task_urn": "urn:aios:acme:task:01J9Z8QK7J8K9L0M1N2P3Q4R5S",
  "grace_period_ms": 5000,
  "decided_at": "2026-07-20T12:11:00.000Z"
}
```

---

## 8. Consultando a decisão (proveniência) de uma requisição

```bash
curl -sS https://api.aios.local/v1/scheduler/requests/01J9Z8QB1C2D3E4F5G6H7J8K9N \
  -H "X-AIOS-Tenant: acme"
```

```json
{
  "decision_id": "01J9Z8QC2D3E4F5G6H7J8K9L0M",
  "request_id": "01J9Z8QB1C2D3E4F5G6H7J8K9N",
  "outcome": "SCHEDULED",
  "effective_priority": 58,
  "cost_score": 0.4123,
  "alpha": 0.5, "beta": 0.3, "gamma": 0.2,
  "placement_target": {"shard": 17, "node_id": "rt-pool-03", "runtime_pool": "python-default"},
  "policy_id": "heuristic",
  "policy_version": "heuristic@1",
  "decided_at": "2026-07-20T12:00:00.012Z",
  "latency_decision_ms": 4.7
}
```

---

## 9. Lidando com backpressure (`AIOS-SCHED-0003`) com retry e jitter

```python
# Exemplo avançado: cliente que respeita Retry-After com backoff + jitter,
# em vez de martelar o Scheduler durante saturação (evita efeito manada).
import random
import time
from aios_sdk import SchedulerClient, SchedulingRequest, SchedulerError

client = SchedulerClient(tenant="acme")
request = SchedulingRequest(
    task_urn="urn:aios:acme:task:01J9Z8QM9L0M1N2P3Q4R5S6T7U",
    agent_urn="urn:aios:acme:agent:01J9Z8QM9L0M1N2P3Q4R5S6T8V",
    service_class="BACKGROUND",
    priority_hint=20,
)

max_attempts = 5
for attempt in range(1, max_attempts + 1):
    try:
        decision = client.submit(request)
        print(f"Admitido na tentativa {attempt}: {decision.outcome}")
        break
    except SchedulerError as err:
        if err.code == "AIOS-SCHED-0003" and err.retriable and attempt < max_attempts:
            base_ms = err.retry_after_ms or 1500
            jitter_ms = random.uniform(0, base_ms * 0.3)
            sleep_s = (base_ms + jitter_ms) / 1000.0
            print(f"Backpressure (tentativa {attempt}); aguardando {sleep_s:.2f}s")
            time.sleep(sleep_s)
            continue
        raise
```

---

## 10. Streaming do estado das filas (observabilidade)

```bash
# gRPC streaming server-side: observa profundidade de fila por shard/classe.
grpcurl -plaintext \
  -H "x-aios-tenant: acme" \
  -d '{"tenant_id": "acme", "service_class": "INTERACTIVE"}' \
  scheduler.aios.internal:7009 aios.scheduler.v1.SchedulerService/StreamQueueState
```

Saída (uma mensagem por atualização, fluxo contínuo):

```json
{"shard": 3,  "service_class": "INTERACTIVE", "queue_depth": 12, "oldest_wait_ms": 340, "pressure": 0.42}
{"shard": 7,  "service_class": "INTERACTIVE", "queue_depth": 41, "oldest_wait_ms": 1210, "pressure": 0.81}
```

Equivalente REST (snapshot pontual, não streaming):

```bash
curl -sS "https://api.aios.local/v1/scheduler/queues?tenant=acme&service_class=INTERACTIVE" \
  -H "X-AIOS-Tenant: acme"
```

---

## 11. Ajustando pesos de custo/latência por tenant (avançado)

Requer role `scheduler-admin` e é governado por `022-Policy` (mudanças de
peso/cota por tenant DEVEM ser autorizadas).

```bash
# Lê a política ativa (pesos alpha/beta/gamma) do tenant.
curl -sS https://api.aios.local/v1/scheduler/policy?tenant=acme \
  -H "X-AIOS-Tenant: acme"

# Atualiza os pesos: tenant prioriza fortemente custo (beta) sobre latência.
curl -sS -X PUT https://api.aios.local/v1/scheduler/policy \
  -H "Content-Type: application/json" \
  -H "X-AIOS-Tenant: acme" \
  -H "Authorization: Bearer <token-com-role-scheduler-admin>" \
  -H "Idempotency-Key: 01J9Z8QN0M1N2P3Q4R5S6T7U8V" \
  -d '{
        "tenant_id": "acme",
        "weights": {"alpha": 0.2, "beta": 0.7, "gamma": 0.1}
      }'
```

> Os pesos DEVEM satisfazer `alpha + beta + gamma ≈ 1` — o servidor normaliza
> na leitura, mas rejeita configurações grosseiramente inconsistentes com
> `AIOS-SCHED-0005` (requisição inválida).

---

## 12. Confirmação de sinal de runtime (`ReportRuntimeSignal`) — uso interno do 007

Este método é chamado pelo **Runtime Supervisor (007)**, não por clientes de
agente comuns; incluído aqui para completude de integração.

```bash
grpcurl -plaintext \
  -H "x-aios-tenant: acme" \
  -d '{
        "task_urn": "urn:aios:acme:task:01J9Z8QB1C2D3E4F5G6H7J8K9L",
        "signal": "STARTED",
        "signal_id": "01J9Z8QO1N2P3Q4R5S6T7U8V9W",
        "node_id": "rt-pool-03"
      }' \
  scheduler.aios.internal:7009 aios.scheduler.v1.SchedulerService/ReportRuntimeSignal
```

---

## 13. Ciclo completo — do "hello world" ao avançado (fluxo único)

Resumo integrando os exemplos anteriores num único fluxo mental, do ponto de
vista de um desenvolvedor de agentes usando o SDK Python:

```python
from aios_sdk import SchedulerClient, SchedulingRequest, SchedulerError

client = SchedulerClient(tenant="acme")

# 1. Submeter (hello world)
req = SchedulingRequest(
    task_urn="urn:aios:acme:task:01J9Z8QP2P3Q4R5S6T7U8V9W0X",
    agent_urn="urn:aios:acme:agent:01J9Z8QP2P3Q4R5S6T7U8V9W1Y",
    service_class="INTERACTIVE",
    priority_hint=70,
    quality_floor=0.75,
)
decision = client.submit(req)
print(decision.outcome, decision.effective_priority)

# 2. Acompanhar via evento (assinatura assíncrona; ver exemplo 5)
# 3. Consultar proveniência a qualquer momento (ver exemplo 8)
snapshot = client.get_decision(decision.request_id)
print(snapshot.placement_target)

# 4. Cancelar se não for mais necessária (ver exemplo 6)
if snapshot.outcome in ("ADMITTED", "SCHEDULED"):
    client.cancel(decision.request_id)
```

---

## 14. Referências

- Contratos completos de API: `./API.md`
- Catálogo de eventos: `./Events.md`
- Catálogo de erros: `./API.md` §Erros / `_DESIGN_BRIEF.md` §5.4
- Máquina de estados: `./StateMachine.md`
- Configuração (pesos, limiares de backpressure): `./Configuration.md`

---

*Fim dos Exemplos do Módulo 009-Scheduler.*
