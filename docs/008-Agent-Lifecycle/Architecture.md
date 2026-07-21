---
Documento: Architecture
Módulo: 008-Agent-Lifecycle
Status: Draft
Versão: 0.1
Última atualização: 2026-07-20
Responsável (RACI-A): Arquiteto do Módulo 008 — Agent Lifecycle
ADRs relacionados: ADR-0080, ADR-0081, ADR-0082, ADR-0083, ADR-0084, ADR-0085, ADR-0086, ADR-0087, ADR-0088, ADR-0089
RFCs relacionados: RFC-0001 (baseline); RFC-0080 (Cold Agent Hibernation & Checkpoint/Restore Protocol, a propor)
Depende de: 006-Kernel, 007-Agent-Runtime, 009-Scheduler, 010-Memory, 020-Communication, 021-Security, 022-Policy, 024-Observability, 025-Audit, 027-Cluster
---

# 008-Agent-Lifecycle — Arquitetura

> Este documento detalha a arquitetura interna do **Lifecycle Manager** do AIOS
> em notação C4 adaptada para ASCII. Ele **não redefine** contratos centrais
> (URN, envelope de evento, envelope de erro, idempotência, correlação,
> subjects) — estes são consumidos de
> `../003-RFC/RFC-0001-Architecture-Baseline.md`. Este documento é derivado de
> `./_DESIGN_BRIEF.md` (fonte única de verdade do módulo) e não pode
> contradizê-lo.

## Índice

1. Papel do Agent Lifecycle na arquitetura global
2. C4 Nível 1 — Contexto do Sistema
3. C4 Nível 2 — Contêineres
4. C4 Nível 3 — Componentes
5. Tabela de responsabilidades por componente
6. Fronteiras e regras de dependência
7. Padrões arquiteturais adotados
8. Tecnologias e justificativas
9. Caminho quente (hot path) × caminho frio (cold path)
10. Riscos arquiteturais e trade-offs
11. Alternativas descartadas
12. Referências e ADRs

---

## 1. Papel do Agent Lifecycle na Arquitetura Global

O módulo `008-Agent-Lifecycle` é o **Lifecycle Manager** do AIOS — o análogo
funcional de `init`/`systemd` em um sistema operacional clássico
(`../000-Vision/Vision.md` §7.1). Ele é a **autoridade única** sobre a máquina
de estados canônica do agente (`Created → Ready → Running ↔ Suspended →
Hibernated → Migrating → Terminated/Failed`, ver `./StateMachine.md`) e o
**único** componente autorizado a persistir transições de estado do **Agent
Control Block (ACB)**.

O módulo **NÃO** executa o loop cognitivo do agente (isso é do
`007-Agent-Runtime`), **NÃO** decide *quem*/*quando*/*onde* executar
(isso é do `009-Scheduler`) e **NÃO** expõe syscalls cognitivas nem gerencia
cotas (isso é do `006-Kernel`/`026-Cost-Optimizer`). O 008 **aplica** decisões
de outros módulos sobre o *estado de ciclo de vida* do agente: materializa,
suspende, hiberniza, faz checkpoint/restore e migra — sempre de forma
transacional, idempotente e auditável.

A responsabilidade mais crítica do módulo, do ponto de vista de escala, é
sustentar `≥ 10⁶` agentes por cluster com `≤ 5%` ativos em RAM simultaneamente
(NFR-004), via **hibernação de cold agents** e **materialização sob demanda**
com `p99 ≤ 250 ms` (NFR-001).

---

## 2. C4 — Nível 1: Contexto do Sistema

```
                     gRPC (interno, mTLS)
     ┌───────────────────┐                       ┌────────────────────────┐
     │   006-Kernel        │──── acb.created ────▶│                        │
     │  (dono da criação   │◀── confirma estado ──│                        │
     │   do ACB)            │                      │                        │
     └───────────────────┘                       │                        │
                                                   │                        │
     ┌───────────────────┐   REST (externo,      │                        │  gRPC   ┌──────────────────┐
     │  API Gateway         │   via YARP)          │  008-AGENT-LIFECYCLE   │────────▶│  022-Policy (PDP) │
     │  (021-Security)       │─────────────────────▶│  (Lifecycle Manager)   │◀────────│                  │
     └───────────────────┘                       │                        │         └──────────────────┘
                                                   │                        │
     ┌───────────────────┐  admissão/placement    │                        │  spawn/suspend/  ┌──────────────────┐
     │  009-Scheduler        │◀───────────────────│                        │  materialize     │ 007-Agent-Runtime │
     │                       │────────────────────▶│                        │─────────────────▶│  (data plane)     │
     └───────────────────┘  decisão               │                        │◀─────────────────│ heartbeat/exit    │
                                                   │                        │                  └──────────────────┘
     ┌───────────────────┐  ponteiro/snapshot     │                        │
     │  010-Memory            │◀───────────────────│                        │
     │                       │────────────────────▶│                        │
     └───────────────────┘  working memory ref    │                        │
                                                   │                        │
     ┌───────────────────┐  placement/draining    │                        │  eventos (async)  ┌──────────────────┐
     │  027-Cluster           │◀───────────────────│                        │─────────────────▶│  NATS/JetStream    │
     │                       │────────────────────▶│                        │◀─────────────────│  (020)             │
     └───────────────────┘                       │                        │                  └──────────────────┘
                                                   │                        │
     ┌───────────────────┐   telemetria/audit     │                        │
     │ 024-Observability/    │◀───────────────────│                        │
     │ 025-Audit               │                     └────────────────────────┘
     └───────────────────┘
```

**Atores e sistemas em fronteira**

| Ator/Sistema | Papel na interação com o Agent Lifecycle | Protocolo |
|--------------|-------------------------------------------|-----------|
| `006-Kernel` | Cria o ACB (fonte de identidade); consome eventos de estado para refletir *lifecycle state* no ACB. | gRPC + eventos NATS |
| API Gateway (YARP, `021`) | *Front door* externo: valida OAuth2/OIDC, propaga claims, roteia REST `/v1/agents/**`. | REST/HTTPS |
| `022-Policy` (PDP) | Autoridade de decisão consultada por `LifecyclePolicyEnforcer` para toda mutação. | gRPC |
| `009-Scheduler` | Decide admissão/placement/preempção; o 008 negocia slot e aplica a decisão. | gRPC + eventos |
| `007-Agent-Runtime` | Processo de execução materializado/pausado/encerrado pelo 008; emite heartbeat/exit consumidos para reconciliação. | gRPC + eventos |
| `010-Memory` | Dono da materialização da working memory; o 008 apenas serializa/restaura o *ponteiro/snapshot*. | gRPC |
| `027-Cluster` | Fornece decisão/saúde de nós/shards para migração e drenagem em massa. | gRPC + eventos |
| NATS/JetStream (`020`) | Barramento de eventos `agent.lifecycle.*` (produção) e de placement/heartbeat/idle (consumo). | NATS/JetStream |
| `024-Observability` / `025-Audit` | Destino de telemetria OTel e de eventos auditáveis (ex.: expurgo LGPD). | OTLP / gRPC |

---

## 3. C4 — Nível 2: Contêineres

> O Agent Lifecycle é publicado como **um único serviço de control plane**
> (stateless, escalado horizontalmente por shard), acompanhado dos
> armazenamentos que o suportam.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                     008-AGENT-LIFECYCLE — CONTÊINERES                            │
│                                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────────┐    │
│  │        Lifecycle Service  (.NET 10, AOT, stateless, N réplicas)          │    │
│  │  Expõe: REST /v1/agents/** (via YARP) · gRPC aios.lifecycle.v1 (interno) │    │
│  └───────────┬─────────────────┬──────────────────┬───────────────┬────────┘    │
│              │                 │                  │               │             │
│   ┌──────────▼─────────┐ ┌────▼──────────┐ ┌──────▼──────────┐ ┌─▼───────────┐ │
│   │   PostgreSQL 16      │ │     Redis      │ │      MinIO       │ │NATS/       │ │
│   │  schema `lifecycle`  │ │ (lease+fencing,│ │ (blobs de        │ │JetStream   │ │
│   │  (ACB proj. estado,  │ │  ACB hot cache,│ │  checkpoint/     │ │(Outbox →   │ │
│   │  transitions,        │ │  índice de     │ │  snapshot,       │ │eventos     │ │
│   │  checkpoint idx,     │ │  cold agents)  │ │  cifrados        │ │lifecycle)  │ │
│   │  hibernation,         │ │                │ │  AES-256-GCM)    │ │            │ │
│   │  migration_job,       │ │                │ │                  │ │            │ │
│   │  outbox) — RLS por    │ │                │ │                  │ │            │ │
│   │  tenant_id             │ │                │ │                  │ │            │ │
│   └─────────────────────┘ └────────────────┘ └──────────────────┘ └────────────┘ │
│                                                                                    │
│   ┌──────────────────────────────────────────────────────────────────────────┐   │
│   │  Sidecar de Telemetria: OTel Collector (traces/metrics) → 024;            │   │
│   │  Serilog → Seq (024); ambos fora do caminho crítico de transição.         │   │
│   └──────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

| Contêiner | Responsabilidade | Persistência | Escala |
|-----------|-------------------|---------------|--------|
| Lifecycle Service | Hospeda todos os componentes internos (§4); expõe REST/gRPC; stateless. | Nenhuma local (tudo externalizado). | Horizontal, por shard determinístico (`hash(tenant,agent) mod N`, §9 do brief); sem afinidade obrigatória. |
| PostgreSQL (`lifecycle` schema) | Fonte da verdade da projeção de ACB (estado de ciclo de vida), `LifecycleTransition` (event-sourced), `Checkpoint`, `HibernationRecord`, `MigrationJob`, `LifecycleOutbox`. | Durável, particionado por `tenant_id`/tempo. | Réplicas de leitura + streaming replication (HA, `027`). |
| Redis | Lease/fencing por agente (`LeaseManager`), projeção quente do ACB (leitura sub-ms), índice mínimo (`≤ 4 KiB`) de cold agents para roteamento/wake. | Volátil (reconstruível a partir do PostgreSQL). | Cluster com réplicas. |
| MinIO | Payload serializado de checkpoints/snapshots (ACB + working memory), cifrado AES-256-GCM, comprimido `msgpack+zstd`. | Durável, versionado por `generation`. | Object storage distribuído (`027`). |
| NATS/JetStream | Transporte durável de eventos `agent.lifecycle.*` publicados via Outbox; consumo de eventos de `009`/`007`/`006`/`014`/`027`. | Streams dedicados (ver `./Events.md`). | Cluster NATS (`020`). |

---

## 4. C4 — Nível 3: Componentes

> Reproduzido e detalhado a partir de `./_DESIGN_BRIEF.md` §2.1. Este é o
> diagrama de componentes autoritativo do módulo.

```
                         REST / gRPC (aios.lifecycle.v1)
                                     │
                       ┌─────────────▼──────────────┐
                       │     LifecycleApiSurface     │  validação · idem-key · RFC7807
                       └─────────────┬──────────────┘
                                     │
                       ┌─────────────▼──────────────┐   consulta PDP
                       │   LifecyclePolicyEnforcer   │─────────────────▶ 022-Policy
                       └─────────────┬──────────────┘
                                     │ comando autorizado
        ┌────────────────────────────▼─────────────────────────────┐
        │                    LifecycleCoordinator                    │
        │   (saga · efeitos · publicação) — adquire lease por agente │
        └───┬───────┬──────────┬──────────┬───────────┬──────────┬───┘
            │       │          │          │           │          │
     ┌──────▼─┐ ┌───▼─────┐ ┌──▼───────┐ ┌▼─────────┐ ┌▼────────┐ │
     │ State  │ │  Lease  │ │  Spawn   │ │Hibernat. │ │Migration│ │
     │Machine │ │ Manager │ │ Manager  │ │Controller│ │Orchestr.│ │
     │Engine  │ │(Redis)  │ │  │       │ │   │      │ │   │     │ │
     └───┬────┘ └─────────┘ └──┼───────┘ └───┼──────┘ └───┼─────┘ │
         │                     │             │            │       │
         │              ┌──────▼─────┐ ┌─────▼──────┐     │       │
         │              │ WarmPool   │ │Checkpoint  │◀────┘       │
         │              │ Manager    │ │ Service    │             │
         │              └──────┬─────┘ └─────┬──────┘             │
         │                     │             │                    │
    ┌────▼─────────────────────▼─────────────▼────────────────────▼───┐
    │                          AcbStore  /  SnapshotStore              │
    │  PostgreSQL (ACB proj., transitions, outbox, snapshot idx) ·     │
    │  Redis (hot) · MinIO (snapshot/checkpoint blobs)                 │
    └───┬──────────────────────────────────────────────────────┬──────┘
        │ Outbox                                                │
   ┌────▼──────────────────┐                        ┌───────────▼─────────────┐
   │ LifecycleEventPublisher│──▶ NATS/JetStream ────│  LifecycleEventConsumer  │
   │  aios.<tenant>.agent.  │    (020)              │  ← 009 placement         │
   │  lifecycle.*           │                       │  ← 007 heartbeat/exit    │
   └───────────────────────┘                        │  ← 006 acb.created       │
                                                     │  ← 014 task idle         │
                                                     │  ← 027 node draining     │
                                                     └──────────┬──────────────┘
   ┌───────────────────────┐                                    │ reativo
   │ReconciliationController│◀───────────────────────────────────┘
   │ (desejado × observado) │──▶ LifecycleCoordinator (compensação)
   └───────────────────────┘
   ┌───────────────────────┐    ┌──────────────────────┐
   │  TombstoneManager      │    │  LifecycleTelemetry   │  (transversal OTel → 024)
   │  (retenção/LGPD purge) │    │  aios_lifecycle_*     │
   └───────────────────────┘    └──────────────────────┘
```

---

## 5. Tabela de Responsabilidades por Componente

| Componente | Responsabilidade primária | Colaboradores diretos | Dependências externas |
|------------|----------------------------|--------------------------|--------------------------|
| **LifecycleApiSurface** | Fachada REST/gRPC do módulo; valida contrato/versão de API, `Idempotency-Key`, mapeia erro para RFC 7807/`google.rpc.Status`. | LifecyclePolicyEnforcer, LifecycleCoordinator | API Gateway (YARP) |
| **LifecyclePolicyEnforcer** | PEP: monta `DecisionContext` por mutação; consulta o PDP; *default deny*. | PolicyClient (interno), LifecycleCoordinator | `022-Policy` |
| **LifecycleCoordinator** | Orquestrador central; adquire lease, invoca a FSM, dispara efeitos (spawn/checkpoint/migrate) via saga, publica eventos. | StateMachineEngine, LeaseManager, AcbStore, *Managers*, LifecycleEventPublisher | — |
| **StateMachineEngine** | Implementa a FSM canônica (`./StateMachine.md`); valida transições, avalia guardas, calcula ações de entrada/saída; determinístico e puro. | AcbStore (leitura) | — |
| **AcbStore** | Repositório da projeção de estado de ciclo de vida do ACB e do histórico de transições (event-sourced + CQRS). | PostgreSQL, Redis | — |
| **SpawnManager** | Materializa agente novo ou frio: negocia slot com `009`, pede runtime a `007`, aplica snapshot, aquece contexto/memória (`010`/`011`). Alvo `p99 ≤ 250 ms`. | WarmPoolManager, SnapshotStore | `009-Scheduler`, `007-Agent-Runtime` |
| **WarmPoolManager** | Mantém pool de runtimes pré-aquecidos por tenant/shard, reduzindo cold-start. | SpawnManager | `007-Agent-Runtime`, Redis |
| **HibernationController** | Detecta agentes ociosos, aplica política de hibernação, dispara checkpoint, libera RAM, marca agente como *cold*. | CheckpointService, SnapshotStore, AcbStore | — |
| **CheckpointService** | Cria/restaura checkpoints consistentes de ACB + working memory; atomicidade, hash de integridade, cifragem. | SnapshotStore, MigrationOrchestrator | `010-Memory` |
| **SnapshotStore** | Persistência do estado serializado: blob em MinIO + metadados/índice em PostgreSQL. | CheckpointService, HibernationController | MinIO, PostgreSQL |
| **MigrationOrchestrator** | Saga de migração entre shards/nós (checkpoint→transfer→materializa→cutover), com corte de escrita e drenagem. | CheckpointService, SpawnManager | `027-Cluster`, `009-Scheduler` |
| **LeaseManager** | Locks/leases distribuídos por `agent_id`; garante um único coordenador por agente; renovação e fencing token. | LifecycleCoordinator | Redis |
| **ReconciliationController** | Loop de reconciliação estado-desejado × observado; detecta runtime morto, checkpoint órfão, migração travada. | AcbStore, LifecycleCoordinator | `007-Agent-Runtime` |
| **LifecycleEventPublisher** | Publica eventos `agent.lifecycle.*` via Outbox transacional; at-least-once + dedupe por `event.id`. | — | PostgreSQL (outbox), NATS/JetStream |
| **LifecycleEventConsumer** | Consome eventos de `009`, `007`, `006`, `014`, `027` para acionar transições reativas. | LifecycleCoordinator | NATS/JetStream |
| **TombstoneManager** | Aplica retenção de agentes `Terminated/Failed`, expurgo de snapshots/checkpoints (LGPD), anti-resurrection. | SnapshotStore, AcbStore | `025-Audit` |
| **LifecycleTelemetry** | Instrumentação OTel transversal; expõe métricas `aios_lifecycle_*`. | Todos os componentes acima (cross-cutting) | `024-Observability` |

---

## 6. Fronteiras e Regras de Dependência

- O módulo 008 **DEVE** tratar `007-Agent-Runtime` e `009-Scheduler` como
  **serviços externos consultados**: o 008 *pede* materialização/admissão, mas
  **NÃO DEVE** implementar o loop cognitivo (N1) nem a decisão de placement
  (N2).
- O 008 **NÃO DEVE** expor syscalls cognitivas (`remember`, `recall`, `plan`,
  `invoke_tool`) nem gerenciar cotas (N3) — apenas lê cotas por referência
  (`quota_ref`) para admitir uma materialização.
- O 008 **NÃO DEVE** persistir ou consolidar camadas de memória (N4); serializa
  apenas o *ponteiro/snapshot da working memory* como parte do checkpoint,
  delegando a materialização de longo prazo ao `010-Memory`.
- O 008 **NÃO DEVE** tomar decisões de política (N5); toda mutação **DEVE**
  atravessar `LifecyclePolicyEnforcer` até o PDP do `022-Policy`.
- O 008 **NÃO DEVE** gerenciar provisão física de nós/pods ou autoscaling de
  infraestrutura (N6) — isso é do `027-Cluster`/`028-Deployment`.
- O 008 **NÃO DEVE** ser trilha de auditoria imutável (N8): apenas *emite*
  eventos; a trilha imutável é responsabilidade do `025-Audit`.
- Toda dependência externa (`009`, `007`, `006`, `010`, `022`, `027`) **DEVE**
  estar isolada por *circuit breaker* e *bulkhead* dedicados no cliente
  correspondente, para que a falha de uma dependência não degrade as demais.
- O módulo **DEVE** ser *stateless* ao nível de processo: todo estado que
  sobrevive a um restart de réplica **DEVE** residir em PostgreSQL/Redis/MinIO/
  NATS, nunca em memória de processo não externalizada.

---

## 7. Padrões Arquiteturais Adotados

| Padrão | Onde é aplicado | Motivação |
|--------|-------------------|-----------|
| **PEP/PDP** | `LifecyclePolicyEnforcer` consulta `022-Policy`. | Separação entre aplicação e decisão de política; *default deny* uniforme (RFC-0001 §5.8). |
| **Saga com compensação** | `MigrationOrchestrator` (quiesce→checkpoint→transfer→materialize→cutover→cleanup); `LifecycleCoordinator` para spawn/hibernação. | Transação distribuída entre 008, Scheduler, Runtime e Cluster sem 2PC (ADR-0085). |
| **Event Sourcing** | `LifecycleTransition` (append-only, `seq` monotônico por agente). | Proveniência total do ciclo de vida; base de auditoria e replay (ADR-0086). |
| **Transactional Outbox** | `LifecycleEventPublisher` grava o evento na mesma transação do estado. | Atomicidade entre persistência e publicação sem *dual write* (ADR-0086). |
| **Lease + Fencing Token** | `LeaseManager` (Redis `SET NX PX`). | Serialização de transições por agente sem lock global; protege contra coordenador rogue (ADR-0084). |
| **Controller/Reconcile** | `ReconciliationController`. | Corrige deriva entre estado desejado e observado (runtime morto sem `Terminated`), padrão robusto de sistemas distribuídos (ADR-0087). |
| **CQRS** | `AcbStore`: escrita append-only de transições; leitura por projeção de estado atual (cache quente em Redis). | Escala leitura de estado sem tocar o log de transições. |
| **Sharding determinístico** | `placement_shard = hash(tenant_id, agent_id) mod N`. | Localidade e distribuição de carga previsível rumo a `10⁶⁺` agentes. |
| **Object Pool (WarmPool)** | `WarmPoolManager` mantém runtimes pré-aquecidos por tenant/shard. | Elimina cold-start de processo; materialização vira *deserialização + attach*, não boot completo (ADR-0083). |
| **Circuit Breaker / Bulkhead** | Clientes de `009`, `007`, `010`, `022`, `027`. | Isola falha de dependência externa; evita cascata. |

---

## 8. Tecnologias e Justificativas

| Tecnologia | Uso no Agent Lifecycle | Justificativa | Alternativa descartada |
|------------|---------------------------|----------------|--------------------------|
| **.NET 10 (AOT)** | Runtime do Lifecycle Service. | Baixa latência de start (crítico para `p99 ≤ 250 ms` de spawn), tipagem forte para a FSM e sagas, alinhado ao control plane do AIOS. | JVM (maior *footprint* de start); Go (ecossistema gRPC/OTel do .NET já padronizado). |
| **gRPC** (`aios.lifecycle.v1`) | Comunicação interna com `006`, `007`, `009`, `010`, `022`, `027`. | Contrato fortemente tipado, streaming (`StreamTransitions`), baixa latência no caminho quente de transição (NFR-002). | REST interno (overhead de serialização/latência maior). |
| **REST (via YARP)** | Superfície externa `/v1/agents/**`. | Interoperabilidade com SDKs/operação; RFC-0001 §5.7 exige versionamento por caminho. | GraphQL (complexidade desnecessária para um conjunto de comandos de ciclo de vida). |
| **PostgreSQL 16** | Fonte da verdade de `LifecycleTransition`, `Checkpoint`, `HibernationRecord`, `MigrationJob`, `LifecycleOutbox`, projeção de ACB. | RLS nativo por `tenant_id`, transações ACID para Outbox, particionamento maduro por tempo/tenant. | MongoDB (sem RLS nativo equivalente; padrão Outbox menos maduro). |
| **Redis** | `LeaseManager` (fencing), projeção quente de ACB, índice mínimo de cold agents. | Latência sub-ms, `SET NX PX` atômico para lease, TTL nativo para renovação. | Memcached (sem operações atômicas ricas o suficiente para fencing). |
| **MinIO** | `SnapshotStore`: payload de checkpoint/snapshot cifrado e comprimido. | Object storage compatível S3, padrão do AIOS para estado durável de agentes *cold* (ADR-0081). | Sistema de arquivos local (não escalável, sem durabilidade multi-nó). |
| **NATS/JetStream** | Transporte de eventos `agent.lifecycle.*` e consumo de eventos de outros módulos. | Baixa latência, streams duráveis, replay, alinhado ao barramento único do AIOS. | Kafka (operação mais pesada; NATS já é o padrão do AIOS). |
| **msgpack+zstd** | Codec de serialização de checkpoint (default). | Compacto e rápido para working memory até `256 MiB`; `json+zstd` disponível como alternativa legível (ADR-0082). | Protobuf puro (menos flexível para payload semi-estruturado de working memory). |
| **AES-256-GCM** | Cifragem em repouso de checkpoints/snapshots, chave por tenant (`021`). | Autenticado (AEAD), padrão de mercado, mitiga *tampering* e *information disclosure* (STRIDE, §12). | AES-CBC sem autenticação (vulnerável a *tampering* não detectado). |
| **OpenTelemetry + Prometheus + Grafana + Serilog/Seq** | `LifecycleTelemetry`: spans por transição, métricas `aios_lifecycle_*`, logs correlacionados. | Padrão transversal do AIOS; correlação `trace_id`/`tenant_id` obrigatória (RFC-0001 §5.6). | APM proprietário (vendor lock-in). |

---

## 9. Caminho Quente (Hot Path) × Caminho Frio (Cold Path)

```
   HOT PATH (agente ativo, RAM)              COLD PATH (agente hibernado)
 ┌───────────────────────┐                 ┌────────────────────────────┐
 │ 007-Agent-Runtime      │                 │  Redis: índice mínimo       │
 │  (Running/Suspended)   │                 │  (≤ 4 KiB) por agent_id      │
 └──────────┬─────────────┘                 └──────────────┬─────────────┘
            │ suspend/hibernate                              │ wake (demanda)
            ▼                                                ▼
 ┌───────────────────────┐   hibernate    ┌────────────────────────────┐
 │ HibernationController  │───────────────▶│  CheckpointService          │
 │                        │                │  (serializa ACB+workmem)    │
 └───────────────────────┘                 └──────────────┬─────────────┘
                                                            ▼
                                             ┌────────────────────────────┐
                                             │  SnapshotStore              │
                                             │  MinIO (blob) + PG (índice) │
                                             └──────────────┬─────────────┘
                                                            │ wake: materialize <250ms
                                                            ▼
                                             ┌────────────────────────────┐
                                             │  SpawnManager + WarmPool    │
                                             │  → Ready/Running (hot path) │
                                             └────────────────────────────┘
```

Este desenho é o que permite `≤ 5%` dos `10⁶` agentes estarem ativos em RAM
simultaneamente (NFR-004): a maioria vive no *cold path* — apenas metadados
mínimos em Redis e o snapshot completo em MinIO/PostgreSQL — sendo
materializada sob demanda pelo *hot path* com latência `p99 ≤ 250 ms`.

---

## 10. Riscos Arquiteturais e Trade-offs

| Risco/Trade-off | Descrição | Mitigação |
|-------------------|-----------|-----------|
| Storm de wake sob pico de demanda | Muitos cold agents acordando simultaneamente pode saturar WarmPool/Runtime. | Backpressure via `migration.max_concurrent_per_node` equivalente para wake; admissão do Scheduler; leases evitam thundering herd. |
| Latência de migração acoplada à saúde do destino | `Migrating` depende de nó/shard destino saudável; se cair no meio, saga precisa compensar. | Saga com timeout (`migration.saga_timeout_ms`) + compensação (reativa origem); `MigrationJob=compensated`. |
| Overhead de checkpoint periódico em agentes muito ativos | `checkpoint.interval_s` frequente pode competir com o caminho quente por I/O. | Checkpoint assíncrono, fora do caminho de resposta ao usuário; meta `p99 ≤ 500 ms` para até 64 MiB (NFR-003). |
| Deriva entre estado desejado e observado | Runtime pode morrer sem notificar (crash, OOM-kill). | `ReconciliationController` com `reconcile.interval_ms` e `runtime_dead_after_ms`; RTO ≤ 15 min (NFR-005). |
| Crescimento de `LifecycleTransition` (event-sourced) | Alto volume de agentes gera grande volume de linhas de histórico. | Particionamento por tempo/tenant (`./Database.md`), retenção configurável, projeção de estado atual em cache. |
| Complexidade de saga de migração | Múltiplas fases (`quiesce/checkpoint/transfer/materialize/cutover/cleanup`) aumentam superfície de falha. | Cada fase é idempotente e observável individualmente; compensação automática por fase (`./StateMachine.md` §6). |

---

## 11. Alternativas Descartadas

| Alternativa | Por que foi descartada |
|-------------|--------------------------|
| Lock global por tenant para serializar transições | Elimina concorrência horizontal; substituído por lease por `agent_id` com fencing token (ADR-0084). |
| 2PC (two-phase commit) entre 008↔009↔007↔027 na migração | Custo de latência e disponibilidade incompatível com NFR-010 (`p99 ≤ 2 s`); substituído por saga com compensação (ADR-0085). |
| Manter working memory completa no ACB (em vez de ponteiro) | Infla o ACB, viola a fronteira de propriedade com `010-Memory` (N4), impede materialização independente da memória de longo prazo. |
| Boot completo de processo a cada `spawn`/`wake` (sem WarmPool) | Incompatível com o alvo `p99 ≤ 250 ms` (NFR-001) em escala de milhões de agentes; substituído por WarmPool + materialização por deserialização (ADR-0083). |
| Réplica síncrona de working memory em vez de checkpoint/restore | Custo de latência e de rede incompatível com o modelo de escala a `10⁶` cold agents; checkpoint assíncrono com RPO ≤ 5 min é o compromisso adotado (ADR-0081/0082). |

---

## 12. Referências e ADRs

- Fonte de verdade do módulo: `./_DESIGN_BRIEF.md`
- Arquitetura global: `../001-Architecture/Architecture.md`
- Contratos centrais: `../003-RFC/RFC-0001-Architecture-Baseline.md`
- Glossário: `../040-Glossary/Glossary.md`
- ADRs do módulo (faixa `ADR-0080`–`ADR-0089`): ver `./ADR.md` para o índice
  detalhado — FSM canônica (ADR-0080), hibernação em MinIO/PostgreSQL
  (ADR-0081), codec/cifragem de checkpoint (ADR-0082), materialização com
  WarmPool (ADR-0083), lease+fencing (ADR-0084), migração como saga
  (ADR-0085), event sourcing + Outbox (ADR-0086), reconciliação
  (ADR-0087), fronteira 008×006×009×007 (ADR-0088), retenção/tombstone/LGPD
  (ADR-0089).
- RFCs do módulo: ver `./RFC.md` — RFC-0080 (Cold Agent Hibernation &
  Checkpoint/Restore Protocol, a propor).
- Componentes acoplados: `../006-Kernel/`, `../007-Agent-Runtime/`,
  `../009-Scheduler/`, `../010-Memory/`, `../020-Communication/`,
  `../021-Security/`, `../022-Policy/`, `../024-Observability/`,
  `../025-Audit/`, `../027-Cluster/`.

*Fim do documento `Architecture.md` do módulo 008-Agent-Lifecycle.*
