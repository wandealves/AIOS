---
Documento: Scalability
Módulo: 007-Agent-Runtime
Status: Draft
Versão: 0.1
Última atualização: 2026-07-21
Responsável (RACI-A): Arquiteto do Módulo 007 — Agent Runtime
ADRs relacionados: ADR-0073, ADR-0074, ADR-0077, ADR-0079
RFCs relacionados: RFC-0001
Depende de: ./_DESIGN_BRIEF.md, ./Deployment.md, ./Database.md
---

# 007-Agent-Runtime — Scalability

> Modelo de escala e concorrência do Agent Runtime, elaborando
> `./_DESIGN_BRIEF.md` §10. Estratégia central herdada de
> `../001-Architecture/Architecture.md` §12: **agentes efêmeros e
> *cold-storable*** + **sharding determinístico** + **comunicação
> seletiva**.

## Índice

1. Modelo de escala (horizontal/vertical)
2. Particionamento e sharding
3. Concorrência intra-processo
4. Locks e exclusão mútua
5. Backpressure
6. Fan-out de comunicação e isolamento de ruído
7. Caminho de escala (1 → 10⁶+)
8. Limites teóricos
9. Diagrama ASCII — arquitetura de escala
10. Referências

---

## 1. Modelo de Escala (Horizontal/Vertical)

| Eixo | Estratégia |
|------|-------------|
| **Horizontal** (dimensão primária) | Mais **nós de execução** hospedando mais processos runtime, cada um limitado por `runtime.pool.max_sessions_per_node` (default 500, NFR-004). O Runtime Supervisor distribui o *placement* entre nós (fora do escopo deste módulo). |
| **Vertical** (dimensão secundária) | Nós maiores (mais vCPU/RAM) sustentam mais sessões `busy` simultâneas até o limite configurado, mas o principal ganho de escala vem de **liberar RAM de sessões ociosas** (modelo *cold-storable*, §7), não de nós individualmente maiores. |
| **Escala por hibernação** (dimensão dominante em volume) | A maioria dos agentes em um instante qualquer está `suspended` (checkpointado, ~0 RAM) — a escala a 10⁶+ agentes (NFR-004) depende **muito mais** da eficiência de checkpoint/resume do que de CPU/RAM agregada do cluster. |

---

## 2. Particionamento e Sharding

```
shard = hash(tenant_id, agent_id) mod N
```

- Cada `RuntimeInstance`/pool atende um conjunto de shards (afinidade),
  garantindo que sessões de um mesmo agente sejam preferencialmente
  materializadas em runtimes próximos ao seu shard lógico — reduz
  necessidade de mover blobs de checkpoint entre regiões/AZs
  desnecessariamente.
- Sessões de um mesmo agente **sempre** no mesmo shard, evitando
  checkpoints cross-shard e simplificando a reconciliação de `step_cursor`.
- O particionamento físico das tabelas duráveis (`runtime.agent_session`,
  `runtime.react_step`) segue estratégia própria (`HASH(tenant_id)`,
  `RANGE(created_at)`) documentada em `./Database.md` §3 — **independente**
  do `shard` lógico de aplicação usado para afinidade de runtime.

---

## 3. Concorrência Intra-processo

- O loop cognitivo é **assíncrono** (`asyncio`), não-bloqueante para toda
  I/O de LLM/tool/memória — múltiplas operações de I/O pendentes (ex.:
  aguardando `ModelRouterClient` e, em paralelo, gravando telemetria) não
  bloqueiam o *event loop* do processo (ADR-0077).
- **Um agente por processo** (isolamento forte, Princípio 1 da Visão
  Global): não há multiplexação de múltiplos agentes dentro do mesmo
  processo do SO — a concorrência de *agentes* é sempre **entre
  processos**, nunca dentro de um.
- Paralelismo de invocação de tools **dentro** de um mesmo agente é
  limitado por `loop.max_tool_fanout` (default 8) — um agente não pode
  disparar um número ilimitado de chamadas simultâneas mesmo que a
  infraestrutura subjacente suportasse mais.

---

## 4. Locks e Exclusão Mútua

| Recurso protegido | Mecanismo | Granularidade |
|---------------------|-----------|------------------|
| Execução de uma `AgentSession` | *Lease* distribuído (Redis) por `session_id`, com TTL curto renovado por heartbeat. | Por sessão — garante **exatamente um** runtime ativo (ADR-0079). |
| `Resume` concorrente da mesma sessão | Aquisição do *lease* é pré-condição obrigatória (**G9**, `./StateMachine.md` T14). | Idem. |
| Cache de decisão de capability | Sem lock — cache local por processo, invalidado por evento (`aios.<t>.policy.decision.updated`); leitura/escrita local, sem contenção distribuída. | Por processo. |

O runtime **deliberadamente evita** locks pessimistas distribuídos no
caminho quente do loop (passos ReAct não competem por um recurso
compartilhado entre processos) — a única exclusão mútua distribuída
necessária é a de **sessão ativa**, resolvida pelo *lease* de `Resume`.

---

## 5. Backpressure

| Ponto de saturação | Mecanismo de proteção | Efeito ao saturar |
|------------------------|----------------------------|------------------------|
| Admissão de novas sessões por nó | `runtime.pool.max_sessions_per_node` | Novo `Boot` é recusado (`AIOS-RUNTIME-0001`, tratado no nível do Supervisor antes de alcançar a instância). |
| Publicação de eventos | Outbox local com `events.outbox.flush_interval_ms`/`max_retries`; JetStream aplica *flow control* nativo. | Backlog cresce (`aios_runtime_events_outbox_pending`); alerta antes de DLQ (`./FailureRecovery.md` §4). |
| Runtime em `draining` | Recusa novas tarefas. | `AIOS-RUNTIME-0013`. |
| Fan-out de tools | `loop.max_tool_fanout` | Invocação excedente rejeitada antes de despachar (`AIOS-RUNTIME-0012`). |
| Admissão global (fora deste módulo) | `009-Scheduler` decide fila/preempção antes de qualquer `Boot` chegar ao runtime. | Fora do escopo — o runtime é sempre a "última milha" da decisão de admissão. |

---

## 6. Fan-out de Comunicação e Isolamento de Ruído

- Eventos de execução (`task.execution.step`) são de **alta frequência**
  por sessão; para evitar custo O(N) de observabilidade proporcional ao
  número de agentes ativos crescendo sem controle, `telemetry.sampling_ratio`
  permite reduzir a taxa de amostragem de **traces** (não de eventos de
  negócio, que continuam 100% publicados — a amostragem afeta apenas a
  granularidade de tracing detalhado, preservando o rastro funcional
  completo via `ReActStep`).
- Heartbeat (`heartbeat.interval_ms`) é enviado por instância, não por
  sessão — o custo de heartbeat escala com o número de **processos**, não
  com o número de agentes lógicos (a maioria hibernados não gera
  heartbeat algum, pois não ocupam um processo).
- **Bulkhead por sandbox:** `cgroups v2` garante que um agente com
  comportamento anômalo (CPU/RAM/PIDs) não degrada outros agentes no mesmo
  nó — isolamento de ruído por construção, não por convenção operacional.

---

## 7. Caminho de Escala (1 → 10⁶+)

| Faixa de agentes ativos+hibernados | Estratégia dominante |
|----------------------------------------|---------------------------|
| **1 – 10³** | Pool único, `warm_size` pequeno, um nó de execução é suficiente; foco em correção funcional. |
| **10³ – 10⁵** | Múltiplos pools/nós; sharding de sessão por `hash(tenant_id, agent_id) mod N`; cluster NATS com múltiplos nós; maioria das sessões já em ciclo ativo/suspenso equilibrado. |
| **10⁵ – 10⁶+** | Maioria dos agentes **majoritariamente `cold`** (`suspended`, checkpointado); pools distribuídos por AZ/região; `Resume` sob demanda com meta agressiva (< 250 ms, alinhada a NFR-001, ainda que o caminho de `Resume` não seja idêntico ao de `Boot` frio — ambos compartilham a mesma meta de latência de materialização); custo de infraestrutura dominado por **armazenamento de checkpoint** (MinIO/PG), não por CPU/RAM de processos ativos. |

Este caminho é o pré-requisito estrutural direto da meta **V-R1** da Visão
Global (10⁶+ agentes concorrentes distribuídos) — ver `./Vision.md` §6.

---

## 8. Limites Teóricos

| Dimensão | Limite | Fator limitante |
|-----------|--------|--------------------|
| Sessões `busy` por nó | `runtime.pool.max_sessions_per_node` (config, default 500) | CPU/RAM/PIDs agregados do nó sob `cgroups v2`, respeitando `sandbox.*` por sessão. |
| Passos de loop por sessão | `loop.max_steps` (config, default 50) | Orçamento cognitivo — não é um limite de infraestrutura, é um limite de **governança** (evita loops improdutivos). |
| Fan-out de tools por passo | `loop.max_tool_fanout` (config, default 8) | Contenção de custo de coordenação O(N²) dentro de um único agente. |
| Throughput de eventos por nó | ≥ 5.000 msg/s (NFR-007) | Capacidade de flush do outbox local + capacidade do cluster NATS/JetStream. |
| Agentes hibernados totais | Sem limite teórico rígido do runtime em si | Limitado pela capacidade de armazenamento durável (PostgreSQL `runtime.*` + MinIO), fora deste módulo. |

---

## 9. Diagrama ASCII — Arquitetura de Escala

```
                         Região/AZ 1                          Região/AZ 2
                 ┌─────────────────────────┐          ┌─────────────────────────┐
                 │  Pool de nós (shards 0-31)│          │  Pool de nós (shards 32-63)│
                 │  ┌─────┐ ┌─────┐ ┌─────┐  │          │  ┌─────┐ ┌─────┐ ┌─────┐  │
                 │  │ nó 1 │ │ nó 2 │ │ nó N │  │  ...     │  │ nó 1 │ │ nó 2 │ │ nó N │  │
                 │  └─────┘ └─────┘ └─────┘  │          │  └─────┘ └─────┘ └─────┘  │
                 └───────────┬─────────────┘          └───────────┬─────────────┘
                              │                                     │
                              ▼                                     ▼
                 ┌─────────────────────────────────────────────────────────┐
                 │        NATS/JetStream cluster · Redis cluster              │
                 └─────────────────────────────────────────────────────────┘
                              │
                              ▼
                 ┌─────────────────────────────────────────────────────────┐
                 │  PostgreSQL (runtime.*, RLS) · MinIO (checkpoints)          │
                 │  10⁵-10⁶+ AgentSession, majoritariamente `suspended`         │
                 └─────────────────────────────────────────────────────────┘
```

---

## 10. Referências

- Estratégia central (fonte): `./_DESIGN_BRIEF.md` §10.
- Modelo de dados e particionamento físico: `./Database.md` §3.
- Metas numéricas de escala: `./NonFunctionalRequirements.md` (NFR-004, NFR-010).
- Benchmarks de densidade/cold start: `./Benchmark.md`.
- *Lease* distribuído e prevenção de dupla execução: `./FailureRecovery.md` §2.
- Arquitetura global de escala: `../001-Architecture/Architecture.md` §12.
