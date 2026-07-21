---
Documento: Design Brief (Fonte Única de Verdade Interna)
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0095, ADR-0096, ADR-0097 (a propor); ADR-0001..ADR-0010 (herdadas)
RFCs relacionados: RFC-0001 (baseline); RFC-0090, RFC-0091 (a propor)
Depende de: 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster; consome contratos de 000-Vision, 001-Architecture, 040-Glossary, 003-RFC/RFC-0001
---

# 009-Scheduler — Design Brief (Fonte Única de Verdade)

> **Autoridade.** Este documento é a **fonte única de verdade interna** do módulo
> 009-Scheduler. Os 26 documentos obrigatórios (ver `README.md` e
> `../_templates/MODULE_TEMPLATE.md`) DEVEM ser derivados deste brief e **NÃO
> PODEM contradizê-lo**. Onde este brief for silencioso, prevalecem a
> `../001-Architecture/Architecture.md` e a `../003-RFC/RFC-0001-Architecture-Baseline.md`.
> Este brief NÃO redefine contratos centrais (URN, envelope de evento CloudEvents,
> envelope de erro RFC 7807, `Idempotency-Key`, correlação `traceparent`/`X-AIOS-Tenant`);
> ele os **referencia** e os **especializa** para o domínio de scheduling.

> **Termos normativos** (DEVE, NÃO DEVE, DEVERIA, NÃO DEVERIA, PODE) seguem RFC 2119 / RFC 8174.
> **Domínio de erro/subject deste módulo:** `SCHED` (erros `AIOS-SCHED-NNNN`) e domínio de
> subject `task` / `scheduler` conforme §6.

---

## 0. Posição no AIOS e contrato de fronteira

O Scheduler é o **scheduler cognitivo** do plano de controle (análogo ao CFS do
Linux, mas consciente de custo e qualidade). Ele responde à pergunta canônica:
**"qual tarefa/agente executa, quando, onde (placement), com qual prioridade e a
que custo?"**, sob restrições de cota, orçamento, SLA e capacidade. Ele é o
**guardião da admissão** e da **backpressure** do sistema.

```
   Kernel(006) ── "pede slot para Task" (gRPC) ─────────▶ Scheduler(009)
   Policy(022) ◀── PDP "pode admitir/preemptar?" ───────  Scheduler(009)
   Cost-Optimizer(026) ◀── orçamento/estimativa custo ──  Scheduler(009)
   Cluster(027) ──── inventário de capacidade/shards ───▶ Scheduler(009)
   Scheduler(009) ── "SchedulingDecision (place/preempt)" ▶ Runtime Supervisor(007)
   Scheduler(009) ── eventos aios.<tenant>.task.execution.* ▶ NATS/JetStream
```

O Scheduler **decide**; o **Runtime Supervisor (007)** materializa a decisão
(spawn/suspend/kill de Agent Runtime). O Scheduler NÃO executa agentes, NÃO fala
com LLM (isso é do 017), NÃO cobra (isso é do 026), NÃO aplica sandbox (021/007).

---

## 1. Responsabilidades e Não-Responsabilidades

### 1.1 Responsabilidades (o que o módulo DEVE fazer)

| # | Responsabilidade |
|---|------------------|
| R-01 | **Admission Control**: aceitar ou rejeitar cada `SchedulingRequest` com base em cotas, orçamento (consulta 026), capacidade de shard/nó (consulta 027) e política (PDP 022). |
| R-02 | **Priority/Cost Policy Evaluation**: computar o score multiobjetivo `C = α·L + β·$ + γ·E` e a prioridade efetiva de cada tarefa, respeitando `Q ≥ Q_min`, SLA e orçamento. |
| R-03 | **Placement**: escolher o shard/nó/runtime-pool de destino via sharding determinístico `shard = hash(tenant_id, agent_id) mod N`, respeitando afinidade, anti-afinidade e capacidade residual. |
| R-04 | **Preemption**: suspender tarefas/agentes de menor prioridade efetiva para liberar recurso a tarefas de maior prioridade, de forma segura (preempção compensável, com aviso ao Runtime). |
| R-05 | **Backpressure**: propagar pressão de saturação para produtores (Kernel/Gateway) via limites de admissão e sinais de fila, integrado ao JetStream. |
| R-06 | **Filas de prioridade**: manter filas de execução por (tenant, classe, shard) em Redis ZSET com score = prioridade efetiva. |
| R-07 | **Emissão de eventos de ciclo de decisão**: publicar `aios.<tenant>.task.execution.{admitted,scheduled,preempted,rejected,resumed,completed,failed}` conforme envelope RFC-0001 §5.2. |
| R-08 | **Idempotência de decisão**: garantir que uma mesma `SchedulingRequest` (mesma `Idempotency-Key` / `request_id`) produza no máximo uma admissão efetiva. |
| R-09 | **Fairness e anti-starvation**: garantir progresso mínimo (`Q_min` de vazão) a todo tenant/classe, evitando *starvation* por aging. |
| R-10 | **Registro de proveniência da decisão**: persistir a decisão (features, score, política aplicada, versão) para auditoria (025) e treino futuro (023). |
| R-11 | **Reavaliação e re-scheduling**: reprogramar tarefas em fila diante de mudança de capacidade, preempção externa ou expiração de deadline. |
| R-12 | **Enforcement de SLA/deadline**: rejeitar ou escalonar por deadline (EDF-aware) tarefas com prazo, respeitando classe de serviço. |
| R-13 | **Estabilidade da interface v1→v2**: expor a política de decisão atrás de uma interface `ISchedulingPolicy` que permita trocar heurística (v1) por política aprendida RL (v2) sem mudar API/eventos. |

### 1.2 Não-Responsabilidades (o que o módulo NÃO DEVE fazer)

| # | Não-Responsabilidade | Dono correto |
|---|----------------------|--------------|
| N-01 | Executar o loop cognitivo/ReAct do agente. | 007-Agent-Runtime |
| N-02 | Criar/suspender/matar processos de runtime (materializar placement). | 007 (Runtime Supervisor) |
| N-03 | Definir/avaliar políticas RBAC/ABAC (é PEP, consulta PDP; não é PDP). | 022-Policy |
| N-04 | Escolher o modelo LLM ou falar com provedores de inferência. | 017-Model-Router |
| N-05 | Calcular preço/faturar; ser fonte da verdade de orçamento. | 026-Cost-Optimizer |
| N-06 | Persistir memória cognitiva, contexto ou conhecimento. | 010/011/018/019 |
| N-07 | Prover HA/replicação de NATS/PG/Redis; failover de infraestrutura. | 027-Cluster |
| N-08 | Gerenciar cotas de token dentro da inferência (rate de tokens do LLM). | 026/017 |
| N-09 | Trilha de auditoria imutável (apenas emite eventos que 025 consome). | 025-Audit |
| N-10 | Decidir *o que* a tarefa faz (plano); apenas *quando/onde/prioridade*. | 012-Planning / 014-Workflow |

---

## 2. Decomposição em Componentes Internos

### 2.1 Componentes (PascalCase · responsabilidade · dependências)

| Componente | Responsabilidade | Depende de |
|------------|------------------|------------|
| `SchedulingGateway` | Superfície de entrada (gRPC/REST interno). Recebe `SchedulingRequest`, valida envelope, correlação e `Idempotency-Key`; roteia ao pipeline. É o PEP do módulo. | `PolicyClient`, `IdempotencyStore`, `AdmissionController` |
| `AdmissionController` | Decide admitir/rejeitar/enfileirar: verifica cotas (Redis), orçamento (026), capacidade (`CapacityProvider`), backpressure e política. Emite `admitted`/`rejected`. | `QuotaLedger`, `BudgetClient`, `CapacityProvider`, `PolicyClient`, `BackpressureController` |
| `PriorityCostEvaluator` | Computa `C = α·L + β·$ + γ·E` e prioridade efetiva; aplica classe de serviço, deadline (EDF), aging (anti-starvation). Encapsula `ISchedulingPolicy` (v1 heurística / v2 RL). | `PolicyWeightsProvider`, `CostEstimator`, `ISchedulingPolicy` |
| `ISchedulingPolicy` | Interface estável de decisão. `HeuristicPolicy` (v1) e `LearnedPolicy` (v2, RL) implementam-na sem alterar contrato externo. | `FeatureExtractor` |
| `FeatureExtractor` | Extrai o vetor de features da decisão (carga, fila, custo estimado, folga de SLA, histórico) para política e proveniência/treino. | `QuotaLedger`, `CapacityProvider`, `CostEstimator` |
| `PlacementEngine` | Escolhe shard/nó/runtime-pool via `hash(tenant,agent) mod N`; aplica afinidade/anti-afinidade e capacidade residual; produz `PlacementTarget`. | `ShardRouter`, `CapacityProvider` |
| `ShardRouter` | Função determinística de sharding e mapa shard→nó (rebalanceamento consistente sob mudança de N). | `CapacityProvider` (topologia de 027) |
| `PreemptionManager` | Seleciona vítimas de preempção por prioridade efetiva; orquestra suspensão segura (aviso + grace period); emite `preempted`. | `PriorityQueueStore`, `RuntimeSupervisorClient`, `PolicyClient` |
| `PriorityQueueStore` | Filas de prioridade por (tenant, classe, shard) em Redis ZSET; enqueue/dequeue/peek/aging; leases de dispatch. | Redis |
| `DispatchLoop` | Laço de despacho: retira o topo elegível das filas, revalida capacidade, emite `scheduled` e envia `SchedulingDecision` ao Runtime Supervisor. | `PriorityQueueStore`, `PlacementEngine`, `RuntimeSupervisorClient`, `DecisionJournal` |
| `BackpressureController` | Calcula estado de saturação (fila, lag JetStream, capacidade) e expõe sinal de admissão (aceitar/desacelerar/rejeitar) + `Retry-After`. | `CapacityProvider`, `PriorityQueueStore`, JetStream |
| `QuotaLedger` | Contabiliza consumo de concorrência/slots por tenant/agente/classe em Redis (contadores atômicos com TTL); *reservation* durante admissão. | Redis |
| `CapacityProvider` | Visão de capacidade (slots livres por shard/nó/pool), atualizada por heartbeats do Runtime Supervisor e inventário do 027. | `RuntimeSupervisorClient`, 027 |
| `CostEstimator` | Estima custo/latência/energia esperados por tarefa (chama 026 / cache local); alimenta `C`. | `BudgetClient` (026) |
| `PolicyClient` | Cliente PDP: consulta 022 para autorização de admissão/preempção (*default deny*). | 022-Policy |
| `BudgetClient` | Cliente de orçamento/estimativa junto ao 026. | 026-Cost-Optimizer |
| `RuntimeSupervisorClient` | Cliente gRPC do Runtime Supervisor (007): envia decisão, recebe heartbeats/término. | 007 |
| `IdempotencyStore` | Persiste resultado de decisão por `Idempotency-Key`/`request_id` ≥ 24h (RFC-0001 §5.5). | Redis + PostgreSQL |
| `DecisionJournal` | Outbox transacional: grava a decisão (proveniência) em PostgreSQL e publica evento atômica­mente. | PostgreSQL, NATS/JetStream |
| `EventPublisher` | Serializa e publica eventos no envelope CloudEvents (RFC-0001 §5.2) nos subjects canônicos. | NATS/JetStream, `DecisionJournal` |
| `SchedulerTelemetry` | Instrumentação OTel (traces/metrics/logs Serilog→Seq), golden signals e SLIs. | OpenTelemetry |
| `ReconciliationWorker` | Reconcilia estado (filas × runtime real), expira leases órfãos, faz re-scheduling e detecta decisões perdidas. | `PriorityQueueStore`, `CapacityProvider`, `DecisionJournal` |

### 2.2 Diagrama de Componentes (ASCII)

```
                             SCHEDULER SERVICE (009 · .NET 10)
  ┌────────────────────────────────────────────────────────────────────────────────┐
  │                                                                                  │
  │  gRPC/REST ── (Kernel 006 / Gateway)                                             │
  │      │                                                                           │
  │      ▼                                                                           │
  │  ┌──────────────────┐    idem?   ┌──────────────────┐                            │
  │  │ SchedulingGateway│──────────▶ │ IdempotencyStore │ (Redis+PG)                 │
  │  │      (PEP)       │            └──────────────────┘                            │
  │  └────────┬─────────┘                                                            │
  │           │ SchedulingRequest                                                    │
  │           ▼                                                                       │
  │  ┌───────────────────────┐   PDP    ┌─────────────┐                              │
  │  │  AdmissionController   │────────▶ │ PolicyClient│──▶ Policy Engine (022)       │
  │  │                        │          └─────────────┘                              │
  │  │  ┌──────────────────┐  │  budget  ┌─────────────┐                              │
  │  │  │ QuotaLedger(Redis)│ │────────▶ │ BudgetClient│──▶ Cost-Optimizer (026)      │
  │  │  └──────────────────┘  │          └─────────────┘                              │
  │  │  ┌──────────────────┐  │ capacity ┌─────────────────┐                          │
  │  │  │BackpressureCtrl   │ │◀───────▶ │ CapacityProvider│──▶ Cluster(027)/RTSup(007)│
  │  │  └──────────────────┘  │          └─────────────────┘                          │
  │  └─────────┬──────────────┘                                                       │
  │            │ admitido → avaliar                                                   │
  │            ▼                                                                       │
  │  ┌─────────────────────────┐    ┌──────────────────┐   ┌───────────────┐         │
  │  │  PriorityCostEvaluator  │──▶ │ FeatureExtractor │   │ CostEstimator │         │
  │  │  C=α·L+β·$+γ·E, EDF,aging│    └──────────────────┘   └───────────────┘         │
  │  │  ┌────────────────────┐ │                                                      │
  │  │  │ ISchedulingPolicy  │ │  v1: HeuristicPolicy   v2: LearnedPolicy(RL)         │
  │  │  └────────────────────┘ │                                                      │
  │  └─────────┬───────────────┘                                                      │
  │            │ prioridade efetiva                                                   │
  │            ▼                                                                       │
  │  ┌──────────────────────┐        ┌──────────────────┐                            │
  │  │  PriorityQueueStore   │◀──────▶│ PreemptionManager│──▶ RuntimeSupervisorClient │
  │  │  (Redis ZSET/shard)   │        └──────────────────┘         │ (007)            │
  │  └─────────┬────────────┘                                      │                  │
  │            │ topo elegível                                     ▼                  │
  │            ▼                          ┌──────────────────┐  Runtime Supervisor    │
  │  ┌──────────────────┐   place         │  PlacementEngine │  (spawn/suspend/kill)  │
  │  │   DispatchLoop    │───────────────▶│  + ShardRouter   │                        │
  │  └───────┬──────────┘                 └──────────────────┘                        │
  │          │ decisão                                                                 │
  │          ▼                                                                         │
  │  ┌──────────────────┐  outbox  ┌──────────────┐   ┌────────────────┐              │
  │  │  DecisionJournal │────────▶ │EventPublisher│──▶│ NATS/JetStream │              │
  │  │  (PostgreSQL)    │          └──────────────┘   └────────────────┘              │
  │  └──────────────────┘                                                              │
  │                                                                                    │
  │  ┌────────────────────┐        ┌───────────────────┐                              │
  │  │ ReconciliationWorker│       │ SchedulerTelemetry│ (OTel/Prometheus/Seq)        │
  │  └────────────────────┘        └───────────────────┘                              │
  └────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Modelo de Dados / Entidades Canônico

Alinhado a RFC-0001 §5.1 (URN `urn:aios:<tenant>:<tipo>:<id>`, IDs ULID). Todo
recurso multi-tenant carrega `tenant_id` (slug) obrigatório. Chaves primárias são
ULID. Persistência quente em Redis; fonte da verdade de decisão em PostgreSQL.

### 3.1 `SchedulingRequest` (entrada canônica de escalonamento)

| Campo | Tipo | Chave/Constraint | Notas |
|-------|------|------------------|-------|
| `request_id` | ULID | PK | Idempotência (casado com `Idempotency-Key`). |
| `tenant_id` | string (slug) | NOT NULL, RLS | Fronteira de isolamento. |
| `task_urn` | URN | NOT NULL | `urn:aios:<tenant>:task:<ulid>`. |
| `agent_urn` | URN | NOT NULL | `urn:aios:<tenant>:agent:<ulid>`. |
| `service_class` | enum(`INTERACTIVE`,`BATCH`,`BACKGROUND`,`SYSTEM`) | NOT NULL | Classe de serviço → pesos/preempção. |
| `priority_hint` | int (0–100) | default 50 | Sugestão do chamador (não vinculante). |
| `deadline_at` | timestamptz | NULL | Se presente, ativa lógica EDF. |
| `est_cost_usd` | numeric(12,6) | ≥ 0 | Preenchido por `CostEstimator` (026). |
| `est_latency_ms` | int | ≥ 0 | Estimativa de latência de execução. |
| `est_tokens` | bigint | ≥ 0 | Estimativa de tokens (para orçamento). |
| `quality_floor` | numeric(4,3) | 0–1 | `Q_min` requerido. |
| `affinity` | jsonb | NULL | Dicas de afinidade/anti-afinidade de placement. |
| `payload_ref` | URN/URI | NULL | Ponteiro para payload (MinIO), nunca inline se sensível. |
| `submitted_at` | timestamptz | NOT NULL | Base do aging. |
| `traceparent` | string | NOT NULL | Correlação W3C (RFC-0001 §5.6). |

### 3.2 `SchedulingDecision` (proveniência da decisão — event-sourced)

| Campo | Tipo | Chave/Constraint | Notas |
|-------|------|------------------|-------|
| `decision_id` | ULID | PK | Único por decisão. |
| `request_id` | ULID | FK → SchedulingRequest | 1 request → N decisões (admissão, re-schedule, preempção). |
| `tenant_id` | string | NOT NULL, RLS | — |
| `outcome` | enum(`ADMITTED`,`SCHEDULED`,`REJECTED`,`PREEMPTED`,`RESUMED`,`DEFERRED`) | NOT NULL | Ver §4. |
| `effective_priority` | int | NOT NULL | Prioridade computada (score → ordem ZSET). |
| `cost_score` | numeric(12,6) | NOT NULL | Valor de `C = α·L + β·$ + γ·E`. |
| `alpha` `beta` `gamma` | numeric(6,4) | NOT NULL | Pesos aplicados (política/tenant). |
| `placement_target` | jsonb | NULL | `{shard, node_id, runtime_pool}`. |
| `policy_id` | string | NOT NULL | Identidade da política/versão usada. |
| `policy_version` | string | NOT NULL | `heuristic@1` / `rl@<hash>` (interface estável §1 R-13). |
| `feature_vector` | jsonb | NULL | Features (para auditoria/treino RL). |
| `reason_code` | string | NULL | Ex.: `AIOS-SCHED-0001` quando rejeição. |
| `decided_at` | timestamptz | NOT NULL | Instante da decisão. |
| `latency_decision_ms` | numeric(8,3) | NOT NULL | SLI de p99 ≤ 20 ms. |

### 3.3 `QueueEntry` (Redis ZSET — estado quente)

| Campo | Tipo | Notas |
|-------|------|-------|
| Chave ZSET | `sched:{tenant}:{class}:{shard}` | Uma ZSET por (tenant, classe, shard). |
| Membro | `task_urn` | Elemento da fila. |
| Score | double | `effective_priority` (menor = mais urgente); inclui componente de aging. |
| Hash lateral | `sched:meta:{task_urn}` | `{request_id, deadline_at, submitted_at, lease_owner, lease_until}`. |

### 3.4 `SlotReservation` / `QuotaCounter` (Redis)

| Campo | Tipo | Notas |
|-------|------|-------|
| `sched:quota:{tenant}:concurrency` | int (atômico) | Slots concorrentes em uso vs. `max_concurrency`. |
| `sched:quota:{tenant}:{class}:inflight` | int | Concorrência por classe. |
| `sched:reservation:{request_id}` | hash + TTL | Reserva durante admissão (liberada em confirm/abort). |

### 3.5 `CapacitySnapshot` (Redis + memória)

| Campo | Tipo | Notas |
|-------|------|-------|
| `node_id` | string | Identidade do nó/pool (de 027). |
| `shard` | int | `0..N-1`. |
| `free_slots` | int | Slots livres reportados por heartbeat. |
| `pressure` | numeric(4,3) | 0–1, insumo do BackpressureController. |
| `updated_at` | timestamptz | TTL de frescor (heartbeat). |

### 3.6 `IdempotencyRecord`

| Campo | Tipo | Notas |
|-------|------|-------|
| `idempotency_key` | string | PK; ULID/UUID do cliente (RFC-0001 §5.5). |
| `request_id` | ULID | Vínculo. |
| `response_snapshot` | jsonb | Resultado a repetir. |
| `expires_at` | timestamptz | ≥ 24h. |

**Regras invariantes:** (i) todo registro carrega `tenant_id` com RLS ativo;
(ii) `SchedulingDecision` é append-only (event sourcing, §001 arquitetura);
(iii) `task_urn`/`agent_urn` DEVEM ser URNs válidas RFC-0001; (iv) uma
`SchedulingRequest` NÃO DEVE resultar em mais de uma decisão `SCHEDULED` ativa
simultânea (garantido por lease + idempotência).

---

## 4. Máquina de Estados Canônica

### 4.1 Estados da Tarefa sob a ótica do Scheduler

`RECEIVED → ADMITTED → QUEUED → SCHEDULED → RUNNING → COMPLETED | FAILED`
com ramos `REJECTED`, `PREEMPTED`, `DEFERRED`, `EXPIRED`, `CANCELLED`.

Nota: `RUNNING/COMPLETED/FAILED` refletem sinais do Runtime (007); o Scheduler
os observa para liberar slots e fechar contabilidade — a autoridade de execução
é do 007. Estados **terminais**: `COMPLETED`, `FAILED`, `REJECTED`, `EXPIRED`,
`CANCELLED`.

### 4.2 Diagrama de Estados (ASCII)

```
                        submit (SchedulingRequest)
                                   │
                                   ▼
                             ┌───────────┐
                             │ RECEIVED  │
                             └─────┬─────┘
              guarda: política+cota+orçamento+capacidade
                 admite │                         │ nega
                        ▼                         ▼
                  ┌───────────┐             ┌───────────┐
                  │ ADMITTED  │             │ REJECTED  │ (terminal)
                  └─────┬─────┘             └───────────┘
            enqueue (prioridade efetiva)
                        ▼
                  ┌───────────┐  deadline vencido / cancel
                  │  QUEUED    │───────────────┐───────────────┐
                  └─────┬─────┘                │               │
        dispatch(topo, capacidade ok)          ▼               ▼
                        ▼                 ┌───────────┐   ┌───────────┐
                  ┌───────────┐           │  EXPIRED  │   │ CANCELLED │ (terminais)
                  │ SCHEDULED │           └───────────┘   └───────────┘
                  └─────┬─────┘
        Runtime confirma start │        preempção (vítima escolhida)
                        ▼      └───────────────┐
                  ┌───────────┐                ▼
                  │  RUNNING   │          ┌───────────┐   resume (capacidade/prioridade)
                  └──┬─────┬───┘          │ PREEMPTED │──────────────┐
      completa │     │     │ falha        └─────┬─────┘              │
               ▼     │     ▼                    │ requeue            ▼
        ┌───────────┐│┌───────────┐             └──────────────▶ (QUEUED)
        │ COMPLETED ││ │  FAILED   │
        └───────────┘ └───────────┘ (terminais)
                         │ retry policy (backoff, ≤ maxRetries)
                         └──────────────▶ (QUEUED)  [se retriável]

  DEFERRED: de QUEUED sob backpressure alto → aguarda janela; re-entra em QUEUED.
```

### 4.3 Tabela de Transições, Gatilhos e Guardas

| De | Para | Gatilho | Guarda (DEVE ser verdadeira) | Ação/Evento |
|----|------|---------|------------------------------|-------------|
| RECEIVED | ADMITTED | `submit` | PDP=allow ∧ cota livre ∧ orçamento ≥ custo est. ∧ capacidade ≥ 1 ∧ backpressure≠reject | reserva slot; emite `admitted` |
| RECEIVED | REJECTED | `submit` | qualquer guarda de admissão falha | libera reserva; erro `AIOS-SCHED-*`; emite `rejected` |
| ADMITTED | QUEUED | `enqueue` | prioridade efetiva computada | ZSET add; emite (interno) enqueue |
| QUEUED | SCHEDULED | `dispatch` | topo elegível ∧ capacidade shard ≥ 1 ∧ lease adquirido | envia `SchedulingDecision`; emite `scheduled` |
| QUEUED | DEFERRED | `backpressure` | pressão ≥ limiar_defer | agenda re-tentativa; emite `deferred` (opcional) |
| QUEUED | EXPIRED | `deadline_tick` | `now > deadline_at` | remove da fila; emite `rejected`(reason=deadline) |
| QUEUED / SCHEDULED | CANCELLED | `cancel` | request cancelável | libera slot; emite `rejected`(reason=cancelled) |
| SCHEDULED | RUNNING | `runtime.started` | Runtime confirma boot | marca running |
| SCHEDULED/RUNNING | PREEMPTED | `preempt` | ∃ tarefa maior prioridade ∧ PDP=allow ∧ grace respeitado | sinaliza suspend; emite `preempted` |
| PREEMPTED | QUEUED | `requeue/resume` | recurso/prioridade permite | re-enqueue com aging preservado |
| RUNNING | COMPLETED | `runtime.completed` | — | libera slot; emite `completed` |
| RUNNING | FAILED | `runtime.failed` | — | libera slot; emite `failed` |
| FAILED | QUEUED | `retry` | retriável ∧ tentativas < maxRetries ∧ backoff decorrido | re-enqueue |

---

## 5. Superfície de API (REST + gRPC)

### 5.1 Princípios

- Interno preferencialmente **gRPC** (`aios.scheduler.v1`); externo/admin via
  **REST** `/v1/scheduler/...` sob o Gateway YARP.
- Toda mutação DEVE aceitar `Idempotency-Key`; toda chamada DEVE propagar
  `traceparent` e `X-AIOS-Tenant` (RFC-0001 §5.6). Erros no envelope RFC 7807
  (RFC-0001 §5.4) com `code = AIOS-SCHED-<NNNN>`.

### 5.2 gRPC — serviço `aios.scheduler.v1.SchedulerService`

| Método | Descrição | Idempotente |
|--------|-----------|-------------|
| `Submit(SchedulingRequest) → SchedulingDecision` | Admissão + enfileiramento; caminho quente (SLO p99 ≤ 20 ms). | Sim (por `request_id`) |
| `Cancel(CancelRequest) → Ack` | Cancela tarefa em QUEUED/SCHEDULED. | Sim |
| `Preempt(PreemptRequest) → PreemptResult` | Preempção dirigida (uso administrativo/kernel). | Sim |
| `GetDecision(request_id) → SchedulingDecision` | Consulta proveniência. | Leitura |
| `ReportRuntimeSignal(RuntimeSignal) → Ack` | Runtime informa started/completed/failed/heartbeat. | Sim (por signal id) |
| `StreamQueueState(QueueFilter) → stream QueueStat` | Observabilidade de filas. | Leitura |

### 5.3 REST (admin/observabilidade)

| Método | Rota | Descrição |
|--------|------|-----------|
| `POST` | `/v1/scheduler/requests` | Submeter `SchedulingRequest` (equivalente a `Submit`). |
| `GET` | `/v1/scheduler/requests/{request_id}` | Obter decisão/estado. |
| `DELETE` | `/v1/scheduler/requests/{request_id}` | Cancelar. |
| `GET` | `/v1/scheduler/queues` | Listar filas por tenant/classe/shard (métricas). |
| `POST` | `/v1/scheduler/preemptions` | Preempção dirigida (RBAC: role scheduler-admin). |
| `GET` | `/v1/scheduler/policy` | Política ativa (id/versão/pesos α,β,γ) por tenant. |
| `PUT` | `/v1/scheduler/policy` | Atualizar pesos/classe (governado por 022). |
| `GET` | `/v1/scheduler/capacity` | Snapshot de capacidade/pressão (backpressure). |

### 5.4 Catálogo de Erros `AIOS-SCHED-<NNNN>` (faixa reservada 0001–0099)

| Código | HTTP | `retriable` | Significado |
|--------|------|-------------|-------------|
| `AIOS-SCHED-0001` | 429 | true | Cota de concorrência do tenant excedida. |
| `AIOS-SCHED-0002` | 429 | true | Orçamento insuficiente para custo estimado (026). |
| `AIOS-SCHED-0003` | 503 | true | Backpressure: sistema saturado, sem capacidade. |
| `AIOS-SCHED-0004` | 403 | false | Política negou admissão/preempção (PDP 022). |
| `AIOS-SCHED-0005` | 400 | false | `SchedulingRequest` inválida (URN/campos). |
| `AIOS-SCHED-0006` | 409 | false | Conflito de idempotência (chave reusada com payload divergente). |
| `AIOS-SCHED-0007` | 422 | false | `quality_floor` inatingível sob restrições. |
| `AIOS-SCHED-0008` | 410 | false | Deadline já vencido na admissão (EXPIRED). |
| `AIOS-SCHED-0009` | 404 | false | `request_id`/decisão não encontrada. |
| `AIOS-SCHED-0010` | 503 | true | Nenhum shard/nó elegível para placement. |
| `AIOS-SCHED-0011` | 409 | false | Tarefa em estado terminal; operação não aplicável. |
| `AIOS-SCHED-0012` | 500 | true | Falha interna de decisão (fallback aplicado). |

---

## 6. Catálogo de Eventos NATS

Subjects seguem RFC-0001 §5.3 `aios.<tenant>.<dominio>.<entidade>.<acao>`. Domínio
primário `task`, entidade `execution`. Envelope CloudEvents (RFC-0001 §5.2).
Semântica **at-least-once** + consumidores idempotentes (dedup por `event.id`).
Publicação via `DecisionJournal` (outbox transacional) em JetStream (durável).

### 6.1 Emitidos (produtor = Scheduler)

| Subject | Ação | Quando | `data` (resumo) |
|---------|------|--------|-----------------|
| `aios.<tenant>.task.execution.admitted` | admitted | Admissão aceita | `{taskUrn, decisionId, effectivePriority}` |
| `aios.<tenant>.task.execution.rejected` | rejected | Admissão negada / deadline / cancel | `{taskUrn, reasonCode, retriable}` |
| `aios.<tenant>.task.execution.scheduled` | scheduled | Despacho ao Runtime Supervisor | `{taskUrn, placement:{shard,node,pool}, decisionId}` |
| `aios.<tenant>.task.execution.preempted` | preempted | Vítima preemptada | `{taskUrn, byTaskUrn, gracePeriodMs}` |
| `aios.<tenant>.task.execution.resumed` | resumed | Retomada pós-preempção/backpressure | `{taskUrn, decisionId}` |
| `aios.<tenant>.task.execution.completed` | completed | Runtime concluiu (observado) | `{taskUrn, costUsd, tokens, latencyMs}` |
| `aios.<tenant>.task.execution.failed` | failed | Runtime falhou (observado) | `{taskUrn, errorCode, willRetry}` |
| `aios.<tenant>.scheduler.backpressure.changed` | changed | Estado de saturação muda | `{level, retryAfterMs, shard}` |
| `aios.<tenant>.scheduler.decision.recorded` | recorded | Proveniência gravada (para 025/023) | `{decisionId, policyVersion, featureVectorRef}` |

### 6.2 Consumidos (consumidor = Scheduler)

| Subject | Produtor | Uso no Scheduler |
|---------|----------|------------------|
| `aios.<tenant>.agent.lifecycle.spawned` | Kernel/007 | Confirma materialização de placement → SCHEDULED→RUNNING. |
| `aios.<tenant>.agent.lifecycle.suspended` | 007 | Libera slot; confirma preempção. |
| `aios.<tenant>.agent.lifecycle.terminated` | 007 | Fecha contabilidade de slot. |
| `aios.<tenant>.cost.budget.updated` | 026 | Atualiza orçamento disponível para admissão. |
| `aios.<tenant>.cluster.capacity.changed` | 027 | Atualiza `CapacityProvider`/`ShardRouter` (N muda). |
| `aios.<tenant>.policy.decision.updated` | 022 | Invalida cache de decisões de política. |

---

## 7. Requisitos Funcionais e Não-Funcionais

### 7.1 Requisitos Funcionais (extrato canônico)

| ID | Requisito | Prioridade |
|----|-----------|------------|
| FR-001 | O Scheduler DEVE admitir/rejeitar cada `SchedulingRequest` avaliando cota, orçamento, capacidade e PDP. | MUST |
| FR-002 | O Scheduler DEVE computar prioridade efetiva a partir de `C = α·L + β·$ + γ·E` sob `Q ≥ Q_min`, SLA e orçamento. | MUST |
| FR-003 | O Scheduler DEVE realizar placement determinístico `hash(tenant,agent) mod N` com afinidade/anti-afinidade. | MUST |
| FR-004 | O Scheduler DEVE preemptar tarefas de menor prioridade com grace period e autorização PDP. | MUST |
| FR-005 | O Scheduler DEVE aplicar backpressure e sinalizar `Retry-After`/`AIOS-SCHED-0003` sob saturação. | MUST |
| FR-006 | O Scheduler DEVE manter filas de prioridade em Redis ZSET por (tenant, classe, shard). | MUST |
| FR-007 | O Scheduler DEVE emitir todos os eventos de §6.1 no envelope RFC-0001. | MUST |
| FR-008 | O Scheduler DEVE ser idempotente por `Idempotency-Key`/`request_id` (RFC-0001 §5.5). | MUST |
| FR-009 | O Scheduler DEVE garantir anti-starvation via aging, honrando vazão mínima por tenant/classe. | MUST |
| FR-010 | O Scheduler DEVE registrar proveniência de decisão (features, pesos, política/versão). | MUST |
| FR-011 | O Scheduler DEVE re-escalonar diante de mudança de capacidade, preempção ou expiração. | MUST |
| FR-012 | O Scheduler DEVE suportar EDF para tarefas com `deadline_at`. | SHOULD |
| FR-013 | O Scheduler DEVE expor a decisão atrás de `ISchedulingPolicy` (v1 heurística → v2 RL) sem mudar API/eventos. | MUST |
| FR-014 | O Scheduler DEVE reconciliar filas × runtime real e expirar leases órfãos. | MUST |
| FR-015 | O Scheduler DEVERIA suportar cancelamento de tarefas em QUEUED/SCHEDULED. | SHOULD |

### 7.2 Requisitos Não-Funcionais (com metas numéricas, SLO/SLI)

| ID | Atributo | Meta (SLO) | SLI / Método |
|----|----------|-----------|--------------|
| NFR-001 | Latência de decisão | **p99 ≤ 20 ms**, p50 ≤ 5 ms (caminho `Submit`, sem I/O bloqueante externo no caminho quente) | histograma `aios_scheduler_decision_latency_ms` |
| NFR-002 | Throughput | ≥ **50.000 decisões/s** por réplica; escala linear por réplica/shard | load test sustentado |
| NFR-003 | Disponibilidade | ≥ **99,95%** do endpoint de scheduling (control plane) | uptime probe / error budget |
| NFR-004 | Escalabilidade | 1 → **10⁶⁺** agentes; crescimento sub-linear de coordenação via sharding | benchmark de escala (`Scalability.md`) |
| NFR-005 | Corretude de admissão | **0** overcommit de slot além de `max_concurrency` (reservas atômicas) | teste de concorrência/chaos |
| NFR-006 | Fairness | Todo tenant ativo obtém ≥ `Q_min` (default 1%) de vazão em janela de 60 s | métrica de fairness por tenant |
| NFR-007 | Durabilidade da decisão | RPO ≤ 5 min; decisões críticas em outbox durável (JetStream+PG) | teste de recuperação |
| NFR-008 | Recuperação | RTO ≤ 15 min após falha de réplica; sem perda de trabalho admitido | drill de failover |
| NFR-009 | Custo de preempção | overhead de preempção ≤ 50 ms de decisão; grace period configurável default 2 s | métrica `aios_scheduler_preemption_ms` |
| NFR-010 | Observabilidade | 100% das decisões com trace/span e evento de proveniência | cobertura OTel |
| NFR-011 | Idempotência | 0 admissões duplicadas sob retry/at-least-once | teste de duplicação |
| NFR-012 | Segurança | 100% das mutações passam por PEP→PDP (*default deny*) | auditoria 025 |

---

## 8. Chaves de Configuração Principais

| Chave | Default | Faixa | Escopo | Recarregável |
|-------|---------|-------|--------|--------------|
| `scheduler.shards.count` (N) | 64 | 1–4096 | global/cluster | Não (rebalanceamento coordenado com 027) |
| `scheduler.decision.timeout_ms` | 15 | 1–100 | global | Sim |
| `scheduler.weights.alpha` (latência) | 0.5 | 0–1 | tenant | Sim |
| `scheduler.weights.beta` ($custo) | 0.3 | 0–1 | tenant | Sim |
| `scheduler.weights.gamma` (energia) | 0.2 | 0–1 | tenant | Sim |
| `scheduler.quality_floor` (Q_min) | 0.7 | 0–1 | tenant/classe | Sim |
| `scheduler.max_concurrency.per_tenant` | 1000 | 1–10⁶ | tenant | Sim |
| `scheduler.max_concurrency.per_class.INTERACTIVE` | 400 | 0–10⁶ | tenant | Sim |
| `scheduler.fairness.min_share` (Q_min vazão) | 0.01 | 0–1 | tenant | Sim |
| `scheduler.aging.factor_per_sec` | 1.0 | 0–100 | classe | Sim |
| `scheduler.aging.max_boost` | 40 | 0–100 | classe | Sim |
| `scheduler.preemption.enabled` | true | bool | global | Sim |
| `scheduler.preemption.grace_period_ms` | 2000 | 0–60000 | classe | Sim |
| `scheduler.backpressure.defer_threshold` | 0.80 | 0–1 | shard | Sim |
| `scheduler.backpressure.reject_threshold` | 0.95 | 0–1 | shard | Sim |
| `scheduler.backpressure.retry_after_ms` | 1500 | 100–60000 | global | Sim |
| `scheduler.deadline.edf_enabled` | true | bool | tenant | Sim |
| `scheduler.retry.max_attempts` | 3 | 0–10 | classe | Sim |
| `scheduler.retry.backoff_base_ms` | 500 | 50–60000 | classe | Sim |
| `scheduler.policy.engine` | `heuristic` | `heuristic`\|`rl` | global/tenant | Sim (interface estável §1 R-13) |
| `scheduler.idempotency.ttl_hours` | 24 | 24–168 | global | Sim |
| `scheduler.capacity.heartbeat_ttl_ms` | 5000 | 1000–60000 | global | Sim |
| `scheduler.lease.ttl_ms` | 10000 | 1000–120000 | global | Sim |
| `scheduler.redis.zset_prefix` | `sched` | string | global | Não |

Pesos DEVEM satisfazer `alpha + beta + gamma ≈ 1` (normalizados na leitura).
Alterações de pesos e cotas por tenant DEVEM ser autorizadas via 022.

---

## 9. Modos de Falha e Recuperação

| Modo de falha | Detecção | Estratégia de recuperação |
|---------------|----------|---------------------------|
| Redis indisponível (filas/cotas) | health probe / timeout | Fail-safe: admissão entra em modo conservador (rejeita novas com `AIOS-SCHED-0003`, retriável); leases em PG como fallback; sem overcommit. RPO das filas mitigado por re-derivação do outbox/PG. |
| PostgreSQL indisponível (outbox) | timeout | Buffer local + retry; se persistir, degrada para modo *read-only* de decisões e para admissões que exigem journaling; alerta. |
| NATS/JetStream indisponível | ack timeout / lag | Outbox transacional retém eventos; publicação re-tentada com backoff; consumidores idempotentes toleram duplicação. |
| Runtime Supervisor (007) não responde | heartbeat expira | `ReconciliationWorker` expira lease, marca SCHEDULED→QUEUED (re-schedule), libera slot; evita trabalho perdido (idempotência por `request_id`). |
| Cost-Optimizer (026) indisponível | timeout | Usa última estimativa cacheada + margem conservadora; se sem dado, aplica `AIOS-SCHED-0002` só a orçamentos no limite; degradação graciosa. |
| Policy Engine (022) indisponível | timeout | *Default deny* (RFC-0001 §5.8): admissão/preempção que exijam PDP são negadas (`AIOS-SCHED-0004`), preservando segurança. |
| Decisão duplicada (retry/at-least-once) | `IdempotencyStore` | Retorna snapshot da 1ª decisão; nenhuma reserva/admissão nova (NFR-011). |
| Overcommit por corrida | contador atômico Redis (Lua/`INCR` transacional) | Reserva atômica com CAS; rollback em falha; invariante NFR-005. |
| Starvation de tenant | métrica de fairness | Aging força boost até `max_boost`; garante `min_share`. |
| Lease órfão (worker morreu no dispatch) | TTL de lease | Reconciliação re-enfileira; idempotência evita dupla execução. |
| Loop de preempção (thrashing) | métrica de preempções/s | Histerese: cooldown por tarefa + anti-preempção de vítimas recém-agendadas. |

**Idempotência:** `Submit`, `Cancel`, `Preempt`, `ReportRuntimeSignal` são
idempotentes por chave. **Retries:** backoff exponencial com jitter, `max_attempts`
configurável. **DLQ:** eventos não processáveis vão para stream DLQ do JetStream
para replay. **RTO ≤ 15 min, RPO ≤ 5 min** (herdado de V-R10/NFR-007/008).

---

## 10. Escalabilidade e Concorrência

Estratégia central: **sharding determinístico + estado externalizado + caminho
quente sem I/O bloqueante + backpressure**.

| Dimensão | Estratégia |
|----------|-----------|
| Sharding | `shard = hash(tenant_id, agent_id) mod N` (`ShardRouter`). Filas ZSET particionadas por (tenant, classe, shard). N configurável; rebalanceamento coordenado com 027 (hashing consistente para minimizar migração). |
| Réplicas | Serviço stateless; escala horizontal por réplica. Cada réplica é *owner* de um subconjunto de shards (ownership via lease em Redis) para evitar contenção de dispatch. |
| Locks | Sem locks globais no caminho quente. Reservas de slot por contador atômico (script Lua no Redis). Ownership de shard por lease com TTL; renovação por heartbeat. |
| Concorrência de dispatch | `DispatchLoop` por shard (partição), sem cross-shard locks; ordenação por ZSET; leases de dispatch evitam dupla retirada. |
| Backpressure | `BackpressureController` combina profundidade de fila, lag JetStream e pressão de capacidade; níveis `accept < defer(0.80) < reject(0.95)`; propaga `Retry-After`. |
| Caminho quente | `Submit` evita I/O externo síncrono: cotas/capacidade em Redis (sub-ms), orçamento via cache (026), PDP com cache de curta duração; p99 ≤ 20 ms (NFR-001). |
| Cold agents | Tarefas de agentes suspensos entram em fila normalmente; materialização (spawn <250 ms) é do 007; Scheduler apenas garante slot e placement. |
| Rumo a 10⁶ | Múltiplos shards × réplicas; filas por tenant evitam interferência; fairness por `min_share`; comunicação seletiva (limite de fan-out) para evitar O(N²). |

```
   requests ─▶ [réplica A owner shards 0..31] ─▶ ZSET sched:{t}:{c}:{0..31} ─▶ dispatch
   requests ─▶ [réplica B owner shards 32..63]─▶ ZSET sched:{t}:{c}:{32..63}─▶ dispatch
                         ▲ ownership lease (Redis, TTL) ▲   backpressure feedback ◀── JetStream lag
```

---

## 11. ADRs e RFCs do Módulo (a propor)

Faixa de ADR reservada ao módulo 009: **ADR-0090 a ADR-0099** (regra 009×10).

| ADR | Título proposto |
|-----|-----------------|
| ADR-0090 | Modelo de admission control com reserva atômica de slots em Redis (fail-safe, sem overcommit). |
| ADR-0091 | Função de score multiobjetivo `C = α·L + β·$ + γ·E` e normalização de pesos por tenant/classe. |
| ADR-0092 | Sharding determinístico `hash(tenant,agent) mod N` com hashing consistente e rebalanceamento coordenado com 027. |
| ADR-0093 | Filas de prioridade em Redis ZSET com aging (anti-starvation) e leases de dispatch por shard. |
| ADR-0094 | Modelo de preempção compensável com grace period, histerese e cooldown (anti-thrashing). |
| ADR-0095 | Backpressure em três níveis (accept/defer/reject) integrado a JetStream + `Retry-After`. |
| ADR-0096 | Interface `ISchedulingPolicy` estável: heurística v1 → política aprendida RL v2 sem quebra de contrato. |
| ADR-0097 | Outbox transacional de decisão (event sourcing) para proveniência, auditoria (025) e treino (023). |

| RFC | Título proposto | Relação |
|-----|-----------------|---------|
| RFC-0001 | Architecture Baseline & Core Contracts | Herdada (URN, eventos, erros, idempotência). |
| RFC-0090 | Scheduling Decision Contract & Service-Class Model | Formaliza classes, score e eventos `task.execution.*`. |
| RFC-0091 | Learned Scheduling Policy (RL) Interface & Feature Contract | Contrato de features/telemetria para evolução v2 sem mudar interface. |

---

## 12. Decisões de Segurança

### 12.1 AuthN / AuthZ

- **AuthN:** chamadas externas autenticadas no Gateway (OAuth2/OIDC); interno via
  **mTLS** (RFC-0001 §6). Claims propagadas assinadas; `X-AIOS-Tenant` DEVE casar
  com o tenant autenticado — divergência é rejeitada (`AIOS-SCHED-0004`).
- **AuthZ:** o `SchedulingGateway` é **PEP**; toda admissão/preempção/mutação de
  política consulta o **PDP (022)** — *default deny* (RFC-0001 §5.8). Roles:
  `scheduler-submit` (Submit/Cancel), `scheduler-admin` (Preempt, PUT policy).
- **Isolamento multi-tenant:** `tenant_id` obrigatório em toda entidade, RLS no
  PostgreSQL, namespace NATS por tenant (`aios.<tenant>.*`), filas ZSET por tenant.

### 12.2 Threat Model STRIDE (resumo)

| Ameaça | Vetor no Scheduler | Mitigação |
|--------|--------------------|-----------|
| **S**poofing | Chamador forja tenant/agente | mTLS + OIDC; `X-AIOS-Tenant` validado contra claim; URNs verificadas. |
| **T**ampering | Alterar prioridade/pesos/fila | Mutação de política via PDP + auditoria; ZSET/quota só via serviço; RLS. |
| **R**epudiation | Negar ter submetido/preemptado | `DecisionJournal` append-only + eventos `decision.recorded` → 025 (imutável). |
| **I**nformation disclosure | Vazar payload/PII em eventos/erros | Minimização (RFC-0001 §7); `payload_ref` em vez de inline; `detail` sem PII (§5.4). |
| **D**enial of Service | Flood de `Submit` para exaurir filas | Rate-limit no Gateway + admission control + backpressure + cotas por tenant. |
| **E**levation of privilege | Preempção não autorizada / burlar cota | PDP em preempção; reservas atômicas; role `scheduler-admin` para operações sensíveis. |

### 12.3 LGPD / GDPR

- **Minimização:** o Scheduler NÃO DEVE persistir payload de tarefa; usa
  `payload_ref` (URN/URI para MinIO). `feature_vector` NÃO DEVE conter PII bruta.
- **Retenção:** decisões e idempotência têm TTL/retention definidos
  (`idempotency.ttl_hours`, política de retenção do journal); expurgo rastreável.
- **Direito ao esquecimento:** operação de expurgo por `tenant_id`/`agent_urn`
  propagada e auditada (coordenada com 025), removendo entradas de fila, decisões
  e idempotência associadas.
- **Segregação:** RLS + namespaces NATS garantem que dados de um tenant não
  vazem para decisões de outro.

---

## 13. Rastreabilidade para os 26 Documentos

Este brief alimenta diretamente: `Vision.md`(§1), `Architecture.md`(§2),
`FunctionalRequirements.md`(§7.1), `NonFunctionalRequirements.md`(§7.2),
`StateMachine.md`(§4), `ClassDiagrams.md`(§2/§3), `Database.md`(§3),
`API.md`(§5), `Events.md`(§6), `Configuration.md`(§8), `Security.md`(§12),
`FailureRecovery.md`(§9), `Scalability.md`(§10), `ADR.md`/`RFC.md`(§11). Nenhum
documento derivado PODE contradizer as seções acima.

---

*Fim do Design Brief do Módulo 009-Scheduler. Fonte única de verdade interna.*
