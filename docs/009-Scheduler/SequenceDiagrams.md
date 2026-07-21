---
Documento: SequenceDiagrams
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0093, ADR-0094, ADR-0095 (a propor); ADR-0001..ADR-0010 (herdadas)
RFCs relacionados: RFC-0001 (baseline); RFC-0090 (a propor)
Depende de: 006-Kernel, 007-Agent-Runtime, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — SequenceDiagrams

> Diagramas de sequência ASCII para os fluxos críticos do Scheduler — caminho
> feliz e caminhos de falha — conforme componentes de `Architecture.md`,
> contratos de `API.md`/`Events.md` (a derivar de `_DESIGN_BRIEF.md` §5/§6) e a
> máquina de estados de `StateMachine.md`. Setas `───▶` denotam chamada
> síncrona (gRPC/REST, aguarda resposta); setas `┄┄▶` denotam mensagem
> assíncrona (evento NATS/JetStream); `- - -` denota resposta/retorno.

## Índice

1. Convenções de notação
2. Fluxo 1 (feliz) — Admissão, fila, dispatch e execução completa
3. Fluxo 2 (falha) — Rejeição por cota excedida
4. Fluxo 3 (falha) — Rejeição por orçamento insuficiente (Cost-Optimizer)
5. Fluxo 4 (falha) — Policy Engine indisponível (*default deny*)
6. Fluxo 5 (feliz) — Backpressure: defer sob saturação e retomada
7. Fluxo 6 (feliz) — Preempção compensável com grace period
8. Fluxo 7 (feliz) — Idempotência: retry de `Submit` com mesma `Idempotency-Key`
9. Fluxo 8 (falha) — Deadline expira em fila (EDF/EXPIRED)
10. Fluxo 9 (falha) — Runtime Supervisor não responde (lease órfão / reconciliação)
11. Fluxo 10 (feliz) — Cancelamento de tarefa em `QUEUED`
12. Fluxo 11 (falha) — Falha em execução com retry
13. Timeouts e SLOs por fluxo

---

## 1. Convenções de Notação

```
Participante A     Participante B
     │                    │
     │──── msg síncrona ─▶│   (aguarda resposta; timeout aplicável)
     │◀- - - resposta - - -│
     │┄┄┄▶ evento async ┄┄▶│   (publish/consume via NATS/JetStream, sem aguardar)
     │                    │
     ┌───────────────┐
     │ processamento  │   (caixa de atividade/computação local)
     └───────────────┘
```

Todos os fluxos assumem os cabeçalhos obrigatórios de RFC-0001 §5.6
(`traceparent`, `X-AIOS-Tenant`, `Idempotency-Key` em mutações) mesmo quando
omitidos do diagrama por brevidade.

---

## 2. Fluxo 1 (Feliz) — Admissão, Fila, Dispatch e Execução Completa

**Participantes:** Kernel(006), SchedulingGateway, AdmissionController,
PolicyClient→Policy(022), BudgetClient→CostOptimizer(026),
CapacityProvider→Cluster(027), PriorityCostEvaluator, PriorityQueueStore,
DispatchLoop, PlacementEngine, RuntimeSupervisorClient→RuntimeSupervisor(007),
DecisionJournal, EventPublisher→NATS/JetStream.

```
Kernel   SchedGw   AdmissionCtrl  PolicyClient  BudgetClient  CapacityProv  PriorityEval  QueueStore  DispatchLoop  PlacementEng  RTSupClient  DecisionJournal  NATS
  │  Submit(req,      │              │              │              │            │            │            │            │             │              │
  │  idemKey)          │              │              │              │            │            │            │            │             │              │
  ├───────────────────▶│              │              │              │            │            │            │            │             │              │
  │                    │ valida envelope/idem         │              │            │            │            │            │             │              │
  │                    ├─────────────▶│              │              │            │            │            │            │             │              │
  │                    │              │ Authorize(admit) │           │            │            │            │            │             │              │
  │                    │              ├─────────────▶│ (PDP 022)    │            │            │            │            │             │              │
  │                    │              │◀- allow - - -│              │            │            │            │            │             │              │
  │                    ├─────────────────────────────▶│ GetAvailableBudget       │            │            │            │             │              │
  │                    │              │              ├─────────────▶│ (026)      │            │            │            │             │              │
  │                    │              │              │◀- ok - - - - │            │            │            │            │             │              │
  │                    ├──────────────────────────────────────────▶│ Snapshot(shard)          │            │            │             │              │
  │                    │              │              │              │◀- free≥1 - │            │            │            │             │              │
  │                    │  ┌──────────────────────────────────────────────┐        │            │            │            │             │              │
  │                    │  │ QuotaLedger.TryReserve() [Redis CAS atômico]  │        │            │            │            │             │              │
  │                    │  └──────────────────────────────────────────────┘        │            │            │            │             │              │
  │                    │              │              │              │  admitted  │            │            │            │             │              │
  │                    ├──────────────────────────────────────────────────────────▶│ Evaluate(req)           │            │             │              │
  │                    │              │              │              │            │  C=α·L+β·$+γ·E, aging, EDF│            │             │              │
  │                    │              │              │              │            │◀- effectivePriority - - -│            │             │              │
  │                    │                                                          ├───────────▶│ Enqueue(entry, score)   │             │              │
  │                    │                                                          │              │(ZSET sched:{t}:{c}:{s}) │             │              │
  │                    │  ┌──────────────────────────────────────────────────────────────┐        │            │             │              │
  │                    │  │ DecisionJournal.RecordAndPublish(outcome=ADMITTED)  [outbox]  │        │            │             │              │
  │                    │  └──────────────────────────────────────────────────────────────┘        │            │             │              │
  │                    │                                                                            │       ┄┄┄▶│ task.execution.admitted      │
  │◀── 200 OK (decisionId, effectivePriority) ───────────────────────────────────────────────────────────────────────────────────────────────│
  │                    │                                                                            │            │            │             │              │
  │                    │                                                                            │            ┌──────────────────────┐    │              │
  │                    │                                                                            │            │ DispatchLoop.RunAsync│    │              │
  │                    │                                                                            │            │   (por shard, loop)  │    │              │
  │                    │                                                                            │            └──────────┬───────────┘    │              │
  │                    │                                                                            ├───PeekTop/Dequeue(lease)▶│            │             │              │
  │                    │                                                                            │◀- topo elegível - - - -│            │             │              │
  │                    │                                                                            │                          ├─Resolve()─▶│ placement  │             │              │
  │                    │                                                                            │                          │◀- target - │             │              │
  │                    │                                                                            │                          ├─Dispatch(SchedulingDecision)──────────────▶│              │
  │                    │                                                                            │                          │            │            │◀- Ack - - -│              │
  │                    │                                                                            │  ┌──────────────────────────────────────────────────────┐            │              │
  │                    │                                                                            │  │ DecisionJournal.RecordAndPublish(outcome=SCHEDULED)   │            │              │
  │                    │                                                                            │  └──────────────────────────────────────────────────────┘            │              │
  │                    │                                                                            │                          │            │            │       ┄┄┄▶ task.execution.scheduled│
  │                    │                                                                                                                     │            │              │
  │                    │◀┄┄┄ agent.lifecycle.spawned (RTSup confirma boot) ┄┄┄ NATS ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┘              │
  │                    │  (marca RUNNING internamente — observação)                                                                                                        │
  │                    │◀┄┄┄ task.execution.completed (Runtime concluiu) ┄┄┄ NATS ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┘
  │                    │  QuotaLedger.Release(reservationId)                                                                                                              │
```

**Notas:** o retorno síncrono ao Kernel ocorre logo após `ADMITTED` (não
espera dispatch/execução) — caminho quente com SLO **p99 ≤ 20 ms** (NFR-001).
Dispatch, placement e execução são assíncronos e observados via eventos.

---

## 3. Fluxo 2 (Falha) — Rejeição por Cota Excedida

```
Kernel   SchedulingGateway    AdmissionController    QuotaLedger(Redis)     EventPublisher
  │ Submit(req)     │                    │                    │                    │
  ├────────────────▶│                    │                    │                    │
  │                 ├───────────────────▶│                    │                    │
  │                 │                    ├── InFlight(tenant,class) ─────────────▶│
  │                 │                    │◀── inflight ≥ max_concurrency - - - - │
  │                 │                    │  guarda falha: cota excedida            │
  │                 │                    ├─ RecordAndPublish(outcome=REJECTED,     │
  │                 │                    │    reasonCode=AIOS-SCHED-0001) ────────▶│
  │                 │                    │                    │         ┄┄┄▶ task.execution.rejected
  │◀── 429 Too Many Requests ────────────┤                    │                    │
  │   { code: AIOS-SCHED-0001,           │                    │                    │
  │     retriable: true, retryAfterMs }  │                    │                    │
```

**Guarda:** `InFlight(tenant, class) ≥ scheduler.max_concurrency.per_tenant` ou
`per_class`. **Erro:** `AIOS-SCHED-0001` (HTTP 429, `retriable=true`). Nenhuma
reserva é criada — `QuotaLedger.TryReserve` sequer é chamado.

---

## 4. Fluxo 3 (Falha) — Rejeição por Orçamento Insuficiente

```
Kernel   SchedulingGateway   AdmissionController   BudgetClient    Cost-Optimizer(026)
  │ Submit(req)    │                   │                 │                  │
  ├───────────────▶│                  │                 │                  │
  │                ├─────────────────▶│                 │                  │
  │                │                  ├────────────────▶│ GetAvailableBudget(tenant) ─▶│
  │                │                  │                 │◀── budget=$0.12 - - - -│
  │                │                  │  EstCostUsd (req) = $0.50 > budget disponível  │
  │                │                  │  guarda falha: orçamento insuficiente          │
  │                │                  ├─ RecordAndPublish(outcome=REJECTED,             │
  │                │                  │    reasonCode=AIOS-SCHED-0002) ─────────────────▶ (outbox+NATS)
  │◀── 429 Too Many Requests ─────────┤                 │                  │
  │   { code: AIOS-SCHED-0002, retriable: true }        │                  │
```

**Guarda:** `est_cost_usd > orçamento disponível` (consulta a 026).
**Erro:** `AIOS-SCHED-0002` (HTTP 429, `retriable=true`).
**Degradação (falha do 026):** se `BudgetClient` expira por timeout, usa
última estimativa cacheada com margem conservadora (`_DESIGN_BRIEF.md` §9);
só aplica `AIOS-SCHED-0002` quando o orçamento estiver no limite.

---

## 5. Fluxo 4 (Falha) — Policy Engine Indisponível (*Default Deny*)

```
Kernel   SchedulingGateway   AdmissionController   PolicyClient    Policy Engine(022)
  │ Submit(req)    │                   │                 │                  │
  ├───────────────▶│                  │                 │                  │
  │                ├─────────────────▶│                 │                  │
  │                │                  ├────────────────▶│ Authorize(admit) ─▶│
  │                │                  │                 │      (timeout, sem resposta em decision.timeout_ms)
  │                │                  │                 │◀── TimeoutException  │
  │                │                  │  fail-safe: default deny (RFC-0001 §5.8)│
  │                │                  ├─ RecordAndPublish(outcome=REJECTED,     │
  │                │                  │    reasonCode=AIOS-SCHED-0004) ─────────▶ (outbox+NATS)
  │◀── 403 Forbidden ──────────────────┤                 │                  │
  │   { code: AIOS-SCHED-0004, retriable: false }        │                  │
```

**Guarda:** ausência de resposta do PDP dentro de `scheduler.decision.timeout_ms`
(default 15 ms) é tratada como **negação** — nunca como aprovação implícita.
**Erro:** `AIOS-SCHED-0004` (HTTP 403, `retriable=false`). Este é o único
caminho de falha em que o Scheduler **não** oferece retry automático ao
cliente, pois a segurança prevalece sobre a disponibilidade (SM-07).

---

## 6. Fluxo 5 (Feliz) — Backpressure: Defer sob Saturação e Retomada

**Participantes:** Kernel, SchedulingGateway, AdmissionController,
BackpressureController, CapacityProvider, PriorityQueueStore, DispatchLoop.

```
Kernel   SchedulingGateway   AdmissionController   BackpressureCtrl   CapacityProvider   QueueStore
  │ Submit(req)    │                   │                    │                 │              │
  ├───────────────▶│                  │                    │                 │              │
  │                ├─────────────────▶│                    │                 │              │
  │                │                  ├──────────────────▶│ Evaluate(shard) │              │
  │                │                  │                    ├───────────────▶│ Pressure()   │
  │                │                  │                    │◀── pressure=0.85│              │
  │                │                  │◀── level=Defer ────┤                 │              │
  │                │                  │ (0.80 ≤ pressure < 0.95)                              │
  │                │                  │  admissão prossegue mas fila entra em modo DEFERRED    │
  │                │                  ├──────────────────────────────────────────────────────▶│ Enqueue(entry, deferred=true)
  │◀── 202 Accepted (decisionId, status=DEFERRED, retryAfterMs=1500) ────────────────────────│
  │                │                  │                    │                 │              │
  │                │                  │  ...  janela decorre; capacidade libera (nó reporta heartbeat) ...
  │                │                  │                    │                 │◀┄┄ heartbeat (free_slots↑) ┄┄│
  │                │                  │◀── level=Accept ───┤                 │              │
  │                │                                                                          │  DispatchLoop retoma
  │                │                                                                          ├─────────────▶ (fluxo normal, ver §2)
```

**Guarda:** `pressure(shard) ∈ [defer_threshold, reject_threshold)` →
`Defer`; tarefa é admitida mas o dispatch é retardado; cliente recebe
`Retry-After` calculado por `scheduler.backpressure.retry_after_ms` (default
1500 ms). Se `pressure ≥ reject_threshold` (0.95), a resposta seria `503`
(`AIOS-SCHED-0003`, ver Fluxo equivalente de rejeição).

---

## 7. Fluxo 6 (Feliz) — Preempção Compensável com Grace Period

**Participantes:** DispatchLoop (nova tarefa de alta prioridade),
PriorityCostEvaluator, PreemptionManager, PolicyClient→Policy(022),
PriorityQueueStore, RuntimeSupervisorClient→RuntimeSupervisor(007)→Runtime,
DecisionJournal.

```
DispatchLoop   PriorityEval   PreemptionMgr   PolicyClient   QueueStore   RTSupClient   RuntimeSupervisor(007)   DecisionJournal
     │              │               │              │              │             │                │                   │
     │ nova tarefa alta prioridade sem capacidade livre no shard                                                       │
     ├─────────────▶│ score alto     │              │              │             │                │                   │
     │              ├──────────────▶│ TryPreempt(candidate, running[])           │             │                │                   │
     │              │               ├─────────────▶│ Authorize(preempt, victim) │             │                │                   │
     │              │               │◀── allow ─────┤              │             │                │                   │
     │              │               │  seleciona vítima = menor prioridade efetiva em execução      │                   │
     │              │               │  verifica cooldown/histerese (fora da janela recente)         │                   │
     │              │               ├─────────────────────────────▶│ RequestSuspend(victimUrn, graceMs=2000) ─────────▶│
     │              │               │              │              │             │                ├─ envia sinal de suspensão ao Agent Runtime
     │              │               │  ┌──────────────────────────────────────────────────────────────┐                │
     │              │               │  │ DecisionJournal.RecordAndPublish(outcome=PREEMPTED,           │                │
     │              │               │  │   byTaskUrn=<nova tarefa>, gracePeriodMs=2000)  [outbox]      │                │
     │              │               │  └──────────────────────────────────────────────────────────────┘                │
     │              │               │                                                              ┄┄┄▶ task.execution.preempted
     │              │               │              │              │             │◀── grace period decorre (2000ms) ───│
     │              │               │              │              │             │◀┄┄ agent.lifecycle.suspended ┄┄┄┄┄┄┄┤
     │              │               ├─────────────────────────────▶│ Requeue(victim, preserveAging=true) │             │                │                   │
     │              │               │                                                              ┄┄┄▶ task.execution.resumed (quando recapturar slot)
     │  nova tarefa é despachada no slot liberado (segue Fluxo 1, a partir de "dispatch")            │
```

**Guarda:** `PDP=allow` para a preempção específica ∧ vítima fora do
cooldown ∧ `grace_period_ms` respeitado antes de considerar a suspensão
efetiva. **Ação compensável:** a vítima retorna a `QUEUED` com aging
preservado — nenhum trabalho é descartado, apenas suspenso (SM-05).

---

## 8. Fluxo 7 (Feliz) — Idempotência: Retry de `Submit`

**Cenário:** o cliente (Kernel) não recebe a resposta do primeiro `Submit`
(timeout de rede) e reenvia a mesma requisição com o mesmo `Idempotency-Key`.

```
Kernel   SchedulingGateway   IdempotencyStore    AdmissionController
  │ Submit(req, idemKey=K)   │                    │
  ├─────────────────────────▶│                   │
  │                          ├──TryBegin(K, hash)▶│ (Redis, não existe ainda)
  │                          │◀── lease adquirida │
  │                          ├───────────────────────────────────▶│ (segue Fluxo 1 normalmente)
  │                          │◀── SchedulingDecision(ADMITTED) ───│
  │                          ├──Complete(K, response)──▶│ (grava snapshot, TTL≥24h)
  │◀── 200 OK (decisionId=D1)│                   │
  │                          │                    │
  │  ... timeout de rede no cliente antes de ler a resposta ...
  │                          │                    │
  │ Submit(req, idemKey=K)  [retry idêntico]      │
  ├─────────────────────────▶│                   │
  │                          ├──TryBegin(K, hash)▶│ (Redis: já existe resultado)
  │                          │◀── snapshot(D1) - -│  (não chama AdmissionController novamente)
  │◀── 200 OK (decisionId=D1) [mesmo resultado]  │
```

**Invariante (NFR-011):** nenhuma segunda admissão/reserva de slot é criada; a
resposta é idêntica à primeira. Se o `requestHash` do retry divergir do
original para a mesma `idemKey`, o Scheduler retorna `AIOS-SCHED-0006`
(HTTP 409, conflito de idempotência) em vez do snapshot.

---

## 9. Fluxo 8 (Falha) — Deadline Expira em Fila (EDF/EXPIRED)

**Participantes:** PriorityQueueStore, ReconciliationWorker (ou tarefa
periódica `deadline_tick`), DecisionJournal, EventPublisher.

```
(processo periódico interno)   PriorityQueueStore   DecisionJournal   NATS
        │ deadline_tick (varredura periódica)   │                  │
        ├──────────────────────────────────────▶│                  │
        │                                        │ para cada entry: now > deadline_at?
        │                                        │  sim → remove da ZSET
        │◀── entries expiradas ──────────────────│                  │
        ├─ RecordAndPublish(outcome=REJECTED, reasonCode=deadline,  │
        │    code=AIOS-SCHED-0008) ──────────────────────────────▶│
        │                                                          ┄┄┄▶ task.execution.rejected
```

**Guarda:** `now > deadline_at` enquanto em `QUEUED` (a tarefa nunca chegou a
`SCHEDULED`). **Erro:** `AIOS-SCHED-0008` (HTTP 410, `retriable=false`) —
deadline já vencido é considerado definitivo, sem retry automático pelo
Scheduler (o chamador decide se resubmete com novo `deadline_at`).

---

## 10. Fluxo 9 (Falha) — Runtime Supervisor Não Responde (Lease Órfão)

**Participantes:** DispatchLoop (decisão original), RuntimeSupervisorClient,
ReconciliationWorker, PriorityQueueStore, DecisionJournal.

```
DispatchLoop   RTSupClient   RuntimeSupervisor(007)   ReconciliationWorker   QueueStore   DecisionJournal
     │ Dispatch(decision)     │                        │                       │              │
     ├───────────────────────▶│──────────────────────▶│ (sem resposta;         │              │
     │                        │                        │  heartbeat não chega) │              │
     │                        │  lease de dispatch com TTL (scheduler.lease.ttl_ms) expira      │
     │                        │                                          │                      │
     │                        │                        ┌──────────────────────────┐             │
     │                        │                        │ RunAsync loop periódico   │             │
     │                        │                        └───────────┬──────────────┘             │
     │                        │                        │ ExpireOrphanLeases() detecta lease vencida
     │                        │                        ├───────────────────────────▶│ (lease_owner/lease_until expirados)
     │                        │                        │  QuotaLedger.Release(reservationId)     │
     │                        │                        ├────────────────────────────────────────▶│ Requeue(entry, preserveAging=true)
     │                        │                        │  (volta a QUEUED; idempotência por request_id evita dupla execução)
     │                        │                        ├─ RecordAndPublish(outcome=DEFERRED/re-schedule) ─▶│
     │                        │                        │                                          ┄┄┄▶ (evento interno de reconciliação)
```

**Guarda:** ausência de heartbeat/confirmação dentro de
`scheduler.lease.ttl_ms` (default 10000 ms). **Ação de recuperação:**
`ReconciliationWorker` expira o lease, libera o slot e re-enfileira —
nenhum trabalho é considerado perdido porque a idempotência por `request_id`
impede execução duplicada caso o Runtime Supervisor responda tardiamente
(NFR-008: RTO ≤ 15 min).

---

## 11. Fluxo 10 (Feliz) — Cancelamento de Tarefa em `QUEUED`

```
Kernel   SchedulingGateway   AdmissionController/QueueStore   QuotaLedger   DecisionJournal
  │ Cancel(requestId, idemKey)  │                              │                │
  ├────────────────────────────▶│                             │                │
  │                             ├── verifica estado atual (QUEUED, não terminal) │
  │                             ├── remove de PriorityQueueStore ─▶│            │
  │                             ├── QuotaLedger.Release(reservationId) ────────▶│
  │                             ├─ RecordAndPublish(outcome=REJECTED,           │
  │                             │    reasonCode=cancelled) ─────────────────────▶│
  │◀── 200 OK (Ack) ────────────┤                             │      ┄┄┄▶ task.execution.rejected
```

**Guarda:** a tarefa DEVE estar em `QUEUED` ou `SCHEDULED` (não terminal). Se
já estiver em estado terminal, retorna `AIOS-SCHED-0011` (409). Operação
idempotente por `Idempotency-Key`.

---

## 12. Fluxo 11 (Falha) — Falha em Execução com Retry

```
RuntimeSupervisor(007)   SchedulingGateway (ReportRuntimeSignal)   QuotaLedger   QueueStore   DecisionJournal
       │ runtime.failed(taskUrn, willRetry=true)     │                    │             │              │
       ├─────────────────────────────────────────────▶│                  │             │              │
       │                                              ├─ QuotaLedger.Release() ────────▶│             │
       │                                              ├─ RecordAndPublish(outcome=FAILED observado)   │
       │                                              │                                                ┄┄┄▶ task.execution.failed
       │                                              │  avalia retriável ∧ tentativas < max_attempts   │
       │                                              │  backoff_base_ms decorrido?                      │
       │                                              ├───────────────────────────────▶│ Requeue(entry, attempt+1) │
       │                                              │                                                ┄┄┄▶ (novo ciclo — segue Fluxo 1 a partir de QUEUED)
       │  se tentativas ≥ max_attempts: estado permanece FAILED (terminal)              │
```

**Guarda:** `willRetry=true` ∧ `attempts < scheduler.retry.max_attempts`
(default 3) ∧ backoff (`scheduler.retry.backoff_base_ms`, default 500 ms,
exponencial com jitter) decorrido. Caso contrário, `FAILED` é terminal.

---

## 13. Timeouts e SLOs por Fluxo

| Fluxo | Timeout/SLO aplicável | Comportamento no timeout |
|-------|------------------------|-----------------------------|
| Fluxo 1 (Submit → ADMITTED) | `scheduler.decision.timeout_ms` (15 ms) fim-a-fim no caminho quente; SLO p99 ≤ 20 ms (NFR-001). | Se a soma das checagens exceder o orçamento de tempo, aplica-se fail-safe por dependência (ver Fluxos 3–4). |
| Fluxo 4 (PDP indisponível) | `scheduler.decision.timeout_ms`. | *Default deny* — `AIOS-SCHED-0004`. |
| Fluxo 5 (Backpressure) | Reavaliação periódica; `Retry-After` = `scheduler.backpressure.retry_after_ms` (1500 ms). | Cliente DEVE respeitar `Retry-After` antes de reenviar. |
| Fluxo 6 (Preempção) | `scheduler.preemption.grace_period_ms` (2000 ms); overhead de decisão ≤ 50 ms (NFR-009). | Suspensão forçada após grace period se Runtime não confirmar. |
| Fluxo 9 (Lease órfão) | `scheduler.lease.ttl_ms` (10000 ms); `scheduler.capacity.heartbeat_ttl_ms` (5000 ms). | `ReconciliationWorker` expira e re-enfileira; RTO ≤ 15 min (NFR-008). |
| Fluxo 11 (Retry) | `scheduler.retry.backoff_base_ms` (500 ms, exponencial + jitter); `scheduler.retry.max_attempts` (3). | Após esgotar tentativas, `FAILED` torna-se terminal. |
| Todos (idempotência) | `scheduler.idempotency.ttl_hours` (24h, mínimo RFC-0001 §5.5). | Resultado repetido sem nova efetivação (Fluxo 7). |
