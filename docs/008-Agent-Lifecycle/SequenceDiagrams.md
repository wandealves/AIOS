---
Documento: SequenceDiagrams
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0083, ADR-0084, ADR-0085, ADR-0086, ADR-0087
RFCs relacionados: RFC-0001 (baseline); RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore Protocol, a propor)
Depende de: 008-Agent-Lifecycle/_DESIGN_BRIEF.md, 008-Agent-Lifecycle/StateMachine.md, 008-Agent-Lifecycle/ClassDiagrams.md
---

# 008-Agent-Lifecycle — Diagramas de Sequência

> Este documento detalha, em ASCII, os fluxos críticos de execução do módulo
> 008 — caminho feliz e caminho de falha — com participantes explícitos e
> anotação de mensagens síncronas (`──▶`/`◀──`, requisição/resposta
> bloqueante) e assíncronas (`┈┈▶`, publicação/consumo de evento via
> NATS/JetStream). Todos os fluxos reutilizam os contratos de
> `../003-RFC/RFC-0001-Architecture-Baseline.md` (envelope de evento,
> `traceparent`, `X-AIOS-Tenant`, `Idempotency-Key`, envelope de erro RFC 7807)
> sem redefini-los.

## Índice

1. Convenções de notação
2. Spawn de agente novo com WarmPool (caminho feliz)
3. Spawn — timeout de admissão do Scheduler (caminho de falha)
4. Suspend → Resume (caminho feliz)
5. Hibernação por ociosidade → Wake sob demanda (caminho feliz)
6. Wake — checkpoint corrompido (caminho de falha)
7. Checkpoint sob demanda (caminho feliz e falha de storage)
8. Migração completa entre shards (caminho feliz, saga)
9. Migração — destino cai durante a saga (caminho de falha, compensação)
10. Terminate + expurgo assíncrono (Tombstone/LGPD)
11. Reconciliação — runtime morto sem `Terminated` (detecção e recuperação)
12. Concorrência — duas transições simultâneas (conflito de lease)
13. Referências

---

## 1. Convenções de Notação

```
Participante A          Participante B
     │                        │
     │──── msg síncrona ─────▶│   chamada bloqueante (REST/gRPC), aguarda resposta
     │◀─── resposta ──────────│
     │                        │
     │┈┈┈ msg assíncrona ┈┈▶│   publicação/consumo de evento (NATS/JetStream)
     │                        │
     │  ╔══════════════╗      │
     │  ║  nota/decisão ║      │   comentário sobre guarda/estado
     │  ╚══════════════╝      │
     │ ─ ─ ─ (timeout) ─ ─ ─ ▶│   temporizador armado
```

---

## 2. Spawn de Agente Novo com WarmPool (Caminho Feliz)

**Participantes:** Cliente (API Gateway/SDK), `LifecycleApiSurface`,
`LifecyclePolicyEnforcer`, `LifecycleCoordinator`, `LeaseManager`,
`StateMachineEngine`, `SchedulerClient`(→`009`), `WarmPoolManager`,
`SpawnManager`, `AcbStore`, `LifecycleEventPublisher`, NATS/JetStream.

```
Cliente   ApiSurface  PolicyEnf  Coordinator  LeaseMgr  StateMachine  Scheduler(009)  WarmPool  SpawnMgr  AcbStore  EventPub   NATS
  │           │            │          │           │           │            │            │          │          │        │        │
  │ POST /v1/agents        │          │           │           │            │            │          │          │        │        │
  │ Idempotency-Key: K1    │          │           │           │            │            │          │          │        │        │
  │──────────▶│            │          │           │           │            │            │          │          │        │        │
  │           │──Authorize▶│          │           │           │            │            │          │          │        │        │
  │           │            │  lifecycle:spawn      │           │            │            │          │          │        │        │
  │           │◀── Allow ──│          │           │           │            │            │          │          │        │        │
  │           │───────────────────────▶│           │           │            │            │          │          │        │        │
  │           │            │          │──Acquire──▶│           │            │            │          │          │        │        │
  │           │            │          │◀─Lease OK─│           │            │            │          │          │        │        │
  │           │            │          │──Evaluate(trigger=Spawn)──────────▶│            │          │          │        │        │
  │           │            │          │            │           │  G0: tenant ok,          │          │          │        │        │
  │           │            │          │            │           │  ACB válido              │          │          │        │        │
  │           │            │          │◀── plan: Created ──────│            │            │          │          │        │        │
  │           │            │          │──RequestAdmission──────────────────▶│            │          │          │        │        │
  │           │            │          │            │           │            │ G1: cotas ok│          │          │        │        │
  │           │            │          │◀── AdmissionDecision(slot) ─────────│            │          │          │        │        │
  │           │            │          │   ACB → Ready (T2)      │            │            │          │          │        │        │
  │           │            │          │──AppendTransition(Created→Ready)───────────────────────────▶│        │        │
  │           │            │          │──TryAcquireWarmRuntime─────────────────────────────▶│          │        │        │
  │           │            │          │            │           │            │            │ runtime pré-aquecido│        │        │
  │           │            │          │◀── RuntimeInstanceRef ──────────────────────────────│          │        │        │
  │           │            │          │   ACB → Running (T3), generation+1   │            │          │          │        │        │
  │           │            │          │──AppendTransition(Ready→Running)+EnqueueOutbox(tx)──────────▶│        │        │
  │           │            │          │                                                              │        │        │
  │           │            │          │──RelayPendingBatch────────────────────────────────────────────────────▶│        │
  │           │            │          │                                                              │  publish│
  │           │            │          │                                                              │        │┈┈▶│
  │           │            │          │                                                              │        │agent.  │
  │           │            │          │                                                              │        │lifecycle│
  │           │            │          │                                                              │        │.running│
  │           │◀── 202 Accepted (AgentControlBlock, State=Running) ─────────│            │            │          │        │        │
  │◀──────────│            │          │           │           │            │            │          │          │        │        │
  │           │            │          │──Release(lease)────────────────────▶│           │           │            │            │          │          │        │        │
```

**Notas:**
- Latência-alvo ponta a ponta: `p99 ≤ 250 ms` (NFR-001), viabilizada por
  `WarmPoolManager.TryAcquireWarmRuntime` retornar um runtime já aquecido
  (evita boot completo de processo — ADR-0083).
- `Idempotency-Key: K1` é persistida por `LifecycleApiSurface` (via
  `IIdempotencyStore` equivalente do brief, RFC-0001 §5.5); uma repetição da
  mesma requisição retorna o mesmo `202 Accepted` sem reexecutar o spawn.
- `AppendTransition` + `EnqueueOutbox` ocorrem na **mesma transação** de
  banco de dados (INV2): a publicação em NATS é sempre um passo posterior e
  assíncrono, nunca acoplado à resposta síncrona ao cliente.

---

## 3. Spawn — Timeout de Admissão do Scheduler (Caminho de Falha)

**Participantes:** Cliente, `LifecycleApiSurface`, `LifecycleCoordinator`,
`LeaseManager`, `SchedulerClient`(→`009`), `AcbStore`, `LifecycleEventPublisher`.

```
Cliente    ApiSurface   Coordinator   LeaseMgr   Scheduler(009)   AcbStore   EventPub
  │            │             │            │            │              │          │
  │ POST /v1/agents          │            │            │              │          │
  │───────────▶│             │            │            │              │          │
  │            │────────────▶│            │            │              │          │
  │            │             │──Acquire──▶│            │              │          │
  │            │             │◀─Lease OK─│            │              │          │
  │            │             │   ACB → Created (T1)     │              │          │
  │            │             │──RequestAdmission───────▶│              │          │
  │            │             │  ─ ─ ─ (spawn.timeout_ms = 1500) ─ ─ ─ ▶│          │
  │            │             │            │   sem resposta            │          │
  │            │             │◀── timeout (sem AdmissionDecision) ────│          │
  │            │             │   G1 falha: admissão negada/timeout     │          │
  │            │             │   ACB → Failed (T11, fail[G9])          │          │
  │            │             │──TryUpdate(state=Failed, failure_code=  │          │
  │            │             │   AIOS-LIFECYCLE-0006)─────────────────▶│          │
  │            │             │──AppendTransition+EnqueueOutbox(tx)─────▶│          │
  │            │             │─────────────────────────────────────────────────▶│
  │            │             │                                                    │┈┈▶ agent.
  │            │             │                                                    │    lifecycle.failed
  │            │◀── 202 Accepted (posteriormente consultável via GET,             │          │
  │            │    State=Failed, failure_code=AIOS-LIFECYCLE-0006) ──│           │          │
  │◀───────────│             │            │            │              │          │
  │            │             │──Release(lease)─────────▶│              │          │
```

**Notas:**
- O spawn é aceito como `202 Accepted` (assíncrono por natureza); o cliente
  observa o resultado via `GET /v1/agents/{id}` ou `GET
  /v1/agents/{id}/lifecycle`. Uma falha de admissão **não** é retornada como
  erro síncrono da requisição de criação, mas como estado terminal `Failed`
  do ACB.
- `AIOS-LIFECYCLE-0006` (timeout de materialização) é o código de erro
  persistido em `failure_code`; o evento `agent.lifecycle.failed` carrega
  `reason` com o detalhe.
- A guarda G1 (§3 de `./StateMachine.md`) nunca foi satisfeita — por isso a
  transição efetiva registrada é `Created → Failed`, pulando `Ready`/`Running`
  (consistente com INV5, que só exige passagem por `Ready`/`Running` antes de
  pausa/hibernação/migração — `terminate`/`fail` podem ocorrer de qualquer
  estado não-terminal).

---

## 4. Suspend → Resume (Caminho Feliz)

**Participantes:** Cliente, `LifecycleApiSurface`, `LifecyclePolicyEnforcer`,
`LifecycleCoordinator`, `LeaseManager`, `AcbStore`, `LifecycleEventPublisher`,
`Agent Runtime`(`007`).

```
Cliente   ApiSurface  PolicyEnf  Coordinator  LeaseMgr  Runtime(007)  AcbStore  EventPub
  │           │            │          │           │           │           │        │
  │ POST /v1/agents/{id}/suspend       │           │           │           │        │
  │──────────▶│            │          │           │           │           │        │
  │           │──Authorize▶│(lifecycle:suspend)    │           │           │        │
  │           │◀── Allow ──│          │           │           │           │        │
  │           │───────────────────────▶│           │           │           │        │
  │           │            │          │──Acquire──▶│           │           │        │
  │           │            │          │◀─Lease OK─│           │           │        │
  │           │            │          │  G4: sem operação crítica em curso │        │
  │           │            │          │──Suspend(runtime_instance_id)─────▶│        │
  │           │            │          │◀── ack (working memory preservada)│        │
  │           │            │          │  ACB → Suspended (T4)              │        │
  │           │            │          │──AppendTransition+EnqueueOutbox(tx)▶│        │
  │           │            │          │──RelayPendingBatch─────────────────────────▶│
  │           │◀── 202 Accepted ───────│           │           │           │  ┈┈▶ agent.lifecycle.suspended
  │◀──────────│            │          │──Release(lease)────────▶│           │        │
  │           │            │          │           │           │           │        │
  │ POST /v1/agents/{id}/resume        │           │           │           │        │
  │──────────▶│            │          │           │           │           │        │
  │           │──Authorize▶│(lifecycle:resume)     │           │           │        │
  │           │◀── Allow ──│          │           │           │           │        │
  │           │───────────────────────▶│           │           │           │        │
  │           │            │          │──Acquire──▶│           │           │        │
  │           │            │          │◀─Lease OK─│           │           │        │
  │           │            │          │  G5: slot disponível, working memory íntegra│
  │           │            │          │──Resume(runtime_instance_id)──────▶│        │
  │           │            │          │◀── ack (execução retomada) ───────│        │
  │           │            │          │  ACB → Running (T5)                │        │
  │           │            │          │──AppendTransition+EnqueueOutbox(tx)▶│        │
  │           │◀── 202 Accepted ───────│           │           │           │        │
  │◀──────────│            │          │──Release(lease)────────▶│           │        │
```

**Notas:**
- O contrato de FR-003 é verificado por hash: `resume` **DEVE** recuperar
  estado idêntico ao pré-`suspend` (hash da working memory igual). Essa
  verificação ocorre dentro de `Runtime(007)` no `ack` de `Resume`.
- Nenhuma serialização para MinIO ocorre neste fluxo — a working memory
  permanece em RAM/Redis durante `Suspended` (distinção crucial frente à
  hibernação, §5).

---

## 5. Hibernação por Ociosidade → Wake sob Demanda (Caminho Feliz)

**Participantes:** `HibernationController` (loop periódico),
`LeaseManager`, `CheckpointService`, `SnapshotStore` (MinIO+PG), `AcbStore`,
`LifecycleEventPublisher`, Cliente (requisição posterior), `SpawnManager`,
`WarmPoolManager`.

```
HibernationCtl  LeaseMgr  CheckpointSvc  SnapshotStore  AcbStore  EventPub    (--- tempo depois ---)  Cliente  ApiSurface  Coordinator  SpawnMgr  WarmPool
      │              │           │              │           │         │                                  │         │           │           │          │
      │  (loop, hibernation.idle_ttl_s=300)      │           │         │                                  │         │           │           │          │
      │──ScanIdle(shard)────────▶│              │           │         │                                  │         │           │           │          │
      │  agente X: idle=310s ≥ idle_ttl          │           │         │                                  │         │           │           │          │
      │──Acquire(agentX)────────▶│              │           │         │                                  │         │           │           │          │
      │◀─Lease OK────────────────│              │           │         │                                  │         │           │           │          │
      │──Create(agentX, consistency=Consistent)──▶│           │         │                                  │         │           │           │          │
      │              │           │──Put(payload)──▶│           │         │                                  │         │           │           │          │
      │              │           │◀── storage_uri, content_hash │         │                                  │         │           │           │          │
      │              │           │  G6: checkpoint consistent gravado     │                                  │         │           │           │          │
      │◀── Checkpoint(id=CK1) ───│              │           │         │                                  │         │           │           │          │
      │  ACB → Hibernated (T6); runtime_instance_id→null; RAM liberada    │                                  │         │           │           │          │
      │──AppendTransition+EnqueueOutbox(tx)──────────────────▶│         │                                  │         │           │           │          │
      │──RelayPendingBatch────────────────────────────────────────────▶│                                  │         │           │           │          │
      │                                                                │┈┈▶ agent.lifecycle.hibernated       │         │           │           │          │
      │──Release(agentX)─────────▶│              │           │         │                                  │         │           │           │          │
      │                                                                                                    │         │           │           │          │
      │                                    (--- demanda chega mais tarde ---)                              │         │           │           │          │
      │                                                                                                    │ POST .../wake         │           │          │
      │                                                                                                    │────────▶│           │           │          │
      │                                                                                                    │         │──────────▶│           │          │
      │                                                                                                    │         │           │──Acquire(agentX)─────▶ LeaseMgr
      │                                                                                                    │         │           │◀─Lease OK─────────────│
      │                                                                                                    │         │           │  G5: VerifyIntegrity(CK1)=true
      │                                                                                                    │         │           │──Materialize(agentX, CK1)──▶│
      │                                                                                                    │         │           │           │──TryAcquireWarmRuntime──▶│
      │                                                                                                    │         │           │           │◀── RuntimeInstanceRef ──│
      │                                                                                                    │         │           │◀── ack (< 250ms) ──────│
      │                                                                                                    │         │           │  ACB → Ready→Running (T7); generation+1; wake_count+1
      │                                                                                                    │         │◀── 202 Accepted ──────│           │          │
      │                                                                                                    │◀────────│           │           │          │
```

**Notas:**
- A hibernação **DEVE** garantir checkpoint `Consistent` **antes** de liberar
  RAM (INV4); se o `Create` falhar, o agente permanece `Suspended` e a
  hibernação é adiada (ver §7, falha de storage).
- O *wake* reconstrói o runtime a partir do checkpoint usando o mesmo
  caminho de materialização do spawn (`WarmPoolManager` + `SpawnManager`),
  reforçando a meta `p99 ≤ 250 ms` mesmo a partir de frio (NFR-001, FR-005).
- `generation` incrementa e `wake_count` incrementa a cada wake — usados
  para detectar tentativas de reuso de uma encarnação obsoleta (proteção
  contra *replay* de checkpoint antigo).

---

## 6. Wake — Checkpoint Corrompido (Caminho de Falha)

**Participantes:** Cliente, `LifecycleApiSurface`, `LifecycleCoordinator`,
`CheckpointService`, `SnapshotStore`, `AcbStore`, `LifecycleEventPublisher`.

```
Cliente   ApiSurface  Coordinator  CheckpointSvc  SnapshotStore  AcbStore  EventPub
  │           │            │             │              │           │        │
  │ POST /v1/agents/{id}/wake            │              │           │        │
  │──────────▶│            │             │              │           │        │
  │           │───────────▶│             │              │           │        │
  │           │            │──VerifyIntegrity(CK1)──────▶│              │           │        │
  │           │            │             │──Get(storage_uri)──────────▶│           │        │
  │           │            │             │◀── payload ──────────────│           │        │
  │           │            │             │  sha-256 recomputado ≠ content_hash    │        │
  │           │            │◀── false (corrompido) ──────│              │           │        │
  │           │            │  G5 falha: checkpoint não íntegro          │           │        │
  │           │            │  → tenta último Checkpoint anterior íntegro (se houver)│        │
  │           │            │             │              │           │        │
  │           │            │  (nenhum checkpoint anterior válido disponível)  │        │
  │           │            │  ACB → Failed (T11); failure_code=AIOS-LIFECYCLE-0007   │
  │           │            │──TryUpdate(state=Failed)────────────────────────▶│        │
  │           │            │──AppendTransition+EnqueueOutbox(tx)──────────────▶│        │
  │           │            │──RelayPendingBatch──────────────────────────────────────▶│
  │           │◀── 422 Unprocessable (RFC7807, code=AIOS-LIFECYCLE-0007) ──────│        │
  │◀──────────│            │             │              │           │        │┈┈▶ agent.lifecycle.failed
```

**Notas:**
- `AIOS-LIFECYCLE-0007` (checkpoint corrompido) satisfaz o modo de falha
  descrito em `./_DESIGN_BRIEF.md` §9: rejeita, tenta checkpoint anterior;
  se nenhum íntegro existir, resulta em `Failed` com auditoria (`025`),
  atendendo NFR-009 (`0` restore de checkpoint corrompido não detectado).
- Como `wake` é uma operação idempotente por `Idempotency-Key`, uma
  repetição da mesma requisição retorna o mesmo `422` sem reexecutar a
  verificação de integridade.

---

## 7. Checkpoint sob Demanda (Caminho Feliz e Falha de Storage)

**Participantes:** Cliente, `LifecycleApiSurface`, `LifecycleCoordinator`,
`CheckpointService`, `SnapshotStore` (MinIO), `AcbStore`,
`LifecycleEventPublisher`.

```
== Caminho feliz ==
Cliente   ApiSurface  Coordinator  CheckpointSvc  SnapshotStore  AcbStore  EventPub
  │           │            │             │              │           │        │
  │ POST /v1/agents/{id}/checkpoint      │              │           │        │
  │──────────▶│            │             │              │           │        │
  │           │───────────▶│             │              │           │        │
  │           │            │──Create(agentId, Consistent)─▶│              │           │        │
  │           │            │             │──serializa ACB+working_memory_ref──│        │        │
  │           │            │             │──Put(payload cifrado AES-256-GCM,│        │        │
  │           │            │             │      codec=msgpack+zstd)────────▶│           │        │
  │           │            │             │◀── storage_uri ─────────────────│           │        │
  │           │            │◀── Checkpoint(id=CK2, content_hash) ──│              │           │        │
  │           │            │──TryUpdate(last_checkpoint_id=CK2)────────────────────▶│        │
  │           │            │──AppendTransition(self-loop)+EnqueueOutbox(tx)─────────▶│        │
  │           │◀── 201 Created (CheckpointRef) ─│             │              │           │        │
  │◀──────────│            │             │              │           │        │┈┈▶ agent.lifecycle.checkpointed

== Caminho de falha: storage indisponível ==
Cliente   ApiSurface  Coordinator  CheckpointSvc  SnapshotStore
  │           │            │             │              │
  │ POST /v1/agents/{id}/checkpoint      │              │
  │──────────▶│            │             │              │
  │           │───────────▶│             │              │
  │           │            │──Create(agentId, Consistent)─▶│
  │           │            │             │──Put(payload)──▶│
  │           │            │             │  ─ ─ (retry com backoff exponencial+jitter) ─ ─
  │           │            │             │◀── erro de I/O (MinIO indisponível) ─│
  │           │            │◀── Result.Failure(AIOS-LIFECYCLE-0014) ─│
  │           │◀── 503 Service Unavailable (RFC7807, retriable=true) ─│
  │◀──────────│            │             │              │
```

**Notas:**
- Meta de latência do caminho feliz: `p99 ≤ 500 ms` para working memory
  `≤ 64 MiB` (NFR-003).
- No caminho de falha, o ACB **permanece inalterado** (nenhuma transição é
  registrada) — `AIOS-LIFECYCLE-0014` é `retriable=true`; o cliente pode
  reenviar com o mesmo `Idempotency-Key`. Se o agente estava em processo de
  hibernação (via `HibernationController`), a hibernação é adiada e o
  agente permanece `Suspended` (degradação graciosa, `./_DESIGN_BRIEF.md`
  §9).

---

## 8. Migração Completa entre Shards (Caminho Feliz, Saga)

**Participantes:** `Cluster(027)` (evento de draining ou decisão de
rebalanceamento), `LifecycleCoordinator`, `LeaseManager`,
`MigrationOrchestrator`, `CheckpointService`, `SnapshotStore`, `SpawnManager`
(no shard destino), `AcbStore`, `LifecycleEventPublisher`.

```
Cluster(027)  Coordinator  LeaseMgr  MigrationOrch  CheckpointSvc  SnapshotStore  SpawnMgr(destino)  AcbStore  EventPub
    │              │            │            │              │              │              │              │        │
    │┈┈ node.draining ┈┈▶│      │            │              │              │              │              │        │
    │              │──Acquire(agentX)───────▶│              │              │              │              │        │
    │              │◀─Lease OK──│            │              │              │              │              │        │
    │              │──Start(agentX, target=shardY)──────────▶│              │              │              │        │
    │              │            │            │  G7: destino saudável (027) │              │              │        │
    │              │            │            │  ACB → Migrating (T8)       │              │              │        │
    │              │            │            │──AppendTransition+EnqueueOutbox(tx)────────────────────────▶│        │
    │              │            │            │──Phase=Quiesce (drena syscalls em voo)──────│              │        │
    │              │            │            │──Phase=Checkpoint───────────▶│              │              │        │
    │              │            │            │              │──Put(payload)─▶│              │              │        │
    │              │            │            │              │◀─ storage_uri─│              │              │        │
    │              │            │            │◀── Checkpoint(CK3, Consistent)│              │              │        │
    │              │            │            │──Phase=Transfer (replica blob p/ região destino)────────────│        │
    │              │            │            │──Phase=Materialize──────────────────────────▶│              │        │
    │              │            │            │              │              │◀── RuntimeInstanceRef (shardY)│        │
    │              │            │            │  destino sincronizado, generation confere    │              │        │
    │              │            │            │──Phase=Cutover (G8: origem drenada/invalidada)│              │        │
    │              │            │            │  ACB → Ready/Running (T9, no destino)         │              │        │
    │              │            │            │──AppendTransition+EnqueueOutbox(tx)────────────────────────▶│        │
    │              │            │            │──Phase=Cleanup (libera recursos da origem)     │              │        │
    │              │            │            │  MigrationJob.Status=Succeeded (terminal)      │              │        │
    │              │            │            │──RelayPendingBatch─────────────────────────────────────────────────▶│
    │              │──Release(agentX)────────▶│              │              │              │              │        │┈┈▶ agent.lifecycle.migrated
```

**Notas:**
- Meta: `p99 ≤ 2 s` para agente `≤ 64 MiB`, mesmo DC, sem perda de trabalho
  aceito (NFR-010).
- O `Cutover` é o único ponto de não-retorno da saga (§6.1 de
  `./StateMachine.md`); todas as fases anteriores são compensáveis sem
  perda.
- `LeaseManager` mantém a lease do `agent_id` durante toda a saga — nenhum
  outro coordenador pode iniciar uma transição concorrente sobre o mesmo
  agente (INV1), incluindo uma segunda tentativa de migração
  (`AIOS-LIFECYCLE-0009`).

---

## 9. Migração — Destino Cai Durante a Saga (Caminho de Falha, Compensação)

**Participantes:** `LifecycleCoordinator`, `MigrationOrchestrator`,
`SpawnManager`(destino), `Cluster(027)`, `AcbStore`, `LifecycleEventPublisher`.

```
Coordinator  MigrationOrch  SpawnMgr(destino)  Cluster(027)  AcbStore  EventPub
     │              │                │              │            │        │
     │──Start(agentX, target=shardY)─▶│              │            │        │
     │              │  ACB → Migrating (T8)          │            │        │
     │              │──Phase=Quiesce→Checkpoint→Transfer (ok)─────│        │
     │              │──Phase=Materialize────────────▶│              │        │
     │              │                │  ─ ─ (nó destino cai, sem resposta) ─ ─
     │              │                │┈┈ cluster.node.unhealthy ┈┈▶│            │        │
     │              │◀── timeout (migration.saga_timeout_ms=120000) ───────────│        │
     │              │  G8 falha: destino não saudável                │            │        │
     │              │──Compensate(agentX, reason="target unhealthy")──────────────│        │
     │              │  origem ainda autoritativa (fase < Cutover) → reversível    │        │
     │              │  MigrationJob.Status=Compensated (terminal)                 │        │
     │              │  ACB retorna ao LifecycleState anterior (ex.: Running na origem)│    │
     │              │──AppendTransition(Migrating→Running)+EnqueueOutbox(tx)──────▶│        │
     │              │──RelayPendingBatch──────────────────────────────────────────────────▶│
     │              │                │              │            │        │┈┈▶ agent.lifecycle.
     │              │                │              │            │        │    running (retomado)
```

**Notas:**
- Como a falha ocorreu **antes** do `Cutover`, a compensação é sempre
  possível e o agente **nunca perde trabalho aceito** — garantia central de
  FR-007 e da tabela de modos de falha (`./_DESIGN_BRIEF.md` §9).
- `AIOS-LIFECYCLE-0010` (destino de migração indisponível/insalubre) é o
  código associado à causa raiz, registrado em `MigrationJob`/auditoria.
- Se a falha ocorresse **após** o `Cutover` (raro), não haveria via de
  compensação e o agente transicionaria para `Failed` (T11) — cenário
  tratado como exceção rara na tabela FMEA, mitigada por tornar `Cutover` a
  fase mais curta e mais verificada da saga.

---

## 10. Terminate + Expurgo Assíncrono (Tombstone/LGPD)

**Participantes:** Cliente, `LifecycleApiSurface`,
`LifecyclePolicyEnforcer`, `LifecycleCoordinator`, `AcbStore`,
`LifecycleEventPublisher`, `TombstoneManager` (job periódico),
`SnapshotStore`, `025-Audit`.

```
Cliente   ApiSurface  PolicyEnf  Coordinator  AcbStore  EventPub      (--- retention.terminated_ttl_days depois ---)  TombstoneMgr  SnapshotStore  AcbStore  Audit(025)
  │           │            │          │           │        │                                                              │             │           │        │
  │ DELETE /v1/agents/{id}             │           │        │                                                              │             │           │        │
  │──────────▶│            │          │           │        │                                                              │             │           │        │
  │           │──Authorize▶│(lifecycle:terminate)  │        │                                                              │             │           │        │
  │           │◀── Allow ──│          │           │        │                                                              │             │           │        │
  │           │───────────────────────▶│           │        │                                                              │             │           │        │
  │           │            │  G10: efeitos colaterais drenados          │                                                              │             │           │        │
  │           │            │  ACB → Terminated (T10); terminated_at preenchido; cotas liberadas │                          │             │           │        │
  │           │            │──AppendTransition+EnqueueOutbox(tx)────────▶│        │                                                              │             │           │        │
  │           │            │──RelayPendingBatch─────────────────────────────────▶│                                                              │             │           │        │
  │           │◀── 202 Accepted ───────│           │        │┈┈▶ agent.lifecycle.terminated                                │             │           │        │
  │◀──────────│            │          │           │        │                                                              │             │           │        │
  │           │            │          │           │        │                                                              │             │           │        │
  │           │            │          │           │        │        (--- job periódico de retenção ---)                   │             │           │        │
  │           │            │          │           │        │                                                              │──PurgeExpired(tenant)──▶│           │        │
  │           │            │          │           │        │                                                              │  ACB.state=Terminated, terminated_at + 30d < now │
  │           │            │          │           │        │                                                              │──Delete(storage_uri de todos Checkpoints)──▶│           │        │
  │           │            │          │           │        │                                                              │◀── ok ──────│           │        │
  │           │            │          │           │        │                                                              │──MarkPurged(agentX)────────────────▶│        │
  │           │            │          │           │        │                                                              │──EmitAuditRecord(purge, agentX)──────────────▶│
  │           │            │          │           │        │                                                              │  agent_id NÃO reutilizável (anti-resurrection)│
```

**Notas:**
- `terminate` (T10) **PODE** ser disparado a partir de qualquer estado
  não-terminal (`Created`, `Ready`, `Running`, `Suspended`, `Hibernated`,
  `Migrating`), conforme a FSM canônica.
- O expurgo (`TombstoneManager`) é **assíncrono e desacoplado** do
  `terminate` — ocorre após `retention.terminated_ttl_days` (default 30
  dias) ou sob solicitação explícita de direito ao esquecimento (LGPD/GDPR).
- O evento de expurgo é auditável (`025-Audit`) mas **não** carrega conteúdo
  de working memory — apenas metadados (`./_DESIGN_BRIEF.md` §12.3).

---

## 11. Reconciliação — Runtime Morto sem `Terminated` (Detecção e Recuperação)

**Participantes:** `ReconciliationController` (loop periódico),
`AcbStore`, `Agent Runtime(007)` (ausência de heartbeat),
`LifecycleCoordinator`, `CheckpointService`, `SpawnManager`,
`LifecycleEventPublisher`.

```
ReconcileCtl  AcbStore  Runtime(007)  Coordinator  CheckpointSvc  SpawnMgr  EventPub
     │            │            │             │              │          │        │
     │  (loop, reconcile.interval_ms=2000)    │              │          │        │
     │──ListByTenantAndState(state=Running)──▶│             │              │          │        │
     │◀── [agentX: last_heartbeat há 18s] ────│             │              │          │        │
     │            │            │  runtime_dead_after_ms=15000 excedido     │          │        │
     │            │            │  ╔══ desvio: desejado=Running, observado=morto ══╗   │        │
     │──RequestTransition(agentX, trigger=Reconcile)─────────▶│              │          │        │
     │            │            │             │  tenta recuperar do último checkpoint  │          │
     │            │            │             │──VerifyIntegrity(last_checkpoint_id)──▶│          │        │
     │            │            │             │◀── true ──────│              │          │        │
     │            │            │             │──Materialize(agentX, checkpoint)───────▶│          │
     │            │            │             │◀── RuntimeInstanceRef (novo) ───────────│          │        │
     │            │            │  ACB → Ready→Running (recuperado); generation+1        │          │        │
     │            │            │──AppendTransition+EnqueueOutbox(tx)─────────────────────────────▶│        │
     │            │            │──RelayPendingBatch──────────────────────────────────────────────────────▶│
     │            │            │             │              │          │        │┈┈▶ agent.lifecycle.running
     │            │            │             │              │          │        │    (recuperado, RTO ≤ 15min)
```

**Notas:**
- Este fluxo materializa o modo de falha "Runtime (007) morre em `Running`"
  de `./_DESIGN_BRIEF.md` §9: detecção por ausência de heartbeat além de
  `reconcile.runtime_dead_after_ms`, recuperação a partir do último
  checkpoint válido, atendendo RTO ≤ 15 min (NFR-005) e RPO ≤ 5 min
  (NFR-006, dado `checkpoint.interval_s=120`).
- Se `VerifyIntegrity` falhar ou nenhum checkpoint estiver disponível dentro
  do orçamento (`recovery.rto_budget_s`), o `ReconciliationController`
  aciona `terminate`/`fail` (T11) em vez de recuperação (mesmo padrão do
  fluxo §6).

---

## 12. Concorrência — Duas Transições Simultâneas (Conflito de Lease)

**Participantes:** Cliente A, Cliente B, `LifecycleApiSurface` (duas
requisições concorrentes), `LifecycleCoordinator`, `LeaseManager`.

```
Cliente A   ApiSurface   Coordinator A   LeaseMgr   Coordinator B   ApiSurface   Cliente B
   │             │              │            │            │             │            │
   │ POST .../suspend            │            │            │             │            │
   │────────────▶│               │            │            │             │            │
   │             │──────────────▶│            │            │             │            │
   │             │               │──Acquire(agentX, holder=A)──▶│            │             │            │
   │             │               │◀─ Lease OK (fencing_token=42) ──│            │             │            │
   │             │               │            │            │             │            │
   │             │               │            │                          │ POST .../resume         │
   │             │               │            │                          │────────────▶│            │
   │             │               │            │◀─────────────────────────│            │            │
   │             │               │            │            │──Acquire(agentX, holder=B)──▶│
   │             │               │            │            │◀─ Lease NEGADA (já detida por A, TTL ativo)│
   │             │               │            │            │  AIOS-LIFECYCLE-0003             │
   │             │               │            │            │──────────────────────────▶│            │
   │             │               │            │            │             │◀── 409 Conflict (RFC7807, │
   │             │               │            │            │             │    code=AIOS-LIFECYCLE-0003,│
   │             │               │            │            │             │    retriable=true) ───────│
   │             │               │            │            │             │◀───────────│            │
   │             │               │  G4 ok: sem operação crítica em curso │             │            │
   │             │               │  ACB → Suspended (T4)   │            │             │            │
   │             │               │──Release(agentX)────────▶│            │             │            │
   │             │◀── 202 Accepted ─────────────│           │            │             │            │
   │◀────────────│               │            │            │             │            │
```

**Notas:**
- Este fluxo demonstra FR-010 e INV1: duas transições concorrentes sobre o
  mesmo `agent_id` — uma sucede, a outra recebe
  `AIOS-LIFECYCLE-0003` (lease não adquirida), `retriable=true`. O Cliente B
  **DEVE** reenviar após o TTL da lease (`lease.ttl_ms`) ou após observar
  conclusão via `GET /v1/agents/{id}`.
- `fencing_token=42` protege contra o cenário em que o Coordinator A trava
  (ex.: GC pause) e um novo coordenador assume: qualquer escrita tardia de A
  com token `42` seria rejeitada por `IAcbStore.TryUpdate` se um coordenador
  mais novo já tiver incrementado o token (proteção split-brain, §7 de
  `./StateMachine.md`).

---

## 13. Referências

- Máquina de estados subjacente a todos os fluxos: `./StateMachine.md`
- Estruturas de dados e interfaces referenciadas: `./ClassDiagrams.md`
- Arquitetura de componentes: `./Architecture.md`
- Catálogo de eventos publicados/consumidos: `./_DESIGN_BRIEF.md` §6
  (detalhado em `./Events.md`)
- Catálogo de erros: `./_DESIGN_BRIEF.md` §5.3 (detalhado em `./API.md`)
- Modos de falha e estratégia de recuperação: `./_DESIGN_BRIEF.md` §9
  (detalhado em `./FailureRecovery.md`)
- Contratos centrais (envelope de evento, erro RFC 7807, idempotência,
  correlação): `../003-RFC/RFC-0001-Architecture-Baseline.md`

*Fim do documento `SequenceDiagrams.md` do módulo 008-Agent-Lifecycle.*
