---
Documento: Architecture
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0095, ADR-0096, ADR-0097 (a propor); ADR-0001..ADR-0010 (herdadas)
RFCs relacionados: RFC-0001 (baseline); RFC-0090, RFC-0091 (a propor)
Depende de: 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster
---

# 009-Scheduler — Architecture

> Este documento detalha, para o módulo Scheduler, o modelo C4 (Contexto →
> Contêiner → Componente) fixado em `../001-Architecture/Architecture.md`. Ele
> **não redefine** contratos centrais (URN, envelope de evento, envelope de
> erro, idempotência, correlação) — apenas os especializa para o domínio de
> scheduling, conforme `_DESIGN_BRIEF.md`.

## Índice

1. Posição no AIOS e contrato de fronteira
2. C4 Nível 1 — Contexto do Scheduler
3. C4 Nível 2 — Contêiner (Scheduler Service)
4. C4 Nível 3 — Componentes internos
5. Tabela de responsabilidades por componente
6. Fronteiras e não-responsabilidades
7. Padrões arquiteturais adotados
8. Tecnologias e justificativas
9. Modelo de camadas do caminho de decisão
10. Riscos arquiteturais e trade-offs
11. Alternativas descartadas
12. Referências e ADRs/RFCs

---

## 1. Posição no AIOS e Contrato de Fronteira

O Scheduler (009) é o **scheduler cognitivo** do plano de controle — análogo ao
CFS (*Completely Fair Scheduler*) do Linux, porém consciente de **custo**,
**qualidade** e **orçamento**. Ele responde à pergunta canônica: *"qual
tarefa/agente executa, quando, onde (placement), com qual prioridade e a que
custo?"*, sob restrições de cota, orçamento, SLA e capacidade. É o **guardião
da admissão** e da **backpressure** do sistema.

```
   Kernel(006) ── "pede slot para Task" (gRPC) ─────────▶ Scheduler(009)
   Policy(022) ◀── PDP "pode admitir/preemptar?" ───────  Scheduler(009)
   Cost-Optimizer(026) ◀── orçamento/estimativa custo ──  Scheduler(009)
   Cluster(027) ──── inventário de capacidade/shards ───▶ Scheduler(009)
   Scheduler(009) ── "SchedulingDecision (place/preempt)" ▶ Runtime Supervisor(007)
   Scheduler(009) ── eventos aios.<tenant>.task.execution.* ▶ NATS/JetStream
```

O Scheduler **decide**; o **Runtime Supervisor (007)** materializa a decisão
(spawn/suspend/kill de Agent Runtime). O Scheduler NÃO executa agentes, NÃO
fala com LLM (isso é do 017), NÃO cobra/precifica (isso é do 026), NÃO aplica
sandbox (021/007). Esta fronteira é **invariante arquitetural** — nenhum
componente interno descrito abaixo PODE assumir responsabilidade de outro
módulo.

---

## 2. C4 Nível 1 — Contexto do Scheduler

```
                                  ┌─────────────────────────────────┐
    ┌───────────────┐   gRPC      │                                 │
    │  Kernel (006) │────────────▶│                                 │
    │ (syscall spawn)│            │                                 │
    └───────────────┘             │                                 │
    ┌───────────────┐  admin REST │                                 │
    │ Web Console/  │────────────▶│         SCHEDULER (009)         │
    │ CLI (032/030) │            │      "Scheduler Cognitivo"        │
    └───────────────┘            │                                 │
    ┌───────────────┐  PDP allow │                                 │──gRPC──▶ ┌────────────────────┐
    │ Policy (022)  │◀───────────│  admission · priority · cost    │          │ Runtime Supervisor  │
    └───────────────┘            │  placement · preemption ·       │          │       (007)         │
    ┌───────────────┐  orçamento │  backpressure                    │          └──────────┬──────────┘
    │Cost-Optimizer │◀───────────│                                 │                     │ heartbeats/
    │    (026)      │            │                                 │◀────────────────────┘ sinais
    └───────────────┘            │                                 │
    ┌───────────────┐ inventário │                                 │──NATS──▶ ┌────────────────────┐
    │ Cluster (027) │────────────▶│                                 │          │ JetStream (eventos) │
    └───────────────┘            │                                 │          │ task.execution.*    │
                                  └─────────────────────────────────┘          └──────────┬──────────┘
                                                                                            │ consome
                                                                        ┌───────────────────▼────────┐
                                                                        │ Audit(025) · Learning(023) │
                                                                        │ Observability(024)          │
                                                                        └─────────────────────────────┘
```

**Atores e sistemas no contexto**

| Ator/Sistema | Papel frente ao Scheduler | Protocolo |
|--------------|---------------------------|-----------|
| Kernel (006) | Solicita slot de execução para uma `Task`/`Agent` (`Submit`). | gRPC interno |
| Policy Engine — PDP (022) | Autoriza/nega admissão e preempção (*default deny*). | gRPC interno |
| Cost-Optimizer (026) | Fornece orçamento disponível e estimativa de custo/latência. | gRPC interno |
| Cluster (027) | Fornece inventário de capacidade (nós/shards/pools) e topologia. | gRPC/eventos |
| Runtime Supervisor (007) | Recebe `SchedulingDecision` (place/preempt); reporta heartbeats/sinais de runtime. | gRPC interno |
| Web Console/CLI (032/030) | Operação administrativa: preempção dirigida, política, observabilidade de filas. | REST via Gateway (YARP) |
| Audit (025) / Learning (023) / Observability (024) | Consomem eventos e proveniência de decisão. | NATS/JetStream |

---

## 3. C4 Nível 2 — Contêiner (Scheduler Service)

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                          SCHEDULER SERVICE  (.NET 10, stateless)                     │
│                                                                                      │
│   REST (YARP) ─────────┐            gRPC interno ─────────┐                          │
│                        ▼                                 ▼                          │
│              ┌───────────────────────────────────────────────────┐                  │
│              │            Scheduler API Surface (Gateway/PEP)     │                  │
│              └───────────────────────┬───────────────────────────┘                  │
│                                      ▼                                              │
│              ┌───────────────────────────────────────────────────┐                  │
│              │   Pipeline de decisão (Admission→Priority→         │                  │
│              │   Placement→Preemption→Dispatch)                   │                  │
│              └───────────────────────┬───────────────────────────┘                  │
│                                      │                                              │
└──────────────────────────────────────┼──────────────────────────────────────────────┘
                                       │
        ┌──────────────────────┬──────┴───────┬────────────────────┐
        ▼                      ▼               ▼                    ▼
┌───────────────┐     ┌────────────────┐ ┌───────────────┐  ┌────────────────────┐
│     Redis      │     │  PostgreSQL    │ │ NATS/JetStream│  │  Observability     │
│ filas ZSET ·   │     │ DecisionJournal│ │ eventos       │  │  OTel · Prometheus │
│ cotas · leases │     │ (outbox)       │ │ task.execution│  │  Serilog → Seq     │
└───────────────┘     └────────────────┘ └───────────────┘  └────────────────────┘
```

**Contêiner e responsabilidade**

| Contêiner | Tecnologia | Responsabilidade | Estado |
|-----------|-----------|-------------------|--------|
| Scheduler Service | .NET 10 | Toda a lógica de admissão, prioridade/custo, placement, preempção, backpressure, dispatch. | Stateless (estado externalizado) |
| Redis | Redis 7 | Filas de prioridade (ZSET), cotas/reservas atômicas, leases de shard/dispatch, cache de capacidade. | Volátil, hot state |
| PostgreSQL | PG16 | `DecisionJournal` (outbox transacional, event sourcing de decisão), `IdempotencyRecord` de longo prazo. | Fonte da verdade (frio) |
| NATS/JetStream | NATS | Publicação de eventos `task.execution.*` e `scheduler.*`; consumo de eventos de 007/026/027/022. | Stream durável |
| Observability Stack | OTel/Prometheus/Grafana/Serilog/Seq | Traces/métricas/logs do caminho de decisão. | Time-series |

O Scheduler Service é **stateless** — nenhuma réplica mantém estado que não
possa ser reconstruído a partir de Redis/PostgreSQL. Isso permite escala
horizontal linear (ver §10 do Design Brief e `Scalability.md`).

---

## 4. C4 Nível 3 — Componentes Internos

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

Este diagrama é normativo e idêntico ao fixado em `_DESIGN_BRIEF.md` §2.2 — o
Scheduler DEVE manter esta decomposição; qualquer refatoração estrutural
relevante DEVE ser proposta como nova ADR (faixa `ADR-0090`–`ADR-0099`) antes de
alterar este documento.

---

## 5. Tabela de Responsabilidades por Componente

| Componente | Responsabilidade | Depende de |
|------------|-------------------|------------|
| `SchedulingGateway` | Superfície de entrada (gRPC/REST interno). Recebe `SchedulingRequest`, valida envelope, correlação e `Idempotency-Key`; roteia ao pipeline. É o **PEP** do módulo. | `PolicyClient`, `IdempotencyStore`, `AdmissionController` |
| `AdmissionController` | Decide admitir/rejeitar/enfileirar: verifica cotas (Redis), orçamento (026), capacidade (`CapacityProvider`), backpressure e política. Emite `admitted`/`rejected`. | `QuotaLedger`, `BudgetClient`, `CapacityProvider`, `PolicyClient`, `BackpressureController` |
| `PriorityCostEvaluator` | Computa `C = α·L + β·$ + γ·E` e prioridade efetiva; aplica classe de serviço, deadline (EDF), aging (anti-starvation). Encapsula `ISchedulingPolicy`. | `PolicyWeightsProvider`, `CostEstimator`, `ISchedulingPolicy` |
| `ISchedulingPolicy` | Interface estável de decisão. `HeuristicPolicy` (v1) e `LearnedPolicy` (v2, RL) implementam-na sem alterar contrato externo (API/eventos). | `FeatureExtractor` |
| `FeatureExtractor` | Extrai o vetor de features da decisão (carga, fila, custo estimado, folga de SLA, histórico) para política e proveniência/treino. | `QuotaLedger`, `CapacityProvider`, `CostEstimator` |
| `PlacementEngine` | Escolhe shard/nó/runtime-pool via `hash(tenant,agent) mod N`; aplica afinidade/anti-afinidade e capacidade residual; produz `PlacementTarget`. | `ShardRouter`, `CapacityProvider` |
| `ShardRouter` | Função determinística de sharding e mapa shard→nó (rebalanceamento consistente sob mudança de N). | `CapacityProvider` (topologia de 027) |
| `PreemptionManager` | Seleciona vítimas de preempção por prioridade efetiva; orquestra suspensão segura (aviso + grace period); emite `preempted`. | `PriorityQueueStore`, `RuntimeSupervisorClient`, `PolicyClient` |
| `PriorityQueueStore` | Filas de prioridade por (tenant, classe, shard) em Redis ZSET; enqueue/dequeue/peek/aging; leases de dispatch. | Redis |
| `DispatchLoop` | Laço de despacho: retira o topo elegível das filas, revalida capacidade, emite `scheduled` e envia `SchedulingDecision` ao Runtime Supervisor. | `PriorityQueueStore`, `PlacementEngine`, `RuntimeSupervisorClient`, `DecisionJournal` |
| `BackpressureController` | Calcula estado de saturação (fila, lag JetStream, capacidade) e expõe sinal de admissão (aceitar/desacelerar/rejeitar) + `Retry-After`. | `CapacityProvider`, `PriorityQueueStore`, JetStream |
| `QuotaLedger` | Contabiliza consumo de concorrência/slots por tenant/agente/classe em Redis (contadores atômicos com TTL); *reservation* durante admissão. | Redis |
| `CapacityProvider` | Visão de capacidade (slots livres por shard/nó/pool), atualizada por heartbeats do Runtime Supervisor e inventário do 027. | `RuntimeSupervisorClient`, 027-Cluster |
| `CostEstimator` | Estima custo/latência/energia esperados por tarefa (chama 026 / cache local); alimenta `C`. | `BudgetClient` (026) |
| `PolicyClient` | Cliente PDP: consulta 022 para autorização de admissão/preempção (*default deny*). | 022-Policy |
| `BudgetClient` | Cliente de orçamento/estimativa junto ao 026. | 026-Cost-Optimizer |
| `RuntimeSupervisorClient` | Cliente gRPC do Runtime Supervisor (007): envia decisão, recebe heartbeats/término. | 007-Agent-Runtime |
| `IdempotencyStore` | Persiste resultado de decisão por `Idempotency-Key`/`request_id` ≥ 24h (RFC-0001 §5.5). | Redis + PostgreSQL |
| `DecisionJournal` | Outbox transacional: grava a decisão (proveniência) em PostgreSQL e publica evento atomicamente. | PostgreSQL, NATS/JetStream |
| `EventPublisher` | Serializa e publica eventos no envelope CloudEvents (RFC-0001 §5.2) nos subjects canônicos. | NATS/JetStream, `DecisionJournal` |
| `SchedulerTelemetry` | Instrumentação OTel (traces/metrics/logs Serilog→Seq), golden signals e SLIs. | OpenTelemetry |
| `ReconciliationWorker` | Reconcilia estado (filas × runtime real), expira leases órfãos, faz re-scheduling e detecta decisões perdidas. | `PriorityQueueStore`, `CapacityProvider`, `DecisionJournal` |

---

## 6. Fronteiras e Não-Responsabilidades

O Scheduler é intencionalmente **estreito**: decide, não executa. As
fronteiras abaixo são invioláveis e refletem `_DESIGN_BRIEF.md` §1.2.

| # | Não-Responsabilidade | Dono correto |
|---|----------------------|--------------|
| N-01 | Executar o loop cognitivo/ReAct do agente. | 007-Agent-Runtime |
| N-02 | Criar/suspender/matar processos de runtime (materializar placement). | 007-Agent-Runtime (Runtime Supervisor) |
| N-03 | Definir/avaliar políticas RBAC/ABAC (é PEP, consulta PDP; não é PDP). | 022-Policy |
| N-04 | Escolher o modelo LLM ou falar com provedores de inferência. | 017-Model-Router |
| N-05 | Calcular preço/faturar; ser fonte da verdade de orçamento. | 026-Cost-Optimizer |
| N-06 | Persistir memória cognitiva, contexto ou conhecimento. | 010-Memory / 011-Context / 018-Knowledge / 019-GraphRAG |
| N-07 | Prover HA/replicação de NATS/PG/Redis; failover de infraestrutura. | 027-Cluster |
| N-08 | Gerenciar cotas de token dentro da inferência (rate de tokens do LLM). | 026-Cost-Optimizer / 017-Model-Router |
| N-09 | Trilha de auditoria imutável (apenas emite eventos que 025 consome). | 025-Audit |
| N-10 | Decidir *o que* a tarefa faz (plano); apenas *quando/onde/prioridade*. | 012-Planning / 014-Workflow |

Consequência arquitetural direta: o Scheduler **não mantém** cliente de LLM,
**não mantém** sandbox, e **não é** fonte da verdade de faturamento — qualquer
componente novo proposto que viole esta fronteira DEVE ser rejeitado em code
review e reencaminhado ao módulo dono.

---

## 7. Padrões Arquiteturais Adotados

| Padrão | Onde no Scheduler | Motivo |
|--------|--------------------|--------|
| PEP / PDP | `SchedulingGateway` (PEP) → `PolicyClient` → Policy Engine (022, PDP). | *Default deny* centralizado (RFC-0001 §5.8); Scheduler nunca decide autorização sozinho. |
| Outbox transacional | `DecisionJournal` (grava decisão + publica evento na mesma unidade lógica). | Atomicidade DB+evento; nenhuma decisão é "perdida" entre persistência e notificação. |
| Event Sourcing | `SchedulingDecision` é append-only; estado observável é derivado da sequência de decisões. | Proveniência auditável (025) e dado de treino para política aprendida (023, RFC-0091). |
| CQRS (leitura/escrita) | Escrita via `DispatchLoop`/`AdmissionController`; leitura de estado de fila via `StreamQueueState`/REST `/queues` sem tocar no caminho quente. | Isola leitura de observabilidade do caminho crítico de decisão (p99 ≤ 20 ms). |
| Strategy (política plugável) | `ISchedulingPolicy` com `HeuristicPolicy` (v1) e `LearnedPolicy` (v2, RL). | Evolução de heurística → RL sem mudar API/eventos (FR-013, ADR-0096, RFC-0091). |
| Sharding determinístico | `ShardRouter`: `shard = hash(tenant_id, agent_id) mod N`. | Particionamento sem coordenação central; caminho para 10⁶⁺ agentes (ADR-0092). |
| Lease / Ownership | Ownership de shard e lease de dispatch em Redis com TTL. | Evita locks globais e dupla retirada de fila sob concorrência. |
| Backpressure em camadas | `BackpressureController`: `accept < defer(0.80) < reject(0.95)`. | Protege o sistema de saturação, propaga `Retry-After` ao produtor (ADR-0095). |
| Circuit breaker (implícito via fail-safe) | Modos de falha de dependências externas (022/026/007) em `_DESIGN_BRIEF.md` §9. | Degradação graciosa sem overcommit. |
| Ambassador (Gateway) | Exposição REST via YARP (`/v1/scheduler/...`), gRPC interno direto. | Consistência com o padrão global de entrada única (`001-Architecture`). |

---

## 8. Tecnologias e Justificativas

| Tecnologia | Papel no Scheduler | Justificativa | ADR |
|------------|---------------------|----------------|-----|
| **.NET 10** | Toda a lógica de decisão (control plane). | Desempenho previsível e baixa latência de GC/JIT necessários para p99 ≤ 20 ms (NFR-001); tipagem forte para o contrato `ISchedulingPolicy`; async/await maduro para I/O de Redis/PG/NATS sem bloqueio. | ADR-0003 (herdada) |
| **Redis (ZSET + contadores atômicos)** | `PriorityQueueStore`, `QuotaLedger`, leases de shard/dispatch, cache de capacidade. | Estruturas nativas de fila ordenada (ZSET) com score = prioridade efetiva; operações atômicas (scripts Lua) evitam overcommit sem locks distribuídos; latência sub-ms compatível com o caminho quente. | ADR-0090, ADR-0093 |
| **PostgreSQL** | `DecisionJournal` (outbox/event sourcing), `IdempotencyRecord` de longo prazo. | ACID para garantir que a decisão persistida e o evento publicado sejam atômicos; base relacional já padrão do AIOS (`005-Database`). Não há uso de `pgvector`/`Apache AGE` neste módulo — o Scheduler não armazena embeddings nem grafo de conhecimento (fora de escopo, ver §6). | ADR-0097 (herdada de ADR-0005) |
| **NATS/JetStream** | Publicação de `task.execution.*`/`scheduler.*`; consumo de eventos de 007/026/027/022. | Baixíssima latência de pub/sub; JetStream garante durabilidade e replay para outbox; alinhado ao barramento único do AIOS (ADR-0004). | ADR-0095 (herdada de ADR-0004) |
| **gRPC** | Superfície primária (`aios.scheduler.v1.SchedulerService`) para chamadas control-plane→control-plane (Kernel, Runtime Supervisor). | Baixa latência, contrato fortemente tipado via protobuf, streaming nativo (`StreamQueueState`). | ADR-0001 (herdada) |
| **REST (via YARP)** | Superfície administrativa/observabilidade (`/v1/scheduler/...`). | Compatibilidade externa/ferramentas de operação; versionamento por caminho (`/v1`) conforme RFC-0001 §5.7. | ADR-0001 (herdada) |
| **OpenTelemetry + Prometheus + Grafana** | `SchedulerTelemetry`: traces/métricas do pipeline de decisão. | Padrão vendor-neutral já adotado globalmente; necessário para NFR-010 (100% das decisões com trace/span). | ADR-0001 (herdada) |
| **Serilog → Seq** | Log estruturado do Scheduler. | Consulta estruturada por `tenant_id`/`request_id`/`decision_id`; padrão .NET do AIOS. | ADR-0001 (herdada) |
| **MinIO** | Não usado diretamente pelo Scheduler (apenas referenciado indiretamente via `payload_ref` de `SchedulingRequest`, que aponta para blobs de terceiros). | Minimização de dados: o Scheduler NÃO DEVE persistir payload de tarefa (§12.3 do brief). | — |

---

## 9. Modelo de Camadas do Caminho de Decisão

```
  L3  ENTRADA         SchedulingGateway (PEP) — validação de envelope, correlação, Idempotency-Key
  ─────────────────────────────────────────────────────────────────────────────────────
  L2  DECISÃO         AdmissionController → PriorityCostEvaluator → PlacementEngine →
                       PreemptionManager  (caminho quente, sem I/O bloqueante externo)
  ─────────────────────────────────────────────────────────────────────────────────────
  L1  DESPACHO         PriorityQueueStore (Redis ZSET) → DispatchLoop → RuntimeSupervisorClient
  ─────────────────────────────────────────────────────────────────────────────────────
  L0  PERSISTÊNCIA/     DecisionJournal (PostgreSQL, outbox) → EventPublisher (NATS/JetStream)
      NOTIFICAÇÃO
  ─────────────────────────────────────────────────────────────────────────────────────
  L-  TRANSVERSAL       SchedulerTelemetry (OTel) · ReconciliationWorker · IdempotencyStore
```

Regra de dependência: uma camada só depende da camada imediatamente inferior ou
de componentes transversais (`SchedulerTelemetry`, `IdempotencyStore`). O
caminho L2 (decisão) NÃO DEVE realizar chamadas síncronas bloqueantes a
serviços externos (022/026/027) fora de cache de curta duração — é o que
garante o SLO de p99 ≤ 20 ms (NFR-001, `_DESIGN_BRIEF.md` §10).

---

## 10. Riscos Arquiteturais e Trade-offs

| Risco/Trade-off | Descrição | Decisão |
|------------------|-----------|---------|
| Cache de política/orçamento no caminho quente | Cache de curta duração de PDP/orçamento pode admitir com dado levemente desatualizado. | Aceito: TTL curto (segundos) + reconciliação assíncrona; nunca overcommit de slot (garantido por reserva atômica em Redis), apenas possível leve atraso em refletir mudança de orçamento/política. |
| Redis como fonte de estado quente único | Indisponibilidade de Redis afeta todo o caminho de admissão. | Aceito com fail-safe: modo conservador rejeita novas admissões (`AIOS-SCHED-0003`) em vez de arriscar overcommit (ver `_DESIGN_BRIEF.md` §9). |
| Sharding fixo por `N` configurável | Mudança de `N` exige rebalanceamento coordenado com 027. | Aceito: `scheduler.shards.count` não é recarregável em runtime isoladamente; ADR-0092 define hashing consistente para minimizar migração. |
| Preempção pode gerar *thrashing* | Preempções repetidas entre tarefas de prioridade próxima. | Mitigado por histerese/cooldown (ADR-0094); métrica `aios_scheduler_preemption_ms` monitora overhead. |
| Duas gerações de política coexistindo (`heuristic`/`rl`) | Complexidade operacional de suportar v1 e v2 simultaneamente por tenant. | Aceito: interface `ISchedulingPolicy` isola o impacto; troca é por configuração (`scheduler.policy.engine`), não por deploy (ADR-0096). |

---

## 11. Alternativas Descartadas

| Alternativa | Motivo da recusa | ADR |
|-------------|-------------------|-----|
| Fila única global (sem sharding) | Não escala para 10⁶⁺ agentes; ponto de contenção único; viola NFR-004. | ADR-0092 |
| Lock distribuído (ex.: Redlock) para dispatch | Overhead de latência incompatível com p99 ≤ 20 ms; complexidade de correção sob partição de rede. | ADR-0090, ADR-0093 |
| Preempção não-compensável (kill imediato) | Perde trabalho em curso sem grace period; viola princípio de recuperabilidade do AIOS. | ADR-0094 |
| Política de decisão hardcoded (sem interface plugável) | Bloquearia evolução para RL (v2) sem quebra de contrato; contraria FR-013. | ADR-0096 |
| Kafka como bus de eventos do Scheduler | Latência/operabilidade piores que NATS para o caso de uso de baixa latência; decisão já tomada globalmente. | ADR-0004 (herdada) |
| Persistir payload de tarefa inline em `SchedulingRequest` | Viola minimização de dados (LGPD/GDPR) e infla o caminho quente. | Ver `_DESIGN_BRIEF.md` §12.3 |

---

## 12. Referências e ADRs/RFCs

- Fonte única de verdade do módulo: `_DESIGN_BRIEF.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- ADRs do módulo (a propor, faixa reservada `ADR-0090`–`ADR-0099`): `../002-ADR/`
  — ver índice consolidado em `ADR.md`.
- RFCs do módulo (a propor): `../003-RFC/` — ver índice consolidado em `RFC.md`.
- Diagramas de classe/estrutura: `ClassDiagrams.md`
- Máquina de estados: `StateMachine.md`
- Fluxos críticos: `SequenceDiagrams.md`
- Modelo de escala e sharding: `Scalability.md`
