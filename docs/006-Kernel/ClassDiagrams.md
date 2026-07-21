---
Documento: ClassDiagrams
Módulo: 006-Kernel
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 006-Kernel
ADRs relacionados: ADR-0061, ADR-0062, ADR-0063, ADR-0066, ADR-0068 (deste módulo, a propor)
RFCs relacionados: RFC-0001 (baseline); RFC-0006 (Cognitive Syscall ABI, a propor); RFC-0007 (ACB & Quota Model, a propor)
Depende de: 006-Kernel/Architecture.md, 006-Kernel/StateMachine.md, 003-RFC/RFC-0001-Architecture-Baseline.md
---

# 006-Kernel — Diagramas de Classe / Estrutura

> Este documento detalha as estruturas de dados e interfaces internas do Kernel
> em notação ASCII (adaptação de UML). Os campos e tipos reproduzem
> exatamente o modelo de dados de `./_DESIGN_BRIEF.md` §3 — nenhum campo é
> inventado ou renomeado. As interfaces de componente refletem
> `./Architecture.md` §4–§5.

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

### 2.1 `AgentControlBlock` («entity», tabela `kernel.acb`)

```
┌───────────────────────────────────────────────────────────────┐
│                 AgentControlBlock  «entity»                    │
├───────────────────────────────────────────────────────────────┤
│ + Urn            : string   // urn:aios:<tenant>:agent:<ULID>  │
│ + TenantId        : string   // RLS boundary                   │
│ + AgentId         : Ulid                                       │
│ + State           : AcbState                                   │
│ + Priority        : byte     // 0..9, default 5 (0 = maior)     │
│ + ParentUrn       : string?  // árvore de spawn                │
│ + QuotaRef        : Guid     // FK → Quota.Id                   │
│ + MemoryPtr       : MemoryPointerSet                            │
│ + ContextPtr      : ContextPointer                              │
│ + PolicyRef       : string   // URN/versão do policy binding    │
│ + CapabilitySet   : IReadOnlyList<string>                       │
│ + RuntimeRef      : string?  // handle do Agent Runtime, NULL=cold│
│ + CheckpointRef   : string?                                      │
│ + ShardKey        : int      // hash(tenant_id, agent_id) mod N │
│ + Version         : long     // OCC                             │
│ + CreatedAt       : DateTimeOffset                               │
│ + UpdatedAt       : DateTimeOffset                               │
│ + LastActiveAt    : DateTimeOffset?                              │
│ + TerminatedAt    : DateTimeOffset?                              │
│ + Labels          : IReadOnlyDictionary<string, JsonElement>    │
├───────────────────────────────────────────────────────────────┤
│ + CanTransitionTo(target: AcbState) : bool                      │
│ + WithNextVersion() : AgentControlBlock   // clone imutável +1  │
└───────────────────────────────────────────────────────────────┘
```

Invariantes locais: ver §5 (I1–I4). `AgentControlBlock` é tratado como
**imutável** na camada de aplicação — toda mutação produz uma nova instância
com `Version` incrementado, persistida via OCC (`AcbStore`).

### 2.2 `MemoryPointerSet` («value object», subcampo de `memory_ptr` jsonb)

```
┌───────────────────────────────────────────────┐
│         MemoryPointerSet  «value object»        │
├───────────────────────────────────────────────┤
│ + Working     : string?   // ref → 010, working  │
│ + Short        : string?   // ref → 010, short-term│
│ + Long         : string?   // ref → 010, long-term │
│ + Semantic     : string?   // ref → 010, semantic  │
│ + Episodic     : string?   // ref → 010, episodic  │
│ + Procedural   : string?   // ref → 010, procedural│
└───────────────────────────────────────────────┘
```

### 2.3 `ContextPointer` («value object», subcampo de `context_ptr` jsonb)

```
┌───────────────────────────────────────────┐
│        ContextPointer  «value object»       │
├───────────────────────────────────────────┤
│ + SessionRef   : string   // ref → 011      │
│ + WindowTokens : int      // orçamento atual│
└───────────────────────────────────────────┘
```

### 2.4 `Quota` («entity», tabela `kernel.quota`)

```
┌───────────────────────────────────────────────────────────────┐
│                        Quota  «entity»                          │
├───────────────────────────────────────────────────────────────┤
│ + Id               : Guid                                       │
│ + TenantId          : string                                     │
│ + Scope             : QuotaScope    // {Tenant, Agent}            │
│ + SubjectUrn        : string        // URN do tenant ou agente    │
│ + TokensLimit       : long                                       │
│ + CpuMsLimit        : long                                       │
│ + MemMiBLimit       : int                                        │
│ + CostUsdLimit      : decimal       // numeric(12,4)              │
│ + ToolsLimit        : int                                        │
│ + SyscallsPerSec    : int                                        │
│ + Window            : TimeSpan      // default 1h                │
│ + Enforcement       : EnforcementMode // {Hard, Soft}              │
│ + Version           : long          // OCC                        │
├───────────────────────────────────────────────────────────────┤
│ + IsWithinLimit(usage: QuotaUsage) : bool                        │
└───────────────────────────────────────────────────────────────┘
```

> Nota: os **contadores de consumo** (voláteis, alta frequência) não são um
> campo desta entidade — vivem em Redis como token-buckets por `SubjectUrn`
> (`QuotaUsage`, §2.5), conforme brief §3.2.

### 2.5 `QuotaUsage` («value object», projeção Redis, não persistida em PostgreSQL)

```
┌───────────────────────────────────────────────┐
│           QuotaUsage  «value object»            │
├───────────────────────────────────────────────┤
│ + SubjectUrn        : string                    │
│ + TokensConsumed    : long                       │
│ + CpuMsConsumed     : long                       │
│ + MemMiBInUse       : int                        │
│ + CostUsdConsumed   : decimal                    │
│ + ToolsInvoked      : int                        │
│ + SyscallsInWindow  : int                        │
│ + WindowResetAt     : DateTimeOffset              │
└───────────────────────────────────────────────┘
```

### 2.6 `SyscallRecord` («entity», tabela `kernel.syscall_log`)

```
┌───────────────────────────────────────────────────────────────┐
│                    SyscallRecord  «entity»                      │
├───────────────────────────────────────────────────────────────┤
│ + Id               : Ulid                                        │
│ + TenantId          : string                                      │
│ + AgentUrn          : string                                      │
│ + Verb              : SyscallVerb                                 │
│ + IdempotencyKey    : string?                                     │
│ + Decision          : PepDecision    // {Allow, Deny}               │
│ + DenyCode          : string?        // AIOS-CAP-*/AIOS-QUOTA-*     │
│ + Status            : ExecutionStatus // {Ok, Error}                │
│ + LatencyMs         : int                                          │
│ + TraceId           : string                                       │
│ + CreatedAt         : DateTimeOffset                                │
└───────────────────────────────────────────────────────────────┘
```

### 2.7 `OutboxMessage` («entity», tabela `kernel.outbox`)

```
┌───────────────────────────────────────────────────────────────┐
│                    OutboxMessage  «entity»                      │
├───────────────────────────────────────────────────────────────┤
│ + Id               : Ulid        // = event.id do CloudEvents     │
│ + TenantId          : string                                      │
│ + Subject           : string     // aios.<tenant>.agent.…           │
│ + Payload           : JsonDocument // envelope CloudEvents completo │
│ + Published         : bool       // false até o relay confirmar    │
│ + CreatedAt          : DateTimeOffset                               │
├───────────────────────────────────────────────────────────────┤
│ + MarkPublished() : OutboxMessage                                 │
└───────────────────────────────────────────────────────────────┘
```

---

## 3. Interfaces Públicas dos Componentes

> Assinaturas em pseudocódigo C#-like, refletindo `./Architecture.md` §4–§5.
> `Result<T>` representa o padrão de retorno (sucesso ou `AIOS-<DOMINIO>-<NNNN>`).

### 3.1 `ISyscallGateway` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                     ISyscallGateway  «interface»                    │
├───────────────────────────────────────────────────────────────────┤
│ + Dispatch(request: SyscallEnvelope) : Task<Result<SyscallReply>>   │
└───────────────────────────────────────────────────────────────────┘
```

`SyscallEnvelope` («value object»): `Verb`, `AgentUrn`, `TenantId`,
`TraceParent`, `IdempotencyKey?`, `Payload` (específico por verbo).

### 3.2 `ICapabilityEnforcer` «interface» (PEP)

```
┌───────────────────────────────────────────────────────────────────┐
│                  ICapabilityEnforcer  «interface»                   │
├───────────────────────────────────────────────────────────────────┤
│ + Authorize(ctx: DecisionContext) : Task<Result<PepDecision>>       │
└───────────────────────────────────────────────────────────────────┘
```

`DecisionContext` («value object»): `Subject` (agent URN), `Action` (verbo),
`Resource` (URN alvo), `Environment` (tenant, hora, atributos ABAC).

### 3.3 `IPolicyClient` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                      IPolicyClient  «interface»                     │
├───────────────────────────────────────────────────────────────────┤
│ + Evaluate(ctx: DecisionContext) : Task<Result<PdpVerdict>>         │
│ + InvalidateCache(policyBundleVersion: string) : void               │
└───────────────────────────────────────────────────────────────────┘
```

### 3.4 `IAcbStore` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                        IAcbStore  «interface»                       │
├───────────────────────────────────────────────────────────────────┤
│ + GetByUrn(urn: string) : Task<AgentControlBlock?>                  │
│ + TryUpdate(acb: AgentControlBlock, expectedVersion: long)          │
│     : Task<Result<AgentControlBlock>>   // AIOS-KERNEL-0009 se falhar│
│ + Insert(acb: AgentControlBlock) : Task<Result<AgentControlBlock>>  │
│ + ListByTenantAndState(tenantId: string, state: AcbState)           │
│     : IAsyncEnumerable<AgentControlBlock>                           │
└───────────────────────────────────────────────────────────────────┘
```

### 3.5 `IResourceQuotaManager` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                 IResourceQuotaManager  «interface»                  │
├───────────────────────────────────────────────────────────────────┤
│ + Reserve(subjectUrn: string, request: QuotaRequest)                │
│     : Task<Result<QuotaReservation>>    // AIOS-QUOTA-0001..0003    │
│ + Commit(reservation: QuotaReservation) : Task<Result<Unit>>        │
│ + Release(reservation: QuotaReservation) : Task<Result<Unit>>       │
│ + GetUsage(subjectUrn: string) : Task<QuotaUsage>                   │
└───────────────────────────────────────────────────────────────────┘
```

`QuotaRequest`/`QuotaReservation` («value object»): dimensão (tokens, cpu-ms,
mem, custo, tools, syscalls), quantidade solicitada, `reservationId` (ULID).

### 3.6 `ILifecycleCoordinator` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                ILifecycleCoordinator  «interface»                   │
├───────────────────────────────────────────────────────────────────┤
│ + Spawn(cmd: SpawnCommand) : Task<Result<AgentControlBlock>>        │
│ + Kill(urn: string, reason: string) : Task<Result<Unit>>            │
│ + Suspend(urn: string) : Task<Result<Unit>>                         │
│ + Resume(urn: string) : Task<Result<Unit>>                           │
│ + Checkpoint(urn: string) : Task<Result<CheckpointRef>>              │
└───────────────────────────────────────────────────────────────────┘
```

### 3.7 `ISchedulerClient` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                    ISchedulerClient  «interface»                    │
├───────────────────────────────────────────────────────────────────┤
│ + RequestAdmission(req: AdmissionRequest)                            │
│     : Task<Result<AdmissionDecision>>   // AIOS-KERNEL-0003 se down  │
└───────────────────────────────────────────────────────────────────┘
```

### 3.8 `ILifecycleClient` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                    ILifecycleClient  «interface»                    │
├───────────────────────────────────────────────────────────────────┤
│ + Materialize(urn: string, slot: SlotRef) : Task<Result<RuntimeRef>>│
│ + Hibernate(urn: string) : Task<Result<CheckpointRef>>               │
│ + Restore(urn: string, checkpointRef: string)                        │
│     : Task<Result<RuntimeRef>>                                       │
└───────────────────────────────────────────────────────────────────┘
```

### 3.9 `IResourceBrokerRouter` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                 IResourceBrokerRouter  «interface»                   │
├───────────────────────────────────────────────────────────────────┤
│ + Remember(urn: string, item: MemoryWriteRequest)                    │
│     : Task<Result<MemoryWriteReply>>          // → 010                │
│ + Recall(urn: string, query: MemoryQueryRequest)                      │
│     : Task<Result<MemoryQueryReply>>          // → 010                │
│ + Plan(urn: string, goal: PlanRequest) : Task<Result<PlanReply>>      // → 012 │
│ + InvokeTool(urn: string, call: ToolInvocation)                       │
│     : Task<Result<ToolResult>>                // → 015                │
│ + RouteModel(urn: string, req: InferenceRequest)                      │
│     : Task<Result<InferenceRoute>>            // → 017                │
└───────────────────────────────────────────────────────────────────┘
```

### 3.10 `IIdempotencyStore` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                   IIdempotencyStore  «interface»                    │
├───────────────────────────────────────────────────────────────────┤
│ + TryGetCachedResult(tenantId: string, key: string)                  │
│     : Task<CachedResult?>                                            │
│ + SaveResult(tenantId: string, key: string, payloadHash: string,      │
│     result: SyscallReply) : Task<Unit>       // AIOS-SYSCALL-0002    │
│     // se payloadHash divergente para a mesma key                    │
└───────────────────────────────────────────────────────────────────┘
```

### 3.11 `IEventEmitter` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                       IEventEmitter  «interface»                     │
├───────────────────────────────────────────────────────────────────┤
│ + EnqueueTransactional(evt: CloudEventEnvelope, tx: IDbTransaction)  │
│     : Task<Unit>            // grava em kernel.outbox na mesma tx    │
│ + RelayPendingBatch(maxBatch: int) : Task<int>  // publica no NATS   │
└───────────────────────────────────────────────────────────────────┘
```

### 3.12 `ICheckpointManager` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                  ICheckpointManager  «interface»                    │
├───────────────────────────────────────────────────────────────────┤
│ + CreateCheckpoint(urn: string) : Task<Result<CheckpointRef>>        │
│ + ValidateCheckpoint(checkpointRef: string) : Task<bool>              │
└───────────────────────────────────────────────────────────────────┘
```

### 3.13 `IKernelTelemetry` «interface»

```
┌───────────────────────────────────────────────────────────────────┐
│                    IKernelTelemetry  «interface»                    │
├───────────────────────────────────────────────────────────────────┤
│ + StartSyscallSpan(verb: SyscallVerb, agentUrn: string) : ISpan      │
│ + RecordSyscallLatency(verb: SyscallVerb, ms: double) : void         │
│ + EmitAuditRecord(record: AuditEntry) : Task<Unit>   // → 025        │
└───────────────────────────────────────────────────────────────────┘
```

---

## 4. Contratos (Pré-condições e Pós-condições)

| Operação | Pré-condição | Pós-condição | Erro em violação |
|----------|--------------|---------------|--------------------|
| `IAcbStore.TryUpdate` | `expectedVersion == acb.Version` no momento da leitura anterior. | ACB persistido com `Version = expectedVersion + 1`; evento de mudança enfileirado no Outbox na mesma transação. | `AIOS-KERNEL-0009` (conflito OCC). |
| `ILifecycleCoordinator.Spawn` | Capability `agent:spawn` concedida (`ICapabilityEnforcer.Authorize` = Allow) **e** cota `agent_count` do tenant disponível. | ACB criado em `Pending`; `SchedulerClient.RequestAdmission` disparado; evento `agent.lifecycle.spawned` no Outbox. | `AIOS-CAP-0001` (capability); `AIOS-QUOTA-0001` (cota). |
| `ILifecycleCoordinator.Kill` | Capability `agent:kill` concedida **e** (`caller == owner` **ou** `role == admin`). | ACB → `Terminating`; drenagem agendada; ao concluir, → `Terminated` com `terminated_at` preenchido e cotas de agente liberadas. | `AIOS-CAP-0001`; `AIOS-KERNEL-0002` se estado atual não permite `kill`. |
| `IResourceQuotaManager.Reserve` | `subjectUrn` possui `Quota` ativa; `request` bem formado (dimensão válida, quantidade > 0). | Token-bucket decrementado atomicamente (script Lua); `QuotaReservation` retornada com `reservationId` único. | `AIOS-QUOTA-0001`/`0002`/`0003` conforme dimensão excedida. |
| `IResourceQuotaManager.Commit` | `reservation` existe e não expirou (TTL de reserva). | Consumo confirmado; contador não é revertido. | `AIOS-KERNEL-0010` se `reservation` inválida/expirada. |
| `IResourceQuotaManager.Release` | `reservation` existe (mesmo expirada). | Quantidade reservada devolvida ao saldo disponível. | Idempotente — chamada repetida é *no-op* seguro. |
| `IPolicyClient.Evaluate` | `DecisionContext` completo (sujeito, ação, recurso, ambiente). | `PdpVerdict` retornado e, se aplicável, cacheado por `kernel.pep.decision_cache_ttl_ms`. | `AIOS-CAP-0003` se PDP indisponível e `fail_mode=closed`. |
| `IEventEmitter.EnqueueTransactional` | Chamado dentro de uma transação de banco de dados ativa (`tx`). | Linha inserida em `kernel.outbox` com `published=false`, no mesmo commit do estado de domínio. | Falha de transação reverte ambos (estado + evento) atomicamente. |
| `IIdempotencyStore.SaveResult` | `key` não persistida anteriormente **ou** `payloadHash` idêntico ao registrado. | Resultado persistido/retornado por ≥ 24h (`kernel.idempotency.retention_h` ≥ 24). | `AIOS-SYSCALL-0002` se `payloadHash` divergente para a mesma `key`. |
| `ICheckpointManager.CreateCheckpoint` | ACB em `Running` ou `Suspended`; capability `agent:checkpoint` concedida. | Snapshot durável concluído via `LifecycleClient`; `checkpoint_ref` atualizado no ACB; evento `agent.checkpoint.created`. | `AIOS-KERNEL-0005` se falha no broker de Lifecycle/MinIO. |

---

## 5. Invariantes

> Reproduzidas literalmente de `./_DESIGN_BRIEF.md` §4.2, aplicadas ao nível de
> estrutura de dados/classe:

- **I1** — Um `AgentControlBlock` **DEVE** ter `RuntimeRef` não-nulo **se e
  somente se** `State ∈ {Running, Suspended}`. Em qualquer outro estado,
  `RuntimeRef` **DEVE** ser `null`.
- **I2** — Todo `AgentControlBlock` em estado terminal (`Terminated`, `Failed`)
  **DEVE** ter `TerminatedAt` preenchido e a `Quota` de escopo `agent`
  liberada (contador zerado/removido).
- **I3** — Toda escrita em `AcbStore` **DEVE** usar `TryUpdate` com
  `expectedVersion`; escrita direta sem checagem de versão é uma violação de
  contrato (viola OCC) e **NÃO DEVE** existir no código do Kernel.
- **I4** — Toda transição de estado do ACB **DEVE** produzir exatamente um
  `OutboxMessage` correspondente, na mesma transação de banco de dados que
  altera o ACB — nunca zero, nunca mais de um.
- **I5** (derivada, nível de `Quota`) — `QuotaUsage.TokensConsumed ≤
  Quota.TokensLimit` **DEVE** ser verdadeiro ao final de qualquer janela
  reconciliada quando `Enforcement = Hard`; overshoot é tolerado apenas de
  forma transitória sob concorrência extrema, limitado a ≤ 1% (NFR-009).
- **I6** (derivada, nível de `SyscallRecord`) — Toda syscall privilegiada
  **DEVE** produzir exatamente um `SyscallRecord`, com `Decision` preenchida
  **antes** de `Status`, refletindo que o PEP sempre precede a execução.

---

## 6. Diagrama de Relações

```
        ┌────────────────┐  1        1  ┌────────────────┐
        │     Tenant      │◆────────────│     Quota       │  (scope=tenant)
        └───────┬─────────┘              └─────────────────┘
                │ 1
                │
                │ *
        ┌───────▼─────────────────┐   1        1   ┌────────────────┐
        │  AgentControlBlock       │◆──────────────▶│     Quota       │ (scope=agent)
        │  «entity»                 │                └────────────────┘
        └──┬───────┬───────┬────────┘
           │        │        │
           │ *      │ *      │ 0..1 (parent_urn)
           ▼        ▼        ▼
   ┌───────────┐ ┌──────────────┐ ┌───────────────────────┐
   │SyscallRecord│ │OutboxMessage │ │ AgentControlBlock (self)│
   │  «entity»   │ │  «entity»    │ │  árvore de spawn        │
   └───────────┘ └──────────────┘ └───────────────────────┘

   AgentControlBlock ──▶ MemoryPointerSet   (composição, embutido em memory_ptr)
   AgentControlBlock ──▶ ContextPointer     (composição, embutido em context_ptr)
   Quota              ──▷ usa QuotaUsage    (dependência, projeção em Redis)

   ┌──────────────┐ uses  ┌────────────────────┐ uses  ┌───────────────┐
   │ SyscallGateway│──────▶│ CapabilityEnforcer  │──────▶│ PolicyClient  │
   └───────┬───────┘       └──────────┬──────────┘       └───────────────┘
           │ uses                     │ uses
           ▼                          ▼
   ┌──────────────┐          ┌────────────────┐
   │IdempotencyStore│          │   AcbStore      │
   └──────────────┘          └────────────────┘

   ┌──────────────────────┐ uses ┌────────────────┐ uses ┌────────────────┐
   │ LifecycleCoordinator   │──────▶│ SchedulerClient │      │ LifecycleClient │
   └───────────┬────────────┘      └────────────────┘      └────────────────┘
               │ uses                                              ▲
               └──────────────────────────────────────────────────┘
               │ uses
               ▼
       ┌────────────────┐         ┌───────────────────┐
       │  EventEmitter    │◀────────│ CheckpointManager   │
       └────────────────┘         └───────────────────┘
               ▲
               │ uses (todos os componentes acima publicam via EventEmitter)
       ┌───────┴────────┐
       │ResourceQuotaMgr │
       └────────────────┘
               ▲
               │ uses
       ┌───────┴──────────────┐
       │ ResourceBrokerRouter    │
       └───────────────────────┘
```

---

## 7. Enumerações e Value Objects

### 7.1 `AcbState` (enum)

```
enum AcbState {
  Pending, Admitted, Running, Suspended, Hibernated,
  Terminating, Terminated, Failed
}
```

> Definição completa de estados/transições/guardas: ver `./StateMachine.md`.
> Estados terminais: `Terminated`, `Failed`.

### 7.2 `SyscallVerb` (enum)

```
enum SyscallVerb {
  Spawn, Kill, Suspend, Resume, Remember, Recall,
  Plan, InvokeTool, RouteModel, GetQuota, Checkpoint
}
```

### 7.3 `PepDecision` (enum)

```
enum PepDecision { Allow, Deny }
```

### 7.4 `QuotaScope` (enum)

```
enum QuotaScope { Tenant, Agent }
```

### 7.5 `EnforcementMode` (enum)

```
enum EnforcementMode { Hard, Soft }
```

### 7.6 `ExecutionStatus` (enum)

```
enum ExecutionStatus { Ok, Error }
```

### 7.7 `QuotaDimension` (enum, value object auxiliar de `QuotaRequest`)

```
enum QuotaDimension {
  Tokens, CpuMs, MemMiB, CostUsd, Tools, SyscallsPerSec
}
```

---

## 8. Referências

- Modelo de dados canônico: `./_DESIGN_BRIEF.md` §3
- Máquina de estados: `./StateMachine.md`
- Fluxos de execução: `./SequenceDiagrams.md`
- Arquitetura de componentes: `./Architecture.md`
- Contratos centrais (URN, envelope, erro, idempotência): `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`

*Fim do documento `ClassDiagrams.md` do módulo 006-Kernel.*
