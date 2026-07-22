---
Documento: Examples
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0072
RFCs relacionados: RFC-0001, RFC-0070
Depende de: ./API.md, ./Configuration.md, ../031-SDK/
---

# 007-Agent-Runtime — Examples

> O Agent Runtime é um serviço **interno** (§N-08, `./Vision.md` §3.2); não
> há SDK de usuário final para ele. Os exemplos abaixo mostram como o
> **Runtime Supervisor**/control plane interage com o runtime via
> `aios.runtime.v1`, e como um desenvolvedor de plataforma (não um usuário
> final) inspeciona/testa uma instância localmente. Do "hello world" (boot +
> um passo simples) ao avançado (checkpoint/resume, streaming, tratamento de
> falha), todos os exemplos usam os endpoints reais de `./API.md`.

## Índice

1. Pré-requisitos
2. Exemplo 1 — "Hello World": Boot + SubmitTask simples
3. Exemplo 2 — Streaming de passos com `StreamSteps`
4. Exemplo 3 — Suspensão e retomada (checkpoint)
5. Exemplo 4 — Tratamento de erro de cota
6. Exemplo 5 — Consulta administrativa via REST
7. Exemplo 6 — Testando localmente com `grpcurl`
8. Referências

---

## 1. Pré-requisitos

- Ambiente `docker-compose.dev.yml` de `./Deployment.md` §1.1 em execução
  (`agent-runtime-pool`, `runtime-supervisor`, `nats`, `redis`).
- Cliente gRPC gerado a partir do `.proto` de `./API.md` §2 (Python: `grpcio-tools`;
  .NET: `Grpc.Tools`, usado pelo Runtime Supervisor real).
- Certificados mTLS de desenvolvimento provisionados por `021-Security`
  (ou `--insecure` apenas em ambiente local isolado).

---

## 2. Exemplo 1 — "Hello World": Boot + `SubmitTask` Simples

### 2.1 Cliente Python (simulando o Runtime Supervisor para fins de teste)

```python
import grpc
from aios.runtime.v1 import runtime_control_pb2 as rc, runtime_control_pb2_grpc as rc_grpc
from aios.runtime.v1 import agent_execution_pb2 as ae, agent_execution_pb2_grpc as ae_grpc

channel = grpc.secure_channel("agent-runtime-1:50051", grpc.ssl_channel_credentials())
control = rc_grpc.RuntimeControlStub(channel)
execution = ae_grpc.AgentExecutionStub(channel)

# 1. Boot — materializa o agente a partir de um AgentSpec mínimo
boot_reply = control.Boot(rc.BootRequest(
    idempotency_key="01J9Z9AABBCCDDEEFFGGHHJJKK",
    tenant_id="acme",
    agent_urn="urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    agent_spec=b'{"goal": "Responder ao usuario: qual a capital da Franca?"}',
    traceparent="00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
))
print(f"session_id={boot_reply.session_id} state={boot_reply.state} cold_start_ms={boot_reply.cold_start_ms}")

# 2. SubmitTask — inicia o loop ReAct e consome o stream de resultado
task = ae.TaskRequest(
    idempotency_key="01J9Z9TASK0000000000000001",
    tenant_id="acme",
    task_urn="urn:aios:acme:task:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
    agent_urn="urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    goal_payload=b'{"objective": "Responder a pergunta do usuario com uma frase curta."}',
    traceparent="00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
)
for event in execution.SubmitTask(task):
    if event.HasField("step"):
        print(f"[step {event.step.index}] phase={event.step.phase} status={event.step.status} latency_ms={event.step.latency_ms}")
    elif event.HasField("completed"):
        print(f"DONE: cost_usd={event.completed.cost_usd} tokens={event.completed.tokens}")
    elif event.HasField("failed"):
        print(f"FAILED: {event.failed.code} — {event.failed.reason} (retriable={event.failed.retriable})")
```

**Saída esperada (fluxo feliz):**

```
session_id=01J9ZA1AGENTSESSN000000001 state=ready cold_start_ms=187
[step 0] phase=think status=ok latency_ms=8
[step 1] phase=act status=ok latency_ms=6
[step 2] phase=observe status=ok latency_ms=3
[step 3] phase=think status=ok latency_ms=5
DONE: cost_usd=0.0021 tokens=412
```

---

## 3. Exemplo 2 — Streaming de Passos com `StreamSteps`

> Uso típico: um serviço intermediário do control plane (nunca o usuário
> final diretamente) observa o progresso de uma sessão em andamento para
> repassar ao WebConsole.

```python
from aios.runtime.v1 import agent_execution_pb2 as ae

for step_event in execution.StreamSteps(ae.SessionRef(session_id="01J9ZA1AGENTSESSN000000001")):
    print(f"step={step_event.step_id} index={step_event.index} phase={step_event.phase} status={step_event.status}")
```

Se a conexão cair e for reaberta, o consumidor **DEVE** deduplicar por
`step_id` (semântica *at-least-once*, RFC-0001 §5.5) — reabrir o stream pode
reentregar os últimos passos já vistos.

---

## 4. Exemplo 3 — Suspensão e Retomada (Checkpoint)

```python
# Supervisor decide suspender (ex.: preempção do Scheduler)
suspend_reply = control.Suspend(rc.SuspendRequest(
    session_id="01J9ZA1AGENTSESSN000000001",
    reason="preemption",
))
print(f"checkpoint_id={suspend_reply.checkpoint_id} step_cursor={suspend_reply.step_cursor}")

# ... mais tarde, em outra instância de runtime (mesmo ou outro nó) ...
resume_reply = control.Resume(rc.ResumeRequest(
    checkpoint_id=suspend_reply.checkpoint_id,
))
print(f"session_id={resume_reply.session_id} state={resume_reply.state}")
```

**Saída esperada:**

```
checkpoint_id=01J9ZCHKPNT000000000000001 step_cursor=4
session_id=01J9ZA1AGENTSESSN000000001 state=thinking
```

A sessão retoma exatamente do `step_cursor=4`, sem repetir os passos 0–3
(ver `./SequenceDiagrams.md` §6 para o fluxo completo, incluindo o caso de
crash entre `Suspend` e o `Resume` planejado).

---

## 5. Exemplo 4 — Tratamento de Erro de Cota

```python
try:
    for event in execution.SubmitTask(task_com_orcamento_baixo):
        if event.HasField("failed") and event.failed.code == "AIOS-RUNTIME-0004":
            print("Cota local esgotada — verificar budget.max_tokens/max_cost_usd do tenant.")
            # Ação recomendada: consultar GetStatus para ver `consumed` vs `budget`
            status = control.GetStatus(rc.StatusRequest(runtime_id=boot_reply.runtime_id))
            print(f"consumed={status.consumed} budget={status.budget}")
except grpc.RpcError as e:
    print(f"gRPC error: {e.code()} — {e.details()}")
```

---

## 6. Exemplo 5 — Consulta Administrativa via REST

```bash
# Readiness (usado internamente pelo Supervisor, não por usuário final)
curl -sS --cert client.pem --key client-key.pem \
  https://agent-runtime-1.internal:8443/v1/runtime/readyz

# Status de uma instância
curl -sS --cert client.pem --key client-key.pem \
  https://agent-runtime-1.internal:8443/v1/runtime/01J9ZA1RVNTMEXEC0000000001/status | jq .

# Snapshot de sessão (respeitando RLS por tenant)
curl -sS --cert client.pem --key client-key.pem \
  -H "X-AIOS-Tenant: acme" \
  https://agent-runtime-1.internal:8443/v1/runtime/sessions/01J9ZA1AGENTSESSN000000001 | jq .
```

---

## 7. Exemplo 6 — Testando Localmente com `grpcurl`

```bash
grpcurl -cacert ca.pem -cert client.pem -key client-key.pem \
  -d '{
        "idempotency_key": "01J9Z9AABBCCDDEEFFGGHHJJKK",
        "tenant_id": "acme",
        "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
        "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
      }' \
  agent-runtime-1.internal:50051 aios.runtime.v1.RuntimeControl/GetStatus
```

---

## 8. Referências

- Contratos completos (proto/REST): `./API.md`.
- Chaves de configuração usadas pelos exemplos (`budget.*`, `loop.*`): `./Configuration.md`.
- Fluxos de sequência correspondentes: `./SequenceDiagrams.md`.
- Perguntas frequentes sobre uso e limites: `./FAQ.md`.
