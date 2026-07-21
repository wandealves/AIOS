---
Documento: ADR.md
Módulo: 009-Scheduler
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 009-Scheduler
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0008, ADR-0009, ADR-0010, ADR-0090, ADR-0091, ADR-0092, ADR-0093, ADR-0094, ADR-0095, ADR-0096, ADR-0097
RFCs relacionados: RFC-0001
Depende de: 001-Architecture, 006-Kernel, 022-Policy, 026-Cost-Optimizer, 027-Cluster, 002-ADR
---

# 009-Scheduler — Índice de ADRs

> **Escopo deste documento.** Este documento **NÃO** decide nada por si — é um
> **índice navegável** das *Architecture Decision Records* (ADR) que afetam o
> módulo 009-Scheduler. As decisões em si residem em `../002-ADR/`, no formato
> definido por `../002-ADR/TEMPLATE.md` (Contexto · Problema · Alternativas ·
> Análise · Escolha · Consequências · Riscos · Trade-offs). Este arquivo **DEVE**
> ser mantido sincronizado com `../009-Scheduler/_DESIGN_BRIEF.md` §11 e com
> `../002-ADR/README.md`; qualquer divergência é um defeito de documentação.

> **Convenção de numeração.** Conforme `./_DESIGN_BRIEF.md` §11, a faixa
> `ADR-0090`–`ADR-0099` está **reservada** ao módulo 009-Scheduler (regra
> `módulo × 10`, evitando colisão com faixas de outros módulos — ex.:
> `006-Kernel` usa `ADR-0060`–`ADR-0069`). Nenhuma outra numeração PODE ser
> usada por este módulo sem atualizar este índice e `../002-ADR/README.md`. Das
> dez posições reservadas, oito estão em uso (`ADR-0090`–`ADR-0097`); `ADR-0098`
> e `ADR-0099` permanecem **livres** para decisões futuras (ex.: uma eventual
> ADR sobre política multi-região de placement) e NÃO DEVEM ser preenchidas
> senão por proposta formal seguindo o processo da §4.

---

## 1. ADRs Globais que Restringem o Scheduler

Estas ADRs foram decididas na fundação canônica do AIOS (`../002-ADR/`) e
**DEVEM** ser respeitadas por qualquer ADR específica do Scheduler — elas não
são redecididas aqui, apenas referenciadas e especializadas para o domínio de
escalonamento.

| ADR | Título | Status | Como restringe o Scheduler |
|-----|--------|--------|------------------------------|
| [ADR-0001](../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md) | AIOS é um SO, não um framework | Accepted | O Scheduler é o **scheduler cognitivo** do AIOS — análogo ao CFS de um kernel Linux, não uma fila de jobs de terceiros embutida numa lib de orquestração. Admission control, placement e preempção (§1 do brief) são syscalls de sistema, não plugins de framework. |
| [ADR-0002](../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md) | Microserviços + split control/data plane | Accepted | O Scheduler vive inteiramente no **control plane** (.NET 10); ele **decide**, nunca **executa** — materialização de placement (spawn/suspend/kill) é sempre delegada ao `007-Agent-Runtime` (data plane) via `RuntimeSupervisorClient` (N-01/N-02 do brief). |
| [ADR-0003](../002-ADR/ADR-0003-DotNet-Control-Python-Runtime.md) | .NET 10 no controle, Python no runtime | Accepted | O `SchedulingGateway` e todo o pipeline de decisão (`AdmissionController` → `PriorityCostEvaluator` → `PlacementEngine` → `DispatchLoop`) são implementados em **.NET 10**, garantindo o caminho quente de baixa latência exigido por NFR-001 (p99 ≤ 20 ms). |
| [ADR-0004](../002-ADR/ADR-0004-NATS-como-Barramento.md) | NATS como barramento primário | Accepted | Todo evento do ciclo de decisão (`aios.<tenant>.task.execution.*`, §6 do brief) é publicado via **NATS/JetStream**, nunca via um bus alternativo; consumido de forma at-least-once com dedup por `event.id`. |
| [ADR-0005](../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md) | PostgreSQL + pgvector + Apache AGE | Accepted | O `DecisionJournal` usa **PostgreSQL** como fonte da verdade da proveniência de decisão (`SchedulingDecision`, event-sourced, append-only). O Scheduler não usa `pgvector`/AGE diretamente (não é seu domínio), mas herda a decisão de unificação de armazenamento relacional para dados frios/auditáveis. |
| [ADR-0006](../002-ADR/ADR-0006-Redis-Estado-Quente.md) | Redis para estado quente e locks | Accepted | `PriorityQueueStore` (ZSET), `QuotaLedger` (contadores atômicos), `CapacitySnapshot` e leases de dispatch/ownership de shard usam **Redis** exclusivamente — nenhum cache alternativo é introduzido no caminho quente. |
| [ADR-0008](../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md) | Governança por política, default deny | Accepted | Fundamenta diretamente o papel do `SchedulingGateway`/`AdmissionController`/`PreemptionManager` como **PEP**: toda admissão e preempção consulta o PDP (`022-Policy`) via `PolicyClient`, com postura *default deny* — se o PDP está indisponível, nega (`AIOS-SCHED-0004`), nunca admite por omissão. |
| [ADR-0009](../002-ADR/ADR-0009-Model-Router-Multiobjetivo.md) | Model Router multiobjetivo | Accepted | Estabelece o precedente arquitetural de **decisão multiobjetivo sob restrição** que o `PriorityCostEvaluator` especializa para escalonamento: onde o Model Router otimiza custo×qualidade×latência na escolha de modelo, o Scheduler otimiza `C = α·L + β·$ + γ·E` na escolha de prioridade/placement — mesma filosofia, domínio distinto (ver ADR-0091). |
| [ADR-0010](../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md) | Observabilidade e auditoria por construção | Accepted | Todo o `SchedulerTelemetry` (spans OTel, métricas `aios_scheduler_*`, logs Serilog→Seq) e a emissão de `aios.<tenant>.scheduler.decision.recorded` (para `025-Audit`) implementam esta ADR — observabilidade e proveniência não são adicionadas a posteriori, são parte do contrato de decisão (R-10 do brief). |

---

## 2. ADRs Específicas do Módulo 009-Scheduler (Propostas)

> **Estado de redação.** As ADRs abaixo estão **reservadas** na faixa
> `ADR-0090`–`ADR-0097` e seu conteúdo (Contexto/Problema/Alternativas/Análise/
> Escolha/Consequências/Riscos/Trade-offs) está **descrito no
> `./_DESIGN_BRIEF.md` §11** deste módulo, que é a fonte única de verdade até
> que cada ADR seja escrita como arquivo próprio em `../002-ADR/` e tenha seu
> `Status` promovido de `Proposed` para `Accepted`. Nenhum destes arquivos
> existe fisicamente em `../002-ADR/` no momento desta versão do índice; este
> documento resume o conteúdo pretendido para orientar quem for redigi-las.

| ADR | Título | Status | Resumo da decisão (1 linha) | Módulos afetados |
|-----|--------|--------|------------------------------|-------------------|
| ADR-0090 | Admission control com reserva atômica de slots em Redis (fail-safe, sem overcommit) | Proposed | Toda admissão reserva um slot via script Lua atômico em Redis (`sched:quota:*`) antes de confirmar `ADMITTED`; falha do Redis degrada para rejeição conservadora, nunca overcommit. | 009, 006, 026 |
| ADR-0091 | Função de score multiobjetivo `C = α·L + β·$ + γ·E` e normalização de pesos por tenant/classe | Proposed | Adotar a soma ponderada `C = α·L + β·$ + γ·E` (latência, custo, energia) com `α+β+γ ≈ 1` normalizados na leitura, configuráveis por tenant/classe, como função única de custo que alimenta a prioridade efetiva. | 009, 026 |
| ADR-0092 | Sharding determinístico `hash(tenant,agent) mod N` com hashing consistente e rebalanceamento coordenado com 027 | Proposed | Particionar filas e ownership de dispatch por `shard = hash(tenant_id, agent_id) mod N`; mudanças de `N` usam hashing consistente e são coordenadas com `027-Cluster` para minimizar migração de shard. | 009, 027 |
| ADR-0093 | Filas de prioridade em Redis ZSET com aging (anti-starvation) e leases de dispatch por shard | Proposed | Manter filas por `(tenant, classe, shard)` em ZSET do Redis, com score = prioridade efetiva + componente de aging monotônico, e leases de dispatch (TTL) para exclusão mútua sem locks globais. | 009 |
| ADR-0094 | Modelo de preempção compensável com grace period, histerese e cooldown (anti-thrashing) | Proposed | Preempção é sempre **compensável** (aviso + `grace_period_ms` antes de suspender de fato) e sujeita a histerese/cooldown por tarefa, para evitar *thrashing* de preempção sob carga oscilante. | 009, 007 |
| ADR-0095 | Backpressure em três níveis (accept/defer/reject) integrado a JetStream + `Retry-After` | Proposed | `BackpressureController` classifica saturação em três níveis (`accept < defer(0.80) < reject(0.95)`) combinando profundidade de fila, lag de JetStream e pressão de capacidade, propagando `Retry-After` ao chamador. | 009, 020 |
| ADR-0096 | Interface `ISchedulingPolicy` estável: heurística v1 → política aprendida RL v2 sem quebra de contrato | Proposed | A decisão de prioridade/placement é encapsulada atrás de `ISchedulingPolicy`; `HeuristicPolicy` (v1) é a implementação inicial e `LearnedPolicy` (v2, RL) pode substituí-la sem alterar API externa nem contrato de eventos. | 009, 023 |
| ADR-0097 | Outbox transacional de decisão (event sourcing) para proveniência, auditoria (025) e treino (023) | Proposed | `DecisionJournal` grava `SchedulingDecision` em PostgreSQL e publica o evento correspondente atomicamente (padrão Outbox), garantindo que toda decisão tenha proveniência auditável e reusável como dado de treino. | 009, 023, 025 |

### 2.1 Justificativa de cada ADR proposta (contexto resumido)

| ADR | Por que uma ADR (e não um detalhe de implementação)? |
|-----|-------------------------------------------------------|
| ADR-0090 | A escolha entre reserva atômica local (Redis+Lua) vs. reserva coordenada por um serviço externo de cotas afeta diretamente NFR-005 (0 overcommit) e a filosofia de "caminho quente sem I/O bloqueante" (§10 do brief) — decisão estrutural de corretude sob concorrência, não um detalhe de código. |
| ADR-0091 | A forma da função de custo (soma ponderada linear vs. alternativas não-lineares/Pareto) determina como tenants configuram trade-offs latência×custo×energia e é compartilhada com `026-Cost-Optimizer` — mudar a forma da função depois de adotada tem custo de migração de configuração e de modelos de política (v2 RL) treinados sobre ela. |
| ADR-0092 | Sharding determinístico vs. alternativas (hash ring dinâmico gerido externamente, particionamento por range) define a estratégia de escala a 10⁶⁺ agentes (NFR-004) e o custo de rebalanceamento quando `N` muda — decisão que atravessa a fronteira com `027-Cluster`. |
| ADR-0093 | ZSET com aging vs. alternativas (fila FIFO por classe, heap em memória por réplica, fila em banco relacional) determina a garantia de fairness (NFR-006) e o comportamento sob restart de réplica — escolha de modelo de dados com implicações de disponibilidade. |
| ADR-0094 | Preempção compensável com grace period vs. preempção imediata (kill sem aviso) é uma decisão de **contrato de comportamento** visível ao Runtime Supervisor (007) e aos SLAs de tarefas preemptadas — afeta NFR-009 e a experiência de tenants sob contenção. |
| ADR-0095 | O modelo de três níveis de backpressure vs. um simples "aceita/rejeita" binário define a suavidade de degradação do sistema sob carga (parte central da filosofia de scheduler cognitivo consciente de saturação, §0 do brief) — decisão arquitetural com impacto direto em NFR-002 e na experiência de produtores (Kernel/Gateway). |
| ADR-0096 | Preservar `ISchedulingPolicy` como fronteira estável é o que permite evoluir de heurística determinística para política aprendida (RL) **sem** quebrar API/eventos consumidos por outros módulos — decisão de compatibilidade evolutiva de longo prazo (base de FR-013/NFR-013), não um detalhe de implementação de uma classe. |
| ADR-0097 | Outbox transacional vs. alternativas (2PC, publish-then-persist, CDC sobre WAL) define a garantia de atomicidade entre decisão e evento sob falha parcial — decisão que afeta diretamente NFR-007 (RPO ≤ 5 min) e a confiabilidade da trilha consumida por `025-Audit` e `023-Learning`. |

---

## 3. Rastreabilidade ADR → Componente → Requisito

| ADR | Componente(s) afetado(s) (ver `Architecture.md` §2) | Requisito(s) relacionado(s) |
|-----|-------------------------------------------------------|-------------------------------|
| ADR-0090 | `AdmissionController`, `QuotaLedger` | FR-001, NFR-005, NFR-011 |
| ADR-0091 | `PriorityCostEvaluator`, `CostEstimator` | FR-002, NFR-001 |
| ADR-0092 | `PlacementEngine`, `ShardRouter`, `CapacityProvider` | FR-003, NFR-004, NFR-014 |
| ADR-0093 | `PriorityQueueStore`, `DispatchLoop` | FR-006, FR-009, NFR-006 |
| ADR-0094 | `PreemptionManager` | FR-004, NFR-009 |
| ADR-0095 | `BackpressureController` | FR-005, FR-022, NFR-002 |
| ADR-0096 | `ISchedulingPolicy`, `HeuristicPolicy`, `LearnedPolicy`, `FeatureExtractor` | FR-013, NFR-013 |
| ADR-0097 | `DecisionJournal`, `EventPublisher` | FR-007, FR-010, NFR-007, NFR-010 |

---

## 4. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `ADR-00NN-<slug>.md` em `../002-ADR/` usando
   `../002-ADR/TEMPLATE.md`, preenchendo todas as seções obrigatórias
   (Contexto, Problema, Alternativas, Análise, Escolha, Consequências, Riscos,
   Trade-offs).
2. O conteúdo **NÃO PODE** contradizer o `./_DESIGN_BRIEF.md` deste módulo; caso
   a análise revele necessidade de divergência, o brief DEVE ser atualizado
   primeiro (o brief é a fonte única de verdade).
3. Revisão pelo Arquiteto-Chefe e pelo(s) módulo(s) afetado(s) listados na
   tabela da §3, com atenção especial a `006-Kernel` (consumidor de
   `SchedulingDecision`), `022-Policy` (PDP consultado em ADR-0090/0094) e
   `027-Cluster` (coordenação de rebalanceamento em ADR-0092).
4. Ao ser aceita, o `Status` muda para `Accepted` no arquivo da ADR e neste
   índice; o `../002-ADR/README.md` global é atualizado com a nova linha.
5. ADRs aceitas **NÃO SÃO** editadas — mudanças de rumo geram uma nova ADR que
   marca a anterior como `Superseded by ADR-XXXX`. Qualquer proposta de uso das
   posições livres `ADR-0098`/`ADR-0099` DEVE seguir este mesmo processo antes
   de reivindicar o número.

---

## 5. Referências

- Índice global de ADRs: `../002-ADR/README.md`
- Template de ADR: `../002-ADR/TEMPLATE.md`
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §11
- Arquitetura do módulo: `./Architecture.md`
- Requisitos: `./FunctionalRequirements.md`, `./NonFunctionalRequirements.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Índice de RFCs do módulo: `./RFC.md`

*Fim do índice de ADRs do módulo 009-Scheduler.*
