---
Documento: Scalability
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0081 (Cold Agent Hibernation em MinIO+PostgreSQL), ADR-0083 (WarmPool/materialização < 250 ms), ADR-0084 (Lease + fencing token), ADR-0085 (Migração como saga), ADR-0086 (Event sourcing + Outbox)
RFCs relacionados: RFC-0001 (baseline), RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore Protocol — a propor)
Depende de: 001-Architecture, 006-Kernel, 009-Scheduler, 020-Communication, 027-Cluster
---

# 008-Agent-Lifecycle — Scalability

> **Escopo.** Este documento especifica o modelo de escala do módulo Agent
> Lifecycle: como ele escala horizontalmente, como particiona o estado
> (`AgentControlBlock`), como o modelo hot/cold de hibernação sustenta
> `≥ 10⁶` agentes, como resolve concorrência sem locks pessimistas
> (lease + fencing granular por agente), como aplica backpressure, e quais
> são os limites teóricos conhecidos no caminho para milhões de agentes.
> Consolida e aprofunda `_DESIGN_BRIEF.md` §10, sem contradizê-lo.

---

## 1. Modelo de Escala Geral

O serviço Agent Lifecycle é projetado como **stateless horizontalmente
escalável**: nenhuma réplica mantém estado que não possa ser reconstruído a
partir de PostgreSQL (`AcbStore`, fonte da verdade), Redis (lease, projeção
quente) ou MinIO (`SnapshotStore`, blobs de checkpoint). Isso permite que a
capacidade do módulo cresça **linearmente com o número de réplicas**,
limitada apenas pela capacidade das dependências compartilhadas
(PostgreSQL, Redis, MinIO, NATS, PDP, Scheduler, Runtime).

| Dimensão | Estratégia de escala |
|----------|-------------------------|
| Escala do **serviço** | **Horizontal** — réplicas idênticas sem afinidade obrigatória (§2). |
| Escala do **estado do ACB** | **Sharding determinístico** + separação hot/cold via hibernação (§3). |
| Escala de **concorrência por agente** | **Lease granular + fencing token** (Redis), sem lock global (§4). |
| Escala de **eventos** | Outbox transacional desacoplado do caminho quente + JetStream (§6). |
| Escala de **cold-start** | WarmPool por tenant/shard elimina boot completo de processo de runtime (§5). |

---

## 2. Escala Horizontal do Serviço

```
                     ┌──────────────── Gateway (YARP) — health-aware LB ────────────────┐
                     ▼                          ▼                            ▼
             ┌───────────────┐         ┌───────────────┐            ┌───────────────┐
             │ Lifecycle      │  ...    │ Lifecycle      │    ...     │ Lifecycle      │
             │ réplica i      │         │ réplica j      │            │ réplica k      │
             │ (stateless)    │         │ (stateless)    │            │ (stateless)    │
             └───────┬────────┘         └───────┬────────┘            └───────┬────────┘
                     │                          │                              │
                     └──────────────┬───────────┴───────────────┬─────────────┘
                                    ▼                            ▼
                         Redis (lease/fencing,          PostgreSQL (ACB, transições,
                         ACB quente) — hash slot         outbox, ckpt idx) — particionado
                         nativo do Redis Cluster         por tenant_id + placement_shard
                                    │
                                    ▼
                              MinIO (blobs de checkpoint,
                              particionado por bucket/prefixo de tenant)
```

- Qualquer réplica **DEVE** poder processar qualquer comando de qualquer
  tenant/agente — não há particionamento de tráfego obrigatório no Gateway.
- Réplicas **PODEM** manter cache local em memória de projeções recém
  acessadas do ACB (otimização de latência), mas esse cache **NÃO É** fonte
  da verdade e **DEVE** ser invalidável (TTL curto ou invalidação por
  evento de transição).
- Adicionar/remover réplicas **NÃO REQUER** coordenação de estado — é uma
  operação puramente de infraestrutura (ver `Deployment.md` §9).

---

## 3. Particionamento (Sharding) e Modelo Hot/Cold

### 3.1 Função de sharding

```
placement_shard = hash(tenant_id, agent_id) mod N     (N conforme ../001-Architecture/Architecture.md §12)
```

- `placement_shard` é um campo **gerado e persistido** no `AgentControlBlock`
  (`_DESIGN_BRIEF.md` §3.1), calculado na criação do ACB (herdado da decisão
  de placement do Scheduler/Kernel, não recalculado pelo módulo 008).
- O sharding serve a dois propósitos: (a) **localidade de cache** — uma
  réplica que atendeu recentemente um shard tem maior probabilidade de cache
  hit local; (b) **particionamento físico do PostgreSQL** por
  `tenant_id`/`placement_shard` (`Database.md`), permitindo *table
  partitioning* e paralelismo de manutenção (vacuum, reindex) sem contenção
  global.

### 3.2 Estado quente (hot) vs. frio (cold)

| Estado (`LifecycleState`) | Onde reside | Latência de acesso | Custo de RAM/infra |
|-----------------------------|--------------|------------------------|------------------------|
| `Created`/`Ready` | PostgreSQL (durável) + projeção Redis | Sub-milissegundo (Redis) | Baixo (aguardando slot). |
| `Running`/`Suspended` | Redis (projeção quente) + PostgreSQL (durável); working memory em RAM/Redis via `010` | Sub-milissegundo (Redis) | Alto por agente ativo. |
| `Migrating` | Estado transitório em ambos os lados (origem/destino) durante a saga | Variável (fase da saga) | Transitório. |
| `Hibernated` | Somente PostgreSQL (metadado) + snapshot durável em MinIO | Dezenas–centenas de ms (materialização sob demanda) | Baixíssimo — **RAM liberada** (R-05 do brief), apenas linha em PostgreSQL. |
| `Terminated`/`Failed` | Somente PostgreSQL (retido para auditoria até expurgo) | Não aplicável (somente leitura histórica) | Mínimo. |

A transição para `Hibernated` (T6 da FSM, `_DESIGN_BRIEF.md` §4.3) é o
mecanismo central que permite suportar **≥ 10⁶ agentes** (NFR-004) com custo
de infraestrutura sub-linear: em qualquer instante, `≤ 5%` dos agentes estão
ativos em RAM (`Running`/`Suspended`); a vasta maioria está `Hibernated`,
custando aproximadamente uma linha de PostgreSQL + um blob comprimido/cifrado
em MinIO, sem consumir Redis ou RAM de runtime.

```
                        distribuição típica de 10⁶ agentes em um instante
   ┌──────────────────────────────────────────────────────────────────────────────┐
   │ Hibernated  ████████████████████████████████████████████████████████  ≥ 95% │  ← PostgreSQL + MinIO, 0 RAM
   │ Running/Suspended ███                                                  ≤ 5%  │  ← Redis + RAM
   │ Created/Ready/Migrating/Terminated/Failed  ▏                         ~0%    │  ← transitório/histórico
   └──────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Índice quente mínimo para roteamento

Mesmo agentes `Hibernated` mantêm um **índice quente mínimo** em Redis
(`≤ 4 KiB` por agente, NFR-011) contendo apenas o necessário para roteamento
rápido e decisão de wake (`agent_id`, `tenant_id`, `state`, `placement_shard`,
`last_checkpoint_id`, `cold_since`) — não o conteúdo da working memory, que
permanece exclusivamente no `Checkpoint` cifrado em MinIO. Esse índice é o
que permite que a decisão "posso materializar este agente e em quanto tempo"
não exija uma consulta de leitura pesada ao PostgreSQL no caminho crítico de
`wake`.

---

## 4. Concorrência por Agente (Lease + Fencing Token)

### 4.1 Por que lease granular, não lock global

Conflitos de escrita concorrente sobre o **mesmo** agente (ex.: `suspend` e
`migrate` disparados quase simultaneamente) são a única forma de corrida que
o módulo 008 precisa serializar — não há necessidade de coordenação global
entre agentes distintos, que são independentes por design. Um lock global
seria um gargalo artificial de escala; a arquitetura usa **um lease por
`agent_id`** (INV1 do brief: uma transição por agente por vez):

```
   Comando A (suspend)                LeaseManager (Redis)              Comando B (migrate)
        │                                     │                                │
        │  SET NX PX aios:{t}:lc:lease:{id}   │                                │
        │────────────────────────────────────▶│                                │
        │◀── OK, fencing_token=42 ─────────────│                                │
        │                                     │                                │
        │  aplica transição com token=42       │   SET NX PX (mesma chave)      │
        │                                     │◀───────────────────────────────│
        │                                     │──── falha (chave já existe) ───▶│
        │                                     │                                │  AIOS-LIFECYCLE-0003
        │  renova lease (PX) periodicamente    │                                │  (409, retriable)
        │────────────────────────────────────▶│                                │
        │  libera lease ao concluir            │                                │
        │────────────────────────────────────▶│                                │
```

- `LeaseManager` usa `SET NX PX` (Redis) com TTL curto
  (`lease.ttl_ms`, default 5000) e renovação periódica
  (`lease.renew_ms`, default 1500) enquanto o efeito estiver em andamento.
- O `fencing_token` retornado ao adquirir a lease é **monotônico** e
  persistido no `AgentControlBlock` (`fencing_token`). Toda escrita de efeito
  colateral externo (checkpoint, migração, chamada a `007`/`009`) **DEVE**
  validar que o token que está usando ainda é o corrente antes de persistir
  seu resultado — se um coordenador perde a lease (ex.: por GC pause ou
  falha de rede) e outro a adquire, o token antigo se torna inválido e
  qualquer escrita tardia do coordenador antigo é rejeitada com
  `AIOS-LIFECYCLE-0013` (409). Ver ADR-0084.
- Este desenho escala **horizontalmente por shard**: não existe um único
  ponto de serialização global — cada agente tem sua própria chave de lease,
  distribuída pelo hash slot nativo do Redis Cluster.

### 4.2 Custo de latência do lease no caminho quente

Transições curtas e determinísticas da FSM (decisão pura, sem I/O externo)
são o alvo de NFR-002 (`p99 ≤ 20 ms` excluindo efeitos I/O) — o custo de
adquirir/renovar/liberar o lease é parte desse orçamento e **DEVE** ser
medido separadamente (`aios_lifecycle_transition_duration_ms`) do custo dos
efeitos colaterais (spawn/checkpoint/migração), que têm seus próprios SLOs
(NFR-001, NFR-003, NFR-010).

---

## 5. WarmPool e Redução de Cold-Start

### 5.1 Problema

Materializar um agente frio (`Hibernated → Running`) ou novo (`Created →
Running`) exige, no caminho ingênuo, provisionar um processo de runtime
completo (`007-Agent-Runtime`) antes mesmo de restaurar o checkpoint — custo
tipicamente muito acima do SLO de `p99 ≤ 250 ms` (NFR-001).

### 5.2 Solução: pool de runtimes pré-aquecidos

```
   WarmPoolManager mantém, por tenant/shard:
   ┌─────────────────────────────────────────────────────────┐
   │  [runtime pronto]  [runtime pronto]  [runtime pronto]   │  ← warmpool.min_per_shard (default 4)
   │        ...  até warmpool.max_per_shard (default 64)     │
   └─────────────────────────────────────────────────────────┘
                          │
                          ▼ SpawnManager retira 1 runtime do pool
             ┌─────────────────────────────────────┐
             │ 1. attach de contexto/memória (010/011)│  ← apenas deserialização,
             │ 2. aplica snapshot (se cold)           │     não boot de processo
             │ 3. runtime assume identidade do agente │
             └─────────────────────────────────────┘
                          │
                          ▼
                    generation += 1 → Running
```

- A materialização torna-se **deserialização + attach**, não *boot*
  completo de processo — o custo dominante passa a ser I/O de leitura do
  checkpoint (MinIO) e restauração de ponteiros de memória (010/011), ambos
  dentro do orçamento de `p50 ≤ 80 ms` / `p99 ≤ 250 ms` (NFR-001).
- `warmpool.min_per_shard`/`warmpool.max_per_shard` (`_DESIGN_BRIEF.md` §8)
  controlam a reserva mínima e o teto de runtimes ociosos por tenant/shard;
  reserva excessiva desperdiça recursos ociosos, reserva insuficiente
  degrada o p99 de spawn sob rajada — dimensionamento é responsabilidade de
  `Benchmark.md`.
- Runtimes do WarmPool **NÃO DEVEM** reter identidade ou dado de agente
  algum antes da atribuição — são fungíveis entre agentes até o momento do
  `attach`.

---

## 6. Escala de Eventos e CQRS

- **CQRS**: escrita append-only de `LifecycleTransition` (event-sourced);
  leitura por projeção `AgentControlBlock` (cache quente em Redis) — escala
  leitura de estado sem tocar o log de transições, e permite reconstrução
  de auditoria histórica sem impacto no caminho quente.
- **Outbox + JetStream**: a publicação de eventos `agent.lifecycle.*` é
  desacoplada do caminho quente de transição via `LifecycleOutbox`
  (`outbox.publish_batch`, default 256) — a mutação do ACB é confirmada
  independentemente da disponibilidade do NATS no instante da escrita,
  permitindo throughput `≥ 50.000 transições/s` (NFR-008) sem que o
  publicador se torne o gargalo.
- **Nenhum lock distribuído global** — apenas leases de granularidade por
  agente (§4); transições da FSM são operações curtas e determinísticas.

---

## 7. Backpressure e Isolamento de Noisy Neighbor

| Camada | Mecanismo | Efeito |
|--------|-----------|--------|
| Materialização por nó | `migration.max_concurrent_per_node` (default 8) | Um storm de `wake`/`migrate` não satura um único nó de destino; excesso é enfileirado, não rejeitado abruptamente. |
| Admissão de `spawn`/`wake` | Delegada ao `009-Scheduler` (admission control) | O módulo 008 propaga a rejeição do Scheduler como `AIOS-LIFECYCLE-0005` (429, retriable); não tenta contornar a decisão de capacidade. |
| Hibernação sob pressão de RAM | `hibernation.ram_pressure_threshold_pct` (default 85) | `HibernationController` prioriza candidatos por `priority_class`/idle quando a pressão de RAM do cluster cruza o limiar, hibernando proativamente antes de esgotar memória. |
| Saga de migração | `migration.saga_timeout_ms` (default 120000) | Migração travada é compensada (origem reativada) em vez de reter recursos indefinidamente em ambos os lados. |
| Fan-out de eventos consumidos | Consumo via JetStream com backpressure nativo | Um produtor rápido de `scheduler.placement.decided`/`cluster.node.draining` não esgota a capacidade de processamento do módulo 008 — consumidores aplicam *pull* controlado. |

**Princípio de rejeição precoce:** sob saturação, o módulo 008 **DEVERIA**
rejeitar rapidamente com `429`/`503` (retriable) em vez de enfileirar
comandos indefinidamente ou degradar a latência de todas as requisições
igualmente — rejeição precoce e localizada preserva o SLO das demais
requisições (isolamento de *noisy neighbor*).

---

## 8. Limites Teóricos e Capacidade

| Métrica | Meta/limite conhecido | Fonte |
|---------|--------------------------|-------|
| Agentes suportados por cluster | `≥ 10⁶`, `≤ 5%` ativos em RAM simultaneamente | NFR-004 |
| Latência p99 de spawn/materialização (cold→Running) | `≤ 250 ms` (`p50 ≤ 80 ms`) | NFR-001 |
| Latência p99 de decisão de transição (excl. I/O) | `≤ 20 ms` | NFR-002 |
| Latência p99 de checkpoint (working memory ≤ 64 MiB) | `≤ 500 ms` | NFR-003 |
| Throughput de transições por cluster | `≥ 50.000 transições/s` | NFR-008 |
| Latência p99 de migração (agente ≤ 64 MiB, mesmo DC) | `≤ 2 s` | NFR-010 |
| Overhead de ACB frio em índice quente (Redis) | `≤ 4 KiB` | NFR-011 |

### 8.1 Capacity Planning — estimativa por réplica

| Cenário | Réplicas estimadas (referência inicial, sujeita a `Benchmark.md`) |
|---------|------------------------------------------------------------------------|
| 50.000 transições/s agregadas (mix de controle) | ≥ 5 réplicas (com margem de 20% acima do throughput nominal por réplica). |
| 10⁶ agentes totais, `≤ 5%` ativos simultaneamente | ~50.000 agentes quentes em Redis no pior caso; réplicas dimensionadas pelo throughput de transição/materialização, não pela contagem total de agentes (majoritariamente `Hibernated`, custo quase nulo por unidade). |
| Pico de `wake` (rajada de retomada em massa, ex.: reabertura de negócio de um tenant) | Absorvido primariamente por `WarmPoolManager` + admissão do `009-Scheduler`; módulo 008 aplica backpressure via `migration.max_concurrent_per_node` equivalente para materialização em massa. |
| Nó em drenagem (`cluster.node.draining`) com milhares de agentes quentes | Migração em lote respeita `migration.max_concurrent_per_node`; tempo total de drenagem é função do número de agentes quentes no nó, não do total do cluster. |

> Estes números são **estimativas de referência**; a validação empírica
> definitiva (carga sintética, ponto de saturação real, ponto de ruptura) é
> responsabilidade de `./Benchmark.md`, que **DEVE** ser mantido consistente
> com estas metas ou justificar desvio via ADR.

---

## 9. Caminho para Milhões de Agentes

A combinação de estratégias que permite escalar o módulo 008 de milhares
para milhões de agentes geridos é:

1. **Predominância de agentes *cold*** — a maioria dos agentes em qualquer
   instante está `Hibernated`, custando apenas uma linha de PostgreSQL + um
   blob em MinIO (§3.2).
2. **Sharding determinístico** — `hash(tenant,agent) mod N` distribui carga
   e habilita particionamento físico do banco (§3.1, `Database.md`).
3. **Lease granular + fencing token** em vez de lock global — elimina
   contenção de coordenação como gargalo de escala horizontal (§4).
4. **WarmPool** — reduz materialização a deserialização + attach, não boot
   completo de processo, mantendo o SLO de `spawn`/`wake` sob escala (§5).
5. **CQRS + Outbox assíncrono** — desacopla o caminho quente de transição da
   publicação de eventos, sustentando alto throughput (§6).
6. **Backpressure em múltiplas camadas** (Scheduler, RAM, saga de migração)
   — evita que rajadas se transformem em degradação generalizada (§7).
7. **Índice quente mínimo** (`≤ 4 KiB`/agente hibernado) — permite roteamento
   e decisão de wake rápidos mesmo com `10⁶` agentes indexados
   simultaneamente (§3.3).

Esta combinação é consistente com a estratégia global de escala descrita em
`../001-Architecture/Architecture.md` §12, à qual este documento se alinha
sem redefinir.

---

## 10. Riscos de Escala e Mitigação

| Risco | Descrição | Mitigação |
|-------|-----------|-------------|
| Hot shard | Um único `tenant`/`placement_shard` concentra volume anormal de agentes ativos simultaneamente. | Sharding herdado da decisão do Scheduler/Kernel (fora do controle direto do 008); cotas por tenant limitam o volume máximo individual de agentes ativos. |
| Storm de hibernação sob pressão de RAM | Muitos agentes elegíveis a `hibernate` ao mesmo tempo sobrecarregam `CheckpointService`/`SnapshotStore`. | `HibernationController` prioriza por `priority_class`/idle; paralelismo de checkpoint limitado; backpressure na admissão de novos agentes (009). |
| Storm de `wake` (thundering herd) | Muitos agentes hibernados são acordados simultaneamente (ex.: início de janela de negócio de um tenant grande). | Leases evitam materialização duplicada do mesmo agente; `migration.max_concurrent_per_node`-equivalente limita paralelismo de materialização por nó de destino; WarmPool absorve parte do pico. |
| Crescimento não-linear de índices no PostgreSQL sob `10⁶+` linhas | Índices `(tenant_id, state)`, `(agent_id, seq)` degradam sem particionamento físico. | Particionamento de tabela por `tenant_id`/`placement_shard` (`Database.md`); manutenção (vacuum/reindex) por partição, não global. |
| MinIO como ponto de contenção sob storm de checkpoint | Alto volume de upload/download concorrente de blobs de checkpoint. | Cluster MinIO distribuído; compressão (`zstd`) reduz volume de I/O; paralelismo de `CheckpointService` limitado por configuração para não saturar largura de banda. |
| Reparticionamento custoso | Mudança na função de sharding (fora do escopo direto do 008, mas com impacto em `placement_shard`) exige recálculo em massa. | Migração planejada coordenada com `009-Scheduler`/`027-Cluster`; não é operação de configuração *hot* do módulo 008. |

---

## 11. Referências

- Modelo de dados e sharding do ACB: `./_DESIGN_BRIEF.md` §3, §10.
- Máquina de estados (hibernação/wake/migração): `./_DESIGN_BRIEF.md` §4.
- Requisitos não-funcionais relacionados: `./NonFunctionalRequirements.md` (NFR-001, NFR-002, NFR-003, NFR-004, NFR-008, NFR-010, NFR-011).
- Modelo físico de banco e particionamento: `./Database.md`.
- Topologia de implantação e autoescalonamento operacional: `./Deployment.md`.
- Modos de falha sob perda de Redis/MinIO/NATS: `./FailureRecovery.md`.
- Estratégia global de escala do AIOS: `../001-Architecture/Architecture.md` §12.
- Glossário: `../040-Glossary/Glossary.md` (Shard, Hot State, Cold Agent, Fan-out, Backpressure).
