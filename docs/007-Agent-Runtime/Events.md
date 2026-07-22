---
Documento: Events
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0076
RFCs relacionados: RFC-0001
Depende de: ../003-RFC/RFC-0001, ../020-Communication/, ../008-Agent-Lifecycle/, ../009-Scheduler/, ../022-Policy/, ../015-Tool-Manager/, ../025-Audit/
---

# 007-Agent-Runtime — Events

> **Escopo.** Catálogo completo de eventos NATS/JetStream produzidos e
> consumidos pelo Agent Runtime. Subjects seguem
> `aios.<tenant>.<dominio>.<entidade>.<acao>` e o envelope CloudEvents de
> `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2/§5.3 — **não
> redefinidos aqui**. Entrega **at-least-once**; todo consumidor DEVE
> deduplicar por `event.id`. Domínios usados por este módulo: `agent` e
> `task`.

## Índice

1. Convenções e envelope
2. Eventos emitidos (produtor: Agent Runtime)
3. Eventos consumidos (consumidor: Agent Runtime)
4. Padrão Outbox e garantias de entrega
5. Ordenação
6. Versionamento de schema
7. Exemplos completos de envelope
8. Referências

---

## 1. Convenções e Envelope

Todo evento publicado por este módulo usa o envelope CloudEvents-compatível
de RFC-0001 §5.2:

```json
{
  "specversion": "1.0",
  "id": "<ULID do evento>",
  "source": "urn:aios:<tenant>:service:agent-runtime",
  "type": "aios.agent.runtime.booted",
  "subject": "urn:aios:<tenant>:agent:<ULID>",
  "time": "<RFC 3339 UTC>",
  "tenant": "<tenant>",
  "traceparent": "<W3C Trace Context>",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.runtime.booted/1",
  "data": { "...": "payload específico, ver §2" }
}
```

- `subject` do envelope aponta para o **agente** (`agent_urn`) nos eventos de
  ciclo de vida do runtime, e para a **tarefa** (`task_urn`) nos eventos de
  execução — ver coluna "Subject de negócio" abaixo.
- `type` segue `aios.<dominio>.<entidade>.<acao>` (RFC-0001 §5.3).
- Subjects NATS de publicação seguem `aios.<tenant>.<dominio>.<entidade>.<acao>`.

---

## 2. Eventos Emitidos (Produtor: Agent Runtime)

| # | Subject NATS | `type` | Quando é emitido | `data` (campos principais) |
|---|---------------|--------|---------------------|------------------------------|
| 1 | `aios.<t>.agent.runtime.booted` | `aios.agent.runtime.booted` | Transição `boot→ready` (T1). | `runtime_id`, `session_id`, `agent_urn`, `cold_start_ms` |
| 2 | `aios.<t>.agent.runtime.suspended` | `aios.agent.runtime.suspended` | Transição `suspending→suspended` (T13). | `session_id`, `checkpoint_id`, `step_cursor` |
| 3 | `aios.<t>.agent.runtime.resumed` | `aios.agent.runtime.resumed` | Transição `resuming→(estado salvo)` (T15). | `session_id`, `checkpoint_id` |
| 4 | `aios.<t>.agent.runtime.terminated` | `aios.agent.runtime.terminated` | Transição `terminating→done`/`failed` (T17). | `session_id`, `reason`, `terminal_state` |
| 5 | `aios.<t>.agent.runtime.heartbeat` | `aios.agent.runtime.heartbeat` | Periódico (`heartbeat.interval_ms`). | `runtime_id`, `state`, `consumed`, `budget` |
| 6 | `aios.<t>.task.execution.started` | `aios.task.execution.started` | Transição `ready→thinking`, 1º passo (T3). | `task_urn`, `session_id`, `plan_urn` |
| 7 | `aios.<t>.task.execution.step` | `aios.task.execution.step` | Cada `ReActStep` produzido. | `step_id`, `phase`, `status`, `latency_ms` |
| 8 | `aios.<t>.task.execution.completed` | `aios.task.execution.completed` | Transição `thinking→done` (T5). | `task_urn`, `status=SUCCEEDED`, `costUsd`, `tokens` |
| 9 | `aios.<t>.task.execution.failed` | `aios.task.execution.failed` | Transição `*→failed` (T2/T9/T11). | `task_urn`, `code`, `reason`, `retriable` |
| 10 | `aios.<t>.agent.runtime.quota_exceeded` | `aios.agent.runtime.quota_exceeded` | `QuotaGovernor` esgota uma cota local. | `session_id`, `resource`, `limit`, `consumed` |
| 11 | `aios.<t>.agent.runtime.sandbox_violation` | `aios.agent.runtime.sandbox_violation` | Syscall/egress proibido detectado. | `session_id`, `violation_type` |

### 2.1 Consumidores conhecidos por evento

| Evento | Consumidor(es) primário(s) | Uso |
|--------|-------------------------------|-----|
| `agent.runtime.booted` | `009-Scheduler`, `024-Observability` | Confirmar materialização; medir `cold_start_ms`. |
| `agent.runtime.suspended` / `.resumed` | `008-Agent-Lifecycle`, `009-Scheduler` | Atualizar visão de disponibilidade/hibernação do agente. |
| `agent.runtime.terminated` | `008-Agent-Lifecycle`, `009-Scheduler`, `025-Audit` | Fechar ciclo de vida; liberar cotas de nível de pool. |
| `agent.runtime.heartbeat` | Runtime Supervisor, `024-Observability` | Watchdog de saúde do processo; dashboards de saturação. |
| `task.execution.started`/`.step`/`.completed`/`.failed` | Control plane (WebConsole via serviço intermediário), `010-Memory` (consolidação), `025-Audit` | Rastreabilidade fim-a-fim da execução do agente. |
| `agent.runtime.quota_exceeded` | `026-Cost-Optimizer`, `009-Scheduler`, `024-Observability` | Contenção de custo; decisões de admissão futuras. |
| `agent.runtime.sandbox_violation` | `025-Audit`, `021-Security`, `022-Policy` | Investigação de segurança; possível revogação de capability. |

---

## 3. Eventos Consumidos (Consumidor: Agent Runtime)

| Subject NATS | Origem | Ação no runtime |
|---------------|--------|--------------------|
| `aios.<t>.agent.lifecycle.spawn_requested` | `009-Scheduler`/`008-Agent-Lifecycle` | Aciona `Boot` (via `SupervisorChannel`, em conjunto com o comando gRPC do Supervisor). |
| `aios.<t>.agent.lifecycle.suspend_requested` | `009-Scheduler` (preempção) | Aciona `Suspend`. |
| `aios.<t>.agent.lifecycle.resume_requested` | `009-Scheduler`/`008-Agent-Lifecycle` | Aciona `Resume`. |
| `aios.<t>.agent.lifecycle.kill_requested` | `006-Kernel`/`008-Agent-Lifecycle` | Aciona `Kill`. |
| `aios.<t>.policy.decision.updated` | `022-Policy` | Invalida cache de *capabilities* do `CapabilityEnforcer` (PEP local). |
| `aios.<t>.tool.registry.updated` | `015-Tool-Manager` | Recarrega o catálogo de ferramentas do `McpHost` sem exigir *restart* (NFR-014). |

**Deduplicação:** todo consumo acima **DEVE** deduplicar por `event.id`
(RFC-0001 §5.5); o `CapabilityEnforcer`, por exemplo, ignora reentregas do
mesmo `policy.decision.updated` já processado (idempotência de efeito:
invalidar um cache já invalidado é um no-op seguro).

---

## 4. Padrão Outbox e Garantias de Entrega

O `EventPublisher` grava todo evento em um **outbox transacional local**
(SQLite/WAL efêmero, ver `./Database.md` §5) na mesma unidade de trabalho
que a `ExecutionStateMachine` aplica a transição de estado correspondente, e
só então publica de forma assíncrona no NATS/JetStream, com retry
(`events.outbox.max_retries`) e backoff. Esta é a garantia de **atomicidade
estado↔evento** exigida por R-08 do brief (ADR-0076):

```
1. ExecutionStateMachine.transition(evento)      -- aplica no processo
2. EventPublisher.append_to_outbox(evento)        -- grava local (SQLite/WAL)
3. [retorno ao chamador / prossegue o loop]        -- não bloqueia em rede
4. (assíncrono) EventPublisher.flush_outbox()      -- publica no NATS
5. (assíncrono) marca outbox.published = true       -- após ACK do JetStream
```

Se o processo falhar entre os passos 2 e 5, o evento permanece pendente no
outbox local; como esse arquivo é efêmero (não sobrevive à morte do
processo), a garantia formal é: **nenhum evento é publicado sem a transição
correspondente ter sido aplicada** (nunca "evento fantasma"), mas **um
evento pode nunca ser publicado** se o processo morrer antes do flush — esse
gap é coberto pela meta de RPO ≤ 1 passo (NFR-006) e pela retomada via
checkpoint (`./FailureRecovery.md`), não pelo outbox em si.

---

## 5. Ordenação

- Dentro de uma mesma `session_id`, os eventos `task.execution.step` **DEVEM**
  ser publicados na ordem de `ReActStep.index` (garantido pela natureza
  sequencial do `CognitiveLoopEngine` — um único *event loop* `asyncio` por
  processo/agente).
- Entre `session_id` diferentes (agentes distintos), não há garantia nem
  necessidade de ordenação — cada sessão é uma linha do tempo independente.
- Consumidores que dependem de ordem estrita (ex.: reconstrução de trilha de
  auditoria) **DEVEM** ordenar por `(session_id, index)` no lado do
  consumidor, não assumir ordem de chegada de rede.

---

## 6. Versionamento de Schema

- Todo evento referencia `dataschema` (ex.: `aios://schemas/task.execution.step/1`).
- Mudanças **aditivas** (novo campo opcional em `data`) não incrementam a
  versão do schema; consumidores DEVEM tolerar campos desconhecidos
  (RFC-0001 §5.7, NFR-014).
- Mudanças **incompatíveis** (remoção/renomeação de campo, mudança de tipo)
  **DEVEM** incrementar a versão do schema e coexistir por ≥ 2 versões
  major, com o produtor publicando ambas as formas durante a janela de
  migração (mesma política de `../006-Kernel/Events.md`, se existente, e de
  RFC-0001 §9).

---

## 7. Exemplos Completos de Envelope

### 7.1 `agent.runtime.booted`

```json
{
  "specversion": "1.0",
  "id": "01J9ZEVT0000000000000001",
  "source": "urn:aios:acme:service:agent-runtime",
  "type": "aios.agent.runtime.booted",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-21T12:00:00.120Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.runtime.booted/1",
  "data": {
    "runtime_id": "01J9ZA1RVNTMEXEC0000000001",
    "session_id": "01J9ZA1AGENTSESSN000000001",
    "agent_urn": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
    "cold_start_ms": 187
  }
}
```

### 7.2 `task.execution.failed`

```json
{
  "specversion": "1.0",
  "id": "01J9ZEVT0000000000000099",
  "source": "urn:aios:acme:service:agent-runtime",
  "type": "aios.task.execution.failed",
  "subject": "urn:aios:acme:task:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
  "time": "2026-07-21T12:05:32.001Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-1a2b3c4d5e6f7081-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/task.execution.failed/1",
  "data": {
    "task_urn": "urn:aios:acme:task:01J9Z8Q7B0C2D4E6F8G0H2J4K6",
    "code": "AIOS-RUNTIME-0012",
    "reason": "loop.max_steps (50) excedido antes da resposta final.",
    "retriable": false
  }
}
```

### 7.3 `agent.runtime.sandbox_violation`

```json
{
  "specversion": "1.0",
  "id": "01J9ZEVT0000000000000123",
  "source": "urn:aios:acme:service:agent-runtime",
  "type": "aios.agent.runtime.sandbox_violation",
  "subject": "urn:aios:acme:agent:01J9Z8Q6H7K2M4N6P8R0S2T4V6",
  "time": "2026-07-21T12:07:11.442Z",
  "tenant": "acme",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-2b3c4d5e6f708192-01",
  "datacontenttype": "application/json",
  "dataschema": "aios://schemas/agent.runtime.sandbox_violation/1",
  "data": {
    "session_id": "01J9ZA1AGENTSESSN000000001",
    "violation_type": "egress_denied",
    "detail_ref": "urn:aios:acme:audit:01J9ZAVDT00000000000000001"
  }
}
```

---

## 8. Referências

- Catálogo autoritativo (fonte): `./_DESIGN_BRIEF.md` §6.
- Transições que disparam cada evento: `./StateMachine.md` §2, §5.
- Fluxos de sequência que publicam/consomem eventos: `./SequenceDiagrams.md`.
- Envelope CloudEvents e subjects: `../003-RFC/RFC-0001-Architecture-Baseline.md` §5.2, §5.3.
- Barramento NATS/JetStream: `../020-Communication/`.
- Consumidor de auditoria: `../025-Audit/`.
