---
Documento: Events.md
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0093, ADR-0095, ADR-0097 (a propor)
RFCs relacionados: RFC-0001 (baseline); RFC-0090 (a propor)
Depende de: 003-RFC/RFC-0001, 004-API, 006-Kernel, 007-Agent-Runtime, 020-Communication, 022-Policy, 025-Audit, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — Events

> **Escopo.** Catálogo canônico dos eventos NATS/JetStream produzidos e
> consumidos pelo Scheduler. O envelope (CloudEvents-compatível), a convenção
> de subjects `aios.<tenant>.<dominio>.<entidade>.<acao>` e a semântica de
> entrega/idempotência **DEVEM** seguir
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3, §5.5 — **não
> redefinidas aqui**. Consistência absoluta com `./_DESIGN_BRIEF.md` §6.

---

## 1. Convenções gerais

| Aspecto | Regra |
|---------|-------|
| Envelope | CloudEvents 1.0-compatível, campos obrigatórios `specversion, id, source, type, subject, time, tenant, traceparent, datacontenttype, dataschema, data` (RFC-0001 §5.2). |
| Domínios usados | `task` (entidade `execution`) e `scheduler` (entidades `backpressure`, `decision`). |
| `source` | `urn:aios:<tenant>:service:scheduler`. |
| Semântica de entrega | **At-least-once**; consumidores DEVEM deduplicar por `event.id` (RFC-0001 §5.5). |
| Ordenação | Garantida **por subject/chave de partição** (`task_urn`), não globalmente — ver §4. |
| Publicação | Via `DecisionJournal` (outbox transacional): grava `SchedulingDecision` em PostgreSQL e publica atomicamente no mesmo commit lógico (padrão *Outbox*, `../040-Glossary/Glossary.md`). |
| Versionamento de schema | `dataschema` no formato `aios://schemas/<type-sem-prefixo-aios>/<versão>`; evolução aditiva (novos campos opcionais); breaking change incrementa versão e coexiste ≥ 1 versão (RFC-0001 §5.7). |
| Namespace | `aios.<tenant>.*` isola tráfego por tenant (contas NATS, RFC-0001 §5.3). |
| DLQ | Eventos não processáveis (schema inválido, handler falha após `max_deliver`) vão para stream DLQ `aios-dlq.scheduler` para replay manual/automatizado (ver `FailureRecovery.md`). |

---

## 2. Eventos Emitidos (produtor = Scheduler)

### 2.1 Catálogo resumido

| # | Subject | Ação | Produzido por | Quando |
|---|---------|------|----------------|--------|
| 1 | `aios.<tenant>.task.execution.admitted` | `admitted` | `AdmissionController` | Admissão aceita (RECEIVED→ADMITTED) |
| 2 | `aios.<tenant>.task.execution.rejected` | `rejected` | `AdmissionController` / `DispatchLoop` | Admissão negada, deadline vencido ou cancelamento |
| 3 | `aios.<tenant>.task.execution.scheduled` | `scheduled` | `DispatchLoop` | Despacho ao Runtime Supervisor (QUEUED→SCHEDULED) |
| 4 | `aios.<tenant>.task.execution.preempted` | `preempted` | `PreemptionManager` | Vítima escolhida para preempção |
| 5 | `aios.<tenant>.task.execution.resumed` | `resumed` | `PreemptionManager` / `BackpressureController` | Retomada pós-preempção/backpressure |
| 6 | `aios.<tenant>.task.execution.completed` | `completed` | `EventPublisher` (observado do Runtime) | Runtime concluiu execução |
| 7 | `aios.<tenant>.task.execution.failed` | `failed` | `EventPublisher` (observado do Runtime) | Runtime falhou execução |
| 8 | `aios.<tenant>.scheduler.backpressure.changed` | `changed` | `BackpressureController` | Nível de saturação muda (accept/defer/reject) |
| 9 | `aios.<tenant>.scheduler.decision.recorded` | `recorded` | `DecisionJournal` | Proveniência de decisão gravada (para 025/023) |

### 2.2 `aios.<tenant>.task.execution.admitted`

**Produtor:** `AdmissionController`. **Consumidores conhecidos:** 006-Kernel
(atualiza ACB), 025-Audit (trilha), 024-Observability (métricas de admissão).

```json
{
  "specversion": "1.0",
  "id": "01J9ZE000000000000000001",
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.task.execution.admitted",
  "subject": "urn:aios:acme:task:01J9ZA100000000000000010",
  "time": "2026-07-20T12:00:00.010Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/task.execution.admitted/1",
  "data": {
    "taskUrn": "urn:aios:acme:task:01J9ZA100000000000000010",
    "requestId": "01J9ZA000000000000000010",
    "decisionId": "01J9ZA300000000000000012",
    "effectivePriority": 68,
    "serviceClass": "INTERACTIVE"
  }
}
```

**Schema (`data`):** `taskUrn` (string, URN, obrigatório), `requestId` (ULID,
obrigatório), `decisionId` (ULID, obrigatório), `effectivePriority` (int 0–140,
obrigatório — inclui boost de aging), `serviceClass` (enum, obrigatório).

### 2.3 `aios.<tenant>.task.execution.rejected`

**Produtor:** `AdmissionController` (negação de admissão), `DispatchLoop`
(deadline/cancelamento). **Consumidores conhecidos:** 006-Kernel, 025-Audit,
024-Observability.

```json
{
  "specversion": "1.0",
  "id": "01J9ZE000000000000000002",
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.task.execution.rejected",
  "subject": "urn:aios:acme:task:01J9ZA100000000000000020",
  "time": "2026-07-20T12:01:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b8-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/task.execution.rejected/1",
  "data": {
    "taskUrn": "urn:aios:acme:task:01J9ZA100000000000000020",
    "requestId": "01J9ZA000000000000000020",
    "reasonCode": "AIOS-SCHED-0003",
    "retriable": true,
    "retryAfterMs": 1500
  }
}
```

**Schema (`data`):** `taskUrn` (string, obrigatório), `requestId` (ULID,
obrigatório), `reasonCode` (string `AIOS-SCHED-<NNNN>`, obrigatório),
`retriable` (bool, obrigatório), `retryAfterMs` (int, opcional — presente
apenas quando `retriable=true`).

### 2.4 `aios.<tenant>.task.execution.scheduled`

**Produtor:** `DispatchLoop`. **Consumidores conhecidos:** 007-Agent-Runtime
(Runtime Supervisor materializa o placement), 024-Observability, 025-Audit.

```json
{
  "specversion": "1.0",
  "id": "01J9ZE000000000000000003",
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.task.execution.scheduled",
  "subject": "urn:aios:acme:task:01J9ZA100000000000000010",
  "time": "2026-07-20T12:00:00.020Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b9-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/task.execution.scheduled/1",
  "data": {
    "taskUrn": "urn:aios:acme:task:01J9ZA100000000000000010",
    "decisionId": "01J9ZA300000000000000012",
    "placement": { "shard": 17, "node": "node-07", "pool": "python-default" }
  }
}
```

**Schema (`data`):** `taskUrn` (string, obrigatório), `decisionId` (ULID,
obrigatório), `placement.shard` (int 0..N-1, obrigatório),
`placement.node` (string, obrigatório), `placement.pool` (string, obrigatório).
> **Nota:** o consumo deste evento pelo Runtime Supervisor (007) para
> materializar o *spawn* é o gatilho canônico da transição SCHEDULED→RUNNING
> (confirmada de volta via `aios.<tenant>.agent.lifecycle.spawned`, §3.1).

### 2.5 `aios.<tenant>.task.execution.preempted`

**Produtor:** `PreemptionManager`. **Consumidores conhecidos:**
007-Agent-Runtime (executa suspend), 025-Audit, 024-Observability.

```json
{
  "specversion": "1.0",
  "id": "01J9ZE000000000000000004",
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.task.execution.preempted",
  "subject": "urn:aios:acme:task:01J9ZA100000000000000030",
  "time": "2026-07-20T12:02:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902ba-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/task.execution.preempted/1",
  "data": {
    "taskUrn": "urn:aios:acme:task:01J9ZA100000000000000030",
    "byTaskUrn": "urn:aios:acme:task:01J9ZA100000000000000031",
    "gracePeriodMs": 2000
  }
}
```

**Schema (`data`):** `taskUrn` (string, vítima, obrigatório), `byTaskUrn`
(string, tarefa que motivou a preempção, obrigatório), `gracePeriodMs` (int
≥ 0, obrigatório).

### 2.6 `aios.<tenant>.task.execution.resumed`

**Produtor:** `PreemptionManager` / `BackpressureController` (após liberação de
capacidade). **Consumidores conhecidos:** 007-Agent-Runtime, 025-Audit.

```json
{
  "specversion": "1.0",
  "id": "01J9ZE000000000000000005",
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.task.execution.resumed",
  "subject": "urn:aios:acme:task:01J9ZA100000000000000030",
  "time": "2026-07-20T12:04:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902bb-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/task.execution.resumed/1",
  "data": {
    "taskUrn": "urn:aios:acme:task:01J9ZA100000000000000030",
    "decisionId": "01J9ZA300000000000000033"
  }
}
```

**Schema (`data`):** `taskUrn` (string, obrigatório), `decisionId` (ULID,
obrigatório — nova decisão gerada pelo re-scheduling).

### 2.7 `aios.<tenant>.task.execution.completed`

**Produtor:** `EventPublisher`, ao observar `ReportRuntimeSignal(type=COMPLETED)`
do Runtime Supervisor. **Consumidores conhecidos:** 026-Cost-Optimizer
(fecha contabilidade), 023-Learning (treino), 025-Audit, 024-Observability.

```json
{
  "specversion": "1.0",
  "id": "01J9ZE000000000000000006",
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.task.execution.completed",
  "subject": "urn:aios:acme:task:01J9ZA100000000000000010",
  "time": "2026-07-20T12:05:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902bc-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/task.execution.completed/1",
  "data": {
    "taskUrn": "urn:aios:acme:task:01J9ZA100000000000000010",
    "costUsd": 0.0187,
    "tokens": 1340,
    "latencyMs": 412
  }
}
```

**Schema (`data`):** `taskUrn` (string, obrigatório), `costUsd` (number ≥ 0,
obrigatório), `tokens` (int ≥ 0, obrigatório), `latencyMs` (int ≥ 0,
obrigatório).

### 2.8 `aios.<tenant>.task.execution.failed`

**Produtor:** `EventPublisher`, ao observar `ReportRuntimeSignal(type=FAILED)`.
**Consumidores conhecidos:** 025-Audit, 024-Observability, 023-Learning
(análise de falhas).

```json
{
  "specversion": "1.0",
  "id": "01J9ZE000000000000000007",
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.task.execution.failed",
  "subject": "urn:aios:acme:task:01J9ZA100000000000000041",
  "time": "2026-07-20T12:03:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902bd-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/task.execution.failed/1",
  "data": {
    "taskUrn": "urn:aios:acme:task:01J9ZA100000000000000041",
    "errorCode": "AIOS-RUNTIME-0007",
    "willRetry": true
  }
}
```

**Schema (`data`):** `taskUrn` (string, obrigatório), `errorCode` (string,
obrigatório), `willRetry` (bool, obrigatório — reflete `retry.max_attempts`
não esgotado).

### 2.9 `aios.<tenant>.scheduler.backpressure.changed`

**Produtor:** `BackpressureController`. **Consumidores conhecidos:**
006-Kernel (ajusta admissão upstream), 020-Communication (Gateway/produtores),
024-Observability (alertas).

```json
{
  "specversion": "1.0",
  "id": "01J9ZE000000000000000008",
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.scheduler.backpressure.changed",
  "subject": "urn:aios:acme:shard:17",
  "time": "2026-07-20T12:06:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902be-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/scheduler.backpressure.changed/1",
  "data": {
    "level": "reject",
    "retryAfterMs": 1500,
    "shard": 17
  }
}
```

**Schema (`data`):** `level` (enum `accept`\|`defer`\|`reject`, obrigatório),
`retryAfterMs` (int ≥ 0, obrigatório — `0` quando `level=accept`), `shard`
(int, obrigatório).

### 2.10 `aios.<tenant>.scheduler.decision.recorded`

**Produtor:** `DecisionJournal` (outbox), imediatamente após persistir a
`SchedulingDecision`. **Consumidores conhecidos:** 025-Audit (trilha imutável),
023-Learning (dataset de treino da `LearnedPolicy` v2).

```json
{
  "specversion": "1.0",
  "id": "01J9ZE000000000000000009",
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.scheduler.decision.recorded",
  "subject": "urn:aios:acme:task:01J9ZA100000000000000010",
  "time": "2026-07-20T12:00:00.015Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902bf-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/scheduler.decision.recorded/1",
  "data": {
    "decisionId": "01J9ZA300000000000000012",
    "policyVersion": "heuristic@1",
    "featureVectorRef": "urn:aios:acme:decision:01J9ZA300000000000000012#features"
  }
}
```

**Schema (`data`):** `decisionId` (ULID, obrigatório), `policyVersion`
(string, obrigatório), `featureVectorRef` (string URN, obrigatório — ponteiro
para o `feature_vector` completo em `scheduling_decision`, evitando payload de
evento inchado; ver `Database.md`).

---

## 3. Eventos Consumidos (consumidor = Scheduler)

### 3.1 Catálogo resumido

| # | Subject | Produtor | Consumido por (interno) | Uso |
|---|---------|----------|--------------------------|-----|
| 1 | `aios.<tenant>.agent.lifecycle.spawned` | 006/007 | `ReconciliationWorker` | Confirma materialização de placement (SCHEDULED→RUNNING). |
| 2 | `aios.<tenant>.agent.lifecycle.suspended` | 007 | `PreemptionManager` | Confirma preempção; libera slot. |
| 3 | `aios.<tenant>.agent.lifecycle.terminated` | 007 | `QuotaLedger` / `CapacityProvider` | Fecha contabilidade de slot. |
| 4 | `aios.<tenant>.cost.budget.updated` | 026 | `BudgetClient` (cache) | Atualiza orçamento disponível para admissão. |
| 5 | `aios.<tenant>.cluster.capacity.changed` | 027 | `CapacityProvider` / `ShardRouter` | Atualiza topologia/capacidade (N shards, nós). |
| 6 | `aios.<tenant>.policy.decision.updated` | 022 | `PolicyClient` (cache) | Invalida cache de decisões de política (PDP). |

### 3.2 `aios.<tenant>.agent.lifecycle.spawned`

**Produtor:** 006-Kernel/007-Agent-Runtime. **Uso:** o `ReconciliationWorker`
casa o evento por `taskUrn`/`decisionId` embutido no `subject`/`data` para
confirmar que o Runtime materializou o placement; a ausência deste evento
dentro de `scheduler.lease.ttl_ms` aciona expiração de lease e re-schedule
(§9 do `_DESIGN_BRIEF.md`). **Ordenação exigida:** por `task_urn` (o Scheduler
NÃO DEVE processar `spawned` antes do `scheduled` correspondente; se ocorrer
fora de ordem, o handler reconsulta `GetDecision` antes de agir).

### 3.3 `aios.<tenant>.agent.lifecycle.suspended`

**Produtor:** 007-Agent-Runtime. **Uso:** confirma que a suspensão solicitada
por `PreemptionManager` foi efetivada; libera o slot reservado apenas após
esta confirmação (evita liberar capacidade antes do Runtime concluir o
suspend, prevenindo overcommit).

### 3.4 `aios.<tenant>.agent.lifecycle.terminated`

**Produtor:** 007-Agent-Runtime. **Uso:** fecha a contabilidade de slot em
`QuotaLedger`/`CapacityProvider` independentemente do desfecho observado via
`ReportRuntimeSignal` (defesa em profundidade contra sinais perdidos).

### 3.5 `aios.<tenant>.cost.budget.updated`

**Produtor:** 026-Cost-Optimizer. **Uso:** atualiza o cache local de
orçamento consultado por `CostEstimator`/`BudgetClient` no caminho quente de
`Submit`, evitando chamada síncrona bloqueante a 026 (NFR-001, p99 ≤ 20 ms).
Consumo assíncrono — o cache pode estar até `scheduler.capacity.heartbeat_ttl_ms`
desatualizado antes da próxima atualização; a decisão aplica margem
conservadora nesse intervalo (ver modo de falha em `FailureRecovery.md`).

### 3.6 `aios.<tenant>.cluster.capacity.changed`

**Produtor:** 027-Cluster. **Uso:** atualiza `CapacityProvider` e, quando
`N` (número de shards) muda, aciona rebalanceamento coordenado no
`ShardRouter` (hashing consistente, ADR-0092 a propor) para minimizar
migração de filas.

### 3.7 `aios.<tenant>.policy.decision.updated`

**Produtor:** 022-Policy. **Uso:** invalida o cache de curta duração de
`PolicyClient`, forçando reavaliação do PDP na próxima admissão/preempção
relevante ao escopo alterado (tenant/role/recurso).

---

## 4. Semântica de Entrega e Ordenação

- **Entrega:** todos os eventos deste catálogo usam **at-least-once**
  (JetStream com `ack_policy=explicit`, `max_deliver` configurável — ver
  `Configuration.md`). Consumidores DEVEM deduplicar por `event.id` (ULID),
  conforme RFC-0001 §5.5.
- **Ordenação:** garantida **apenas dentro do mesmo `subject`** (mesma
  `task_urn` ou mesmo `shard`, conforme o evento). O JetStream consumer usa
  `task_urn` como chave de particionamento lógico (subject completo inclui o
  URN da tarefa) para preservar a ordem causal
  `admitted → scheduled → (preempted → resumed)* → completed|failed`.
  **NÃO há** garantia de ordem entre `task_urn`s distintas nem entre tenants.
- **Efeitos colaterais idempotentes:** consumidores de
  `task.execution.scheduled` (Runtime Supervisor) e de
  `agent.lifecycle.*` (Scheduler) DEVEM tratar duplicatas como no-op quando o
  estado já reflete o efeito do evento (ex.: `spawned` recebido duas vezes
  para o mesmo `decisionId` não deve re-disparar reconciliação).
- **Outbox transacional:** a publicação de `task.execution.*` e
  `scheduler.decision.recorded` ocorre **atomicamente** com a escrita da
  `SchedulingDecision` em PostgreSQL via `DecisionJournal` — não há janela em
  que a decisão exista sem o evento correspondente ser eventualmente
  publicado (retry com backoff em caso de falha transitória do JetStream).
- **DLQ e replay:** eventos que excedem `max_deliver` (consumo pelo lado do
  Runtime/025/026/023) são movidos para `aios-dlq.scheduler.<subject>`;
  reprocessamento é manual/coordenado com o time operador (ver
  `FailureRecovery.md`).

---

## 5. Configuração de Stream (JetStream) — referência

| Stream | Subjects | Retenção | Réplicas | Uso |
|--------|----------|----------|----------|-----|
| `AIOS_TASK_EXECUTION` | `aios.*.task.execution.>` | limits (por tempo, default 7 dias hot) | 3 (RF, coordenado com 027) | Eventos §2.2–2.8. |
| `AIOS_SCHEDULER_INTERNAL` | `aios.*.scheduler.>` | limits (30 dias — suporta auditoria/treino) | 3 | Eventos §2.9–2.10. |
| `aios-dlq.scheduler` | `aios-dlq.scheduler.>` | limits (90 dias) | 3 | Falhas de consumo (§4). |

> Parâmetros exatos de retenção/réplicas são operados por `027-Cluster`;
> este documento fixa apenas o contrato de subjects/schema. Consumidores
> configuram `ack_wait`, `max_deliver` e `deliver_policy` conforme sua própria
> `Configuration.md`.

---

## 6. Referências

- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2, §5.3, §5.5.
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §6.
- Superfície de API que originam estes eventos: `./API.md`.
- Modelo de dados/proveniência (`SchedulingDecision`, `feature_vector`):
  `./Database.md`.
- Chaves de configuração de backpressure/lease/heartbeat: `./Configuration.md`.
- Máquina de estados correlacionada às transições: `./StateMachine.md`.
- Modos de falha, DLQ e replay: `./FailureRecovery.md`.
