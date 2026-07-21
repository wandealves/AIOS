---
Documento: SequenceDiagrams
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0061, ADR-0062, ADR-0063, ADR-0064, ADR-0066, ADR-0067, ADR-0068 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline); RFC-0006 (Cognitive Syscall ABI, a propor); RFC-0007 (ACB & Quota Model, a propor)
Depende de: 006-Kernel/StateMachine.md, 006-Kernel/ClassDiagrams.md, 006-Kernel/Architecture.md, 009-Scheduler, 008-Lifecycle, 022-Policy
---

# 006-Kernel — Diagramas de Sequência

> Fluxos críticos do Kernel, em caminho feliz e em falha, com participantes,
> mensagens síncronas (`──▶`/`◀──`, chamada+retorno) e assíncronas
> (`┄┄▶`, *fire-and-forget*/evento), e timeouts relevantes. Todos os fluxos
> respeitam a FSM de `./StateMachine.md`, as interfaces de `./ClassDiagrams.md`
> e os contratos de `../003-RFC/RFC-0001-Architecture-Baseline.md`
> (`traceparent`, `X-AIOS-Tenant`, `Idempotency-Key`, envelope CloudEvents,
> envelope de erro RFC 7807).

## Índice

1. Convenções de notação
2. `spawn` — caminho feliz (Pending → Admitted → Running)
3. `spawn` — falha de admissão (Scheduler indisponível/timeout)
4. `spawn` — falha de materialização (boot timeout)
5. Syscall privilegiada negada pelo PEP (capability)
6. `suspend` → `resume` — caminho feliz
7. Hibernação por ociosidade e `resume` de agente *cold*
8. `kill` — encerramento com compensação
9. `invoke_tool` — brokering de recurso, caminho feliz
10. `invoke_tool` — cota excedida (falha)
11. `checkpoint` — caminho feliz e falha de snapshot
12. Replay idempotente de uma mutação (`Idempotency-Key` repetida)
13. Recuperação do Outbox após crash entre commit e publish
14. Referências

---

## 1. Convenções de Notação

```
Participante A          Participante B
     │                        │
     │──── msg síncrona ─────▶│   chamada bloqueante (REST/gRPC), aguarda retorno
     │◀─── retorno ───────────│
     │                        │
     │┄┄┄ msg assíncrona ┄┄┄▶│   publicação de evento (NATS/JetStream), sem retorno direto
     │                        │
     ╎── timeout T ──╎         marcador de temporizador/deadline
```

Prefixos de latência-alvo citados nos diagramas referem-se às metas de
`./_DESIGN_BRIEF.md` §7.2 (NFR-001..NFR-003).

---

## 2. `spawn` — Caminho Feliz (Pending → Admitted → Running)

**Pré-condição:** chamador autenticado (claims OIDC validadas no Gateway);
`Idempotency-Key` presente; capability `agent:spawn` a verificar.
**Pós-condição:** ACB em `Running`, `runtime_ref` atribuído, 3 eventos
publicados (`spawned`, `admitted`, `running`).

```
Cliente(007)   SyscallGateway  CapabilityEnforcer  PolicyClient  AcbStore  ResourceQuotaManager  LifecycleCoordinator  SchedulerClient  LifecycleClient  EventEmitter   022-Policy   009-Scheduler   008-Lifecycle
    │               │                │                │            │             │                     │                  │                │              │              │             │               │
    │─POST /agents ─▶│                │                │            │             │                     │                  │                │              │              │             │               │
    │ (Idem-Key,     │                │                │            │             │                     │                  │                │              │              │             │               │
    │  traceparent)  │──IsCached?────▶│(IdempotencyStore)            │             │                     │                  │                │              │              │             │               │
    │                │◀── miss ───────│                │            │             │                     │                  │                │              │              │             │               │
    │                │──Authorize────▶│                │            │             │                     │                  │                │              │              │             │               │
    │                │  (spawn)       │──Evaluate─────▶│            │             │                     │                  │                │              │              │             │               │
    │                │                │                │──gRPC─────────────────────────────────────────────────────────────────────────────────────────────▶│             │               │
    │                │                │                │◀── allow ──────────────────────────────────────────────────────────────────────────────────────────│             │               │
    │                │◀── allow ──────│                │            │             │                     │                  │                │              │              │             │               │
    │                │──Spawn(cmd)────────────────────────────────────────────────────────────────────▶│                  │                │              │              │             │               │
    │                │                │                │            │◀Reserve(agent_count)──────────────│                     │                  │                │              │              │             │               │
    │                │                │                │            │             │──ok────────────────▶│                  │                │              │              │             │               │
    │                │                │                │            │──Insert(ACB,Pending,v0)──────────▶│                     │                  │                │              │              │             │               │
    │                │                │                │            │◀── ok ─────│                     │                  │                │              │              │             │               │
    │                │                │                │            │             │                     │──EnqueueTx(spawned)─────────────────────────────────▶│              │             │               │
    │                │                │                │            │             │                     │──RequestAdmission──────────────────▶│                │              │              │             │               │
    │                │                │                │            │             │                     │                  │──gRPC──────────────────────────────────────────▶│               │
    │                │                │                │            │             │                     │  ╎── admission_timeout_ms (200ms) ──╎                │              │              │             │               │
    │                │                │                │            │             │                     │                  │◀── slot granted ──────────────────────────────│               │
    │                │                │                │            │             │                     │◀── AdmissionDecision(ok) ──────────│                │              │              │             │               │
    │                │                │                │            │──TryUpdate(Admitted,v0→v1)────────▶│                     │                  │                │              │              │             │               │
    │                │                │                │            │◀── ok ─────│                     │                  │                │              │              │             │               │
    │                │                │                │            │             │                     │──EnqueueTx(admitted)─────────────────────────────────▶│              │             │               │
    │                │                │                │            │             │                     │──Materialize(urn,slot)──────────────────────────────▶│                │              │             │               │
    │                │                │                │            │             │                     │                                    │──gRPC──────────────────────────────────────────▶│
    │                │                │                │            │             │                     │  ╎── boot_timeout_ms (5000ms) ──╎    │                │              │              │             │               │
    │                │                │                │            │             │                     │                                    │◀── runtime_ref ────────────────────────────────│
    │                │                │                │            │             │                     │◀── RuntimeRef ─────────────────────────────────────│                │              │              │             │               │
    │                │                │                │            │──TryUpdate(Running,v1→v2,runtime_ref)▶│                  │                │              │              │             │               │
    │                │                │                │            │◀── ok ─────│                     │                  │                │              │              │             │               │
    │                │                │                │            │             │                     │──EnqueueTx(running)──────────────────────────────────▶│              │             │               │
    │                │                │                │            │             │                     │                  │                │──RelayBatch─▶│  (async, ~50ms)              │
    │                │                │                │            │             │                     │                  │                │              │──publish──▶ NATS/JetStream    │
    │                │◀── ACB{Running}────────────────────────────────────────────│                     │                  │                │              │              │             │               │
    │                │──SaveResult(key,result)──────────▶│(IdempotencyStore)      │                     │                  │                │              │              │             │               │
    │◀─201 Created───│                │                │            │             │                     │                  │                │              │              │             │               │
    │  ACB{urn,Running}                │                │            │             │                     │                  │                │              │              │             │               │
```

**Notas:**
- Latência-alvo ponta a ponta: p99 ≤ 250 ms (NFR-001), dominada pelo `boot
  timeout` do Lifecycle no pior caso; caminho feliz é tipicamente muito mais
  rápido (materialização de runtime *warm-pooled*).
- Os três `EnqueueTx(...)` ocorrem cada um **na mesma transação de banco** que
  a respectiva mutação do ACB (padrão Outbox, I4 de `./StateMachine.md`); a
  publicação real no NATS é assíncrona via `EventEmitter.RelayPendingBatch`.
- `SaveResult` garante que uma repetição do mesmo `POST` com o mesmo
  `Idempotency-Key` retorna `201` idêntico sem reexecutar a saga (ver §12).

---

## 3. `spawn` — Falha de Admissão (Scheduler indisponível/timeout)

**Gatilho:** `009-Scheduler` não responde dentro de
`kernel.spawn.admission_timeout_ms` (default 200 ms) ou retorna rejeição.
**Pós-condição:** ACB → `Failed` (T-03); cota `agent_count` liberada
(compensação); evento `agent.lifecycle.failed` publicado.

```
Cliente(007)   SyscallGateway   LifecycleCoordinator   ResourceQuotaManager   AcbStore   SchedulerClient   EventEmitter   009-Scheduler
    │               │                   │                     │                 │              │              │              │
    │──POST /agents─▶│                   │                     │                 │              │              │              │
    │                │──Spawn(cmd)──────▶│                     │                 │              │              │              │
    │                │                   │──Reserve(agent_count)──────────────▶│                 │              │              │
    │                │                   │◀── ok ──────────────│                 │              │              │              │
    │                │                   │──Insert(Pending,v0)─────────────────────────────────▶│              │              │
    │                │                   │◀── ok ──────────────────────────────│                 │              │              │
    │                │                   │──RequestAdmission──────────────────────────────────────────────────▶│              │
    │                │                   │  ╎────────── admission_timeout_ms (200ms) excedido ──────────────╎  │              │
    │                │                   │◀── AIOS-KERNEL-0003 (timeout, retriable) ─────────────────────────│              │
    │                │                   │──TryUpdate(Pending→Failed,v0→v1)────────────────────▶│              │              │
    │                │                   │◀── ok ──────────────────────────────│                 │              │              │
    │                │                   │──Release(agent_count reservation)───▶│                 │              │              │
    │                │                   │◀── ok ──────────────────────────────│                 │              │              │
    │                │                   │──EnqueueTx(agent.lifecycle.failed)───────────────────────────────────▶│              │
    │                │◀── AIOS-KERNEL-0003 ────────────────────│                 │              │              │              │
    │◀─503 Service Unavailable──────────│                     │                 │              │              │              │
    │   (retriable=true, retryAfterMs)  │                     │                 │              │              │              │
```

**Notas:**
- A compensação (`Release`) **DEVE** ocorrer antes da transição para `Failed`
  ser confirmada como concluída, garantindo que nenhuma cota fique presa (I2
  de `./StateMachine.md`).
- O cliente **DEVE** poder repetir o `spawn` com um **novo**
  `Idempotency-Key` (o ACB anterior está em `Failed`, terminal); repetir com a
  **mesma** chave apenas reproduz o erro já registrado (idempotência do erro).
- Envelope de erro segue RFC-0001 §5.4: `code=AIOS-KERNEL-0003`,
  `status=503`, `retriable=true`.

---

## 4. `spawn` — Falha de Materialização (boot timeout)

**Gatilho:** `008-Lifecycle` não confirma materialização dentro de
`kernel.spawn.boot_timeout_ms` (default 5000 ms), após admissão bem-sucedida.
**Pós-condição:** ACB `Admitted` → `Failed` (T-05); slot do Scheduler liberado;
cota liberada.

```
LifecycleCoordinator   LifecycleClient   SchedulerClient   AcbStore   EventEmitter   008-Lifecycle   009-Scheduler
        │                     │                 │              │            │              │              │
        │──Materialize(urn,slot)───────────────▶│              │            │              │              │
        │                     │                 │              │            │              │──accept?────▶│(fica preso/lento)
        │  ╎──────────── boot_timeout_ms (5000ms) excedido ─────────────────────────────────╎              │
        │◀── AIOS-KERNEL-0004 (timeout, retriable) ─────────────│              │              │              │
        │──TryUpdate(Admitted→Failed,v1→v2)────────────────────▶│            │              │              │
        │◀── ok ──────────────────────────────────────────────│              │              │              │
        │──ReleaseSlot(slot)───────────────────────────────────────────────▶│              │              │              │
        │◀── ok ──────────────────────────────────────────────────────────│              │              │              │
        │──EnqueueTx(agent.lifecycle.failed)────────────────────────────────────────────▶│              │              │
        │  (cota de agent_count já havia sido reservada em §2; liberada aqui via Release) │              │              │
```

**Notas:**
- `AIOS-KERNEL-0004` (504) é `retriable=true`: o cliente pode reemitir `spawn`
  com novo `Idempotency-Key`.
- A liberação do `slot` junto ao Scheduler é obrigatória para não vazar
  capacidade reservada (compensação da saga, ADR-0064).

---

## 5. Syscall Privilegiada Negada pelo PEP (Capability)

**Fluxo genérico**, aplicável a qualquer verbo privilegiado (`spawn`, `kill`,
`suspend`, `resume`, `remember`, `recall`, `plan`, `invoke_tool`,
`route_model`, `checkpoint`). Exemplo: `kill` sem capability.

```
Cliente(007)   SyscallGateway   CapabilityEnforcer   PolicyClient   022-Policy   EventEmitter   025-Audit
    │               │                  │                  │              │            │              │
    │──DELETE /agents/{urn}─▶│           │                  │              │            │              │
    │                │──Authorize(kill,urn)───────────────▶│              │              │            │              │
    │                │                  │──Evaluate───────▶│              │              │            │              │
    │                │                  │                  │──gRPC───────▶│              │            │              │
    │                │                  │                  │◀── deny ─────│              │            │              │
    │                │                  │◀── deny(reason)─│              │              │            │              │
    │                │◀── AIOS-CAP-0001 (deny) ───────────│              │              │            │              │
    │                │──EnqueueTx(agent.syscall.denied)────────────────────────────────▶│              │
    │                │──EmitAuditRecord(deny,kill,urn)─────────────────────────────────────────────────▶│
    │◀─403 Forbidden─│                  │                  │              │            │              │
    │  code=AIOS-CAP-0001               │                  │              │            │              │
```

**Notas:**
- **Default deny**: qualquer indisponibilidade do PDP com
  `kernel.pep.fail_mode=closed` produz o mesmo resultado final (`deny`), com
  `code=AIOS-CAP-0003` em vez de `AIOS-CAP-0001` (ver variante abaixo).
- O evento `agent.syscall.denied` e o registro de auditoria **DEVEM** ser
  emitidos mesmo em caso de negação — negações são eventos de governança de
  primeira classe (R-09, NFR-010).

**Variante: PDP indisponível, `fail_mode=closed`**

```
CapabilityEnforcer   PolicyClient        022-Policy
        │                  │                  │
        │──Evaluate───────▶│                  │
        │                  │──gRPC───────────▶│ (timeout/circuit aberto)
        │                  │  ╎── CB timeout ──╎
        │                  │◀── erro/CB open ─│
        │◀── AIOS-CAP-0003 (fail-closed) ────│
```

---

## 6. `suspend` → `resume` — Caminho Feliz

**Pré-condição:** ACB em `Running`; capability `agent:suspend`/`agent:resume`
concedida.
**Pós-condição:** ACB transita `Running → Suspended → Running` (T-06, T-07).

```
Cliente(007)   SyscallGateway   CapabilityEnforcer   AcbStore   SchedulerClient   EventEmitter
    │               │                  │                │              │              │
    │──POST :suspend─▶│                  │                │              │              │
    │                │──Authorize(suspend)─────────────▶│                │              │              │
    │                │◀── allow ────────│                │              │              │
    │                │──TryUpdate(Running→Suspended,v→v+1)▶│              │              │
    │                │◀── ok ───────────────────────────│                │              │              │
    │                │──EnqueueTx(agent.lifecycle.suspended)─────────────────────────────▶│
    │◀─200 OK────────│                  │                │              │              │
    │       ...        (agente ocioso, estado quente preservado em Redis)                │
    │──POST :resume──▶│                  │                │              │              │
    │                │──Authorize(resume)──────────────▶│                │              │              │
    │                │◀── allow ────────│                │              │              │
    │                │──RequestAdmission(re-reserve slot)──────────────▶│              │
    │                │◀── slot granted ─────────────────────────────────│              │              │
    │                │──TryUpdate(Suspended→Running,v+1→v+2)▶│              │              │
    │                │◀── ok ───────────────────────────│                │              │              │
    │                │──EnqueueTx(agent.lifecycle.resumed)───────────────────────────────▶│
    │◀─200 OK────────│                  │                │              │              │
```

**Notas:** meta de latência p99 ≤ 20 ms para `suspend`/`resume` (NFR-002),
pois nenhuma materialização de runtime é necessária (estado permanece quente
em Redis).

---

## 7. Hibernação por Ociosidade e `resume` de Agente *Cold*

**Gatilho da hibernação:** processo assíncrono periódico do
`LifecycleCoordinator` varre ACBs `Suspended` com
`now - last_active_at > hibernation.idle_ttl_ms` (T-08). **Não** é uma syscall
do cliente.

```
Varredura periódica   LifecycleCoordinator   CheckpointManager   LifecycleClient   AcbStore   EventEmitter   008-Lifecycle(MinIO)
        │                     │                     │                  │              │              │              │
        │──scan(Suspended,idle_ttl)─▶│               │                  │              │              │              │
        │                     │──CreateCheckpoint(urn)──────────────▶│                  │              │              │              │
        │                     │                     │──Hibernate(urn)─────────────────▶│              │              │              │
        │                     │                     │                  │              │              │──snapshot──▶│
        │                     │                     │                  │◀── checkpoint_ref ───────────│              │
        │                     │◀── checkpoint_ref ──│                  │              │              │              │
        │                     │──TryUpdate(Suspended→Hibernated,runtime_ref=null,checkpoint_ref)───▶│              │              │
        │                     │◀── ok ──────────────│                  │              │              │              │
        │                     │──EnqueueTx(agent.lifecycle.hibernated)──────────────────────────────▶│              │
```

**`resume` de agente *cold* (T-09), disparado por syscall do cliente:**

```
Cliente(007)   SyscallGateway   LifecycleCoordinator   SchedulerClient   LifecycleClient   AcbStore   EventEmitter
    │               │                   │                     │                │              │              │
    │──POST :resume─▶│                   │                     │                │              │              │
    │                │──Resume(urn)─────▶│                     │                │              │              │
    │                │                   │──RequestAdmission───────────────────▶│                │              │              │
    │                │                   │◀── slot granted ────────────────────│                │              │              │
    │                │                   │──TryUpdate(Hibernated→Admitted,v→v+1)──────────────▶│              │              │
    │                │                   │◀── ok ──────────────────────────────────────────────│              │              │
    │                │                   │──Restore(urn,checkpoint_ref)────────────────────────────────────▶│              │
    │                │                   │  ╎── alvo p99 < 250ms (materialização sob demanda) ──╎           │              │
    │                │                   │◀── runtime_ref ──────────────────────────────────────────────────│              │
    │                │                   │──TryUpdate(Admitted→Running,runtime_ref)───────────────────────▶│              │
    │                │                   │◀── ok ──────────────────────────────────────────────│              │              │
    │                │                   │──EnqueueTx(agent.lifecycle.running)────────────────────────────────────────────▶│
    │◀─200 OK────────│                   │                     │                │              │              │
```

---

## 8. `kill` — Encerramento com Compensação

**Pré-condição:** ACB em `Running`, `Suspended` ou `Hibernated`; capability
`agent:kill` e (`caller = owner` ou `role = admin`).
**Pós-condição:** ACB → `Terminating` → `Terminated` (T-10, T-11); toda cota
de escopo `agent` liberada; runtime finalizado.

```
Cliente(007)   SyscallGateway   CapabilityEnforcer   LifecycleCoordinator   LifecycleClient   ResourceQuotaManager   AcbStore   EventEmitter   008-Lifecycle
    │               │                  │                     │                    │                  │              │              │              │
    │──DELETE /agents/{urn}─▶│           │                     │                    │                  │              │              │              │
    │                │──Authorize(kill,owner-or-admin)───────▶│                    │                  │              │              │              │
    │                │◀── allow ────────│                     │                    │                  │              │              │              │
    │                │──Kill(urn,reason)──────────────────────▶│                    │                  │              │              │              │
    │                │                  │                     │──TryUpdate(→Terminating)─────────────────────────▶│              │              │
    │                │                  │                     │◀── ok ────────────────────────────────────────────│              │              │
    │                │                  │                     │  (drenagem: syscalls em voo finalizadas ou abortadas)             │              │
    │                │                  │                     │──Hibernate/Stop(urn)──────────────▶│                  │              │              │
    │                │                  │                     │                                    │              │              │──stop process─▶│
    │                │                  │                     │◀── stopped ────────────────────────│                  │              │              │
    │                │                  │                     │──Release(all agent-scoped reservations)──────────────▶│              │              │
    │                │                  │                     │◀── ok ─────────────────────────────────────────────│              │              │
    │                │                  │                     │──TryUpdate(Terminating→Terminated,terminated_at)──────────────────▶│              │
    │                │                  │                     │◀── ok ────────────────────────────────────────────│              │              │
    │                │                  │                     │──EnqueueTx(agent.lifecycle.terminated)───────────────────────────────────────────▶│
    │◀─204 No Content│                  │                     │                    │                  │              │              │              │
```

**Notas:**
- Se o chamador não for `owner` nem tiver `role=admin`, o PEP nega antes de
  qualquer efeito (`AIOS-CAP-0001`), conforme fluxo genérico do §5.
- Se `kill` for chamado a partir de `Pending`/`Admitted` (antes do boot
  concluir), o Kernel **DEVE** primeiro aguardar a resolução de T-02/T-03/T-04
  ou tratar como cancelamento de admissão — este é um fluxo alternativo não
  coberto pela FSM básica e **DEVE** ser tratado como `AIOS-KERNEL-0002` se
  solicitado fora de ordem, exigindo que o cliente aguarde o estado
  estabilizar.

---

## 9. `invoke_tool` — Brokering de Recurso, Caminho Feliz

**Pré-condição:** ACB em `Running`; capability `tool:invoke:<tool_id>`
concedida; cota `tools_limit` disponível.

```
Cliente(007)   SyscallGateway   CapabilityEnforcer   ResourceQuotaManager   ResourceBrokerRouter   EventEmitter   015-Tool-Manager
    │               │                  │                     │                     │                  │              │
    │──POST /tools:invoke─▶│            │                     │                     │                  │              │
    │  (Idem-Key)    │──IsCached?──────▶│(IdempotencyStore)   │                     │                  │              │
    │                │◀── miss ────────│                     │                     │                  │              │
    │                │──Authorize(invoke_tool,tool_id)──────▶│                     │                  │              │
    │                │◀── allow ────────│                     │                     │                  │              │
    │                │──Reserve(tools,1)───────────────────────────────────────▶│                     │                  │              │
    │                │◀── Reserved(reservationId) ──────────│                     │                     │                  │              │
    │                │──InvokeTool(urn,call)──────────────────────────────────────────────────────▶│                  │              │
    │                │                     │                     │                     │──gRPC──────▶│
    │                │                     │                     │                     │◀── ToolResult ─│
    │                │◀── ToolResult ──────────────────────────────────────────────────│                  │              │
    │                │──Commit(reservationId)──────────────────────────────────────▶│                     │                  │              │
    │                │◀── ok ────────────│                     │                     │                  │              │
    │                │──EnqueueTx(agent.syscall.completed)*───────────────────────────────────────────────▶│
    │                │──SaveResult(key,ToolResult)─────────▶│(IdempotencyStore)   │                     │                  │              │
    │◀─200 OK────────│                  │                     │                     │                  │              │
    │  ToolResult     │                  │                     │                     │                  │              │
```

`* Nota: eventos de syscall de recurso concluída são detalhados em
./Events.md; este diagrama ilustra a sequência de controle, não o catálogo
completo de eventos.`

---

## 10. `invoke_tool` — Cota Excedida (Falha)

**Gatilho:** `tools_limit` da janela corrente já consumido (ou `syscalls_per_sec`
excedido). **Pós-condição:** nenhuma reserva efetivada; erro `429`.

```
Cliente(007)   SyscallGateway   CapabilityEnforcer   ResourceQuotaManager   EventEmitter
    │               │                  │                     │                  │
    │──POST /tools:invoke─▶│            │                     │                  │
    │                │──Authorize(invoke_tool)───────────────▶│                  │              │
    │                │◀── allow ────────│                     │                  │
    │                │──Reserve(tools,1)───────────────────────────────────────▶│
    │                │                     (script Lua: saldo insuficiente)      │
    │                │◀── AIOS-QUOTA-0003 (tools limit) ─────│                  │
    │                │──EnqueueTx(agent.quota.exceeded)───────────────────────────────────────▶│
    │◀─429 Too Many Requests─│              │                     │                  │
    │  code=AIOS-QUOTA-0003, retriable=true, retryAfterMs=<janela restante>          │
```

**Notas:**
- A capability já havia sido concedida (`allow`) — a negação aqui é
  puramente de **cota**, não de autorização; os domínios de erro são
  distintos (`AIOS-CAP-*` vs. `AIOS-QUOTA-*`), permitindo ao cliente
  diferenciar "não posso" de "não tenho orçamento agora".
- Quando `enforcement=soft`, o mesmo cenário **NÃO DEVE** retornar `429`:
  a reserva é concedida, um evento `agent.quota.warning` é emitido, e a
  syscall prossegue (comportamento alternativo, não ilustrado acima).

---

## 11. `checkpoint` — Caminho Feliz e Falha de Snapshot

**Caminho feliz** (self-loop T-13, ACB permanece em `Running`/`Suspended`):

```
Cliente(007)   SyscallGateway   CapabilityEnforcer   CheckpointManager   LifecycleClient   AcbStore   EventEmitter   008-Lifecycle(MinIO)
    │               │                  │                     │                  │              │              │              │
    │──POST :checkpoint─▶│              │                     │                  │              │              │              │
    │                │──Authorize(checkpoint)────────────────▶│                  │              │              │              │
    │                │◀── allow ────────│                     │                  │              │              │              │
    │                │──CreateCheckpoint(urn)─────────────────────────────────▶│                  │              │              │
    │                │                     │                     │──snapshot(memory_ptr,context_ptr)──▶│              │              │
    │                │                     │                     │              │              │──put object──▶│
    │                │                     │                     │◀── checkpoint_ref ───────────│              │
    │                │                     │◀── checkpoint_ref ──│                  │              │              │
    │                │                     │──TryUpdate(checkpoint_ref)──────────────────────────▶│              │              │
    │                │                     │◀── ok ──────────────────────────────────────────────│              │              │
    │                │                     │──EnqueueTx(agent.checkpoint.created)────────────────────────────────────────────▶│
    │◀─200 OK────────│                     │  checkpoint_ref     │                  │              │              │              │
```

**Falha de snapshot** (MinIO/Lifecycle indisponível):

```
CheckpointManager   LifecycleClient   008-Lifecycle(MinIO)
        │                  │                  │
        │──snapshot(...)──▶│                  │
        │                  │──put object─────▶│ (falha/timeout)
        │                  │◀── erro ─────────│
        │◀── AIOS-KERNEL-0005 (broker de recurso, retriable) ─│
```
```
CapabilityEnforcer não é reconsultado; o ACB permanece no estado atual
(nenhuma transição ocorreu — checkpoint é self-loop, então falha simplesmente
não atualiza checkpoint_ref); o cliente recebe 502 e PODE tentar novamente
(idempotente: uma nova tentativa de checkpoint apenas sobrescreve o snapshot
mais recente).
```

---

## 12. Replay Idempotente de uma Mutação (`Idempotency-Key` Repetida)

**Cenário:** o cliente reenvia o mesmo `POST /agents` (mesmo
`Idempotency-Key`) após não receber a resposta original (timeout de rede no
cliente), enquanto o Kernel já havia processado a primeira chamada com
sucesso.

```
Cliente(007)   SyscallGateway   IdempotencyStore
    │               │                  │
    │──POST /agents (Idem-Key=K)─▶│      │
    │                │──TryGetCachedResult(K)──▶│
    │                │◀── hit: result(201,ACB{urn,Running}) ──│
    │◀─201 Created (idêntico à 1ª resposta)──│  (nenhuma nova saga de spawn é executada)
    │               │                  │
```

**Cenário de conflito** (mesma chave, payload divergente — ex.: `priority`
diferente na segunda chamada):

```
Cliente(007)   SyscallGateway   IdempotencyStore
    │               │                  │
    │──POST /agents (Idem-Key=K, priority=3)──▶│  (1ª chamada, priority=5)
    │◀─201 Created───│                  │
    │──POST /agents (Idem-Key=K, priority=3)──▶│  (2ª chamada, priority=3 ≠ 5)
    │                │──TryGetCachedResult(K)──▶│
    │                │◀── hit, payloadHash diverge ──│
    │◀─409 Conflict──│                  │
    │  code=AIOS-SYSCALL-0002           │
```

---

## 13. Recuperação do Outbox após Crash entre Commit e Publish

**Cenário de falha:** o Kernel Service falha (crash de processo/nó) após
commitar a transação que grava o ACB **e** o `OutboxMessage` associado, mas
antes do relay confirmar a publicação no NATS (`published=true`).

```
LifecycleCoordinator   PostgreSQL(kernel.outbox)   EventEmitter(relay)   NATS/JetStream
        │                       │                         │                    │
        │──commit(ACB + Outbox row, published=false)──▶│                         │                    │
        │◀── commit ok ─────────│                         │                    │
        ╎══ CRASH do processo do Kernel (antes do relay publicar) ══╎           │
        │                       │                         │                    │
        (nova réplica do Kernel assume; estado no PostgreSQL está intacto)      │
        │                       │◀── poll(published=false, a cada relay_interval_ms=50ms) │
        │                       │──── linha pendente ────▶│                    │
        │                       │                         │──publish(subject,payload)──▶│
        │                       │                         │◀── ack ──────────│
        │                       │◀── mark published=true ─│                    │
```

**Notas:**
- Nenhum estado é perdido: o ACB já estava commitado no PostgreSQL antes do
  crash; apenas a publicação assíncrona estava pendente (NFR-007: perda de
  evento = 0 sob crash).
- Consumidores (`009`, `008`, `010`..`017`, `024`, `025`) **DEVEM** deduplicar
  por `event.id` (ULID do `OutboxMessage.Id`), pois o relay **PODE**, em
  cenários raros de crash durante o próprio `publish`+`mark`, reenviar um
  evento já publicado (semântica **at-least-once** — RFC-0001 §5.2/§5.5).
- Este fluxo materializa a meta de recuperação RTO ≤ 15 min / RPO ≤ 5 min do
  brief §9: o RPO real deste caminho específico é limitado apenas pelo
  `kernel.outbox.relay_interval_ms` (default 50 ms), muito abaixo da meta.

---

## 14. Referências

- Máquina de estados subjacente: `./StateMachine.md`
- Interfaces e contratos por operação: `./ClassDiagrams.md` §4
- Componentes e suas dependências: `./Architecture.md` §4–§5
- Catálogo de erros: `./_DESIGN_BRIEF.md` §5.2 (a detalhar em `./API.md`)
- Catálogo de eventos: `./_DESIGN_BRIEF.md` §6 (a detalhar em `./Events.md`)
- Contratos centrais (envelope de evento/erro, idempotência, correlação):
  `../003-RFC/RFC-0001-Architecture-Baseline.md`

*Fim do documento `SequenceDiagrams.md` do módulo 006-Kernel.*
