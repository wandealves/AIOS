---
Documento: ClassDiagrams
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0084, ADR-0085, ADR-0086
RFCs relacionados: RFC-0001 (baseline); RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore Protocol, a propor)
Depende de: 008-Agent-Lifecycle/_DESIGN_BRIEF.md, 008-Agent-Lifecycle/Architecture.md, 008-Agent-Lifecycle/StateMachine.md
---

# 008-Agent-Lifecycle — Diagramas de Classe / Estrutura

> Este documento detalha as estruturas de dados e interfaces internas do
> Agent Lifecycle em notação ASCII (adaptação de UML). Os campos e tipos
> reproduzem exatamente o modelo de dados de `./_DESIGN_BRIEF.md` §3 —
> nenhum campo é inventado ou renomeado. As interfaces de componente refletem
> `./Architecture.md` §4–§5. O módulo 008 **NÃO** é dono da criação do ACB
> (isso é do `006-Kernel`) — as estruturas abaixo representam a **visão de
> ciclo de vida** do ACB e as entidades próprias do módulo.

## Índice

1. Convenções de notação
2. Entidades de domínio (estruturas de dados)
3. Interfaces públicas dos componentes
4. Contratos (pré-condições, pós-condições)
5. Invariantes
6. Diagrama de relações (composição/agregação/dependência)
7. Enumerações e Value Objects
8. Referências

---

## 1. Convenções de Notação

```
┌─────────────────────────┐
│      NomeDaClasse        │   « stereotype »  ex.: «entity», «interface», «value object»
├─────────────────────────┤
│ + campoPublico : Tipo    │   + público   - privado   # protegido
│ - campoPrivado : Tipo    │
├─────────────────────────┤
│ + Metodo(param: Tipo)    │
│     : TipoRetorno        │
└─────────────────────────┘

  A ──▶ B      dependência (A usa B)
  A ──◇ B      agregação (A contém referência a B, ciclo de vida independente)
  A ──◆ B      composição (A possui B, ciclo de vida acoplado)
  A ──▷ B      A implementa a interface B
  A ──1..* B   cardinalidade
```

---

## 2. Entidades de Domínio (Estruturas de Dados)

### 2.1 `AgentControlBlock` («entity», projeção — tabela `lifecycle.acb_projection`)

> O 006-Kernel é dono da *criação* do ACB; o 008 é dono do *estado de ciclo
> de vida* dentro do ACB e de seu versionamento por `generation` (R2 do
> brief). Esta projeção é a visão que o módulo 008 mantém e mutaciona.

```
┌───────────────────────────────────────────────────────────────────┐
│               AgentControlBlock  «entity» (projeção 008)            │
├───────────────────────────────────────────────────────────────────┤
│ + AgentId              : Ulid    // urn:aios:<tenant>:agent:<id>    │
│ + TenantId              : string  // RLS boundary                   │
│ + State                 : LifecycleState                             │
│ + Generation             : long    // incrementa a cada spawn/       │
│                                     // restore/migração               │
│ + DesiredState          : LifecycleState  // alvo da reconciliação   │
│ + PriorityClass          : PriorityClass                              │
│ + PlacementShard         : int     // hash(tenant_id,agent_id) mod N  │
│ + PlacementNode          : string?  // nó/host atual (se quente)      │
│ + RuntimeInstanceId      : Ulid?   // instância 007 associada         │
│ + WorkingMemoryRef       : string?  // URN → 010                     │
│ + ContextRef             : string?  // URN → 011                     │
│ + PolicyRef              : string   // URN/versão → 022                │
│ + QuotaRef               : string   // URN → 006/026                  │
│ + LastCheckpointId       : Ulid?   // FK → Checkpoint                 │
│ + ColdSince              : DateTimeOffset?                             │
│ + WakeCount              : int                                         │
│ + FencingToken           : long    // token de lease vigente           │
│ + CreatedAt              : DateTimeOffset                              │
│ + UpdatedAt              : DateTimeOffset                              │
│ + TerminatedAt           : DateTimeOffset?                              │
│ + FailureCode            : string?  // AIOS-LIFECYCLE-<NNNN>          │
│ + Labels                 : IReadOnlyDictionary<string, JsonElement>   │
├───────────────────────────────────────────────────────────────────┤
│ + CanTransitionTo(target: LifecycleState) : bool                     │
│ + WithNextGeneration() : AgentControlBlock   // clone imutável +1     │
└───────────────────────────────────────────────────────────────────┘
```

Invariantes locais: ver §5 (I1–I4). Tratado como **imutável** na camada de
aplicação — toda mutação produz uma nova instância persistida via
`AcbStore.TryUpdate` com verificação de `FencingToken`/`Generation`.

### 2.2 `LifecycleTransition` («entity», event-sourced, append-only — tabela `lifecycle.transition`)

```
┌───────────────────────────────────────────────────────────────────┐
│                  LifecycleTransition  «entity»                       │
├───────────────────────────────────────────────────────────────────┤
│ + TransitionId          : Ulid       // = event.id                   │
│ + AgentId                : Ulid                                       │
│ + TenantId                : string                                    │
│ + Seq                     : long      // único (agent_id, seq)         │
│ + FromState               : LifecycleState                             │
│ + ToState                 : LifecycleState                              │
│ + Trigger                 : LifecycleTrigger                            │
│ + Actor                   : string    // "api:<sub>","scheduler",...   │
│ + GuardResult             : JsonDocument                               │
│ + Reason                  : string?                                     │
│ + Generation              : long                                        │
│ + Correlation             : CorrelationContext                          │
│ + CreatedAt                : DateTimeOffset                              │
└───────────────────────────────────────────────────────────────────┘
```

### 2.3 `Checkpoint` («entity», tabela `lifecycle.checkpoint`)

```
┌───────────────────────────────────────────────────────────────────┐
│                       Checkpoint  «entity»                            │
├───────────────────────────────────────────────────────────────────┤
│ + CheckpointId            : Ulid    // urn:aios:<tenant>:checkpoint:<id>│
│ + AgentId                  : Ulid                                       │
│ + TenantId                  : string                                     │
│ + Generation                : long    // encarnação capturada             │
│ + StorageUri                 : string  // minio://<bucket>/<tenant>/...   │
│ + SizeBytes                  : long                                       │
│ + ContentHash                : string  // sha-256                          │
│ + Codec                      : CheckpointCodec                             │
│ + Encryption                  : EncryptionAlgorithm                        │
│ + WorkingMemoryRef            : string  // URN → 010                       │
│ + Consistency                 : ConsistencyLevel                           │
│ + CreatedAt                    : DateTimeOffset                             │
│ + ExpiresAt                    : DateTimeOffset?                            │
├───────────────────────────────────────────────────────────────────┤
│ + VerifyIntegrity(payload: byte[]) : bool  // recomputa sha-256           │
└───────────────────────────────────────────────────────────────────┘
```

### 2.4 `HibernationRecord` («entity», tabela `lifecycle.hibernation_record`)

```
┌───────────────────────────────────────────────────────┐
│               HibernationRecord  «entity»                │
├───────────────────────────────────────────────────────┤
│ + AgentId               : Ulid    // PK composta c/ tenant│
│ + TenantId               : string                          │
│ + ColdSince               : DateTimeOffset                  │
│ + StorageTier             : StorageTier                     │
│ + CheckpointId            : Ulid    // FK → Checkpoint       │
│ + LastWakeLatencyMs       : int                              │
│ + WakeCount               : int                              │
└───────────────────────────────────────────────────────┘
```

### 2.5 `MigrationJob` («entity», saga — tabela `lifecycle.migration_job`)

```
┌───────────────────────────────────────────────────────────────────┐
│                       MigrationJob  «entity»                          │
├───────────────────────────────────────────────────────────────────┤
│ + MigrationId              : Ulid   // urn:aios:<tenant>:migration:<id> │
│ + AgentId                   : Ulid                                       │
│ + TenantId                   : string                                     │
│ + SourceShard                 : int                                        │
│ + SourceNode                  : string                                     │
│ + TargetShard                 : int                                        │
│ + TargetNode                  : string                                     │
│ + Phase                       : MigrationPhase                             │
│ + Status                      : MigrationStatus                            │
│ + CheckpointId                 : Ulid   // FK → Checkpoint                  │
│ + StartedAt                    : DateTimeOffset                             │
│ + FinishedAt                    : DateTimeOffset?                            │
├───────────────────────────────────────────────────────────────────┤
│ + AdvancePhase(next: MigrationPhase) : Result<MigrationJob>              │
│ + Compensate(reason: string) : MigrationJob   // → Status=Compensated    │
└───────────────────────────────────────────────────────────────────┘
```

### 2.6 `LifecycleOutbox` («entity», tabela `lifecycle.outbox`)

```
┌───────────────────────────────────────────────────────────────────┐
│                     LifecycleOutbox  «entity»                        │
├───────────────────────────────────────────────────────────────────┤
│ + EventId                : Ulid       // = event.id do CloudEvents    │
│ + TenantId                 : string                                     │
│ + Subject                  : string    // aios.<tenant>.agent.lifecycle.*│
│ + Payload                  : JsonDocument  // envelope CloudEvents      │
│ + Published                 : bool     // false até relay confirmar     │
│ + CreatedAt                  : DateTimeOffset                            │
├───────────────────────────────────────────────────────────────────┤
│ + MarkPublished() : LifecycleOutbox                                    │
└───────────────────────────────────────────────────────────────────┘
```

### 2.7 `Lease` («value object», projeção Redis — não persistida em PostgreSQL)

```
┌───────────────────────────────────────────────┐
│              Lease  «value object»              │
├───────────────────────────────────────────────┤
│ + Key             : string  // aios:{tenant}:lc:  │
│                      // lease:{agent_id}           │
│ + Holder           : string  // id do coordenador   │
│ + FencingToken     : long    // monotônico           │
│ + ExpiresAt        : DateTimeOffset                  │
└───────────────────────────────────────────────┘
```

### 2.8 `CorrelationContext` («value object», subcampo de `correlation` jsonb)

```
┌───────────────────────────────────────────┐
│     CorrelationContext  «value object»       │
├───────────────────────────────────────────┤
│ + TraceParent      : string                   │
│ + TenantHeader      : string  // X-AIOS-Tenant  │
│ + IdempotencyKey     : string?                  │
└───────────────────────────────────────────┘
```

---

## 3. Interfaces Públicas dos Componentes

> Assinaturas em pseudocódigo C#-like, refletindo `./Architecture.md` §4–§5.
> `Result<T>` representa o padrão de retorno (sucesso ou
> `AIOS-LIFECYCLE-<NNNN>`).

### 3.1 `ILifecycleApiSurface` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                  ILifecycleApiSurface  «interface»                    │
├───────────────────────────────────────────────────────────────────┤
│ + Handle(request: LifecycleRequestEnvelope)                            │
│     : Task<Result<LifecycleReply>>                                     │
└───────────────────────────────────────────────────────────────────┘
```

`LifecycleRequestEnvelope` («value object»): `Action` (verbo), `AgentId`,
`TenantId`, `TraceParent`, `IdempotencyKey?`, `Payload` (específico por
verbo).

### 3.2 `ILifecyclePolicyEnforcer` «interface» (PEP)

```
┌───────────────────────────────────────────────────────────────────┐
│               ILifecyclePolicyEnforcer  «interface»                   │
├───────────────────────────────────────────────────────────────────┤
│ + Authorize(ctx: DecisionContext) : Task<Result<PepDecision>>          │
└───────────────────────────────────────────────────────────────────┘
```

`DecisionContext` («value object»): `Subject` (chamador), `Action`
(`lifecycle:spawn`, `:suspend`, ...), `Resource` (agent URN), `Environment`
(tenant, hora, atributos ABAC).

### 3.3 `ILifecycleCoordinator` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                  ILifecycleCoordinator  «interface»                   │
├───────────────────────────────────────────────────────────────────┤
│ + Spawn(cmd: SpawnCommand) : Task<Result<AgentControlBlock>>            │
│ + Suspend(agentId: Ulid) : Task<Result<Unit>>                           │
│ + Resume(agentId: Ulid) : Task<Result<Unit>>                            │
│ + Hibernate(agentId: Ulid) : Task<Result<CheckpointRef>>                │
│ + Wake(agentId: Ulid) : Task<Result<AgentControlBlock>>                 │
│ + Migrate(cmd: MigrateCommand) : Task<Result<MigrationJob>>             │
│ + Checkpoint(agentId: Ulid) : Task<Result<CheckpointRef>>                │
│ + Restore(agentId: Ulid, checkpointId: Ulid)                             │
│     : Task<Result<AgentControlBlock>>                                   │
│ + Terminate(agentId: Ulid, reason: string) : Task<Result<Unit>>          │
└───────────────────────────────────────────────────────────────────┘
```

### 3.4 `IStateMachineEngine` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                    IStateMachineEngine  «interface»                    │
├───────────────────────────────────────────────────────────────────┤
│ + Evaluate(current: AgentControlBlock, trigger: LifecycleTrigger)        │
│     : Result<LifecycleTransitionPlan>   // AIOS-LIFECYCLE-0002 se inválida│
└───────────────────────────────────────────────────────────────────┘
```

`LifecycleTransitionPlan` («value object»): `FromState`, `ToState`,
`EntryActions` (lista), `ExitActions` (lista), `GuardResult`.

### 3.5 `IAcbStore` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                        IAcbStore  «interface»                          │
├───────────────────────────────────────────────────────────────────┤
│ + GetById(tenantId: string, agentId: Ulid)                              │
│     : Task<AgentControlBlock?>                                          │
│ + TryUpdate(acb: AgentControlBlock, expectedFencingToken: long)          │
│     : Task<Result<AgentControlBlock>>  // AIOS-LIFECYCLE-0013 se falhar  │
│ + AppendTransition(t: LifecycleTransition, tx: IDbTransaction)           │
│     : Task<Unit>                                                        │
│ + ListByTenantAndState(tenantId: string, state: LifecycleState)          │
│     : IAsyncEnumerable<AgentControlBlock>                               │
└───────────────────────────────────────────────────────────────────┘
```

### 3.6 `ISpawnManager` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                      ISpawnManager  «interface»                        │
├───────────────────────────────────────────────────────────────────┤
│ + Materialize(agentId: Ulid, checkpointId: Ulid?)                        │
│     : Task<Result<RuntimeInstanceRef>>  // AIOS-LIFECYCLE-0005/0006      │
└───────────────────────────────────────────────────────────────────┘
```

### 3.7 `IWarmPoolManager` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                    IWarmPoolManager  «interface»                       │
├───────────────────────────────────────────────────────────────────┤
│ + TryAcquireWarmRuntime(tenantId: string, shard: int)                    │
│     : Task<RuntimeInstanceRef?>       // null → cold-start completo      │
│ + Replenish(tenantId: string, shard: int) : Task<Unit>                   │
└───────────────────────────────────────────────────────────────────┘
```

### 3.8 `IHibernationController` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                  IHibernationController  «interface»                   │
├───────────────────────────────────────────────────────────────────┤
│ + EvaluateIdleCandidates(shard: int)                                     │
│     : IAsyncEnumerable<AgentControlBlock>                                │
│ + Hibernate(agentId: Ulid) : Task<Result<CheckpointRef>>                 │
└───────────────────────────────────────────────────────────────────┘
```

### 3.9 `ICheckpointService` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                    ICheckpointService  «interface»                     │
├───────────────────────────────────────────────────────────────────┤
│ + Create(agentId: Ulid, consistency: ConsistencyLevel)                   │
│     : Task<Result<Checkpoint>>          // AIOS-LIFECYCLE-0011           │
│ + Restore(checkpointId: Ulid)                                            │
│     : Task<Result<RestoredState>>       // AIOS-LIFECYCLE-0007/0008      │
└───────────────────────────────────────────────────────────────────┘
```

### 3.10 `ISnapshotStore` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                      ISnapshotStore  «interface»                       │
├───────────────────────────────────────────────────────────────────┤
│ + Put(checkpointId: Ulid, payload: byte[])                               │
│     : Task<Result<string>>   // retorna storage_uri; AIOS-LIFECYCLE-0014 │
│ + Get(storageUri: string) : Task<Result<byte[]>>                         │
│ + Delete(storageUri: string) : Task<Unit>   // usado por TombstoneManager│
└───────────────────────────────────────────────────────────────────┘
```

### 3.11 `IMigrationOrchestrator` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                  IMigrationOrchestrator  «interface»                   │
├───────────────────────────────────────────────────────────────────┤
│ + Start(agentId: Ulid, target: PlacementTarget)                          │
│     : Task<Result<MigrationJob>>                                        │
│ + AdvancePhase(migrationId: Ulid) : Task<Result<MigrationJob>>           │
│ + Compensate(migrationId: Ulid, reason: string)                          │
│     : Task<Result<MigrationJob>>       // AIOS-LIFECYCLE-0009/0010       │
└───────────────────────────────────────────────────────────────────┘
```

### 3.12 `ILeaseManager` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                      ILeaseManager  «interface»                        │
├───────────────────────────────────────────────────────────────────┤
│ + Acquire(agentId: Ulid, holder: string, ttlMs: int)                     │
│     : Task<Result<Lease>>              // AIOS-LIFECYCLE-0003           │
│ + Renew(lease: Lease) : Task<Result<Lease>>                              │
│ + Release(lease: Lease) : Task<Unit>                                     │
└───────────────────────────────────────────────────────────────────┘
```

### 3.13 `IReconciliationController` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                IReconciliationController  «interface»                   │
├───────────────────────────────────────────────────────────────────┤
│ + ReconcileShard(shard: int) : Task<ReconciliationReport>                │
└───────────────────────────────────────────────────────────────────┘
```

### 3.14 `ILifecycleEventPublisher` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                ILifecycleEventPublisher  «interface»                   │
├───────────────────────────────────────────────────────────────────┤
│ + EnqueueTransactional(evt: CloudEventEnvelope, tx: IDbTransaction)      │
│     : Task<Unit>              // grava em lifecycle.outbox na mesma tx  │
│ + RelayPendingBatch(maxBatch: int) : Task<int>  // publica no NATS       │
└───────────────────────────────────────────────────────────────────┘
```

### 3.15 `ITombstoneManager` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                     ITombstoneManager  «interface»                     │
├───────────────────────────────────────────────────────────────────┤
│ + PurgeExpired(tenantId: string) : Task<int>   // retorna qtd. expurgada│
│ + PurgeOnRequest(agentId: Ulid, reason: string) : Task<Result<Unit>>    │
└───────────────────────────────────────────────────────────────────┘
```

### 3.16 `ILifecycleTelemetry` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                    ILifecycleTelemetry  «interface»                    │
├───────────────────────────────────────────────────────────────────┤
│ + StartTransitionSpan(trigger: LifecycleTrigger, agentId: Ulid) : ISpan │
│ + RecordTransitionLatency(trigger: LifecycleTrigger, ms: double) : void │
│ + RecordSpawnLatency(ms: double, cold: bool) : void                     │
└───────────────────────────────────────────────────────────────────┘
```

---

## 4. Contratos (Pré-condições e Pós-condições)

| Operação | Pré-condição | Pós-condição | Erro em violação |
|----------|--------------|---------------|--------------------|
| `ILeaseManager.Acquire` | Nenhum outro `holder` detém lease ativa para `agentId`, **ou** lease anterior expirou (`ExpiresAt < now`). | `Lease` criada/renovada em Redis (`SET NX PX`) com `FencingToken` monotônico incrementado. | `AIOS-LIFECYCLE-0003` (lease não adquirida). |
| `IAcbStore.TryUpdate` | `expectedFencingToken == acb.FencingToken` no momento da leitura anterior. | ACB persistido com `Generation`/`FencingToken` atualizados; `LifecycleTransition` e `LifecycleOutbox` gravados na mesma transação (INV2). | `AIOS-LIFECYCLE-0013` (fencing/`generation` obsoletos). |
| `ILifecycleCoordinator.Spawn` | Capability `lifecycle:spawn` concedida; lease adquirida para `agentId`. | ACB → `Ready`/`Running`; `SpawnManager.Materialize` concluído; evento `lifecycle.running` emitido. | `AIOS-LIFECYCLE-0005` (admissão negada); `AIOS-LIFECYCLE-0006` (timeout). |
| `ICheckpointService.Create` | ACB em `Running` ou `Suspended`; capability `lifecycle:checkpoint` concedida. | `Checkpoint` persistido com `ContentHash` válido; `SnapshotStore.Put` concluído; ACB.`LastCheckpointId` atualizado. | `AIOS-LIFECYCLE-0011` (falha de serialização); `AIOS-LIFECYCLE-0014` (storage indisponível). |
| `ICheckpointService.Restore` | `checkpointId` existe e pertence ao `tenant_id`/`agent_id` do chamador. | `RestoredState` (ACB + `working_memory_ref`) reconstruído; `ContentHash` verificado. | `AIOS-LIFECYCLE-0007` (hash divergente); `AIOS-LIFECYCLE-0008` (checkpoint inexistente). |
| `IHibernationController.Hibernate` | ACB em `Suspended`; idle ≥ `hibernation.idle_ttl_s` **ou** pressão de RAM ≥ `ram_pressure_threshold_pct`. | Checkpoint `Consistency=Consistent` gravado; ACB → `Hibernated`; `RuntimeInstanceId → null`; RAM liberada. | `AIOS-LIFECYCLE-0002` (guarda G6 falhou); `AIOS-LIFECYCLE-0011`/`0014` (falha de checkpoint). |
| `ISpawnManager.Materialize` (a partir de `Hibernated`) | `Checkpoint` associado íntegro (`VerifyIntegrity` = true); Scheduler concede slot. | Novo `RuntimeInstanceId`; `Generation + 1`; ACB → `Ready`/`Running`; evento `lifecycle.woken`. | `AIOS-LIFECYCLE-0007` (checkpoint corrompido); `AIOS-LIFECYCLE-0006` (timeout de materialização). |
| `IMigrationOrchestrator.Start` | Lease adquirida; decisão de placement válida de `009`/`027`; destino saudável. | `MigrationJob` criado em `Phase=Quiesce`, `Status=Running`; ACB → `Migrating`. | `AIOS-LIFECYCLE-0009` (migração já em andamento); `AIOS-LIFECYCLE-0010` (destino indisponível). |
| `IMigrationOrchestrator.Compensate` | `MigrationJob.Status == Running` e `saga_timeout_ms` excedido, ou falha irrecuperável em uma fase. | `MigrationJob.Status → Compensated`; origem reativada; ACB retorna ao estado anterior à migração. | Idempotente — chamada repetida é *no-op* seguro após compensação concluída. |
| `ILifecycleEventPublisher.EnqueueTransactional` | Chamado dentro de uma transação de banco de dados ativa (`tx`). | Linha inserida em `lifecycle.outbox` com `Published=false`, no mesmo commit do estado de domínio. | Falha de transação reverte ambos (estado + evento) atomicamente. |
| `ITombstoneManager.PurgeExpired` | ACB em estado terminal (`Terminated`/`Failed`) há mais de `retention.terminated_ttl_days`. | Snapshots/checkpoints expurgados de MinIO/PostgreSQL; evento auditável emitido (`025`); `agent_id` marcado como não-reutilizável. | Nenhum (operação de manutenção; falhas parciais são retentadas). |

---

## 5. Invariantes

> Reproduzidas literalmente de `./_DESIGN_BRIEF.md` §3.1/§4.3, aplicadas ao
> nível de estrutura de dados/classe:

- **I1** — `(TenantId, AgentId)` **DEVE** ser único em `AgentControlBlock`.
- **I2** — `State` terminal (`Terminated`/`Failed`) **DEVE** implicar
  `DesiredState` também terminal **e** `RuntimeInstanceId = null`.
- **I3** — `State = Hibernated` **DEVE** implicar `LastCheckpointId ≠ null`
  apontando para um `Checkpoint` com `Consistency = Consistent`.
- **I4** (INV1 do brief) — Apenas uma transição por agente pode estar em
  curso por vez; garantido por `ILeaseManager` — nenhuma escrita em
  `IAcbStore.TryUpdate` **DEVE** ocorrer sem uma `Lease` válida detida pelo
  chamador.
- **I5** (INV2 do brief) — Toda transição **DEVE** produzir exatamente um
  `LifecycleTransition` **e** um `LifecycleOutbox`, na mesma transação de
  banco de dados que altera a projeção do ACB.
- **I6** (INV3 do brief) — Estados terminais são absorventes: nenhuma
  transição de saída **PODE** existir a partir de `Terminated` ou `Failed`
  em `IStateMachineEngine.Evaluate`.
- **I7** (INV4 do brief) — `Hibernated` e `Migrating` **DEVEM** possuir
  checkpoint `Consistent` válido antes de liberar RAM/invalidar a origem —
  verificado por `ICheckpointService.VerifyIntegrity` antes de qualquer
  `AdvancePhase` que libere recurso.
- **I8** (derivada, nível de `MigrationJob`) — Toda `MigrationJob` **DEVE**
  transicionar para exatamente um estado terminal (`Succeeded`, `Failed` ou
  `Compensated`); nenhuma migração permanece `Running` além de
  `migration.saga_timeout_ms` sem intervenção do `ReconciliationController`.
- **I9** (derivada, nível de `LifecycleTransition`) — `Seq` **DEVE** ser
  monotonicamente crescente por `AgentId`, sem lacunas dentro de uma mesma
  `Generation` — condição necessária para replay determinístico do histórico.

---

## 6. Diagrama de Relações

```
        ┌────────────────┐  1        1  ┌────────────────┐
        │ AgentControlBlock│◆────────────│  HibernationRecord│ (0..1, se cold)
        │  «entity»          │              └────────────────┘
        └───┬─────┬─────┬────┘
            │ 1..* │ 0..1  │ 0..*
            │      │       │
            ▼      ▼       ▼
   ┌──────────────┐ ┌──────────┐ ┌────────────────┐
   │LifecycleTrans.│ │Checkpoint│ │  MigrationJob    │
   │  «entity»      │ │ «entity» │ │   «entity»       │
   └──────────────┘ └────┬─────┘ └───────┬────────┘
                          │ 1              │ 0..1
                          │                │
                          ▼                ▼
                   ┌──────────────┐  usa Checkpoint (checkpoint_id)
                   │ SnapshotStore  │◀───────────────────────────────┘
                   │ (MinIO + PG)   │
                   └──────────────┘

   AgentControlBlock ──▶ LifecycleOutbox   (via LifecycleCoordinator, mesma tx)
   LifecycleTransition ──▷ usa CorrelationContext  (composição, embutido)
   Lease («value object», Redis) ──▶ AgentControlBlock  (referencia por agent_id, não é FK relacional)

   ┌──────────────┐ uses  ┌──────────────────────┐ uses  ┌───────────────┐
   │LifecycleApi   │──────▶│LifecyclePolicyEnforcer│──────▶│  022-Policy    │
   │Surface        │       └──────────┬────────────┘       │  (PDP externo) │
   └───────┬───────┘                  │ allow                └───────────────┘
           │ dispatch                  ▼
           ▼                   ┌────────────────────┐
   ┌──────────────┐   uses     │ LifecycleCoordinator │
   │LeaseManager    │◀─────────│                      │
   └──────────────┘           └──┬───┬───┬───┬───┬───┘
                                  │   │   │   │   │
                    ┌─────────────┘   │   │   │   └─────────────────┐
                    ▼                  ▼   ▼   ▼                     ▼
           ┌────────────────┐ ┌──────────┐ ┌─────────────┐ ┌────────────────────┐
           │StateMachineEngine│ │SpawnMgr  │ │HibernationCtl│ │MigrationOrchestrator│
           └────────────────┘ └────┬─────┘ └──────┬───────┘ └──────────┬─────────┘
                                    │ uses          │ uses                │ uses
                                    ▼               ▼                     ▼
                             ┌────────────┐  ┌────────────────┐  ┌──────────────┐
                             │WarmPoolMgr │  │CheckpointService│◀─│CheckpointService│
                             └────────────┘  └────────┬───────┘  └──────────────┘
                                                       │ uses
                                                       ▼
                                              ┌────────────────┐
                                              │ SnapshotStore    │
                                              └────────────────┘

   ┌────────────────────────┐ uses ┌──────────────────────┐
   │ ReconciliationController │──────▶│ AcbStore / LifecycleCoordinator│
   └────────────────────────┘      └──────────────────────┘

   ┌────────────────┐ uses ┌────────────────┐
   │ TombstoneManager │──────▶│ SnapshotStore  │
   └────────────────┘      └────────────────┘
           │ uses
           ▼
   ┌────────────────┐
   │  AcbStore        │
   └────────────────┘
```

---

## 7. Enumerações e Value Objects

### 7.1 `LifecycleState` (enum)

```
enum LifecycleState {
  Created, Ready, Running, Suspended, Hibernated,
  Migrating, Terminated, Failed
}
```

> Definição completa de estados/transições/guardas: ver `./StateMachine.md`.
> Estados terminais: `Terminated`, `Failed`.

### 7.2 `LifecycleTrigger` (enum)

```
enum LifecycleTrigger {
  Spawn, Ready, Start, Suspend, Resume, Hibernate, Wake,
  Migrate, Cutover, Terminate, Fail, Reconcile
}
```

### 7.3 `PriorityClass` (enum)

```
enum PriorityClass { System, Interactive, Batch, BestEffort }
```

### 7.4 `CheckpointCodec` (enum)

```
enum CheckpointCodec { JsonZstd, MsgpackZstd }
```

### 7.5 `EncryptionAlgorithm` (enum)

```
enum EncryptionAlgorithm { Aes256Gcm }
```

### 7.6 `ConsistencyLevel` (enum)

```
enum ConsistencyLevel { Consistent, Crash }
```

### 7.7 `StorageTier` (enum)

```
enum StorageTier { Hot, Warm, Cold }
```

### 7.8 `MigrationPhase` (enum)

```
enum MigrationPhase {
  Quiesce, Checkpoint, Transfer, Materialize, Cutover, Cleanup
}
```

### 7.9 `MigrationStatus` (enum)

```
enum MigrationStatus { Pending, Running, Succeeded, Failed, Compensated }
```

### 7.10 `PepDecision` (enum)

```
enum PepDecision { Allow, Deny }
```

---

## 8. Referências

- Modelo de dados canônico: `./_DESIGN_BRIEF.md` §3
- Máquina de estados: `./StateMachine.md`
- Fluxos de execução: `./SequenceDiagrams.md`
- Arquitetura de componentes: `./Architecture.md`
- Contratos centrais (URN, envelope, erro, idempotência): `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`

*Fim do documento `ClassDiagrams.md` do módulo 008-Agent-Lifecycle.*
