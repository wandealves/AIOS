---
Documento: StateMachine
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0093, ADR-0094 (a propor); ADR-0001..ADR-0010 (herdadas)
RFCs relacionados: RFC-0001 (baseline); RFC-0090 (a propor)
Depende de: 006-Kernel, 007-Agent-Runtime, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — StateMachine

> Esta é a máquina de estados canônica da **tarefa sob a ótica do Scheduler**,
> reproduzida sem alteração de `_DESIGN_BRIEF.md` §4. O Scheduler é a
> autoridade sobre os estados `RECEIVED`→`ADMITTED`→`QUEUED`→`SCHEDULED` e sobre
> os ramos `REJECTED`/`PREEMPTED`/`DEFERRED`/`EXPIRED`/`CANCELLED`. Os estados
> `RUNNING`/`COMPLETED`/`FAILED` são **observados** (a autoridade de execução é
> do Runtime Supervisor, 007-Agent-Runtime) — o Scheduler reage a eles para
> liberar slots e fechar contabilidade, mas não os produz.

## Índice

1. Escopo e autoridade da máquina de estados
2. Estados
3. Diagrama de estados (ASCII)
4. Tabela de transições, gatilhos e guardas
5. Ações de entrada e saída por estado
6. Estados terminais
7. Invariantes da máquina de estados
8. Relação com `SchedulingDecision.outcome`
9. Casos especiais (aging, EDF, thrashing)

---

## 1. Escopo e Autoridade da Máquina de Estados

Esta máquina de estados representa o ciclo de vida de uma **tarefa
escalonável** (`Task`, conforme `../040-Glossary/Glossary.md`) desde a
submissão de uma `SchedulingRequest` até um desfecho terminal. Ela é
implementada de forma distribuída:

| Estados | Autoridade | Onde a transição é decidida |
|---------|-----------|-------------------------------|
| `RECEIVED`, `ADMITTED`, `QUEUED`, `SCHEDULED`, `REJECTED`, `PREEMPTED`, `DEFERRED`, `EXPIRED`, `CANCELLED` | **Scheduler (009)** | `AdmissionController`, `PriorityQueueStore`, `DispatchLoop`, `PreemptionManager`, `BackpressureController`. |
| `RUNNING`, `COMPLETED`, `FAILED` | **Runtime Supervisor (007)** | Sinalizado ao Scheduler via `ReportRuntimeSignal` / eventos `agent.lifecycle.*`; o Scheduler apenas observa para liberar slot e fechar contabilidade. |

Nenhum documento derivado PODE introduzir estado, transição ou guarda que
contradiga o exposto aqui — qualquer extensão exige nova ADR e atualização do
`_DESIGN_BRIEF.md` primeiro.

---

## 2. Estados

| Estado | Significado | Tipo |
|--------|-------------|------|
| `RECEIVED` | `SchedulingRequest` chegou ao `SchedulingGateway`, validada estruturalmente, ainda não avaliada por admissão. | Transitório |
| `ADMITTED` | Admissão aceita (PDP + cota + orçamento + capacidade + backpressure ok); slot reservado. | Transitório |
| `QUEUED` | Tarefa na `PriorityQueueStore` (ZSET), aguardando dispatch por prioridade efetiva. | Transitório |
| `SCHEDULED` | `DispatchLoop` retirou a tarefa do topo, resolveu placement e enviou `SchedulingDecision` ao Runtime Supervisor. | Transitório |
| `RUNNING` | Runtime confirmou início de execução (observado via sinal/evento). | Transitório (autoridade externa) |
| `COMPLETED` | Runtime concluiu com sucesso (observado). | **Terminal** |
| `FAILED` | Runtime concluiu com falha (observado); pode retornar a `QUEUED` via política de retry. | **Terminal** (salvo retry) |
| `REJECTED` | Admissão negada (guarda de §4 falhou) ou cancelamento/expiração tratados como rejeição com `reasonCode`. | **Terminal** |
| `PREEMPTED` | Tarefa em `SCHEDULED`/`RUNNING` suspensa para liberar recurso a tarefa de maior prioridade efetiva. | Transitório (retoma para `QUEUED`) |
| `DEFERRED` | Tarefa em `QUEUED` sob backpressure alto, aguardando janela de reavaliação. | Transitório (retorna a `QUEUED`) |
| `EXPIRED` | Deadline (`deadline_at`) vencido antes do dispatch. | **Terminal** |
| `CANCELLED` | Cancelamento explícito (`Cancel`) a partir de `QUEUED`/`SCHEDULED`. | **Terminal** |

---

## 3. Diagrama de Estados (ASCII)

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

**Nota de leitura:** os estados `QUEUED` e `SCHEDULED` também podem transicionar
para `PREEMPTED` (arco não desenhado explicitamente do `QUEUED` acima por
clareza visual — ver tabela §4, linha `SCHEDULED/RUNNING → PREEMPTED`; na
prática a preempção alveja predominantemente tarefas já despachadas ou em
execução, não tarefas ainda em fila, pois estas não consomem slot).

---

## 4. Tabela de Transições, Gatilhos e Guardas

| De | Para | Gatilho | Guarda (DEVE ser verdadeira) | Ação/Evento |
|----|------|---------|------------------------------|-------------|
| `RECEIVED` | `ADMITTED` | `submit` | PDP=allow ∧ cota livre ∧ orçamento ≥ custo estimado ∧ capacidade ≥ 1 ∧ backpressure ≠ reject | Reserva slot (`QuotaLedger.TryReserve`); emite `aios.<tenant>.task.execution.admitted`. |
| `RECEIVED` | `REJECTED` | `submit` | Qualquer guarda de admissão falha | Libera reserva (se houver); mapeia para `AIOS-SCHED-000x`; emite `aios.<tenant>.task.execution.rejected`. |
| `ADMITTED` | `QUEUED` | `enqueue` | Prioridade efetiva computada (`PriorityCostEvaluator.Evaluate` concluído) | `PriorityQueueStore.Enqueue`; sem evento externo dedicado (interno ao pipeline). |
| `QUEUED` | `SCHEDULED` | `dispatch` | Topo elegível da ZSET ∧ capacidade do shard ≥ 1 ∧ lease de dispatch adquirido | `DispatchLoop` envia `SchedulingDecision` ao Runtime Supervisor; emite `aios.<tenant>.task.execution.scheduled`. |
| `QUEUED` | `DEFERRED` | `backpressure` | Pressão do shard ≥ `scheduler.backpressure.defer_threshold` (default 0.80) | Agenda reavaliação; emite (opcional) `aios.<tenant>.scheduler.backpressure.changed`. |
| `QUEUED` | `EXPIRED` | `deadline_tick` | `now > deadline_at` | Remove da fila; emite `aios.<tenant>.task.execution.rejected` (`reasonCode=deadline`, `AIOS-SCHED-0008`). |
| `QUEUED` / `SCHEDULED` | `CANCELLED` | `cancel` | Requisição em estado cancelável (não terminal) | Libera slot/lease; emite `aios.<tenant>.task.execution.rejected` (`reasonCode=cancelled`). |
| `SCHEDULED` | `RUNNING` | `runtime.started` | Runtime confirma boot (heartbeat/evento `agent.lifecycle.spawned`) | Marca `RUNNING` (observação); nenhum evento adicional emitido pelo Scheduler. |
| `SCHEDULED` / `RUNNING` | `PREEMPTED` | `preempt` | ∃ tarefa de maior prioridade efetiva ∧ PDP=allow ∧ `grace_period_ms` respeitado ∧ vítima fora de cooldown | `PreemptionManager` sinaliza `RequestSuspend` ao Runtime; emite `aios.<tenant>.task.execution.preempted`. |
| `PREEMPTED` | `QUEUED` | `requeue`/`resume` | Recurso liberado ou prioridade volta a permitir execução | Re-enqueue com aging preservado; emite `aios.<tenant>.task.execution.resumed` (quando aplicável). |
| `RUNNING` | `COMPLETED` | `runtime.completed` | — (observação de sinal externo) | Libera slot (`QuotaLedger.Release`); emite `aios.<tenant>.task.execution.completed`. |
| `RUNNING` | `FAILED` | `runtime.failed` | — (observação de sinal externo) | Libera slot; emite `aios.<tenant>.task.execution.failed`. |
| `FAILED` | `QUEUED` | `retry` | Retriável ∧ tentativas < `scheduler.retry.max_attempts` ∧ backoff decorrido (`scheduler.retry.backoff_base_ms`) | Re-enqueue com contador de tentativas incrementado. |

---

## 5. Ações de Entrada e Saída por Estado

| Estado | Ação de entrada (*on-enter*) | Ação de saída (*on-exit*) |
|--------|-------------------------------|------------------------------|
| `RECEIVED` | Validar envelope (URN, `traceparent`, `X-AIOS-Tenant`); registrar `IdempotencyStore.TryBegin`. | Encerra checagem de idempotência para o path corrente. |
| `ADMITTED` | Confirmar `SlotReservation`; iniciar `FeatureExtractor`. | Publicar features para `PriorityCostEvaluator`. |
| `QUEUED` | `ZADD` na `PriorityQueueStore` com score = prioridade efetiva; gravar `sched:meta:{task_urn}`. | Adquirir lease de dispatch (`lease_owner`, `lease_until`). |
| `SCHEDULED` | Persistir `SchedulingDecision` (outcome=`SCHEDULED`) via `DecisionJournal` (outbox atômico). | Aguardar confirmação do Runtime (timeout monitorado pelo `ReconciliationWorker`). |
| `RUNNING` | Nenhuma ação de admissão adicional; apenas contabilidade/telemetria. | — |
| `PREEMPTED` | Registrar `SchedulingDecision` (outcome=`PREEMPTED`) com `byTaskUrn`; iniciar temporizador de `grace_period_ms`. | Reaplicar aging preservado ao re-enfileirar. |
| `DEFERRED` | Registrar métrica de saturação; agendar nova tentativa de dispatch. | Retornar a `QUEUED` sem perda de posição relativa de aging. |
| `COMPLETED` | Liberar slot; persistir métricas de custo/latência observadas. | *(terminal)* |
| `FAILED` | Liberar slot; avaliar elegibilidade de retry. | *(terminal, salvo retry)* |
| `REJECTED` | Persistir `SchedulingDecision` (outcome=`REJECTED`) com `reasonCode`. | *(terminal)* |
| `EXPIRED` | Persistir decisão com `reasonCode=deadline`; remover de todas as estruturas de fila. | *(terminal)* |
| `CANCELLED` | Persistir decisão com `reasonCode=cancelled`; liberar slot/lease. | *(terminal)* |

---

## 6. Estados Terminais

Os estados terminais desta máquina são: **`COMPLETED`, `FAILED`
(salvo retry elegível), `REJECTED`, `EXPIRED`, `CANCELLED`**. Uma vez em
estado terminal (exceto `FAILED` retriável), nenhuma nova transição é
permitida — qualquer operação subsequente sobre o mesmo `request_id`
(`Cancel`, `Preempt`) DEVE retornar erro `AIOS-SCHED-0011` ("tarefa em estado
terminal; operação não aplicável").

**Regra de reentrância de `FAILED`:** `FAILED` é terminal apenas quando a
tarefa não é retriável ou já esgotou `scheduler.retry.max_attempts`; caso
contrário, é um estado transitório que retorna a `QUEUED`.

---

## 7. Invariantes da Máquina de Estados

| # | Invariante |
|---|-----------|
| SM-01 | Uma `SchedulingRequest` NÃO DEVE estar em mais de um estado simultaneamente (a máquina é determinística por `request_id`). |
| SM-02 | Uma transição para `SCHEDULED` DEVE coincidir com exatamente uma reserva de slot ativa (garantida por `QuotaLedger` + lease de dispatch) — nunca duas. |
| SM-03 | Toda transição para estado terminal DEVE liberar quaisquer reservas de slot/lease pendentes (sem vazamento de cota). |
| SM-04 | Toda transição DEVE ser acompanhada, quando aplicável (§4), da emissão do evento correspondente no envelope RFC-0001 §5.2, publicado atomicamente com a persistência via `DecisionJournal` (outbox). |
| SM-05 | O aging acumulado de uma tarefa NÃO DEVE ser perdido em transições `QUEUED → PREEMPTED/DEFERRED → QUEUED` (preservação de posição relativa/anti-starvation, FR-009). |
| SM-06 | Uma tarefa em `PREEMPTED` NÃO PODE ser re-selecionada como vítima de nova preempção antes do cooldown configurado (anti-*thrashing*). |
| SM-07 | Transições que dependem de PDP (`RECEIVED→ADMITTED`, `SCHEDULED/RUNNING→PREEMPTED`) DEVEM negar por padrão (*default deny*) se o Policy Engine (022) estiver indisponível. |
| SM-08 | `RUNNING`, `COMPLETED`, `FAILED` são estritamente **observados** — o Scheduler NUNCA DEVE forçar essas transições localmente sem sinal do Runtime Supervisor (007). |

---

## 8. Relação com `SchedulingDecision.outcome`

Cada transição relevante desta máquina gera (ou está associada a) um registro
`SchedulingDecision` com `outcome` correspondente (`_DESIGN_BRIEF.md` §3.2):

| Estado alcançado | `SchedulingDecision.outcome` |
|-------------------|-------------------------------|
| `ADMITTED` | `ADMITTED` |
| `SCHEDULED` | `SCHEDULED` |
| `REJECTED` / `EXPIRED` / `CANCELLED` | `REJECTED` (com `reasonCode` distinguindo a causa) |
| `PREEMPTED` | `PREEMPTED` |
| `QUEUED` (retomando de `PREEMPTED`/`DEFERRED`) | `RESUMED` (quando aplicável) ou implícito no re-enqueue |
| `QUEUED` (aguardando sob backpressure) | `DEFERRED` |

`RUNNING`/`COMPLETED`/`FAILED` não geram novo `SchedulingDecision` — são
registrados como observação/evento (`task.execution.completed`/`.failed`) sem
nova linha de decisão, pois a autoridade de execução pertence ao 007
(ver `_DESIGN_BRIEF.md` §4.1).

---

## 9. Casos Especiais

### 9.1 Aging (anti-starvation)

Enquanto uma tarefa permanece em `QUEUED`, seu `Score` na ZSET é recomputado
periodicamente aplicando `scheduler.aging.factor_per_sec`, até o teto
`scheduler.aging.max_boost`. Isso garante progresso mínimo (`Q_min` de vazão,
NFR-006) mesmo para tarefas de baixa prioridade original — sem alterar a
máquina de estados (o aging afeta apenas o critério de ordenação dentro de
`QUEUED`, não é uma transição de estado).

### 9.2 EDF (*Earliest Deadline First*)

Quando `scheduler.deadline.edf_enabled=true` e `deadline_at` está presente, a
prioridade efetiva é ajustada para favorecer tarefas com prazo mais próximo
(FR-012). A transição `QUEUED → EXPIRED` é o mecanismo de enforcement de
SLA/deadline (FR-012, `AIOS-SCHED-0008`).

### 9.3 Anti-*thrashing* de preempção

`SCHEDULED/RUNNING → PREEMPTED` está sujeita a histerese: uma tarefa recém
agendada (dentro do cooldown, ver ADR-0094 a propor) NÃO DEVE ser escolhida
como vítima novamente, evitando ciclos de preempção mútua entre tarefas de
prioridade próxima (NFR-009 — overhead de preempção ≤ 50 ms de decisão).

### 9.4 Retry após `FAILED`

A transição `FAILED → QUEUED` só ocorre se a falha for marcada como
retriável pelo Runtime Supervisor E o número de tentativas ainda não atingiu
`scheduler.retry.max_attempts` (default 3), com backoff mínimo de
`scheduler.retry.backoff_base_ms` (default 500 ms) antes do re-enqueue.
