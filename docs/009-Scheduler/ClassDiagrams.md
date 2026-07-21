---
Documento: ClassDiagrams
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0096, ADR-0097 (a propor); ADR-0001..ADR-0010 (herdadas)
RFCs relacionados: RFC-0001 (baseline); RFC-0090, RFC-0091 (a propor)
Depende de: 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — ClassDiagrams

> Estruturas, interfaces, contratos e invariantes dos componentes internos do
> Scheduler descritos em `Architecture.md` e em `_DESIGN_BRIEF.md` §2–§3. Os
> diagramas usam notação UML adaptada para ASCII: `◆──` composição,
> `◇──` agregação, `- - ▷` dependência/uso, `──▷` implementação de interface,
> `──▶` associação dirigida.

## Índice

1. Convenções de notação
2. Diagrama de classes — Pipeline de decisão
3. Interface `ISchedulingPolicy` e implementações
4. Diagrama de classes — Placement e Preempção
5. Diagrama de classes — Estado quente (Queue/Quota/Capacity)
6. Diagrama de classes — Clientes de dependências externas
7. Diagrama de classes — Persistência e proveniência
8. Entidades de dados canônicas (contratos de mensagem)
9. Invariantes globais
10. Relações entre componentes (visão consolidada)

---

## 1. Convenções de Notação

```
┌───────────────────────┐
│      ClassName         │   « estereótipo »  (ex.: «service», «interface», «entity», «value object»)
├───────────────────────┤
│ - campoPrivado: Tipo   │
│ + campoPublico: Tipo   │
├───────────────────────┤
│ + Metodo(args): Retorno│
└───────────────────────┘

  A ──▷ B     : A depende de/usa B
  A ──▶ B     : A tem associação dirigida com B (referência)
  A ◆── B     : A é composto por B (ciclo de vida acoplado)
  A ◇── B     : A agrega B (ciclo de vida independente)
  A ┈┈▷ I     : A implementa a interface I
```

Visibilidade: `+` público, `-` privado, `#` protegido. Tipos seguem convenção
.NET (`string`, `int`, `decimal`, `DateTimeOffset`, `Guid`/`Ulid`,
`IReadOnlyList<T>`, `Task<T>` para métodos assíncronos).

---

## 2. Diagrama de Classes — Pipeline de Decisão

```
┌────────────────────────────────────┐
│         SchedulingGateway           │ «service · PEP»
├────────────────────────────────────┤
│ - _idempotency: IIdempotencyStore   │
│ - _admission: IAdmissionController  │
├────────────────────────────────────┤
│ + Submit(req: SchedulingRequest,    │
│     idemKey: string): Task<SchedulingDecision> │
│ + Cancel(req: CancelRequest): Task<Ack>         │
│ + Preempt(req: PreemptRequest): Task<PreemptResult> │
│ + GetDecision(requestId: Ulid): Task<SchedulingDecision> │
│ + ReportRuntimeSignal(sig: RuntimeSignal): Task<Ack>     │
└────────────────┬────────────────────────────────────────┘
                  │ ┈┈▷ valida envelope (traceparent, X-AIOS-Tenant)
                  ▼
┌────────────────────────────────────┐
│        AdmissionController          │ «service»
├────────────────────────────────────┤
│ - _quota: IQuotaLedger              │
│ - _budget: IBudgetClient            │
│ - _capacity: ICapacityProvider       │
│ - _policy: IPolicyClient            │
│ - _backpressure: IBackpressureController │
├────────────────────────────────────┤
│ + Admit(req: SchedulingRequest):    │
│     Task<AdmissionResult>            │
│ - ReserveSlot(tenantId, class): Task<SlotReservation> │
│ - ReleaseReservation(reservationId): Task            │
└────────────────┬────────────────────────────────────┘
                  │ admitido ──▶
                  ▼
┌────────────────────────────────────┐
│       PriorityCostEvaluator          │ «service»
├────────────────────────────────────┤
│ - _weights: IPolicyWeightsProvider   │
│ - _costEstimator: ICostEstimator     │
│ - _policy: ISchedulingPolicy         │
│ - _featureExtractor: IFeatureExtractor │
├────────────────────────────────────┤
│ + Evaluate(req: SchedulingRequest): │
│     Task<EffectivePriority>          │
│ - ComputeScore(l, cost, e,           │
│     alpha, beta, gamma): decimal     │  // C = α·L + β·$ + γ·E
│ - ApplyAging(basePriority, waitMs): int │
│ - ApplyEdf(priority, deadline): int  │
└────────────────┬────────────────────┘
                  │ prioridade efetiva ──▶
                  ▼
┌────────────────────────────────────┐
│          PriorityQueueStore          │ «service»
├────────────────────────────────────┤
│ + Enqueue(entry: QueueEntry): Task    │
│ + PeekTop(tenant, class, shard):     │
│     Task<QueueEntry?>                │
│ + Dequeue(tenant, class, shard,      │
│     leaseId): Task<QueueEntry?>      │
│ + Requeue(entry: QueueEntry,         │
│     preserveAging: bool): Task        │
└────────────────┬────────────────────┘
                  │ topo elegível ──▶
                  ▼
┌────────────────────────────────────┐
│            DispatchLoop              │ «service · worker»
├────────────────────────────────────┤
│ - _queue: IPriorityQueueStore        │
│ - _placement: IPlacementEngine       │
│ - _runtimeClient: IRuntimeSupervisorClient │
│ - _journal: IDecisionJournal          │
├────────────────────────────────────┤
│ + RunAsync(shard: int,                │
│     ct: CancellationToken): Task      │
│ - TryDispatch(entry: QueueEntry):     │
│     Task<SchedulingDecision>          │
└────────────────────────────────────┘
```

**Contrato de `Submit`:** DEVE ser idempotente por `request_id`/`Idempotency-Key`
(RFC-0001 §5.5); DEVE retornar em ≤ 20 ms (p99, NFR-001) no caminho quente; NÃO
DEVE realizar I/O bloqueante síncrono fora de Redis/cache de curta duração.

---

## 3. Interface `ISchedulingPolicy` e Implementações

```
┌──────────────────────────────────────────┐
│           «interface» ISchedulingPolicy    │
├──────────────────────────────────────────┤
│ + PolicyId: string { get; }                │
│ + PolicyVersion: string { get; }            │
├──────────────────────────────────────────┤
│ + ComputePriority(features: FeatureVector,  │
│     weights: PolicyWeights): PriorityScore  │
│ + SelectPreemptionVictims(                  │
│     candidate: QueueEntry,                   │
│     running: IReadOnlyList<QueueEntry>):     │
│     IReadOnlyList<QueueEntry>                │
└───────────────────┬──────────────────────┘
                     △ implementa
        ┌────────────┴─────────────┐
        │                          │
┌───────────────────────┐  ┌────────────────────────┐
│   HeuristicPolicy       │  │    LearnedPolicy (RL)   │
│  «policy v1 · default»  │  │   «policy v2 · opt-in»  │
├───────────────────────┤  ├────────────────────────┤
│ + PolicyId = "heuristic"│  │ + PolicyId = "rl"        │
│ + PolicyVersion="1"     │  │ + PolicyVersion=<hash>   │
│ - _alpha/_beta/_gamma    │  │ - _model: IInferenceRef  │
├───────────────────────┤  ├────────────────────────┤
│ + ComputePriority(...)  │  │ + ComputePriority(...)   │
│   → C=α·L+β·$+γ·E,      │  │   → score aprendido a    │
│     aging, EDF           │  │     partir de feature    │
│ + SelectPreemptionVictims│  │     vector + política     │
│   → menor prioridade     │  │     de exploração          │
│     efetiva primeiro     │  │     limitada (safe RL)     │
└───────────────────────┘  └────────────────────────┘
```

**Invariantes da interface (contrato estável — FR-013, ADR-0096, RFC-0091):**

1. `ISchedulingPolicy` NÃO DEVE expor tipos específicos de implementação (v1/v2)
   em sua assinatura pública — apenas `FeatureVector`, `PolicyWeights`,
   `PriorityScore`, `QueueEntry`.
2. A troca de `HeuristicPolicy` para `LearnedPolicy` DEVE ser feita por
   configuração (`scheduler.policy.engine`), NÃO DEVE exigir mudança de API
   externa nem de schema de eventos (`SchedulingDecision.policy_id`/
   `policy_version` absorvem a variação).
3. Toda implementação DEVE ser determinística o suficiente para ser
   auditável: para os mesmos `FeatureVector`/`PolicyWeights`/seed, o mesmo
   `PriorityScore` DEVE ser reproduzível para fins de auditoria (025) —
   `LearnedPolicy` DEVE registrar a versão do modelo (`policy_version`) usada
   em cada decisão.
4. `SelectPreemptionVictims` NÃO DEVE retornar a própria tarefa candidata nem
   tarefas em estado terminal.

---

## 4. Diagrama de Classes — Placement e Preempção

```
┌────────────────────────────────────┐
│           PlacementEngine            │ «service»
├────────────────────────────────────┤
│ - _shardRouter: IShardRouter          │
│ - _capacity: ICapacityProvider        │
├────────────────────────────────────┤
│ + Resolve(req: SchedulingRequest):    │
│     Task<PlacementTarget>              │
│ - ApplyAffinity(candidates,           │
│     affinity: AffinityHint):          │
│     IReadOnlyList<CapacitySnapshot>    │
└────────────────┬────────────────────┘
                  │ ◇── usa
                  ▼
┌────────────────────────────────────┐
│            ShardRouter               │ «service · stateless»
├────────────────────────────────────┤
│ + N: int { get; }  // scheduler.shards.count │
├────────────────────────────────────┤
│ + ResolveShard(tenantId: string,      │
│     agentId: string): int             │  // hash(tenant,agent) mod N
│ + ShardOwners(shard: int):            │
│     IReadOnlyList<NodeId>              │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│          PreemptionManager           │ «service»
├────────────────────────────────────┤
│ - _queue: IPriorityQueueStore         │
│ - _runtimeClient: IRuntimeSupervisorClient │
│ - _policy: IPolicyClient              │
│ - _policyStrategy: ISchedulingPolicy   │
├────────────────────────────────────┤
│ + TryPreempt(candidate: QueueEntry,    │
│     runningInShard: IReadOnlyList<QueueEntry>): │
│     Task<PreemptResult>                │
│ - RequestGracePeriod(victim: QueueEntry, │
│     graceMs: int): Task                 │
└────────────────────────────────────┘
```

**Invariantes de placement (ADR-0092):**

1. `ResolveShard` DEVE ser puramente determinístico — mesma entrada
   `(tenant_id, agent_id)` DEVE sempre produzir o mesmo `shard` para um dado
   `N` (propriedade exigida para consistência de fila e idempotência).
2. `PlacementTarget` DEVE conter `{shard, node_id, runtime_pool}` e NÃO DEVE
   ser resolvido para um nó sem capacidade residual (`free_slots ≥ 1`).
3. Mudança de `N` (rebalanceamento) DEVE usar hashing consistente para
   minimizar migração de shard — coordenada com 027-Cluster; NÃO é
   recarregável isoladamente em runtime (ver `_DESIGN_BRIEF.md` §8).

**Invariantes de preempção (ADR-0094):**

1. `TryPreempt` NÃO DEVE selecionar vítima sem autorização do PDP (`PolicyClient`).
2. Toda preempção DEVE respeitar `grace_period_ms` (default 2000 ms) antes de
   a suspensão ser considerada efetiva no `RuntimeSupervisorClient`.
3. Uma tarefa recém-agendada (dentro da janela de cooldown) NÃO DEVE ser
   escolhida como vítima novamente (anti-*thrashing*, histerese).

---

## 5. Diagrama de Classes — Estado Quente (Queue / Quota / Capacity)

```
┌────────────────────────────────────┐       ┌────────────────────────────────────┐
│       «value object» QueueEntry      │       │   «value object» EffectivePriority   │
├────────────────────────────────────┤       ├────────────────────────────────────┤
│ + TaskUrn: Urn                        │       │ + Score: int                          │
│ + RequestId: Ulid                     │       │ + CostScore: decimal                   │
│ + Score: double  // menor = urgente   │       │ + Alpha/Beta/Gamma: decimal             │
│ + LeaseOwner: string?                 │       │ + FeatureVector: JsonDocument            │
│ + LeaseUntil: DateTimeOffset?         │       └────────────────────────────────────┘
│ + DeadlineAt: DateTimeOffset?         │
│ + SubmittedAt: DateTimeOffset         │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│           QuotaLedger                │ «service»
├────────────────────────────────────┤
│ + TryReserve(tenantId, class,          │
│     n: int = 1): Task<SlotReservation?> │  // CAS atômico (script Lua)
│ + Release(reservationId: Guid): Task    │
│ + InFlight(tenantId, class): Task<int>  │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│         CapacityProvider             │ «service»
├────────────────────────────────────┤
│ + Snapshot(shard: int):                │
│     Task<IReadOnlyList<CapacitySnapshot>> │
│ + OnHeartbeat(snapshot: CapacitySnapshot): Task │
│ + Pressure(shard: int): Task<decimal>   │  // 0..1, insumo do BackpressureController
└────────────────────────────────────┘

┌────────────────────────────────────┐
│      BackpressureController           │ «service»
├────────────────────────────────────┤
│ - _capacity: ICapacityProvider         │
│ - _queue: IPriorityQueueStore           │
│ - _jetStreamLagProvider: ILagProvider   │
├────────────────────────────────────┤
│ + Evaluate(shard: int):                │
│     Task<BackpressureLevel>             │  // Accept | Defer | Reject
│ + RetryAfterMs(level: BackpressureLevel): int │
└────────────────────────────────────┘
```

**Invariantes de estado quente (NFR-005, ADR-0090, ADR-0093):**

1. `QuotaLedger.TryReserve` DEVE ser atômico (script Lua/transação Redis) — a
   soma de reservas ativas NÃO DEVE jamais exceder
   `scheduler.max_concurrency.per_tenant`/`per_class`.
2. `QueueEntry.Score` DEVE ser recomputado a cada aplicação de aging
   (`scheduler.aging.factor_per_sec`), respeitando o teto
   `scheduler.aging.max_boost`.
3. `BackpressureController.Evaluate` DEVE retornar `Reject` quando pressão
   ≥ `scheduler.backpressure.reject_threshold` (default 0.95), `Defer` quando
   ≥ `scheduler.backpressure.defer_threshold` (default 0.80), caso contrário
   `Accept`.
4. Toda leitura de `CapacitySnapshot` DEVE respeitar TTL de frescor
   (`scheduler.capacity.heartbeat_ttl_ms`); snapshot expirado é tratado como
   capacidade zero (fail-safe).

---

## 6. Diagrama de Classes — Clientes de Dependências Externas

```
┌────────────────────────────────────┐   ┌────────────────────────────────────┐
│           PolicyClient                │   │           BudgetClient                │
│         «adapter · gRPC client»        │   │         «adapter · gRPC client»        │
├────────────────────────────────────┤   ├────────────────────────────────────┤
│ + Authorize(subject, action,          │   │ + GetAvailableBudget(tenantId):        │
│     resource, context): Task<PdpDecision> │   │     Task<decimal>                       │
│                                        │   │ + EstimateCost(req):                   │
│  PdpDecision: Allow | Deny             │   │     Task<CostEstimate>                  │
└────────────────────────────────────┘   └────────────────────────────────────┘

┌────────────────────────────────────┐   ┌────────────────────────────────────┐
│      RuntimeSupervisorClient          │   │          CostEstimator                │
│         «adapter · gRPC client»        │   │         «service · cache local»        │
├────────────────────────────────────┤   ├────────────────────────────────────┤
│ + Dispatch(decision:                   │   │ + Estimate(req: SchedulingRequest):     │
│     SchedulingDecision): Task<Ack>      │   │     Task<CostEstimate>                  │
│ + RequestSuspend(taskUrn: Urn,          │   │  // fallback: última estimativa cacheada │
│     graceMs: int): Task<Ack>            │   │  //  + margem conservadora sob timeout   │
│ + Heartbeats(): IAsyncEnumerable<        │   └────────────────────────────────────┘
│     RuntimeSignal>                     │
└────────────────────────────────────┘
```

**Invariantes de fronteira externa:**

1. Nenhum destes *clients* DEVE conter lógica de decisão de scheduling —
   apenas tradução de contrato (adapter) para o serviço remoto correspondente.
2. Toda chamada DEVE propagar `traceparent`/`X-AIOS-Tenant` (RFC-0001 §5.6).
3. Sob timeout/indisponibilidade, cada cliente DEVE aplicar a estratégia de
   `_DESIGN_BRIEF.md` §9 (ex.: `PolicyClient` → *default deny*;
   `BudgetClient`/`CostEstimator` → cache + margem conservadora).

---

## 7. Diagrama de Classes — Persistência e Proveniência

```
┌────────────────────────────────────┐
│          DecisionJournal              │ «service · outbox»
├────────────────────────────────────┤
│ - _db: NpgsqlConnection                │
│ - _publisher: IEventPublisher           │
├────────────────────────────────────┤
│ + RecordAndPublish(                    │
│     decision: SchedulingDecision,       │
│     evt: CloudEvent): Task              │  // transação única: INSERT + outbox row
└────────────────┬────────────────────┘
                  │ ◆── compõe
                  ▼
┌────────────────────────────────────┐
│           EventPublisher              │ «service»
├────────────────────────────────────┤
│ + Publish(subject: string,             │
│     evt: CloudEvent): Task              │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│           IdempotencyStore             │ «service»
├────────────────────────────────────┤
│ + TryBegin(idemKey: string,            │
│     requestHash: string):              │
│     Task<IdempotencyLease>              │  // Redis (rápido) + PostgreSQL (≥24h)
│ + Complete(idemKey: string,            │
│     response: JsonDocument): Task        │
│ + GetIfExists(idemKey: string):         │
│     Task<JsonDocument?>                  │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│         ReconciliationWorker           │ «service · worker»
├────────────────────────────────────┤
│ - _queue: IPriorityQueueStore           │
│ - _capacity: ICapacityProvider          │
│ - _journal: IDecisionJournal            │
├────────────────────────────────────┤
│ + RunAsync(ct: CancellationToken): Task  │
│ - ExpireOrphanLeases(): Task<int>        │
│ - DetectLostDecisions(): Task<int>       │
└────────────────────────────────────┘
```

**Invariantes de persistência/proveniência (ADR-0097, N-09):**

1. `SchedulingDecision` é **append-only** — nenhuma linha PODE ser atualizada
   ou removida (event sourcing); correções são novas decisões
   (`outcome=DEFERRED`/`RESUMED` etc.).
2. `RecordAndPublish` DEVE ser atômico: se a escrita em PostgreSQL falhar, o
   evento NÃO DEVE ser publicado; se a publicação falhar após commit, o
   outbox DEVE reter o evento para republicação com backoff (at-least-once).
3. `IdempotencyStore.TryBegin` com `requestHash` divergente para a mesma
   `idemKey` DEVE falhar com `AIOS-SCHED-0006` (conflito de idempotência).
4. `ReconciliationWorker` NÃO DEVE criar uma nova admissão — apenas
   re-enfileirar (`Requeue`) tarefas cujo lease expirou, preservando
   idempotência via `request_id`.

---

## 8. Entidades de Dados Canônicas (Contratos de Mensagem)

```
┌──────────────────────────────────────────────┐
│      «entity» SchedulingRequest                │
├──────────────────────────────────────────────┤
│ + RequestId: Ulid              «PK»            │
│ + TenantId: string              «RLS»           │
│ + TaskUrn: Urn                                  │
│ + AgentUrn: Urn                                 │
│ + ServiceClass: ServiceClass    (enum)          │
│ + PriorityHint: int  (0..100, default 50)       │
│ + DeadlineAt: DateTimeOffset?                    │
│ + EstCostUsd: decimal (≥0)                       │
│ + EstLatencyMs: int (≥0)                         │
│ + EstTokens: long (≥0)                           │
│ + QualityFloor: decimal (0..1)                   │
│ + Affinity: JsonDocument?                        │
│ + PayloadRef: Urn?                                │
│ + SubmittedAt: DateTimeOffset                     │
│ + Traceparent: string                             │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│      «entity» SchedulingDecision  (event-sourced)│
├──────────────────────────────────────────────┤
│ + DecisionId: Ulid              «PK»            │
│ + RequestId: Ulid               «FK»            │
│ + TenantId: string               «RLS»           │
│ + Outcome: DecisionOutcome        (enum)          │
│ + EffectivePriority: int                          │
│ + CostScore: decimal                              │
│ + Alpha, Beta, Gamma: decimal                      │
│ + PlacementTarget: JsonDocument?                   │
│ + PolicyId: string                                 │
│ + PolicyVersion: string                            │
│ + FeatureVector: JsonDocument?                      │
│ + ReasonCode: string?                               │
│ + DecidedAt: DateTimeOffset                          │
│ + LatencyDecisionMs: decimal                         │
└──────────────────────────────────────────────┘

ServiceClass = INTERACTIVE | BATCH | BACKGROUND | SYSTEM
DecisionOutcome = ADMITTED | SCHEDULED | REJECTED | PREEMPTED | RESUMED | DEFERRED
```

`SchedulingRequest` ◇── `SchedulingDecision` (agregação 1→N: uma requisição
pode gerar várias decisões ao longo do ciclo de vida — admissão, re-schedule,
preempção). Ver `Database.md` para DDL completo e `StateMachine.md` para o
mapeamento outcome↔estado.

---

## 9. Invariantes Globais

| # | Invariante | Onde é garantida |
|---|-----------|-------------------|
| INV-01 | Todo registro (`SchedulingRequest`, `SchedulingDecision`, `QueueEntry`) carrega `tenant_id` válido; isolamento por RLS/namespace. | `SchedulingGateway`, PostgreSQL RLS, Redis keyspace prefixado por tenant. |
| INV-02 | `task_urn`/`agent_urn` DEVEM ser URNs válidas (`urn:aios:<tenant>:<tipo>:<id>`, RFC-0001 §5.1). | Validação em `SchedulingGateway`. |
| INV-03 | Uma `SchedulingRequest` NÃO DEVE resultar em mais de uma decisão `SCHEDULED` ativa simultânea. | Lease em `PriorityQueueStore` + `IdempotencyStore`. |
| INV-04 | A soma de slots reservados por tenant/classe NÃO DEVE exceder `max_concurrency`. | `QuotaLedger` (reserva atômica). |
| INV-05 | `ISchedulingPolicy` é a única via de cálculo de prioridade — nenhum componente calcula score fora dela. | `PriorityCostEvaluator`. |
| INV-06 | `SchedulingDecision` é append-only; nunca é alterada ou apagada. | `DecisionJournal`. |
| INV-07 | `ResolveShard` é determinístico para um dado `N`. | `ShardRouter`. |
| INV-08 | Toda preempção passa por autorização do PDP antes de sinalizar suspensão. | `PreemptionManager` → `PolicyClient`. |
| INV-09 | `alpha + beta + gamma ≈ 1` (normalizados na leitura). | `PolicyWeightsProvider`. |
| INV-10 | Nenhum componente do pipeline de decisão persiste payload de tarefa — apenas `payload_ref`. | `SchedulingRequest` (contrato). |

---

## 10. Relações entre Componentes (Visão Consolidada)

```
SchedulingGateway ◆── IdempotencyStore
SchedulingGateway ──▶ AdmissionController

AdmissionController ◇── QuotaLedger
AdmissionController ◇── BudgetClient
AdmissionController ◇── CapacityProvider
AdmissionController ◇── PolicyClient
AdmissionController ◇── BackpressureController
AdmissionController ──▶ PriorityCostEvaluator

PriorityCostEvaluator ◇── FeatureExtractor
PriorityCostEvaluator ◇── CostEstimator
PriorityCostEvaluator ◆── ISchedulingPolicy
HeuristicPolicy ┈┈▷ ISchedulingPolicy
LearnedPolicy   ┈┈▷ ISchedulingPolicy
PriorityCostEvaluator ──▶ PriorityQueueStore

PriorityQueueStore ◇── PreemptionManager  (associação bidirecional)
PreemptionManager ◇── RuntimeSupervisorClient
PreemptionManager ◇── PolicyClient

DispatchLoop ◇── PriorityQueueStore
DispatchLoop ◇── PlacementEngine
DispatchLoop ◇── RuntimeSupervisorClient
DispatchLoop ◆── DecisionJournal

PlacementEngine ◇── ShardRouter
PlacementEngine ◇── CapacityProvider

DecisionJournal ◆── EventPublisher
EventPublisher ──▶ "NATS/JetStream"

ReconciliationWorker ◇── PriorityQueueStore
ReconciliationWorker ◇── CapacityProvider
ReconciliationWorker ◇── DecisionJournal

SchedulerTelemetry - - ▷ (todos os componentes, instrumentação transversal)
```

Legenda de leitura: `◆──` indica que o ciclo de vida do componente à direita é
gerenciado pelo componente à esquerda (composição); `◇──` indica dependência
injetada (agregação, ciclo de vida independente, tipicamente via DI container
do .NET). Nenhuma seta cruza a fronteira do módulo sem passar por um dos
*clients* de §6 — reforçando a fronteira descrita em `Architecture.md` §6.
