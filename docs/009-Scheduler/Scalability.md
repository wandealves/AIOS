---
Documento: Scalability
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0090 (Admission control com reserva atômica), ADR-0092 (Sharding determinístico), ADR-0093 (Filas ZSET com leases), ADR-0095 (Backpressure em três níveis)
RFCs relacionados: RFC-0001 (baseline); RFC-0090 (a propor)
Depende de: 001-Architecture, 006-Kernel, 007-Agent-Runtime, 027-Cluster
---

# 009-Scheduler — Scalability

> **Escopo.** Este documento especifica o modelo de escala do Scheduler:
> particionamento/sharding, réplicas, concorrência, ausência de locks globais
> no caminho quente, backpressure e o caminho de crescimento até 10⁶⁺ agentes
> (NFR-004). Reproduz e detalha `_DESIGN_BRIEF.md` §10 sem contradizê-lo; não
> redefine os contratos centrais de RFC-0001.

---

## Índice

1. Princípio Central de Escala
2. Sharding Determinístico
3. Réplicas, Ownership e Ausência de Locks Globais
4. Estado Externalizado (Stateless Service)
5. Concorrência no Caminho de Dispatch
6. Backpressure em Três Níveis
7. Caminho Quente: Orçamento de Latência
8. Cold Agents e Materialização
9. Rebalanceamento de Shards (mudança de N)
10. Limites Teóricos e Rumo a 10⁶⁺ Agentes
11. Fairness sob Escala (Anti-Starvation)
12. Riscos de Escala e Mitigações
13. Metodologia de Validação (Benchmark)
14. Referências

---

## 1. Princípio Central de Escala

A estratégia de escala do Scheduler assenta-se em quatro pilares
combinados, todos derivados de `_DESIGN_BRIEF.md` §10:

```
   ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
   │ Sharding             │  │ Estado externalizado │  │ Caminho quente sem   │  │ Backpressure         │
   │ determinístico        │  │ (Redis/PostgreSQL)   │  │ I/O bloqueante       │  │ em camadas           │
   │ hash(tenant,agent)    │  │ — serviço stateless  │  │ — p99 ≤ 20 ms        │  │ accept/defer/reject  │
   │ mod N                  │  │ escala horizontal    │  │ (NFR-001)             │  │                       │
   └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘
              └───────────────────────┴───────────────────────┴───────────────────────┘
                                              │
                                   Escala sub-linear de coordenação
                                   rumo a 10⁶⁺ agentes (NFR-004)
```

Nenhum destes pilares funciona isoladamente: sharding sem estado externalizado
recriaria afinidade de estado por réplica (fragilidade a falha); estado
externalizado sem caminho quente rápido violaria o SLO de decisão; e nenhum
dos dois impede saturação sem backpressure. A combinação é o que permite
crescimento sub-linear de custo de coordenação com o número de agentes/tarefas.

---

## 2. Sharding Determinístico

### 2.1 Função de sharding

```
shard = hash(tenant_id, agent_id) mod N          (N = scheduler.shards.count, default 64)
```

- `ResolveShard` (componente `ShardRouter`) **DEVE** ser **puramente
  determinístico**: mesma entrada `(tenant_id, agent_id)` produz sempre o
  mesmo `shard` para um dado `N` — sem estado, sem I/O (INV-07,
  `ClassDiagrams.md` §4).
- Filas de prioridade (`PriorityQueueStore`, Redis ZSET) são particionadas por
  `(tenant, service_class, shard)`: chave `sched:{tenant}:{class}:{shard}`.
  Isso significa que o trabalho de um `(tenant, agent)` está sempre
  concentrado em um único shard, permitindo dispatch local sem coordenação
  cross-shard.
- `PlacementEngine` usa o shard resolvido para escolher nó/pool de runtime
  dentro daquele shard, respeitando afinidade/anti-afinidade e capacidade
  residual (`CapacityProvider`).

### 2.2 Por que sharding e não fila única global

| Alternativa | Por que foi descartada |
|-------------|---------------------------|
| Fila única global (ZSET única para todo o sistema) | Todo `enqueue`/`dequeue` competiria pela mesma estrutura — ponto de contenção único; não escala para 10⁶⁺ agentes (viola NFR-004). Descartada, ver `Architecture.md` §11. |
| Lock distribuído (ex.: Redlock) para coordenar dispatch entre réplicas | Overhead de latência incompatível com p99 ≤ 20 ms; complexidade de correção sob partição de rede (split-brain). Descartada. |
| Particionamento apenas por `tenant_id` (sem `shard` explícito) | Tenants muito grandes (*noisy neighbor*) concentrariam carga em uma única partição; sharding por `hash(tenant,agent) mod N` distribui a carga mesmo dentro de um tenant grande. |

---

## 3. Réplicas, Ownership e Ausência de Locks Globais

O Scheduler Service é **stateless**: qualquer réplica **PODE**, em princípio,
atender qualquer tenant/shard, recorrendo a Redis/PostgreSQL para o estado
necessário. Na prática, para evitar contenção de dispatch (duas réplicas
tentando retirar o mesmo topo de fila simultaneamente), cada réplica assume a
posse de um subconjunto de shards via **lease em Redis com TTL**
(`scheduler.lease.ttl_ms`):

```
   requests ─▶ [réplica A owner shards 0..31] ─▶ ZSET sched:{t}:{c}:{0..31} ─▶ dispatch
   requests ─▶ [réplica B owner shards 32..63]─▶ ZSET sched:{t}:{c}:{32..63}─▶ dispatch
                         ▲ ownership lease (Redis, TTL) ▲   backpressure feedback ◀── JetStream lag
```

| Propriedade | Garantia |
|-------------|-----------|
| Sem lock global | Nenhuma operação do caminho de admissão/dispatch adquire um lock que abranja mais de um shard. |
| Ownership é otimização, não requisito de correção | Se a réplica *owner* de um shard falhar, o lease expira e outra réplica assume — nenhuma tarefa fica presa a uma réplica específica (ver `Deployment.md` §5). |
| Reserva atômica de slot | `QuotaLedger.TryReserve` usa script Lua atômico no Redis (CAS), garantindo que a soma de reservas nunca exceda `max_concurrency` mesmo sob múltiplas réplicas concorrentes (NFR-005, ADR-0090). |
| Lease de dispatch | `DispatchLoop` adquire um lease por entrada de fila antes de despachar, evitando dupla retirada (`Dequeue` é atômico via `EVALSHA`). |

---

## 4. Estado Externalizado (Stateless Service)

| Estado | Onde vive | Por que não fica na réplica |
|--------|-----------|-------------------------------|
| Filas de prioridade (`QueueEntry`) | Redis ZSET | Precisa ser visível/mutável por qualquer réplica que assuma ownership de um shard; sobrevive a reinício/substituição de réplica. |
| Cotas/reservas (`QuotaLedger`) | Redis (contadores atômicos) | Contabilidade global por tenant/classe não pode ser particionada por réplica sem risco de overcommit. |
| Proveniência de decisão (`SchedulingDecision`) | PostgreSQL (`DecisionJournal`) | Fonte da verdade durável, auditável, append-only — não pode residir apenas em memória de réplica. |
| Cache de capacidade (`CapacitySnapshot`) | Redis (com TTL de frescor) | Compartilhado entre réplicas; heartbeats do Runtime Supervisor/Cluster atualizam uma visão única. |
| Cache local de decisão PDP/orçamento | Memória de réplica (TTL curto) | Única exceção intencional: cache de curtíssima duração para reduzir I/O síncrono no caminho quente — nunca fonte de verdade, invalidado por evento (`Security.md` §3.2). |

Consequência direta: **escala horizontal linear** — adicionar réplicas
aumenta capacidade de processamento de `Submit`/dispatch proporcionalmente,
sem exigir replicação de estado entre réplicas (o estado já está
externalizado e compartilhado).

---

## 5. Concorrência no Caminho de Dispatch

```
                    ┌─────────────────────────────────────────────┐
                    │            DispatchLoop (por shard)           │
                    │  1. PeekTop(tenant,class,shard)                │
                    │  2. Revalida capacidade (CapacityProvider)     │
                    │  3. Dequeue com lease atômico                  │
                    │  4. PlacementEngine.Resolve → PlacementTarget  │
                    │  5. RuntimeSupervisorClient.Dispatch           │
                    │  6. DecisionJournal.RecordAndPublish (outbox)  │
                    └─────────────────────────────────────────────┘
```

- Cada shard tem, no máximo, **um `DispatchLoop` ativo por vez** (garantido
  pelo lease de ownership de shard, §3) — isso elimina a necessidade de
  coordenar múltiplos despachadores concorrentes sobre a mesma ZSET.
- Dentro de um shard, o `Dequeue` é atômico (script Lua: `ZPOPMIN` + registro
  de lease em uma única operação), eliminando corrida entre o `DispatchLoop`
  e um eventual `ReconciliationWorker` reenfileirando a mesma entrada.
- Não há necessidade de *two-phase commit* entre Redis e PostgreSQL: a
  reserva de slot é confirmada antes do enqueue (Redis), e a decisão de
  dispatch é persistida via outbox transacional (PostgreSQL) **depois** da
  retirada da fila — na pior hipótese de falha entre as duas etapas, o
  `ReconciliationWorker` detecta a divergência e reenfileira (idempotente por
  `request_id`, ver `FailureRecovery.md`).

---

## 6. Backpressure em Três Níveis

| Nível | Condição (pressão do shard) | Comportamento |
|-------|-------------------------------|------------------|
| `accept` | pressão < `scheduler.backpressure.defer_threshold` (default 0.80) | Admissão normal. |
| `defer` | `defer_threshold` ≤ pressão < `reject_threshold` (0.80–0.95) | Tarefa é admitida, porém pode transitar por `DEFERRED` antes de reentrar em `QUEUED` — meio termo entre aceitar sem restrição e rejeitar. |
| `reject` | pressão ≥ `scheduler.backpressure.reject_threshold` (default 0.95) | Nova admissão rejeitada com `AIOS-SCHED-0003` (503) e `retryAfterMs` (default `scheduler.backpressure.retry_after_ms` = 1500 ms). |

`BackpressureController.Evaluate(shard)` combina três sinais:

```
   pressão(shard) = f( profundidade_fila(shard),
                        lag_JetStream(shard),
                        1 - capacidade_residual(shard) )
```

Cada transição de nível **DEVE** publicar
`aios.<tenant>.scheduler.backpressure.changed` (brief §6.1), permitindo que
produtores (Kernel, Gateway) ajustem sua própria taxa de emissão de forma
proativa, não apenas reativa a erros `503`. O sinal de backpressure é
**por shard**, não global — um shard saturado não degrada a admissão em
shards com capacidade residual, preservando isolamento de "vizinho
barulhento" entre partições de tenants/agentes.

---

## 7. Caminho Quente: Orçamento de Latência

O SLO de `Submit` (p99 ≤ 20 ms, p50 ≤ 5 ms — NFR-001) só é sustentável se o
caminho de decisão **não realizar I/O bloqueante síncrono** a serviços
externos fora de cache de curta duração. Orçamento aproximado por etapa:

| Etapa | Fonte de dado | Orçamento típico | Observação |
|-------|----------------|---------------------|-------------|
| Validação de envelope/idempotência | `IdempotencyStore` (Redis) | ~0,5–1 ms | Verificação de chave; cache hit no caminho comum. |
| Verificação de cota | `QuotaLedger` (Redis, script Lua) | ~0,5–1 ms | Operação atômica local ao Redis. |
| Decisão de política (PDP) | `PolicyClient` (cache TTL curto) | ~0,1–1 ms (cache hit) / até timeout configurado em cache miss | Cache miss é o principal risco de latência — mitigado por TTL e invalidação orientada a evento (`Security.md` §3.2). |
| Orçamento/estimativa de custo | `BudgetClient`/`CostEstimator` (cache local) | ~0,1–1 ms (cache hit) | Fallback a estimativa cacheada sob timeout (`_DESIGN_BRIEF.md` §9). |
| Cálculo de score `C = α·L + β·$ + γ·E` | `PriorityCostEvaluator` (in-process) | < 0,5 ms | Puramente computacional, sem I/O. |
| Placement (`hash mod N` + capacidade) | `PlacementEngine`/`CapacityProvider` (cache Redis) | ~0,5–1 ms | Leitura de `CapacitySnapshot` cacheado. |
| Enqueue (ZSET) | `PriorityQueueStore` (Redis) | ~0,5–1 ms | `ZADD` atômico. |
| **Total caminho quente (sem dispatch)** | — | **≈ 3–7 ms típico, ≤ 20 ms p99** | Compatível com NFR-001; dispatch em si ocorre de forma assíncrona no `DispatchLoop`, fora do caminho de resposta de `Submit`. |

Nenhuma etapa acima **DEVE** aguardar uma chamada de rede sem timeout curto
configurado (`scheduler.decision.timeout_ms`, default 15 ms) — timeouts
excedidos acionam o comportamento de fallback/fail-safe correspondente
(`FailureRecovery.md`), nunca um bloqueio indefinido.

---

## 8. Cold Agents e Materialização

O Scheduler trata tarefas de agentes suspensos (*cold agents*, ver Glossário)
exatamente como qualquer outra: elas entram na fila de prioridade normalmente.
A **materialização** do Agent Runtime (spawn, com meta de latência < 250 ms) é
de responsabilidade exclusiva do `007-Agent-Runtime` — o Scheduler apenas
garante a existência de um slot reservado e um `PlacementTarget`, sem
conhecimento do custo de "acordar" o agente. Isso mantém o Scheduler
desacoplado do custo de frio/quente do runtime, permitindo que a estratégia de
hibernação evolua em `007`/`008` sem alterar o modelo de escala do Scheduler.

---

## 9. Rebalanceamento de Shards (mudança de N)

- `scheduler.shards.count` (N) **NÃO É** recarregável isoladamente em
  runtime — é uma operação de manutenção coordenada com `027-Cluster`.
- A função `ResolveShard` **DEVE** usar **hashing consistente** (ADR-0092)
  para que uma mudança de `N` (ex.: 64→65) remapeie apenas uma fração
  próxima de `1/N` das chaves — não uma reorganização completa — minimizando
  migração de filas/tarefas em trânsito.
- Durante o rebalanceamento: (1) o novo mapa shard→nó é publicado via evento
  `aios.<tenant>.cluster.capacity.changed`; (2) réplicas atualizam
  `CapacityProvider`/`ShardRouter`; (3) entradas de fila associadas a shards
  remapeados são gradualmente drenadas e reenfileiradas sob o novo `N`,
  preservando `aging` acumulado (sem penalizar tarefas antigas pela
  reorganização).
- Rebalanceamento **NÃO DEVE** causar overcommit ou perda de tarefas —
  `QuotaLedger` e `PriorityQueueStore` permanecem consistentes durante a
  transição (verificado por teste de rebalanceamento, FR-023).

---

## 10. Limites Teóricos e Rumo a 10⁶⁺ Agentes

| Dimensão | Limite/estratégia |
|----------|----------------------|
| Agentes/tarefas simultâneos | Sharding distribui carga em `N` partições independentes; cada shard suporta um subconjunto finito e previsível de filas ativas — crescimento de agentes é absorvido por aumento de `N` e/ou réplicas, não por degradação da estrutura de dados (ZSET permanece O(log n) por operação). |
| Coordenação entre réplicas | Limitada a aquisição/renovação de lease de ownership de shard (baixa frequência, não por requisição) — **não** há troca de mensagens O(N²) entre réplicas por decisão individual (alinhado ao princípio de *fan-out* limitado do Glossário). |
| Throughput por réplica | Meta ≥ 50.000 decisões/s por réplica (NFR-002); escala linear por réplica/shard adicional — validado em `Benchmark.md`. |
| Comunicação seletiva | O Scheduler não faz *broadcast* de estado a todos os agentes/tenants — cada decisão é ponto-a-ponto (`Submit`→`SchedulingDecision`) ou publicada uma única vez no subject do tenant afetado, evitando custo de coordenação O(N²) mesmo em altíssima escala. |
| Fairness sob alta escala | `min_share` por tenant (§11) garante que o crescimento do número de tenants não permita que um único tenant grande monopolize shards compartilhados. |
| Meta declarada | NFR-004: crescimento de 1 → 10⁶⁺ agentes com crescimento **sub-linear** de coordenação, validado por benchmark de escala incremental (dobrar N/réplicas e medir throughput/latência). |

---

## 11. Fairness sob Escala (Anti-Starvation)

Mesmo sob sharding, um shard específico pode concentrar tarefas de um tenant
com alto volume. Para evitar que isso degrade a experiência de outros
tenants/classes que compartilham o mesmo shard:

- **Aging**: toda `QueueEntry` tem seu `Score` ajustado por
  `scheduler.aging.factor_per_sec`, limitado a `scheduler.aging.max_boost`,
  garantindo que tarefas mais antigas eventualmente sejam despachadas mesmo
  sob fluxo constante de novas tarefas de maior prioridade nominal (FR-009).
- **`min_share`**: `scheduler.fairness.min_share` (default 1%) garante que
  todo tenant ativo receba uma fração mínima de despachos em uma janela de
  observação (default 60 s), mensurada por métrica de fairness por tenant
  (NFR-006).
- Estas garantias são **por shard e agregadas por tenant** — um tenant cujas
  tarefas se espalham por múltiplos shards (via `hash(tenant,agent) mod N`)
  tem sua vazão mínima calculada de forma agregada, não por shard isolado.

---

## 12. Riscos de Escala e Mitigações

| Risco | Descrição | Mitigação |
|-------|-----------|-----------|
| *Hot shard* (tenant/agente com volume desproporcional em um único shard) | Um `agent_id` de altíssimo volume satura seu shard específico, afetando outros tenants colocalizados. | `min_share` (§11) protege outros tenants; monitoramento de profundidade de fila por shard (`Deployment.md` §9) permite intervenção operacional (ex.: isolamento dedicado para tenants outliers, decisão de capacidade em `027`). |
| Cache de política/capacidade desatualizado sob alta taxa de mudança | Sob mudança muito frequente de orçamento/capacidade, o cache de curta duração do caminho quente pode admitir com dado levemente obsoleto. | TTL curto + invalidação orientada a evento; nunca overcommit de slot (garantido por reserva atômica independente do cache, NFR-005). |
| Rebalanceamento sob carga | Mudança de `N` durante pico de tráfego pode temporariamente aumentar latência de placement. | Rebalanceamento é operação planejada (não automática/reativa); executada preferencialmente em janela de baixo tráfego, com hashing consistente minimizando o volume afetado. |
| Preempção em escala (*thrashing* distribuído) | Sob múltiplos shards preemptando simultaneamente, o sistema pode oscilar entre admitir/preemptar as mesmas classes de tarefa. | Histerese e cooldown por tarefa (ADR-0094); métrica `aios_scheduler_preemption_ms`/taxa de preempção monitorada por shard. |
| Contenção de outbox sob alto volume de decisões | Alto throughput de `DecisionJournal.RecordAndPublish` pode gerar contenção de escrita em PostgreSQL. | Particionamento de tabela por tempo (`Database.md`), lote de publicação (*batching*) no `EventPublisher`, e paralelismo do relay de outbox por partição. |

---

## 13. Metodologia de Validação (Benchmark)

A validação empírica dos limites descritos acima é responsabilidade de
`Benchmark.md` (documento de outro batch), que **DEVE** cobrir, no mínimo:

- Throughput sustentado de `Submit` por réplica (meta NFR-002: ≥ 50.000/s).
- Latência de decisão p50/p99 sob carga crescente (meta NFR-001).
- Escala horizontal: throughput agregado ao dobrar réplicas/shards (deve
  aproximar-se de crescimento linear).
- Rebalanceamento de `N`: fração de chaves remapeadas medida empiricamente
  contra a previsão teórica de hashing consistente (~1/N).
- Fairness sob carga adversarial: um tenant satura a fila; demais tenants
  ainda recebem ≥ `min_share` de despachos (NFR-006).
- Comportamento de backpressure sob saturação incremental: transições
  `accept→defer→reject` ocorrem nos limiares configurados, sem overcommit.

---

## 14. Referências

- Modelo de escala fixado na fonte de verdade: `_DESIGN_BRIEF.md` §10.
- Arquitetura e camadas do caminho de decisão: `./Architecture.md` §9.
- Estruturas de dados e invariantes de concorrência: `./ClassDiagrams.md` §5, §9.
- Topologia de implantação e ownership de shard: `./Deployment.md` §5.
- Modos de falha sob saturação/partição: `./FailureRecovery.md`.
- Requisitos não-funcionais de escala: `./NonFunctionalRequirements.md` (NFR-001, NFR-002, NFR-004, NFR-006).
- Glossário: `../040-Glossary/Glossary.md` (termos: Shard, Backpressure, Fan-out, Cold Agent, Hot State).
- Inventário de capacidade/topologia: `../027-Cluster/`.
