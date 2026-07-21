---
Documento: Requirements
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0084, ADR-0085, ADR-0086, ADR-0087, ADR-0088, ADR-0089
RFCs relacionados: RFC-0001, RFC-0080 (a propor)
Depende de: `_DESIGN_BRIEF.md` (deste módulo), `../001-Architecture/Architecture.md`, `../006-Kernel/`, `../007-Agent-Runtime/`, `../009-Scheduler/`, `../010-Memory/`, `../020-Communication/`, `../021-Security/`, `../022-Policy/`, `../024-Observability/`, `../025-Audit/`, `../027-Cluster/`
---

# 008-Agent-Lifecycle — Requirements

> Este documento é o **índice consolidado** de requisitos do módulo Agent
> Lifecycle. Ele não redefine requisitos — apenas organiza, agrupa e rastreia
> os requisitos detalhados em `FunctionalRequirements.md` e
> `NonFunctionalRequirements.md` até os casos de uso (`UseCases.md`) e os
> testes que os verificam (`Testing.md`, `Benchmark.md`). Toda afirmação
> normativa DEVE ser consistente com o `_DESIGN_BRIEF.md` deste módulo, que é
> a fonte única de verdade interna.

## 1. Propósito e Escopo

### 1.1 Propósito

Consolidar, numerar e tornar rastreável o conjunto completo de requisitos do
**Agent Lifecycle** — o *Lifecycle Manager* do AIOS, análogo ao `init`/`systemd`
de um sistema operacional clássico. O módulo 008 é a autoridade única sobre a
máquina de estados canônica do agente (`Created → Ready → Running ↔ Suspended
→ Hibernated → Migrating → Terminated/Failed`), sobre o Agent Control Block
(ACB) como estado de ciclo de vida, sobre a materialização/hibernação de *cold
agents* e sobre checkpoint/restore/migração, permitindo escalar a `10⁶`
agentes com apenas uma fração pequena ativa em RAM.

### 1.2 Escopo (in)

- Requisitos funcionais da FSM canônica de ciclo de vida (§4 do brief):
  transições, guardas, ações de entrada/saída.
- Requisitos funcionais de spawn/materialização (com e sem `WarmPoolManager`),
  suspensão/retomada, hibernação de *cold agents*, checkpoint/restore e
  migração de agentes entre shards/nós.
- Requisitos funcionais de emissão transacional de eventos
  `agent.lifecycle.*`, reconciliação estado-desejado × observado, serialização
  de transições via lease/fencing e retenção/expurgo (tombstone) de agentes
  terminados.
- Requisitos não-funcionais de desempenho (spawn, transição, checkpoint,
  migração), escalabilidade (rumo a milhões de agentes), recuperabilidade
  (RTO/RPO), disponibilidade, throughput, integridade de snapshot,
  observabilidade e idempotência aplicáveis ao módulo 008.
- Casos de uso que materializam esses requisitos em fluxos observáveis
  (felizes e de falha), incluindo os modos de falha do FMEA do brief (§9).

### 1.3 Escopo (out)

- Requisitos de **como** o loop cognitivo do agente executa (ReAct, uso de
  tools) — pertence ao `007-Agent-Runtime`. O 008 apenas inicia/pausa/encerra
  o processo de runtime.
- Requisitos de **quem** executa, quando e onde (prioridade, custo, admissão,
  placement) — pertence ao `009-Scheduler`. O 008 *aplica* a decisão de
  placement, não a toma.
- Requisitos de superfície de **syscalls cognitivas** (`remember`, `recall`,
  `plan`, `invoke_tool`) e de gerenciamento de **cotas** — pertencem ao
  `006-Kernel` e ao `026-Cost-Optimizer`.
- Requisitos de persistência/consolidação de **camadas de memória**
  (working/short/long/semantic/procedural/episódica) — pertencem ao
  `010-Memory`. O 008 serializa apenas o *ponteiro/snapshot* da working
  memory como parte do checkpoint.
- Requisitos de **decisão** de política (RBAC/ABAC) — pertencem ao
  `022-Policy`. O 008 atua como PEP, não como PDP.
- Requisitos de provisão física de nós/pods, autoscaling de infraestrutura ou
  replicação de storage — pertencem ao `027-Cluster` e ao `028-Deployment`.
- Requisitos de roteamento de modelos LLM (`017`) ou de execução de tools
  (`015`) — fora do escopo do módulo 008.

## 2. Stakeholders

| Stakeholder | Papel/Interesse | Documentos primários de interesse |
|-------------|------------------|------------------------------------|
| Arquitetura-Chefe do AIOS | Garantir conformidade com `001-Architecture` e RFC-0001; aprovar ADRs `ADR-0080..0089`. | `Architecture.md`, `ADR.md`, `RFC.md` |
| Equipe do 008-Agent-Lifecycle (dono do módulo) | Implementar, operar e evoluir o serviço; responder pelos SLOs de spawn/hibernação/migração. | Todos os 26 documentos |
| Equipe do 006-Kernel | Consome a FSM/ACB para orquestrar `spawn`/`suspend`/`resume`/`kill`/`checkpoint` como syscalls; dono da criação inicial do ACB. | `API.md`, `StateMachine.md`, `SequenceDiagrams.md` |
| Equipe do 009-Scheduler | Fonte de decisões de admissão/placement/preempção consumidas pelo `SpawnManager`/`MigrationOrchestrator`. | `Events.md`, `API.md`, `SequenceDiagrams.md` |
| Equipe do 007-Agent-Runtime | Contrato de materialização/heartbeat/exit consumido pelo `SpawnManager`/`ReconciliationController`. | `Events.md`, `SequenceDiagrams.md` |
| Equipe do 010-Memory | Contrato de ponteiro/snapshot da working memory usado pelo `CheckpointService`. | `Database.md`, `ClassDiagrams.md` |
| Equipe do 022-Policy (PDP) | Contrato de decisão consumido pelo `LifecyclePolicyEnforcer`. | `API.md`, `Security.md` |
| Equipe do 027-Cluster | Decisões de placement/drenagem de nó consumidas pelo `MigrationOrchestrator`. | `Events.md`, `Scalability.md` |
| Equipe do 024-Observability | Padrões de telemetria, métricas `aios_lifecycle_*` e dashboards. | `Metrics.md`, `Monitoring.md`, `Logging.md` |
| Equipe do 025-Audit | Consumidora dos eventos de ciclo de vida e dos expurgos de `TombstoneManager`. | `Events.md`, `Security.md` |
| Tenants (clientes do AIOS) | Consomem a API de ciclo de vida (spawn/suspend/hibernate/migrate/terminate) para seus agentes. | `API.md`, `Examples.md` |
| Operações/SRE (029) | Opera o serviço em produção; monitora SLOs de RTO/RPO/hibernação. | `Monitoring.md`, `FailureRecovery.md`, `Deployment.md` |
| Compliance/DPO | Garante conformidade LGPD/GDPR sobre snapshots/checkpoints e expurgo de agentes. | `Security.md`, `NonFunctionalRequirements.md` |

## 3. Sumário de Requisitos Funcionais

> Detalhamento completo em `FunctionalRequirements.md`. A numeração
> (`FR-001`..`FR-014`) é global, sequencial e idêntica à do `_DESIGN_BRIEF.md`
> §7.1 — este documento apenas agrupa por categoria funcional.

| Categoria | Faixa de IDs | Resumo |
|-----------|--------------|--------|
| A. Máquina de Estados Canônica | FR-001 | Autoridade única sobre a FSM do ciclo de vida (§4 do brief). |
| B. Spawn e Materialização | FR-002, FR-014 | Spawn de agente novo/frio com `p99 ≤ 250 ms`, negociação de slot e WarmPool. |
| C. Suspensão e Retomada | FR-003 | `suspend`/`resume` preservando working memory. |
| D. Hibernação de Cold Agents | FR-004, FR-005 | Hibernar liberando RAM; materializar sob demanda. |
| E. Checkpoint e Restore | FR-006 | Checkpoint/restore consistentes de ACB + working memory. |
| F. Migração | FR-007 | Migração entre shards/nós via saga. |
| G. Eventos | FR-008 | Emissão transacional (Outbox) de `agent.lifecycle.*`. |
| H. Reconciliação | FR-009 | Reconciliação estado-desejado × observado. |
| I. Concorrência | FR-010 | Serialização de transições por agente via lease. |
| J. Retenção e Expurgo | FR-011 | Tombstone/LGPD de agentes terminados. |
| K. Observabilidade da API | FR-012 | Histórico de ciclo de vida (event-sourced) via API. |
| L. Segurança (PEP) | FR-013 | *Default deny* para toda mutação. |

## 4. Sumário de Requisitos Não-Funcionais

> Detalhamento completo, com metas numéricas (SLO/SLI) e método de
> verificação, em `NonFunctionalRequirements.md` (`NFR-001`..`NFR-017`).
> `NFR-001`..`NFR-013` são idênticos ao `_DESIGN_BRIEF.md` §7.2;
> `NFR-014`..`NFR-017` são elaborações não-contraditórias de atributos de
> qualidade exigidos pelo `MODULE_TEMPLATE.md` (manutenibilidade,
> testabilidade, conformidade, resiliência) que o brief menciona nas seções
> §9/§12 sem numerar explicitamente.

| Atributo de qualidade | Faixa de IDs | Resumo da meta |
|------------------------|--------------|-----------------|
| Desempenho | NFR-001, NFR-002, NFR-003 | `p99 spawn ≤ 250 ms`; `p99 transição ≤ 20 ms`; `p99 checkpoint ≤ 500 ms`. |
| Escalabilidade | NFR-004 | `≥ 10⁶` agentes por cluster, `≤ 5%` ativos em RAM. |
| Recuperabilidade | NFR-005 | `RTO ≤ 15 min`. |
| Durabilidade / RPO | NFR-006 | `RPO ≤ 5 min` de working memory. |
| Disponibilidade | NFR-007 | `≥ 99,95%` do serviço 008. |
| Throughput | NFR-008 | `≥ 50.000 transições/s` por cluster. |
| Integridade | NFR-009 | `0` restore de checkpoint corrompido não detectado. |
| Desempenho de migração | NFR-010 | `p99 migração ≤ 2 s` (agente ≤ 64 MiB, mesmo DC). |
| Eficiência de RAM | NFR-011 | ACB frio `≤ 4 KiB` no índice quente (Redis). |
| Observabilidade | NFR-012 | `100%` das transições com trace OTel correlacionado. |
| Idempotência | NFR-013 | Repetição de mutação ⇒ mesmo resultado, `0` efeito duplicado. |
| Manutenibilidade/Testabilidade | NFR-014, NFR-015 | Componentes testáveis isoladamente; `100%` das transições cobertas por contract test. |
| Segurança | NFR-016 | `100%` das mutações via PEP; `0` bypass. |
| Conformidade LGPD/GDPR | NFR-017 | Expurgo rastreável; `0` reutilização de `agent_id`. |

## 5. Matriz de Rastreabilidade — Requisito → Caso de Uso → Teste

> `UC-NNN` referem-se a `UseCases.md`. `Teste` referencia a estratégia
> descrita em `Testing.md` (unit/integration/contract/e2e/chaos/load) e
> `Benchmark.md` onde aplicável. Esta matriz DEVE ser atualizada a cada novo
> requisito ou caso de uso.

| Requisito | Caso(s) de Uso | Tipo de teste |
|-----------|-----------------|----------------|
| FR-001 | UC-001, UC-015, UC-016 | Unit (FSM), integration (guardas), contract test de transições inválidas |
| FR-002, FR-014 | UC-001, UC-002, UC-003 | Load test (`Benchmark.md` — Spawn Latency), integration (WarmPool hit/miss) |
| FR-003 | UC-004, UC-005 | Integration (hash de working memory pré/pós), e2e |
| FR-004 | UC-006, UC-019 | Integration (verificação de RSS = 0), chaos (storm de hibernação) |
| FR-005 | UC-007 | Load test (latência de materialização fria), integration |
| FR-006 | UC-008, UC-009 | Integration (round-trip checkpoint/restore), chaos (checkpoint corrompido) |
| FR-007 | UC-010, UC-011, UC-020 | Integration (saga de migração), chaos (destino indisponível) |
| FR-008 | Todos os UCs que mutam estado | Chaos (crash entre commit/publish), integration (dedupe por `event.id`) |
| FR-009 | UC-014 | Chaos (runtime morto sem `Terminated`), integration (loop de reconciliação) |
| FR-010 | UC-015 | Integration (concorrência de duas transições), unit (fencing token) |
| FR-011 | UC-017 | Integration (expurgo/tombstone), auditoria de não-reutilização de ID |
| FR-012 | UC-018 | Contract test (paginação e ordenação por `seq`) |
| FR-013 | UC-016 | Integration (PDP `deny`), security test |
| NFR-001, NFR-002, NFR-003 | UC-001, UC-004, UC-005, UC-008 | Load test (`Benchmark.md`) |
| NFR-004, NFR-011 | UC-006, UC-007 | Teste de escala (`Scalability.md`) |
| NFR-005, NFR-006 | UC-009, UC-014 | Chaos test, DR drill |
| NFR-007 | Todos os UCs | Uptime probe sintético |
| NFR-008 | UC-001..UC-014 (mix) | Load test sustentado |
| NFR-009 | UC-008, UC-009 | Verificação de `content_hash` sob corrupção injetada |
| NFR-010 | UC-010, UC-020 | Load test de migração |
| NFR-012 | Todos os UCs | Inspeção de spans/campos de correlação |
| NFR-013 | UC-016 (repetição idempotente em qualquer mutação) | Teste de replay/duplicação |
| NFR-014, NFR-015 | Todos os UCs | Revisão de arquitetura, cobertura de contract test |
| NFR-016 | UC-016 | Pentest, auditoria de cobertura de PEP |
| NFR-017 | UC-017 | Auditoria de expurgo/retenção LGPD |

## 6. Glossário Local

> Este módulo **reutiliza** as definições de `../040-Glossary/Glossary.md` sem
> redefini-las. Os termos abaixo são os mais relevantes para o Agent Lifecycle
> e estão listados apenas como referência de navegação (definição completa no
> glossário global):

| Termo | Ver definição em |
|-------|-------------------|
| ACB (Agent Control Block) | `../040-Glossary/Glossary.md#a` |
| Agent (Agente) | `../040-Glossary/Glossary.md#a` |
| Cold Agent | `../040-Glossary/Glossary.md#c` |
| Lifecycle (Ciclo de Vida) | `../040-Glossary/Glossary.md#l` |
| Placement | `../040-Glossary/Glossary.md#p` |
| Preemption (Preempção) | `../040-Glossary/Glossary.md#p` |
| PEP / PDP | `../040-Glossary/Glossary.md#p` |
| Outbox | `../040-Glossary/Glossary.md#o` |
| Saga | `../040-Glossary/Glossary.md#s` |
| Shard | `../040-Glossary/Glossary.md#s` |
| Hot State (Estado Quente) | `../040-Glossary/Glossary.md#h` |
| Working Memory (Memória de Trabalho) | `../040-Glossary/Glossary.md#w` |
| RTO / RPO | `../040-Glossary/Glossary.md#r` |
| Idempotency (Idempotência) | `../040-Glossary/Glossary.md#i` |
| CQRS | `../040-Glossary/Glossary.md#c` |
| Row-Level Security (RLS) | `../040-Glossary/Glossary.md#r` |
| Event Sourcing | `../040-Glossary/Glossary.md#e` |

## 7. Referências

- Fonte de verdade do módulo: `_DESIGN_BRIEF.md`
- Requisitos funcionais detalhados: `FunctionalRequirements.md`
- Requisitos não-funcionais detalhados: `NonFunctionalRequirements.md`
- Casos de uso: `UseCases.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Módulo do Kernel (dono da criação do ACB): `../006-Kernel/`
- Módulo do Scheduler (dono das decisões de placement/admissão): `../009-Scheduler/`
- Módulo de Memória (dono das camadas de memória): `../010-Memory/`
- Módulo de Cluster/HA/DR (dono da provisão física): `../027-Cluster/`
