---
Documento: Events.md
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0066, ADR-0064
RFCs relacionados: RFC-0001 (baseline), RFC-0006 (Cognitive Syscall ABI, a propor)
Depende de: 001-Architecture, 003-RFC/RFC-0001, 009-Scheduler, 008-Agent-Lifecycle, 022-Policy, 026-Cost-Optimizer, 020-Communication
---

# 006-Kernel — Events

> **Escopo.** Catálogo completo dos eventos NATS/JetStream produzidos e
> consumidos pelo Kernel. O envelope CloudEvents e a convenção de subjects
> (`aios.<tenant>.<dominio>.<entidade>.<acao>`) são definidos em
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3 e **não são
> redefinidos aqui**. `<dominio>` para todos os eventos deste módulo é `agent`
> (lifecycle/checkpoint/quota) ou `agent.syscall` (auditoria de decisão),
> conforme `_DESIGN_BRIEF.md` §6.

---

## 1. Convenções específicas do Kernel

| Aspecto | Regra |
|---------|-------|
| Padrão de publicação | **Outbox transacional** (`EventEmitter` + tabela `kernel.outbox`): o evento é gravado na mesma transação da mutação do ACB/cota e publicado por um relay assíncrono (`kernel.outbox.relay_interval_ms`, default 50 ms). |
| Entrega | **At-least-once** via JetStream. Consumidores DEVEM deduplicar por `event.id` (ULID). |
| Ordenação | Garantida **por `subject` completo** (inclui `agent_urn` implícito via `data.agentUrn`), pois o Kernel publica em ordem de commit por ACB; **não há garantia de ordem entre ACBs distintos**. |
| `source` | `urn:aios:<tenant>:service:kernel`. |
| `dataschema` | `aios://schemas/<type>/<versão>` — versão inicial `1` para todo evento deste catálogo. |
| Streams JetStream | `KERNEL_LIFECYCLE` (retenção 90 dias, réplicas=3), `KERNEL_QUOTA` (retenção 30 dias, réplicas=3), `KERNEL_AUDIT` (retenção 365 dias, réplicas=3, WORM lógico — consumida apenas por `025-Audit`). |
| Versionamento de schema | Evolução aditiva de `data` incrementa a versão do `dataschema` **apenas** em quebra de compatibilidade; consumidores DEVEM tolerar campos desconhecidos (RFC-0001 §5.7). |

---

## 2. Eventos emitidos (produtor: Kernel)

### 2.1 Catálogo resumido

| Subject (`aios.<tenant>.…`) | `type` | Stream | Quando é emitido |
|-----------------------------|--------|--------|--------------------|
| `agent.lifecycle.spawned` | `aios.agent.lifecycle.spawned` | `KERNEL_LIFECYCLE` | ACB criado, transição ∅→`Pending` (T-01). |
| `agent.lifecycle.admitted` | `aios.agent.lifecycle.admitted` | `KERNEL_LIFECYCLE` | Scheduler admitiu, `Pending`→`Admitted` (T-02). |
| `agent.lifecycle.running` | `aios.agent.lifecycle.running` | `KERNEL_LIFECYCLE` | Runtime materializado, `Admitted`→`Running` (T-04). |
| `agent.lifecycle.suspended` | `aios.agent.lifecycle.suspended` | `KERNEL_LIFECYCLE` | `Running`→`Suspended` (T-06). |
| `agent.lifecycle.hibernated` | `aios.agent.lifecycle.hibernated` | `KERNEL_LIFECYCLE` | `Suspended`→`Hibernated` (T-08). |
| `agent.lifecycle.resumed` | `aios.agent.lifecycle.resumed` | `KERNEL_LIFECYCLE` | `Suspended`/`Hibernated`→`Running`/`Admitted` (T-07/T-09). |
| `agent.lifecycle.terminated` | `aios.agent.lifecycle.terminated` | `KERNEL_LIFECYCLE` | `Terminating`→`Terminated` (T-11). |
| `agent.lifecycle.failed` | `aios.agent.lifecycle.failed` | `KERNEL_LIFECYCLE` | Qualquer estado→`Failed` (T-03/T-05/T-12). |
| `agent.checkpoint.created` | `aios.agent.checkpoint.created` | `KERNEL_LIFECYCLE` | `checkpoint` concluído (T-13). |
| `agent.quota.exceeded` | `aios.agent.quota.exceeded` | `KERNEL_QUOTA` | Cota `hard` excedida (rejeição). |
| `agent.quota.warning` | `aios.agent.quota.warning` | `KERNEL_QUOTA` | Cota atinge limiar de alerta (default 80%). |
| `agent.syscall.denied` | `aios.agent.syscall.denied` | `KERNEL_AUDIT` | PEP nega syscall (capability ou tenant). |

### 2.2 Detalhamento por evento

#### 2.2.1 `agent.lifecycle.spawned`

- **Subject**: `aios.<tenant>.agent.lifecycle.spawned`
- **Produtor**: `LifecycleCoordinator` (via `EventEmitter`/Outbox).
- **Consumidores**: `009-Scheduler` (dispara admissão), `025-Audit`, `024-Observability`.
- **Ordenação**: garantida por ACB (é sempre o primeiro evento da árvore de vida daquele `urn`).

```json
{
  "specversion": "1.0",
  "id": "01J9ZD0000000000000000A1",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.lifecycle.spawned",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:00:00.000Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.spawned/1",
  "data": {
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "parentUrn": null,
    "priority": 5,
    "policyRef": "urn:aios:acme:policy:default-agent",
    "quotaRef": "8f14e45f-ceea-467a-8b81-3a4b0f0a1f1a",
    "state": "Pending"
  }
}
```

#### 2.2.2 `agent.lifecycle.admitted`

- **Produtor**: `LifecycleCoordinator`, reagindo a `scheduler.admission.decided` consumido (§3.1).
- **Consumidores**: `008-Agent-Lifecycle` (dispara materialização), `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZD0000000000000000A2",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.lifecycle.admitted",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:00:00.180Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-01a067aa0ba902c8-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.admitted/1",
  "data": {
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "schedulerSlotId": "slot-7f3c",
    "state": "Admitted"
  }
}
```

#### 2.2.3 `agent.lifecycle.running`

- **Produtor**: `LifecycleCoordinator`, reagindo a `lifecycle.runtime.started` consumido.
- **Consumidores**: `007-Agent-Runtime` (confirmação), `024-Observability`, `025-Audit`.

```json
{
  "specversion": "1.0", "id": "01J9ZD0000000000000000A3",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.lifecycle.running",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:00:00.230Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-02b067aa0ba902d9-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.running/1",
  "data": {
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "runtimeRef": "runtime-pod-01J9ZD01",
    "contextRestored": false,
    "state": "Running"
  }
}
```

#### 2.2.4 `agent.lifecycle.suspended`

- **Produtor**: `LifecycleCoordinator` (syscall `suspend` ou reação a `scheduler.preemption.requested`).
- **Consumidores**: `009-Scheduler` (libera slot), `008-Agent-Lifecycle`, `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZD0000000000000000A4",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.lifecycle.suspended",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:10:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-03c067aa0ba902ea-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.suspended/1",
  "data": {
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "reason": "voluntary",
    "state": "Suspended"
  }
}
```

#### 2.2.5 `agent.lifecycle.hibernated`

- **Produtor**: `LifecycleCoordinator` (varredura de ociosidade, `kernel.hibernation.idle_ttl_ms`).
- **Consumidores**: `008-Agent-Lifecycle` (libera RAM/handles), `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZD0000000000000000A5",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.lifecycle.hibernated",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:15:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-04d067aa0ba902fb-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.hibernated/1",
  "data": {
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "checkpointRef": "chk-01J9ZD02",
    "idleMs": 300500,
    "state": "Hibernated"
  }
}
```

#### 2.2.6 `agent.lifecycle.resumed`

- **Produtor**: `LifecycleCoordinator` (syscall `resume`).
- **Consumidores**: `009-Scheduler`, `008-Agent-Lifecycle`, `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZD0000000000000000A6",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.lifecycle.resumed",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:20:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-05e067aa0ba9020c-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.resumed/1",
  "data": {
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "fromState": "Hibernated",
    "toState": "Admitted",
    "checkpointRef": "chk-01J9ZD02"
  }
}
```

#### 2.2.7 `agent.lifecycle.terminated`

- **Produtor**: `LifecycleCoordinator` (fim da saga `kill`, drenagem concluída — T-11).
- **Consumidores**: `009-Scheduler` (libera slot definitivamente), `008-Agent-Lifecycle`, `026-Cost-Optimizer` (fecha contabilização), `025-Audit`, `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZD0000000000000000A7",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.lifecycle.terminated",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T13:00:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-06f067aa0ba9021d-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.terminated/1",
  "data": {
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "reason": "requested",
    "totalTokensUsed": 812400,
    "totalCostUsd": 6.73,
    "state": "Terminated"
  }
}
```

#### 2.2.8 `agent.lifecycle.failed`

- **Produtor**: `LifecycleCoordinator` (T-03/T-05/T-12).
- **Consumidores**: `009-Scheduler` (libera recursos reservados), `025-Audit`, `024-Observability` (alerta).

```json
{
  "specversion": "1.0", "id": "01J9ZD0000000000000000A8",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.lifecycle.failed",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:00:05.200Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-07a067aa0ba9022e-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.lifecycle.failed/1",
  "data": {
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "failedFrom": "Admitted",
    "cause": "boot_timeout",
    "errorCode": "AIOS-KERNEL-0004"
  }
}
```

#### 2.2.9 `agent.checkpoint.created`

- **Produtor**: `CheckpointManager`.
- **Consumidores**: `008-Agent-Lifecycle` (índice de snapshots), `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZD0000000000000000A9",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.checkpoint.created",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:14:59.500Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-08b067aa0ba9023f-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.checkpoint.created/1",
  "data": {
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "checkpointRef": "chk-01J9ZD02",
    "sizeBytes": 4194304,
    "memoryPtrSnapshot": {"working": "ptr-1", "long": "ptr-2"}
  }
}
```

#### 2.2.10 `agent.quota.exceeded`

- **Produtor**: `ResourceQuotaManager`.
- **Consumidores**: `026-Cost-Optimizer` (reavaliação de orçamento), `024-Observability` (alerta), `025-Audit`.

```json
{
  "specversion": "1.0", "id": "01J9ZD0000000000000000B0",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.quota.exceeded",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:30:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-09c067aa0ba90250-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.quota.exceeded/1",
  "data": {
    "subjectUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "dimension": "tokens",
    "limit": 500000,
    "attempted": 512000,
    "enforcement": "hard",
    "errorCode": "AIOS-QUOTA-0001"
  }
}
```

#### 2.2.11 `agent.quota.warning`

- **Produtor**: `ResourceQuotaManager` (limiar configurável, default 80%).
- **Consumidores**: `026-Cost-Optimizer`, `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZD0000000000000000B1",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.quota.warning",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:28:00.000Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-0ad067aa0ba90261-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.quota.warning/1",
  "data": {
    "subjectUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "dimension": "cost_usd",
    "limit": 10.0,
    "current": 8.1,
    "thresholdPct": 80
  }
}
```

#### 2.2.12 `agent.syscall.denied`

- **Produtor**: `CapabilityEnforcer` (PEP).
- **Consumidores**: `025-Audit` (trilha imutável — **obrigatório**, RFC-0001 §5.8), `024-Observability`.

```json
{
  "specversion": "1.0", "id": "01J9ZD0000000000000000B2",
  "source": "urn:aios:acme:service:kernel",
  "type": "aios.agent.syscall.denied",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:11:00.050Z", "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-0be067aa0ba90272-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.syscall.denied/1",
  "data": {
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "verb": "invoke_tool",
    "capabilityRequested": "tool:invoke:web-search",
    "denyCode": "AIOS-CAP-0001",
    "policyRef": "urn:aios:acme:policy:default-agent"
  }
}
```

---

## 3. Eventos consumidos

### 3.1 Catálogo

| Subject assinado | Produtor | Ação do Kernel | Componente que reage |
|-------------------|----------|-----------------|-------------------------|
| `aios.<tenant>.scheduler.admission.decided` | `009-Scheduler` | Dirige T-02 (`Pending`→`Admitted`, admissão aceita) ou T-03 (rejeição/timeout→`Failed`). | `LifecycleCoordinator` |
| `aios.<tenant>.scheduler.preemption.requested` | `009-Scheduler` | Dispara syscall interna `suspend` (T-06). | `LifecycleCoordinator` |
| `aios.<tenant>.lifecycle.runtime.started` | `008-Agent-Lifecycle` | Dirige T-04 (`Admitted`→`Running`), atualiza `runtime_ref`. | `LifecycleCoordinator` |
| `aios.<tenant>.lifecycle.runtime.stopped` | `008-Agent-Lifecycle` | Dirige T-11 (drenagem concluída→`Terminated`) ou T-12 (crash inesperado→`Failed`). | `LifecycleCoordinator` |
| `aios.<tenant>.policy.bundle.updated` | `022-Policy` | Invalida cache de decisões do `CapabilityEnforcer` (TTL forçado a 0 para o bundle afetado). | `CapabilityEnforcer` / `PolicyClient` |
| `aios.<tenant>.cost.budget.updated` | `026-Cost-Optimizer` | Atualiza `cost_usd_limit` na tabela `kernel.quota` (reconciliação de limites). | `ResourceQuotaManager` |

### 3.2 Semântica de consumo

- Todo consumidor do Kernel **DEVE** deduplicar por `event.id` (armazenado em
  tabela de *dedupe* de curta retenção, TTL ≥ janela máxima de reentrega do
  JetStream configurada nos streams de origem).
- Consumo é **at-least-once**; handlers **DEVEM** ser idempotentes — reaplicar
  o mesmo `scheduler.admission.decided` duas vezes não deve gerar duas
  transições de estado (guarda de FSM: transição só ocorre se o estado atual
  do ACB ainda for compatível com a guarda, ver `StateMachine.md`).
- Falha ao processar um evento consumido (ex.: erro transitório ao persistir
  a transição) **DEVE** resultar em NAK/redelivery pelo consumidor JetStream
  (ack explícito somente após commit local bem-sucedido).
- Consumidores usam **durable consumers** nomeados
  (`kernel-lifecycle-consumer`, `kernel-quota-consumer`) para preservar
  posição de leitura entre reinícios do Kernel.

### 3.3 Exemplo — evento consumido `scheduler.admission.decided`

```json
{
  "specversion": "1.0",
  "id": "01J9ZE0000000000000000C1",
  "source": "urn:aios:acme:service:scheduler",
  "type": "aios.scheduler.admission.decided",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-20T12:00:00.150Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/scheduler.admission.decided/1",
  "data": {
    "agentUrn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "decision": "admitted",
    "slotId": "slot-7f3c"
  }
}
```

---

## 4. Ordenação, entrega e garantias

| Garantia | Descrição |
|----------|-----------|
| Ordenação intra-ACB | Eventos emitidos para o mesmo `subject` (mesmo `agentUrn`) são publicados na ordem de commit das transições da FSM (Outbox preserva ordem de inserção por partição lógica `agent_urn`). |
| Ordenação inter-ACB | **Não garantida** — consumidores que precisam de ordem global entre agentes distintos DEVEM implementar sua própria serialização (não é responsabilidade do Kernel). |
| At-least-once | JetStream + Outbox garantem que todo evento aceito pelo Kernel **será** publicado ao menos uma vez, mesmo sob crash entre commit e publish (relay reprocessa `published=false`). |
| Deduplicação | Consumidores deduplicam por `event.id` (ULID único por evento, gerado no momento da gravação no Outbox). |
| Exactly-once efetivo | Resultado de at-least-once + idempotência de consumidor + dedupe por `event.id` (RFC-0001 §5.2, §5.5). |
| Retenção de stream | `KERNEL_LIFECYCLE`: 90 dias. `KERNEL_QUOTA`: 30 dias. `KERNEL_AUDIT`: 365 dias (espelhado de forma durável por `025-Audit`; o Kernel não depende da retenção do stream para prova de auditoria). |

---

## 5. Versionamento de schema de eventos

- `dataschema` segue `aios://schemas/<type>/<versão>`; versão inicial de todo
  evento deste catálogo é `1`.
- Evolução **aditiva** (novo campo opcional em `data`) **não** incrementa a
  versão; consumidores DEVEM ignorar campos desconhecidos (RFC-0001 §5.7).
- Evolução **quebra de compatibilidade** (remoção/renomeação/mudança de tipo
  de campo existente) **DEVE** incrementar a versão (`.../2`) e o Kernel
  **DEVE** publicar em ambas as versões durante uma janela de coexistência
  (mínimo 2 releases minor), sinalizada em `RFC.md` do módulo.
- Registro central de schemas: `../004-API/Events.md` (registro global, RFC-0001 §8).

---

## 6. Referências

- Envelope CloudEvents, subjects e semântica de entrega: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2–§5.3, §5.5.
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §6.
- Máquina de estados que gera as transições descritas: `./StateMachine.md`.
- Tabela `kernel.outbox` (persistência do padrão Outbox): `./Database.md` §3.4.
- Syscalls que originam estes eventos: `./API.md`.
- Barramento e streams JetStream: `../020-Communication/`.
- Consumidor de auditoria imutável: `../025-Audit/`.
