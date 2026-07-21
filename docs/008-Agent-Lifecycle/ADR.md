---
Documento: ADR.md
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0007, ADR-0008, ADR-0010, ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0084, ADR-0085, ADR-0086, ADR-0087, ADR-0088, ADR-0089
RFCs relacionados: RFC-0001, RFC-0080
Depende de: 006-Kernel, 007-Agent-Runtime, 009-Scheduler, 010-Memory, 020-Communication, 021-Security, 022-Policy, 024-Observability, 025-Audit, 027-Cluster, 002-ADR
---

# 008-Agent-Lifecycle — Índice de ADRs

> **Escopo deste documento.** Este documento **NÃO** decide nada por si — é um
> **índice navegável** das *Architecture Decision Records* (ADR) que afetam o
> módulo 008-Agent-Lifecycle. As decisões em si residem em `../002-ADR/`, no
> formato definido por `../002-ADR/TEMPLATE.md` (Contexto · Problema ·
> Alternativas · Análise · Escolha · Consequências · Riscos · Trade-offs). Este
> arquivo **DEVE** ser mantido sincronizado com `../_DESIGN_BRIEF.md` §11 e com
> `../002-ADR/README.md`; qualquer divergência entre este índice e o brief é um
> defeito de documentação — o brief prevalece.

> **Convenção de numeração.** Conforme `../_DESIGN_BRIEF.md` §0 e §11, a faixa
> `ADR-0080`..`ADR-0089` está **reservada** ao módulo 008-Agent-Lifecycle (regra
> `módulo × 10`: `008 × 10 = 0080`, evitando colisão com `009-Scheduler`, que
> usa `ADR-0090`..`ADR-0099`). Nenhuma outra numeração PODE ser usada por este
> módulo sem atualizar este índice e `../002-ADR/README.md`.

---

## 1. ADRs Globais que Restringem o Agent Lifecycle

Estas ADRs foram decididas na fundação canônica do AIOS (`../002-ADR/`) e
**DEVEM** ser respeitadas por qualquer ADR específica do módulo 008 — elas não
são redecididas aqui, apenas referenciadas e interpretadas no contexto do
ciclo de vida do agente.

| ADR | Título | Status | Como restringe o Agent Lifecycle |
|-----|--------|--------|-------------------------------------|
| [ADR-0001](../002-ADR/ADR-0001-Sistema-Operacional-nao-Framework.md) | AIOS é um SO, não um framework | Accepted | O módulo 008 é o **Lifecycle Manager** — análogo a `init`/`systemd` de um SO clássico (`_DESIGN_BRIEF.md` §1.1, R1). A FSM canônica (§4 do brief) é tratada como contrato de sistema operacional, não como conveniência de orquestração de terceiros. |
| [ADR-0002](../002-ADR/ADR-0002-Microservicos-Control-Data-Plane.md) | Microserviços + split control/data plane | Accepted | O `LifecycleCoordinator` e todos os *Managers* (§2 do brief) vivem no **control plane** (.NET 10); o loop cognitivo executado pelo Runtime (007) permanece no **data plane** (Python) — o módulo 008 apenas inicia/pausa/encerra o processo de runtime (N1 do brief), nunca executa raciocínio. |
| [ADR-0003](../002-ADR/ADR-0003-DotNet-Control-Python-Runtime.md) | .NET 10 no controle, Python no runtime | Accepted | `LifecycleApiSurface`, `LifecycleCoordinator`, `StateMachineEngine`, `AcbStore` etc. são implementados em **.NET 10**; comunicação com o Agent Runtime (007, Python) ocorre exclusivamente via gRPC/REST (`SpawnManager`, `MigrationOrchestrator`), nunca por acoplamento de processo. |
| [ADR-0004](../002-ADR/ADR-0004-NATS-como-Barramento.md) | NATS como barramento primário | Accepted | Todo evento `agent.lifecycle.*` (`_DESIGN_BRIEF.md` §6) é publicado via **NATS/JetStream** através do `LifecycleEventPublisher`, nunca via um bus alternativo; consumo reativo de 006/007/009/014/027 via `LifecycleEventConsumer`. |
| [ADR-0005](../002-ADR/ADR-0005-PostgreSQL-pgvector-AGE.md) | PostgreSQL + pgvector + Apache AGE | Accepted | `AcbStore` usa **PostgreSQL** como fonte da verdade do ACB, das `LifecycleTransition` (event-sourced) e do outbox transacional; metadados de `Checkpoint`/`MigrationJob` também residem em PostgreSQL (`_DESIGN_BRIEF.md` §3). O módulo 008 não usa pgvector/AGE diretamente — isso é domínio de `010-Memory` — mas herda a unificação de armazenamento relacional. |
| [ADR-0006](../002-ADR/ADR-0006-Redis-Estado-Quente.md) | Redis para estado quente e locks | Accepted | Projeção quente do ACB (`≤ 4 KiB`, NFR-011), o `LeaseManager` (locks distribuídos por `agent_id` com fencing token) e o índice de roteamento de cold agents usam **Redis**, nunca um cache/lock alternativo. |
| [ADR-0007](../002-ADR/ADR-0007-Memoria-Hierarquica.md) | Memória hierárquica (working/short/long/semantic/procedural/episódica) | Accepted | O módulo 008 serializa **apenas o ponteiro/snapshot da working memory** (`working_memory_ref`) como parte do checkpoint (N4 do brief) — a materialização e consolidação das demais camadas permanece exclusivamente com `010-Memory`. O `CheckpointService` trata a working memory como um *blob opaco* referenciado por URN. |
| [ADR-0008](../002-ADR/ADR-0008-Governanca-por-Politica-Default-Deny.md) | Governança por política, default deny | Accepted | Fundamenta diretamente o papel do `LifecyclePolicyEnforcer` como **PEP**: toda mutação de ciclo de vida (spawn/suspend/resume/hibernate/wake/migrate/checkpoint/restore/terminate) consulta o PDP (`022-Policy`) com postura *default deny*, mapeando para `AIOS-LIFECYCLE-0012` em caso de negação. |
| [ADR-0010](../002-ADR/ADR-0010-Observabilidade-Auditoria-por-Construcao.md) | Observabilidade e auditoria por construção | Accepted | Todo o `LifecycleTelemetry` (spans OTel, métricas `aios_lifecycle_*`, logs Serilog→Seq) e a emissão de eventos auditáveis para `025-Audit` (ex.: expurgo LGPD pelo `TombstoneManager`) implementam esta ADR — observabilidade e auditoria não são adicionadas a posteriori, são parte do contrato do componente. |

---

## 2. ADRs Específicas do Módulo 008-Agent-Lifecycle (Propostas)

> **Estado de redação.** As ADRs abaixo estão **reservadas** na faixa
> `ADR-0080`–`ADR-0089` e seu conteúdo (Contexto/Problema/Alternativas/Análise/
> Escolha/Consequências/Riscos/Trade-offs) está **descrito no
> `../_DESIGN_BRIEF.md` §11.1** deste módulo, que é a fonte única de verdade
> até que cada ADR seja escrita como arquivo próprio em `../002-ADR/` e tenha
> seu `Status` promovido de `Proposed` para `Accepted`. Nenhum destes arquivos
> existe fisicamente em `../002-ADR/` no momento desta versão do índice; este
> documento resume o conteúdo pretendido para orientar quem for redigi-las.

| ADR | Título | Status | Resumo da decisão (1 linha) | Módulos afetados |
|-----|--------|--------|------------------------------|-------------------|
| ADR-0080 | Máquina de estados canônica do ciclo de vida do agente | Proposed | Fixar os 8 estados (`Created, Ready, Running, Suspended, Hibernated, Migrating, Terminated, Failed`) e as 11 transições/guardas (`_DESIGN_BRIEF.md` §4) como contrato imutável do módulo 008, com `StateMachineEngine` puramente determinístico. | 006, 007, 008, 009 |
| ADR-0081 | Cold Agent Hibernation: estado serializado em MinIO + índice em PostgreSQL | Proposed | Hibernar *cold agents* liberando 100% da RAM do runtime, com payload em **MinIO** e metadados/índice em **PostgreSQL**, mantendo apenas uma projeção quente (`≤ 4 KiB`) em Redis para permitir roteamento e `wake` rápido a escala de `10⁶` agentes. | 008, 010, 027 |
| ADR-0082 | Formato e codec de checkpoint (`msgpack+zstd`, AES-256-GCM) | Proposed | Adotar `msgpack+zstd` como codec padrão de serialização (com `json+zstd` como alternativa compatível) e **AES-256-GCM** com chave por tenant para cifragem em repouso de ACB + working memory, com `content_hash` (sha-256) para verificação de integridade. | 008, 010, 021 |
| ADR-0083 | Materialização sob demanda com WarmPool (spawn < 250 ms) | Proposed | Manter um `WarmPoolManager` com reserva mínima/máxima de runtimes pré-aquecidos por tenant/shard, transformando `spawn`/`wake` em *deserialização + attach de memória* em vez de boot completo de processo, para atingir `p99 ≤ 250 ms` (NFR-001). | 007, 008, 009 |
| ADR-0084 | Concorrência por agente via lease + fencing token (Redis) | Proposed | Serializar transições por agente através de lease distribuído (`SET NX PX` + `fencing_token` monotônico) no `LeaseManager`, sem lock global, garantindo **INV1** (uma transição por agente por vez) e escala horizontal por shard. | 008 |
| ADR-0085 | Migração como saga com compensação (checkpoint → transfer → cutover) | Proposed | Modelar migração entre shards/nós como **saga orquestrada** pelo `MigrationOrchestrator` com fases `quiesce → checkpoint → transfer → materialize → cutover → cleanup` e compensação explícita (reativa origem) em caso de falha do destino, garantindo zero perda de trabalho aceito. | 008, 009, 027 |
| ADR-0086 | Event sourcing de transições + Outbox transacional | Proposed | Persistir toda transição como registro `LifecycleTransition` append-only (event-sourced) e publicar eventos `agent.lifecycle.*` via padrão **Outbox** na mesma transação (INV2), garantindo at-least-once com dedupe por `event.id` e throughput `≥ 50.000 transições/s` (NFR-008). | 008, 020 |
| ADR-0087 | Reconciliação estado-desejado × observado (controller pattern) | Proposed | Implementar o `ReconciliationController` segundo o padrão *controller/reconcile* (estado desejado × observado), detectando e corrigindo derivas — ex.: runtime morto sem transição para `Terminated` — dentro de `reconcile.interval_ms`, cumprindo RTO ≤ 15 min (NFR-005). | 007, 008 |
| ADR-0088 | Fronteira 008×006×009×007 (quem decide vs. quem aplica) | Proposed | Delimitar formalmente: 006-Kernel cria o ACB e expõe syscalls cognitivas; 009-Scheduler **decide** placement/admissão/prioridade; 007-Agent-Runtime **executa** o loop cognitivo; 008-Agent-Lifecycle **aplica** as decisões de placement/admissão via FSM e é a única autoridade de persistência de estado de ciclo de vida. | 006, 007, 008, 009 |
| ADR-0089 | Retenção, tombstone e expurgo LGPD de agentes/checkpoints | Proposed | Definir política de retenção (`retention.terminated_ttl_days`, `retention.checkpoint_ttl_days`) e o processo de expurgo rastreável executado pelo `TombstoneManager` para atender ao direito ao esquecimento, com **não-reutilização** de `agent_id`/URN após término. | 008, 010, 021, 025 |

### 2.1 Justificativa de cada ADR proposta (contexto resumido)

| ADR | Por que uma ADR (e não um detalhe de implementação)? |
|-----|-------------------------------------------------------|
| ADR-0080 | A FSM é o **contrato central** consumido por 006 (criação do ACB), 007 (execução do runtime) e 009 (admissão/placement); qualquer alteração de estado/transição exige coordenação multi-módulo — decisão arquitetural, não detalhe de código. |
| ADR-0081 | A escolha de onde e como hibernar determina diretamente se o AIOS atinge a meta de `≥ 10⁶` agentes por cluster com `≤ 5%` ativos em RAM (NFR-004) — é a decisão econômica central do módulo. |
| ADR-0082 | Codec e cifragem de checkpoint afetam simultaneamente performance (NFR-003), integridade (NFR-009) e segurança (LGPD, `_DESIGN_BRIEF.md` §12.3) — mudar o formato depois de adotado exige migração de dados em produção. |
| ADR-0083 | WarmPool vs. cold-start puro é a decisão que define se `p99 ≤ 250 ms` (NFR-001) é atingível; afeta reserva de capacidade de 007 e orçamento de 009. |
| ADR-0084 | Lease com fencing token vs. lock pessimista/transação distribuída determina o modelo de concorrência de todo o módulo (NFR-002, NFR-008) e a ausência de lock global é pré-requisito de escala horizontal. |
| ADR-0085 | Migração como saga (vs. transação distribuída/2PC) é uma escolha estrutural que define como falhas parciais de migração são tratadas sem perda de trabalho aceito (NFR-010). |
| ADR-0086 | Event sourcing + Outbox (vs. CDC ou publicação direta pós-commit) é a garantia de que nenhum evento de ciclo de vida é perdido sob falha — decisão de confiabilidade, não conveniência. |
| ADR-0087 | O padrão controller/reconcile (vs. apenas reação a eventos) é o que garante convergência mesmo sob eventos perdidos/atrasados — central ao SLO de recuperação (NFR-005). |
| ADR-0088 | A fronteira "quem decide vs. quem aplica" entre 006/007/008/009 é fonte recorrente de acoplamento indevido se não for formalizada — decisão de fronteira arquitetural, não de código. |
| ADR-0089 | Retenção/expurgo tem implicação legal direta (LGPD/GDPR) e afeta o modelo de dados (`Checkpoint`, `HibernationRecord`) de forma irreversível — exige registro formal de decisão. |

---

## 3. Rastreabilidade ADR → Componente → Requisito

| ADR | Componente(s) afetado(s) (ver `Architecture.md` §2) | Requisito(s) relacionado(s) |
|-----|-------------------------------------------------------|-------------------------------|
| ADR-0080 | `StateMachineEngine`, `AcbStore`, `LifecycleCoordinator` | FR-001, NFR-002, NFR-012 |
| ADR-0081 | `HibernationController`, `SnapshotStore` | FR-004, FR-005, NFR-004, NFR-011 |
| ADR-0082 | `CheckpointService`, `SnapshotStore` | FR-006, NFR-003, NFR-009 |
| ADR-0083 | `SpawnManager`, `WarmPoolManager` | FR-002, FR-014, NFR-001 |
| ADR-0084 | `LeaseManager`, `LifecycleCoordinator` | FR-010, NFR-002, NFR-008 |
| ADR-0085 | `MigrationOrchestrator` | FR-007, NFR-010 |
| ADR-0086 | `AcbStore`, `LifecycleEventPublisher` | FR-008, FR-012, NFR-008 |
| ADR-0087 | `ReconciliationController` | FR-009, NFR-005 |
| ADR-0088 | `LifecycleCoordinator`, `SpawnManager`, `MigrationOrchestrator` | FR-002, FR-007, FR-013 |
| ADR-0089 | `TombstoneManager` | FR-011 |

---

## 4. Processo de Promoção (Draft → Accepted)

1. O autor redige o arquivo `ADR-00NN-<slug>.md` em `../002-ADR/` usando
   `../002-ADR/TEMPLATE.md`, preenchendo todas as seções obrigatórias
   (Contexto, Problema, Alternativas, Análise, Escolha, Consequências, Riscos,
   Trade-offs).
2. O conteúdo **NÃO PODE** contradizer o `../_DESIGN_BRIEF.md` deste módulo;
   caso a análise revele necessidade de divergência, o brief DEVE ser
   atualizado primeiro (o brief é a fonte única de verdade).
3. Revisão pelo Arquiteto-Chefe e pelos módulos afetados listados na tabela da
   §3 — em particular `006-Kernel`, `007-Agent-Runtime` e `009-Scheduler`,
   dada a fronteira compartilhada (ADR-0088).
4. Ao ser aceita, o `Status` muda para `Accepted` no arquivo da ADR e neste
   índice; o `../002-ADR/README.md` global é atualizado com a nova linha.
5. ADRs aceitas **NÃO SÃO** editadas — mudanças de rumo geram uma nova ADR que
   marca a anterior como `Superseded by ADR-XXXX`.

---

## 5. Referências

- Índice global de ADRs: `../002-ADR/README.md`
- Template de ADR: `../002-ADR/TEMPLATE.md`
- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md` §11.1
- Arquitetura do módulo: `./Architecture.md`
- Máquina de estados do módulo: `./StateMachine.md`
- Requisitos: `./FunctionalRequirements.md`, `./NonFunctionalRequirements.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Índice de RFCs do módulo: `./RFC.md`

*Fim do índice de ADRs do módulo 008-Agent-Lifecycle.*
