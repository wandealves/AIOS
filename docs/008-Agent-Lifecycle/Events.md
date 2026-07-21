---
Documento: Events.md
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0086, ADR-0087
RFCs relacionados: RFC-0001 (baseline), RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore Protocol, a propor)
Depende de: 001-Architecture, 003-RFC/RFC-0001, 006-Kernel, 007-Agent-Runtime, 009-Scheduler, 014-Workflow, 020-Communication, 027-Cluster
---

# 008-Agent-Lifecycle — Events

> **Escopo.** Catálogo completo dos eventos NATS/JetStream produzidos e
> consumidos pelo Lifecycle Manager. O envelope CloudEvents e a convenção de
> subjects (`aios.<tenant>.<dominio>.<entidade>.<acao>`) são definidos em
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3 e **não são
> redefinidos aqui**. `<dominio>` é sempre `agent`, `<entidade>` é sempre
> `lifecycle`, conforme faixa reservada ao módulo (`_DESIGN_BRIEF.md` §0/§6).

---

## 1. Convenções específicas do módulo 008

| Aspecto | Regra |
|---------|-------|
| Padrão de publicação | **Outbox transacional** (`LifecycleOutbox` + `LifecycleEventPublisher`): o evento é gravado na mesma transação da transição do ACB (INV2 do brief) e publicado por um relay assíncrono (`outbox.publish_batch`, default 256 mensagens por lote). |
| Entrega | **At-least-once** via JetStream. Consumidores DEVEM deduplicar por `event.id` (ULID). |
| Ordenação | Garantida **por agente** (mesmo `subject`/`data.agentId`): eventos são publicados na ordem de commit das transições da FSM daquele agente. **Não há garantia de ordem entre agentes distintos.** |
| `source` | `urn:aios:<tenant>:service:agent-lifecycle`. |
| `dataschema` | `aios://schemas/agent.lifecycle.<acao>/1` — versão inicial `1` para todo evento deste catálogo. |
| Stream JetStream | `LIFECYCLE_EVENTS` (retenção 180 dias, réplicas=3, subject filter `aios.*.agent.lifecycle.>`). |
| Consumidores durable | Nomeados por módulo consumidor (ex.: `scheduler-lifecycle-consumer`, `audit-lifecycle-consumer`) para preservar posição de leitura entre reinícios. |
| Minimização de dados | Payloads **NÃO DEVEM** incluir conteúdo de working memory (apenas metadados de estado/ponteiros), conforme `_DESIGN_BRIEF.md` §12.3 e RFC-0001 §7. |

---

## 2. Eventos emitidos (produtor: módulo 008)

### 2.1 Catálogo resumido

| Subject (`aios.<tenant>.agent.lifecycle.<acao>`) | `type` | Transição | Quando é emitido |
|---|---|---|---|
| `…lifecycle.created` | `aios.agent.lifecycle.created` | T1 (`∅→Created`) | ACB registrado no módulo (após 006 criar o ACB). |
| `…lifecycle.ready` | `aios.agent.lifecycle.ready` | T2 (`Created→Ready`) | Scheduler (009) admitiu o agente. |
| `…lifecycle.running` | `aios.agent.lifecycle.running` | T3 (`Ready→Running`) | Runtime (007) iniciou o loop cognitivo. |
| `…lifecycle.suspended` | `aios.agent.lifecycle.suspended` | T4 (`Running→Suspended`) | Agente suspenso (voluntário ou preempção). |
| `…lifecycle.resumed` | `aios.agent.lifecycle.resumed` | T5 (`Suspended→Running`) | Agente retomado. |
| `…lifecycle.hibernated` | `aios.agent.lifecycle.hibernated` | T6 (`Suspended→Hibernated`) | Cold agent; checkpoint consistente gravado, RAM liberada. |
| `…lifecycle.woken` | `aios.agent.lifecycle.woken` | T7 (`Hibernated→Ready/Running`) | Materializado de frio. |
| `…lifecycle.migrating` | `aios.agent.lifecycle.migrating` | T8 (`Running/Suspended→Migrating`) | Saga de migração iniciada. |
| `…lifecycle.migrated` | `aios.agent.lifecycle.migrated` | T9 (`Migrating→destino`) | Cutover concluído no destino. |
| `…lifecycle.checkpointed` | — (efeito colateral) | Checkpoint criado (sob demanda ou automático). |
| `…lifecycle.restored` | `aios.agent.lifecycle.restored` | — (recomposição) | Restore concluído a partir de checkpoint. |
| `…lifecycle.terminated` | `aios.agent.lifecycle.terminated` | T10 (`qualquer→Terminated`) | Encerramento normal. |
| `…lifecycle.failed` | `aios.agent.lifecycle.failed` | T11 (`qualquer→Failed`) | Falha irrecuperável. |

`data` mínimo comum a todo evento: `{ agentId(URN), tenantId, generation,
fromState, toState, trigger, reason?, checkpointId?, placement{shard,node}?
}` — conforme `_DESIGN_BRIEF.md` §6.1.

### 2.2 Detalhamento por evento

#### 2.2.1 `agent.lifecycle.created`

- **Produtor**: `LifecycleCoordinator`, reagindo ao evento consumido
  `agent.control.created` (006).
- **Consumidores**: `009-Scheduler` (dispara admissão), `025-Audit`,
  `024-Observability`.
- **Ordenação**: primeiro evento da árvore de vida daquele `agentId`.

```json
{
  "specversion": "1.0",
  "id": "01J9ZL0000000000000000A1",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.created",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:00:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.created/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 1,
    "fromState": null,
    "toState": "Created",
    "trigger": "spawn",
    "policyRef": "urn:aios:acme:policy:default-agent",
    "quotaRef": "urn:aios:acme:quota:default"
  }
}
```

#### 2.2.2 `agent.lifecycle.ready`

- **Produtor**: `LifecycleCoordinator`, reagindo a
  `scheduler.placement.decided` (009).
- **Consumidores**: `007-Agent-Runtime` (prepara materialização),
  `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000A2",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.ready",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:00:00.120Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-01a067aa0ba902c8-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.ready/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 1,
    "fromState": "Created",
    "toState": "Ready",
    "trigger": "admit",
    "placement": { "shard": 17, "node": null }
  }
}
```

#### 2.2.3 `agent.lifecycle.running`

- **Produtor**: `LifecycleCoordinator`, após `SpawnManager` confirmar boot
  do runtime.
- **Consumidores**: `009-Scheduler` (confirma ocupação do slot),
  `025-Audit`, `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000A3",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.running",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:00:00.230Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-02b067aa0ba902d9-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.running/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 1,
    "fromState": "Ready",
    "toState": "Running",
    "trigger": "start",
    "placement": { "shard": 17, "node": "runtime-node-08" },
    "runtimeInstanceId": "urn:aios:acme:runtime:01J9ZL01"
  }
}
```

#### 2.2.4 `agent.lifecycle.suspended`

- **Produtor**: `LifecycleCoordinator` (API `suspend` ou reação a
  `scheduler.preemption.requested`).
- **Consumidores**: `009-Scheduler` (libera slot), `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000A4",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.suspended",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:10:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-03c067aa0ba902ea-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.suspended/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 1,
    "fromState": "Running",
    "toState": "Suspended",
    "trigger": "suspend",
    "reason": "voluntary"
  }
}
```

#### 2.2.5 `agent.lifecycle.resumed`

- **Produtor**: `LifecycleCoordinator` (API `resume`).
- **Consumidores**: `009-Scheduler`, `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000A5",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.resumed",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:12:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-04d067aa0ba902fb-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.resumed/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 1,
    "fromState": "Suspended",
    "toState": "Running",
    "trigger": "resume"
  }
}
```

#### 2.2.6 `agent.lifecycle.hibernated`

- **Produtor**: `HibernationController` (idle_ttl atingido ou pressão de
  RAM) — sempre precedido por checkpoint `consistent` gravado com sucesso
  (guarda G6, INV4).
- **Consumidores**: `009-Scheduler` (libera slot definitivamente),
  `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000A6",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.hibernated",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:20:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-05e067aa0ba9020c-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.hibernated/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 1,
    "fromState": "Suspended",
    "toState": "Hibernated",
    "trigger": "hibernate",
    "reason": "idle_ttl_exceeded",
    "checkpointId": "urn:aios:acme:checkpoint:01J9ZL02"
  }
}
```

#### 2.2.7 `agent.lifecycle.woken`

- **Produtor**: `SpawnManager` (materialização a partir de checkpoint).
- **Consumidores**: `009-Scheduler` (nova ocupação de slot),
  `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000A7",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.woken",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T14:00:00.180Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-06f067aa0ba9021d-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.woken/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 2,
    "fromState": "Hibernated",
    "toState": "Running",
    "trigger": "wake",
    "checkpointId": "urn:aios:acme:checkpoint:01J9ZL02",
    "wakeLatencyMs": 187,
    "placement": { "shard": 17, "node": "runtime-node-11" }
  }
}
```

#### 2.2.8 `agent.lifecycle.migrating`

- **Produtor**: `MigrationOrchestrator` (início da saga, T8).
- **Consumidores**: `027-Cluster`, `009-Scheduler`, `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000A8",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.migrating",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T15:00:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-07a067aa0ba9022e-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.migrating/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 2,
    "fromState": "Running",
    "toState": "Migrating",
    "trigger": "migrate",
    "reason": "node_draining",
    "migrationId": "urn:aios:acme:migration:01J9ZL03",
    "placement": { "shard": 17, "node": "runtime-node-11" }
  }
}
```

#### 2.2.9 `agent.lifecycle.migrated`

- **Produtor**: `MigrationOrchestrator` (cutover concluído, T9).
- **Consumidores**: `009-Scheduler`, `027-Cluster`, `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000A9",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.migrated",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T15:00:01.800Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-08b067aa0ba9023f-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.migrated/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 3,
    "fromState": "Migrating",
    "toState": "Running",
    "trigger": "cutover",
    "migrationId": "urn:aios:acme:migration:01J9ZL03",
    "placement": { "shard": 42, "node": "runtime-node-23" }
  }
}
```

#### 2.2.10 `agent.lifecycle.checkpointed`

- **Produtor**: `CheckpointService` (sob demanda ou periódico,
  `checkpoint.interval_s`).
- **Consumidores**: `008-Agent-Lifecycle` (índice interno de snapshots,
  auto-consumo apenas para reconciliação), `024-Observability`.
- **Nota**: este evento **não** representa transição da FSM — é um efeito
  colateral publicado para observabilidade e para permitir que
  `009-Scheduler`/`027-Cluster` avaliem RPO corrente do agente.

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000B0",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.checkpointed",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:19:59.500Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-09c067aa0ba90250-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.checkpointed/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 1,
    "checkpointId": "urn:aios:acme:checkpoint:01J9ZL02",
    "sizeBytes": 4194304,
    "contentHash": "sha256:9f86d0818...b91d",
    "consistency": "consistent",
    "codec": "msgpack+zstd"
  }
}
```

#### 2.2.11 `agent.lifecycle.restored`

- **Produtor**: `CheckpointService` (restore concluído, sob demanda ou
  parte de T7/T9).
- **Consumidores**: `010-Memory` (reatacha working memory), `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000B1",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.restored",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T16:00:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-0ad067aa0ba90261-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.restored/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 4,
    "checkpointId": "urn:aios:acme:checkpoint:01J9ZL02",
    "workingMemoryRef": "urn:aios:acme:memory:01J9ZL05"
  }
}
```

#### 2.2.12 `agent.lifecycle.terminated`

- **Produtor**: `LifecycleCoordinator` (T10, fim de saga de terminação,
  drenagem concluída).
- **Consumidores**: `009-Scheduler` (libera recursos), `026-Cost-Optimizer`
  (fecha contabilização), `025-Audit`, `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000B2",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.terminated",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T18:00:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-0be067aa0ba90272-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.terminated/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 4,
    "fromState": "Running",
    "toState": "Terminated",
    "trigger": "terminate",
    "reason": "requested"
  }
}
```

#### 2.2.13 `agent.lifecycle.failed`

- **Produtor**: `LifecycleCoordinator`/`ReconciliationController` (T11).
- **Consumidores**: `009-Scheduler` (libera recursos reservados),
  `025-Audit`, `024-Observability` (alerta).

```json
{
  "specversion": "1.0", "id": "01J9ZL0000000000000000B3",
  "source": "urn:aios:acme:service:agent-lifecycle",
  "type": "aios.agent.lifecycle.failed",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:00:15.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-0cf067aa0ba90283-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.failed/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "tenantId": "acme",
    "generation": 1,
    "fromState": "Ready",
    "toState": "Failed",
    "trigger": "fail",
    "reason": "runtime_lost_beyond_max_recovery",
    "errorCode": "AIOS-LIFECYCLE-0015"
  }
}
```

---

## 3. Eventos consumidos (consumidor: módulo 008)

### 3.1 Catálogo

| Subject consumido | Produtor | Ação disparada | Componente que reage |
|---|---|---|---|
| `aios.<tenant>.agent.control.created` | Kernel (006) | Inicializa ACB → `Created` (T1). | `LifecycleCoordinator` |
| `aios.<tenant>.scheduler.placement.decided` | Scheduler (009) | `Created→Ready` (T2, admissão) ou fornece alvo de migração (T9). | `LifecycleCoordinator` / `MigrationOrchestrator` |
| `aios.<tenant>.scheduler.preemption.requested` | Scheduler (009) | `Running→Suspended` (T4, preempção). | `LifecycleCoordinator` |
| `aios.<tenant>.agent.runtime.heartbeat` | Runtime Supervisor (007) | Reconciliação de liveness (atualiza `last_active_at`). | `ReconciliationController` |
| `aios.<tenant>.agent.runtime.exited` | Runtime Supervisor (007) | `→Terminated` (saída limpa) ou `→Failed` (crash, T11). | `LifecycleCoordinator` / `ReconciliationController` |
| `aios.<tenant>.task.execution.completed` | Workflow/Task (014) | Sinal de idle → candidato a avaliação de hibernação. | `HibernationController` |
| `aios.<tenant>.cluster.node.draining` | Cluster (027) | Dispara migração em massa dos agentes do nó (T8). | `MigrationOrchestrator` |

### 3.2 Semântica de consumo

- Todo consumidor do módulo 008 **DEVE** deduplicar por `event.id`
  (armazenado em tabela/cache de *dedupe* de curta retenção, TTL ≥ janela
  máxima de reentrega do JetStream configurada nos streams de origem).
- Consumo é **at-least-once**; handlers **DEVEM** ser idempotentes —
  reaplicar o mesmo `scheduler.placement.decided` duas vezes não gera duas
  transições (a guarda da FSM em `StateMachine.md` só permite a transição se
  o `state` atual do ACB ainda for compatível).
- Falha ao processar um evento consumido (ex.: erro transitório ao
  persistir a transição) **DEVE** resultar em NAK/redelivery pelo
  consumidor JetStream (ack explícito somente após commit local
  bem-sucedido).
- Consumidores usam **durable consumers** nomeados
  (`lifecycle-scheduler-consumer`, `lifecycle-runtime-consumer`,
  `lifecycle-cluster-consumer`) para preservar posição de leitura entre
  reinícios do `LifecycleEventConsumer`.
- `agent.runtime.heartbeat` tem alto volume e **NÃO** gera transição de
  FSM diretamente — apenas atualiza `last_active_at`/`updated_at` na
  projeção do ACB; a ausência de heartbeat além de
  `reconcile.runtime_dead_after_ms` é o gatilho real de reconciliação
  (ver `FailureRecovery.md`).

### 3.3 Exemplo — evento consumido `scheduler.placement.decided`

```json
{
  "specversion": "1.0",
  "id": "01J9ZM0000000000000000C1",
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.scheduler.placement.decided",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:00:00.100Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/scheduler.placement.decided/1",
  "data": {
    "agentId": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "decision": "admitted",
    "shard": 17,
    "priorityClass": "interactive"
  }
}
```

### 3.4 Exemplo — evento consumido `cluster.node.draining`

```json
{
  "specversion": "1.0",
  "id": "01J9ZM0000000000000000C2",
  "source": "urn:aios:acme:service:cluster",
  "type": "aios.cluster.node.draining",
  "subject": "urn:aios:acme:node:runtime-node-11",
  "time": "2026-07-20T14:55:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-0d0067aa0ba90294-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/cluster.node.draining/1",
  "data": {
    "nodeId": "runtime-node-11",
    "reason": "planned_maintenance",
    "drainDeadline": "2026-07-20T15:30:00.000Z"
  }
}
```

`MigrationOrchestrator` reage disparando `migrate` para cada agente com
`placement_node = runtime-node-11`, respeitando
`migration.max_concurrent_per_node` para não saturar a rede/storage
durante a migração em lote (ver `Scalability.md`).

---

## 4. Ordenação, entrega e garantias

| Garantia | Descrição |
|----------|-----------|
| Ordenação intra-agente | Eventos emitidos para o mesmo `subject` (mesmo `agentId`) são publicados na ordem de commit das transições da FSM (Outbox preserva ordem de inserção por partição lógica `agent_id`, alinhado a `seq` da tabela `LifecycleTransition`). |
| Ordenação inter-agente | **Não garantida** — consumidores que precisam de ordem global entre agentes distintos DEVEM implementar sua própria serialização; não é responsabilidade do módulo 008. |
| At-least-once | JetStream + Outbox garantem que todo evento aceito pelo coordenador **será** publicado ao menos uma vez, mesmo sob crash entre commit e publish (relay reprocessa registros com `published=false`). |
| Deduplicação | Consumidores deduplicam por `event.id` (ULID único por evento, gerado no momento da gravação no `LifecycleOutbox`). |
| Exactly-once efetivo | Resultado de at-least-once + idempotência de consumidor + dedupe por `event.id` (RFC-0001 §5.2, §5.5). |
| Retenção de stream | `LIFECYCLE_EVENTS`: 180 dias. A trilha de auditoria imutável de longo prazo é responsabilidade de `025-Audit`, que consome e arquiva independentemente da retenção do stream. |

---

## 5. Versionamento de schema de eventos

- `dataschema` segue `aios://schemas/agent.lifecycle.<acao>/<versão>`;
  versão inicial de todo evento deste catálogo é `1`.
- Evolução **aditiva** (novo campo opcional em `data`) **não** incrementa a
  versão; consumidores DEVEM ignorar campos desconhecidos (RFC-0001 §5.7).
- Evolução **quebra de compatibilidade** (remoção/renomeação/mudança de
  tipo de campo existente) **DEVE** incrementar a versão (`.../2`) e o
  módulo 008 **DEVE** publicar em ambas as versões durante uma janela de
  coexistência (mínimo 2 releases minor), sinalizada em `RFC.md` do módulo.
- Registro central de schemas: `../004-API/Events.md` (registro global,
  RFC-0001 §8).

---

## 6. Referências

- Envelope CloudEvents, subjects e semântica de entrega: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3, §5.5.
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §6.
- Máquina de estados que gera as transições descritas: `./StateMachine.md`.
- Tabela `lifecycle.outbox` (persistência do padrão Outbox): `./Database.md` §3.6.
- Operações que originam estes eventos: `./API.md`.
- Barramento e streams JetStream: `../020-Communication/`.
- Consumidor de auditoria imutável: `../025-Audit/`.
- Modos de falha de publicação/consumo: `./FailureRecovery.md`.
